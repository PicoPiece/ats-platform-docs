# Tài Liệu Master: Hệ Thống ATS (Automation Test System)

> **Tài liệu tổng hợp giải thích chi tiết cách hệ thống ATS hoạt động**

## Mục Lục

1. [Tổng Quan Hệ Thống](#tổng-quan-hệ-thống)
2. [Kiến Trúc Hệ Thống](#kiến-trúc-hệ-thống)
3. [Luồng Hoạt Động Từng Bước](#luồng-hoạt-động-từng-bước)
4. [Các Thành Phần Chính](#các-thành-phần-chính)
5. [Quy Trình CI/CD](#quy-trình-cicd)
6. [Cơ Chế Test và Validation](#cơ-chế-test-và-validation)
7. [Quản Lý Artifacts và Manifest](#quản-lý-artifacts-và-manifest)
8. [Observability và Metrics](#observability-và-metrics)
9. [Bảo Mật và Best Practices](#bảo-mật-và-best-practices)
10. [Mở Rộng và Scale](#mở-rộng-và-scale)

---

## Tổng Quan Hệ Thống

### ATS Là Gì?

**ATS (Automation Test System)** là một hệ thống tự động hóa kiểm thử firmware cho các thiết bị nhúng (embedded devices), đặc biệt là ESP32. Hệ thống này được thiết kế để:

- **Tự động hóa kiểm thử firmware trên phần cứng thật** (Hardware-in-the-Loop testing)
- **Tích hợp trực tiếp vào CI/CD pipeline** (Jenkins)
- **Loại bỏ kiểm thử thủ công**, giảm thiểu lỗi và tăng tốc độ phát triển
- **Cung cấp metrics và observability** để theo dõi chất lượng firmware theo thời gian

### Vấn Đề Hệ Thống Giải Quyết

Trong các team phát triển firmware và IoT, thường gặp các vấn đề:

1. **CI pipeline chỉ dừng ở build/unit test** - Không test trên hardware thật
2. **Kiểm thử firmware trên hardware là thủ công** - Chậm, dễ sai sót
3. **Lỗi liên quan đến hardware phát hiện muộn** - Tốn kém để sửa
4. **OTA updates rủi ro cao** - Khó reproduce và validate
5. **QA không scale được** - Không theo kịp tốc độ phát triển firmware

**ATS giải quyết bằng cách:**
- Tự động flash firmware lên ESP32 thật
- Chạy các test về power, IO, connectivity, OTA
- Thu thập kết quả qua UART, GPIO, và visual inspection
- Đưa kết quả trở lại CI và hệ thống observability

---

## Kiến Trúc Hệ Thống

### Mô Hình 3 Tầng (3-Tier Architecture)

Hệ thống ATS được chia thành 3 tầng rõ ràng:

```
┌─────────────────────────────────────────────────────────────┐
│         Tầng 1: Control & Build Plane (Xeon Server)         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Jenkins  │  │Prometheus│  │ Grafana  │  │ Build    │     │
│  │ Master   │  │          │  │          │  │ Agent    │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│  - Pipeline orchestration                                   │
│  - Firmware build                                           │
│  - Metrics collection                                       │
│  - Visualization                                            │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ Network (ats-net)
                          │
┌─────────────────────────────────────────────────────────────┐
│      Tầng 2: Execution Plane (ATS Nodes - Raspberry Pi)     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  ATS Node 1  │  │  ATS Node 2  │  │  ATS Node N  │       │
│  │  - Jenkins   │  │  - Jenkins   │  │  - Jenkins   │       │
│  │    Agent     │  │    Agent     │  │    Agent     │       │
│  │  - Test      │  │  - Test      │  │  - Test      │       │
│  │    Runner    │  │    Runner    │  │    Runner    │       │
│  │  - Metrics   │  │  - Metrics   │  │  - Metrics   │       │
│  │    Exporter  │  │    Exporter  │  │    Exporter  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│  - Hardware access (UART, GPIO, USB)                        │
│  - Firmware flashing                                        │
│  - Test execution                                           │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ USB/UART/GPIO
                          │
┌─────────────────────────────────────────────────────────────┐
│         Tầng 3: Device Under Test (ESP32)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ ESP32-1  │  │ ESP32-2  │  │ ESP32-N  │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
│  - Runs firmware under test                                 │
│  - Observable behavior (UART, GPIO, OLED)                   │
└─────────────────────────────────────────────────────────────┘
```

### Lợi Ích Của Kiến Trúc Phân Tầng

1. **Tách biệt trách nhiệm rõ ràng**
   - Control Plane: Orchestration, không cần hardware access
   - Execution Plane: Hardware interaction, không cần build tools
   - Device: Chỉ chạy firmware và expose behavior

2. **Dễ scale**
   - Thêm ATS node mới chỉ cần cắm Raspberry Pi và đăng ký Jenkins agent
   - Không cần thay đổi pipeline

3. **Reliability**
   - CI orchestration ổn định (không bị ảnh hưởng bởi hardware issues)
   - Hardware access được isolate
   - Có thể chạy parallel tests trên nhiều nodes

---

## Luồng Hoạt Động Từng Bước

### Quy Trình Tổng Thể: Commit → Build → Test → Report

```
1. Developer tạo commit/PR trên GitHub
   │
   ├─> GitHub webhook trigger Jenkins
   │
2. Jenkins Build Pipeline (trên Xeon Server)
   │
   ├─> Checkout source code
   ├─> Build ESP32 firmware (ESP-IDF)
   ├─> Generate firmware binary: firmware-esp32.bin
   ├─> Generate ATS Manifest: ats-manifest.yaml
   ├─> Archive artifacts
   ├─> Create Git tag
   └─> Trigger Test Pipeline (async, non-blocking)
   │
3. Jenkins Test Pipeline (trên ATS Node - Raspberry Pi)
   │
   ├─> Copy artifacts từ build job
   │   ├─> firmware-esp32.bin
   │   └─> ats-manifest.yaml
   ├─> Build ats-node-test Docker container
   ├─> Run container (hardware interaction)
   │   ├─> Load manifest
   │   ├─> Detect ESP32 USB port
   │   ├─> Flash firmware to ESP32
   │   ├─> Invoke test runner
   │   └─> Collect results
   └─> Archive test results
   │
4. Test Execution (trong Docker container)
   │
   ├─> Read ats-manifest.yaml
   ├─> Execute test plan:
   │   ├─> UART boot validation
   │   ├─> GPIO toggle test
   │   ├─> OLED display test
   │   └─> Firmware stability test
   ├─> Capture UART logs
   ├─> Check GPIO states
   └─> Generate structured results
   │
5. Results & Reporting
   │
   ├─> Write results to /results directory:
   │   ├─> ats-summary.json (human-readable)
   │   ├─> junit.xml (CI-consumable)
   │   ├─> serial.log (UART output)
   │   └─> meta.yaml (execution metadata)
   ├─> Update Prometheus metrics
   ├─> Archive in Jenkins
   └─> Display in Grafana dashboards
```

### Chi Tiết Từng Bước

#### Bước 1: Build Pipeline (Jenkinsfile)

**Location:** `ats-fw-esp32-demo/platforms/ESP32/Jenkinsfile`

**Các stage:**

1. **Checkout Source**
   - Agent: `fw-build` (Xeon server với ESP-IDF toolchain)
   - Checkout code từ Git repository

2. **Build ESP32 Firmware**
   - Setup ESP-IDF environment
   - Chạy `idf.py build`
   - Output: `firmware-esp32.bin`

3. **Generate ATS Manifest**
   - Tạo file `ats-manifest.yaml` với:
     - Build metadata (CI system, job name, build number)
     - Git info (repo, commit, branch)
     - Artifact checksum (SHA256)
     - Device target (esp32)
     - Test plan (danh sách tests cần chạy)
   - Đây là **contract duy nhất** giữa build, test, và CI system

4. **Archive Artifacts**
   - Archive `firmware-esp32.bin` và `ats-manifest.yaml`
   - Lưu trong Jenkins build history

5. **Tag Firmware**
   - Tạo Git tag: `esp32-fw-{BUILD_NUMBER}-{COMMIT_SHORT}`
   - Để versioning và traceability

6. **Trigger Test Pipeline** (optional)
   - Trigger test pipeline với build parameters
   - Async, non-blocking (build job không đợi test hoàn thành)

#### Bước 2: Test Pipeline (Jenkinsfile.test)

**Location:** `ats-fw-esp32-demo/platforms/ESP32/Jenkinsfile.test`

**Các stage:**

1. **Copy Firmware Artifact**
   - Agent: `ats-node` (Raspberry Pi)
   - Copy `firmware-esp32.bin` và `ats-manifest.yaml` từ build job
   - Validate artifact presence

2. **Checkout Test Framework**
   - Checkout `ats-test-esp32-demo` repository
   - Chứa test execution logic

3. **Build ATS Node Test Container**
   - Build Docker image từ `ats-ats-node/docker/ats-node-test/`
   - Container này chứa tất cả hardware interaction logic

4. **Run ATS Tests**
   - Chạy container với:
     ```bash
     docker run --device=/dev/ttyUSB0 \
                -v /workspace:/workspace \
                ats-node-test:latest
     ```
   - Container tự động:
     - Load manifest
     - Detect ESP32
     - Flash firmware
     - Invoke test runner
     - Generate results

5. **Archive Test Reports**
   - Archive tất cả files trong `/results`:
     - `ats-summary.json`
     - `junit.xml`
     - `serial.log`
     - `meta.yaml`
   - Publish JUnit XML để Jenkins hiển thị test results

#### Bước 3: Test Execution (trong Container)

**Container:** `ats-node-test` từ `ats-ats-node/docker/ats-node-test/`

**Quy trình trong container:**

1. **Load Manifest** (`manifest.py`)
   - Đọc `/workspace/ats-manifest.yaml`
   - Validate schema v1
   - Extract:
     - `device.target` → biết là ESP32
     - `build.artifact.name` → tên file firmware
     - `test_plan` → danh sách tests cần chạy

2. **Hardware Detection** (`hardware.py`)
   - Scan USB ports: `/dev/ttyUSB0`, `/dev/ttyUSB1`, `/dev/ttyACM0`, etc.
   - Detect ESP32 device
   - Validate GPIO access

3. **Flash Firmware** (`flash_esp32.py`)
   - Sử dụng `esptool.py` để flash
   - Command: `esptool.py --port /dev/ttyUSB0 write_flash ... firmware-esp32.bin`
   - Verify flash success

4. **Invoke Test Runner** (`executor.py`)
   - Chạy test runner từ `ats-test-esp32-demo`
   - Pass manifest và test plan
   - Test runner thực thi:
     - UART boot validation
     - GPIO toggle test
     - OLED display test
     - Firmware stability test

5. **Collect Results** (`results.py`)
   - Thu thập output từ test runner
   - Generate structured results:
     - `ats-summary.json`: Human-readable summary
     - `junit.xml`: CI-consumable test report
     - `serial.log`: UART output
     - `meta.yaml`: Execution metadata

6. **Exit Code**
   - `0`: All tests passed
   - `1`: Tests failed
   - `2`: Execution error (hardware not found, manifest invalid, etc.)

---

## Các Thành Phần Chính

### 1. ats-platform-docs

**Vai trò:** Tài liệu hệ thống và kiến trúc

**Nội dung:**
- System overview
- Architecture documentation
- Manifest specification (v1)
- Test output contract (v1)
- CI structure
- Implementation guides

**Đây là nơi định nghĩa contracts và specifications cho toàn bộ hệ thống.**

### 2. ats-fw-esp32-demo

**Vai trò:** Firmware source code và build pipeline

**Trách nhiệm:**
- ✅ ESP32 firmware source code (ESP-IDF project)
- ✅ Build pipeline (`Jenkinsfile`)
- ✅ Generate firmware binary: `firmware-esp32.bin`
- ✅ Generate ATS manifest: `ats-manifest.yaml`
- ✅ Git tagging cho versioning
- ✅ Trigger test pipeline

**KHÔNG làm:**
- ❌ Hardware testing → `ats-test-esp32-demo`
- ❌ Hardware interaction → `ats-ats-node`
- ❌ CI orchestration → `ats-ci-infra`

**Cấu trúc:**
```
ats-fw-esp32-demo/
├── main/                    # Firmware source code
│   ├── app_main.c
│   ├── gpio_demo.c
│   ├── oled_demo.c
│   └── ota.c
├── platforms/
│   └── ESP32/
│       ├── Jenkinsfile      # Build pipeline
│       └── Jenkinsfile.test # Test pipeline
└── CMakeLists.txt
```

### 3. ats-test-esp32-demo

**Vai trò:** Test execution framework (hardware-agnostic)

**Trách nhiệm:**
- ✅ Pure test execution logic
- ✅ Đọc test parameters từ `ats-manifest.yaml`
- ✅ Execute automated hardware tests (giả định firmware đã được flash)
- ✅ Observe firmware behavior:
  - UART logs (từ serial port)
  - GPIO state (từ environment variables)
  - Visual indicators (LED/OLED)
- ✅ Generate structured test results:
  - `ats-summary.json`
  - `junit.xml`
  - `serial.log`
  - `meta.yaml`

**KHÔNG làm:**
- ❌ Build firmware → `ats-fw-esp32-demo`
- ❌ Flash firmware → `ats-ats-node`
- ❌ Detect USB/hardware → `ats-ats-node`
- ❌ Control GPIO directly → `ats-ats-node`

**Cấu trúc:**
```
ats-test-esp32-demo/
├── agent/
│   ├── flash_fw.sh      # (legacy, sẽ được thay bằng ats-ats-node)
│   ├── run_tests.sh     # Main test execution script
│   ├── read_uart.sh     # UART log capture
│   └── gpio_check.sh    # GPIO validation
├── Dockerfile
└── docker-compose.yml
```

**Test Types:**
- UART boot validation
- GPIO toggle test
- OLED display test
- Firmware stability test (reboot/timing)

### 4. ats-ats-node

**Vai trò:** ATS Node Execution Brain - Hardware interaction và test orchestration

**Trách nhiệm:**
- ✅ Hardware access (UART, GPIO, USB detection)
- ✅ Firmware flashing (via `esptool.py` cho ESP32)
- ✅ Test orchestration (load manifest, flash firmware, invoke test runner)
- ✅ Result generation (structured output trong `/results` directory)
- ✅ Hardware abstraction (Jenkins không cần biết về USB ports, GPIO pins)

**Nguyên tắc:** Jenkins là "dumb" - chỉ chạy `ats-node-test` container. Tất cả hardware logic nằm ở đây.

**Cấu trúc:**
```
ats-ats-node/
├── docker/
│   └── ats-node-test/          # Docker container
│       ├── Dockerfile
│       ├── entrypoint.sh
│       └── ats_node_test/      # Python package
│           ├── manifest.py      # Manifest loading/validation
│           ├── hardware.py      # Hardware detection (USB, GPIO)
│           ├── flash_esp32.py   # Firmware flashing
│           ├── executor.py      # Test orchestration
│           └── results.py       # Result generation
└── agent/                       # Legacy scripts
```

**Container Execution Flow:**
1. Load manifest từ `/workspace/ats-manifest.yaml`
2. Detect ESP32 USB port
3. Flash firmware to ESP32
4. Invoke test runner (`ats-test-esp32-demo`)
5. Collect results và write to `/results`
6. Exit với appropriate exit code

### 5. ats-ci-infra

**Vai trò:** CI/CD infrastructure (Jenkins, Prometheus, Grafana)

**Trách nhiệm:**
- ✅ Jenkins Master (pipeline orchestration)
- ✅ Jenkins Build Agents (firmware build với ESP-IDF)
- ✅ Prometheus (metrics collection)
- ✅ Grafana (visualization và dashboards)
- ✅ Configuration as Code (JCasC) cho Jenkins

**Cấu trúc:**
```
ats-ci-infra/
├── docker-compose.yml
├── jenkins/
│   ├── jenkins_home/          # Jenkins data volume
│   ├── jcasc/                 # Configuration as Code
│   │   ├── jenkins.yaml       # Main JCasC config
│   │   └── setup.sh
│   └── fw-build/
│       └── Dockerfile         # Custom build agent image
├── prometheus/
│   └── prometheus.yml         # Metrics collection config
└── grafana/
    ├── provisioning/
    └── dashboards/
```

**Services:**
- **Jenkins Master**: Port 8080
- **Prometheus**: Port 9090
- **Grafana**: Port 3000
- **Build Agent**: Custom image `ats-fw-build:esp32` với ESP-IDF v5.2 pre-installed

**Design Principles:**
- Jenkins orchestrates, không execute hardware
- Hardware access chỉ ở ATS nodes
- Build environments reproducible
- Metrics và logs là first-class citizens
- Scaling ATS capacity không cần CI redesign

---

## Quy Trình CI/CD

### Jenkins Pipeline Structure

#### Build Pipeline (`Jenkinsfile`)

**Agent:** `fw-build` (Xeon server với ESP-IDF toolchain)

**Stages:**

```groovy
stage('Checkout Source') {
    // Checkout code từ Git
}

stage('Build ESP32 Firmware') {
    // Setup ESP-IDF
    // idf.py build
    // Output: firmware-esp32.bin
}

stage('Archive Firmware Artifact') {
    // Archive firmware binary
}

stage('Generate ATS Manifest') {
    // Tạo ats-manifest.yaml với:
    // - Build metadata
    // - Git info
    // - Artifact checksum
    // - Device target
    // - Test plan
}

stage('Tag Firmware') {
    // Git tag: esp32-fw-{BUILD_NUMBER}-{COMMIT_SHORT}
}

stage('Trigger Test Pipeline') {
    // Trigger test pipeline async (nếu TRIGGER_TEST == true)
}
```

**Parameters:**
- `BRANCH_NAME`: Git branch to build (default: `main`)
- `TRIGGER_TEST`: Auto-trigger test pipeline (default: `true`)
- `TAG_PREFIX`: Prefix cho firmware tags (default: `esp32-fw`)

#### Test Pipeline (`Jenkinsfile.test`)

**Agent:** `ats-node` (Raspberry Pi với hardware access)

**Stages:**

```groovy
stage('Copy Firmware Artifact') {
    // Copy firmware-esp32.bin và ats-manifest.yaml từ build job
}

stage('Checkout Test Framework') {
    // Checkout ats-test-esp32-demo
}

stage('Build ATS Node Test Container') {
    // Build Docker image từ ats-ats-node/docker/ats-node-test/
}

stage('Run ATS Tests') {
    // docker run ats-node-test:latest
    // Container tự động:
    // - Load manifest
    // - Detect ESP32
    // - Flash firmware
    // - Run tests
    // - Generate results
}

stage('Archive Test Reports') {
    // Archive /results/*
    // Publish JUnit XML
}
```

**Parameters:**
- `BUILD_JOB_NAME`: Full path to build job
- `BUILD_NUMBER`: Build number to test
- `FW_ARTIFACT`: Firmware artifact name
- `PLATFORM`: Platform name
- `ATS_NODE_LABEL`: Label cho ATS node pool (default: `ats-node`)

### Configuration as Code (JCasC)

Jenkins configuration được quản lý qua YAML files:

**Location:** `ats-ci-infra/jenkins/jcasc/jenkins.yaml`

**Features:**
- Version control: Tất cả Jenkins config trong Git
- Automated setup: Jobs và folders tự động tạo khi Jenkins restart
- Reproducibility: Same config across environments

**Structure:**
```yaml
jenkins:
  systemMessage: "ATS CI/CD Infrastructure"

jobs:
  - script: |
      folder('platforms') { ... }
      folder('platforms/ESP32') { ... }
      pipelineJob('platforms/ESP32/ats-fw-esp32-demo') { ... }
      pipelineJob('platforms/ESP32/ats-fw-esp32-demo-ESP32-test') { ... }
```

---

## Cơ Chế Test và Validation

### Test Types

#### 1. UART Boot Validation

**Mục đích:** Verify firmware boot successfully và output đúng messages

**Quy trình:**
1. Flash firmware
2. Reset ESP32
3. Capture UART output trong khoảng thời gian nhất định
4. Validate expected messages:
   - Boot messages
   - Firmware version
   - Build number
   - Application-specific messages

**Implementation:**
- Script: `ats-test-esp32-demo/agent/read_uart.sh`
- Retry logic: 3 lần nếu timeout hoặc empty data
- Output: `serial.log` file

#### 2. GPIO Toggle Test

**Mục đích:** Verify GPIO pins hoạt động đúng

**Quy trình:**
1. Set GPIO pin direction (output)
2. Toggle GPIO state (HIGH/LOW)
3. Read GPIO state multiple times (3 samples)
4. Validate state changes
5. Majority voting để tránh transient issues

**Implementation:**
- Script: `ats-test-esp32-demo/agent/gpio_check.sh`
- Multiple samples per check
- Retry logic cho direction setting failures

#### 3. OLED Display Test

**Mục đích:** Verify OLED display hiển thị đúng content

**Quy trình:**
1. Firmware gửi data đến OLED
2. Capture display output (nếu có camera)
3. Validate content (text, graphics)
4. Hoặc validate I2C communication

**Implementation:**
- Visual validation (optional, future: camera + AI)
- I2C communication validation

#### 4. Firmware Stability Test

**Mục đích:** Verify firmware không crash, reboot đúng cách

**Quy trình:**
1. Run firmware trong thời gian dài
2. Monitor UART cho crash messages
3. Test reboot behavior
4. Validate stability metrics

**Implementation:**
- Reboot test failure tracking
- Individual reboot attempt monitoring
- Pattern detection cho inconsistent results

### Test Execution Flow

```
1. Test Runner được invoke bởi ats-node-test container
   │
2. Read ats-manifest.yaml
   ├─> Extract test_plan: [gpio_toggle_test, oled_display_test, uart_boot_test]
   ├─> Extract device.target: esp32
   └─> Extract device.board: esp32-devkit
   │
3. Execute tests theo test_plan
   │
   ├─> uart_boot_test
   │   ├─> Read UART output
   │   ├─> Validate boot messages
   │   └─> Record PASS/FAIL
   │
   ├─> gpio_toggle_test
   │   ├─> Set GPIO direction
   │   ├─> Toggle GPIO state
   │   ├─> Read GPIO state (3 samples)
   │   └─> Record PASS/FAIL
   │
   └─> oled_display_test
       ├─> Validate I2C communication
       └─> Record PASS/FAIL
   │
4. Collect results
   ├─> Test status (PASS/FAIL/SKIP)
   ├─> Failure messages (nếu có)
   ├─> UART logs
   └─> Execution metadata
   │
5. Generate structured output
   ├─> ats-summary.json
   ├─> junit.xml
   ├─> serial.log
   └─> meta.yaml
```

### Retry Logic và Error Handling

#### Retry Logic

**UART Reading:**
- Max retries: 3 (configurable)
- Automatic retry on timeout hoặc empty data
- Clear stale data trước khi retry
- Success validation dựa trên line count

**GPIO Checking:**
- Multiple samples per check (default: 3 samples)
- Majority voting cho stable readings
- Retry on direction setting failures

**Benefits:**
- Reduced flakiness từ transient hardware issues
- More reliable test results
- Better handling của timing-sensitive operations

#### Error Handling

**Enhanced Error Messages:**
```
❌ ESP32 device not found
   Checked ports: /dev/ttyUSB0, /dev/ttyUSB1, /dev/ttyACM0, /dev/ttyACM1
   Available USB devices: [list]
   Troubleshooting:
   - Ensure ESP32 is connected via USB
   - Check USB cable and port
   - Verify device permissions (user should be in dialout group)
```

**Graceful Degradation:**
- Tests continue even if some checks fail
- Partial results reported
- Clear distinction giữa failures và skips

### Test Validation

**Validation Features:**
- Detection của zero tests executed (all skipped)
- Success rate calculation
- Warning cho low success rates (<80%)

**Test Summary:**
```
Test Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Passed: 2
Failed: 1
Total:  3
Success Rate: 66%

⚠️  WARNING: Low success rate (< 80%) - investigate test reliability
```

---

## Quản Lý Artifacts và Manifest

### ATS Manifest (ats-manifest.yaml)

**Vai trò:** Single source of truth cho firmware test execution

**Schema v1:**

```yaml
manifest_version: 1

build:
  ci_system: jenkins
  job_name: platforms/ESP32/ats-fw-esp32-demo
  build_number: 42
  git:
    repo: https://github.com/PicoPiece/ats-fw-esp32-demo.git
    commit: 32317d2a5b215ae2f49b5a5136ba37e9a552a3c1
    branch: origin/main
  artifact:
    name: firmware-esp32.bin
    checksum: sha256:1deed34cb306a96dba6cb07d4f930a6c6f4a3fd8399881511ea9b92ee434700e
    build_node: xeon-fw-build

device:
  target: esp32
  board: esp32-devkit

test_plan:
  - gpio_toggle_test
  - oled_display_test
  - uart_boot_test

timestamps:
  build_time: "2024-12-25T10:30:00Z"
```

**Sử dụng bởi:**

1. **Firmware Build** (`ats-fw-esp32-demo`)
   - Generate manifest với tất cả required fields
   - Archive cùng với firmware binary

2. **ATS Node** (`ats-ats-node`)
   - Read manifest từ `/workspace/ats-manifest.yaml`
   - Extract `device.target` để determine hardware type
   - Extract `build.artifact.name` để locate firmware file
   - Extract `test_plan` để pass to test runner

3. **Test Runner** (`ats-test-esp32-demo`)
   - Read manifest cho test parameters
   - Execute tests listed trong `test_plan`
   - Use `device.target` và `device.board` cho test configuration

4. **CI System** (Jenkins)
   - Archive manifest cùng với firmware artifact
   - Pass both to ATS node via artifact copy
   - KHÔNG parse hoặc modify manifest

**Validation Rules:**
- Tất cả required fields phải present
- Type validation (integer, string, ISO 8601 timestamp)
- Semantic validation (valid SHA, valid URL, known device targets)

### Artifact Flow

```
Build Pipeline
    ↓
Produces: firmware-esp32.bin + ats-manifest.yaml
    ↓
Archived in Jenkins
    ↓
Test Pipeline
    ↓
Copies artifacts via copyArtifacts step
    ↓
ATS Node Container
    ↓
Loads manifest, flashes firmware
    ↓
Runs tests
    ↓
Generates results
```

### Test Output Contract

**Location:** `/results` directory

**Files:**

1. **ats-summary.json** (Required)
   - Human-readable test summary
   ```json
   {
     "status": "PASS",
     "tests": [
       {"name": "uart_boot_test", "status": "PASS"},
       {"name": "gpio_toggle_test", "status": "PASS"}
     ],
     "manifest": {
       "build_number": "42",
       "device_target": "esp32"
     }
   }
   ```

2. **junit.xml** (Required)
   - Standard JUnit XML format cho CI test reporting
   - Jenkins, GitLab CI, GitHub Actions đều consume được

3. **serial.log** (Optional but Recommended)
   - Raw serial/UART output từ device
   - Plain text file với device output

4. **meta.yaml** (Required)
   - Execution metadata cho traceability
   ```yaml
   execution:
     timestamp: "2024-12-25T10:30:00Z"
     exit_code: 0
     status: "PASS"
   manifest:
     version: 1
     build_number: "42"
     device_target: "esp32"
   ```

**Exit Codes:**
- `0`: All tests passed
- `1`: One or more tests failed
- `2`: Execution error (manifest not found, hardware unavailable)

---

## Observability và Metrics

### Prometheus Metrics

**Metrics Exporter:**
- Python-based HTTP server (`metrics_exporter.py`)
- Exposes `/metrics` endpoint trên port 8080 (configurable)
- Automatically started bởi test execution scripts
- Metrics persisted to JSON file cho reliability

**Available Metrics:**

| Metric | Type | Description |
|--------|------|-------------|
| `ats_test_pass_total` | Counter | Total number of passed tests |
| `ats_test_fail_total` | Counter | Total number of failed tests |
| `ats_test_duration_seconds` | Gauge | Duration of last test run |
| `ats_fw_version` | Gauge | Firmware version under test |
| `ats_test_last_run_timestamp` | Gauge | Unix timestamp of last test |
| `ats_test_in_progress` | Gauge | Whether test is running (1) or idle (0) |

**Metrics Endpoints:**
- `/metrics` - Prometheus format metrics
- `/health` - Health check endpoint

### Prometheus Configuration

**Location:** `ats-ci-infra/prometheus/prometheus.yml`

**Features:**
- Scrapes metrics từ tất cả ATS nodes
- Configurable scrape intervals (default: 30s)
- Label-based organization (node, platform)
- External labels cho cluster identification

**Example:**
```yaml
- job_name: 'ats-nodes'
  scrape_interval: 30s
  metrics_path: '/metrics'
  static_configs:
    - targets: ['ats-node-1:8080']
      labels:
        node: 'ats-node-1'
        platform: 'raspberry-pi'
```

### Grafana Dashboards

**Location:** `ats-ci-infra/grafana/dashboards/ats-test-dashboard.json`

**Dashboard Panels:**

1. **Test Pass/Fail Rate** - Rate of test passes và failures over time
2. **Total Tests Passed** - Cumulative count of passed tests
3. **Total Tests Failed** - Cumulative count of failed tests
4. **Test Pass/Fail Trend** - Time series graph của test results
5. **Test Duration** - Duration của test runs over time
6. **Test Success Rate** - Percentage gauge (green >95%, yellow >80%, red <80%)
7. **Last Test Run** - Timestamp của most recent test execution
8. **Test In Progress** - Indicator của current test status

**Provisioning:**
- Automatic dashboard provisioning via JCasC
- Datasource auto-configured to Prometheus
- Dashboards available immediately sau Grafana startup

### Metrics Update Flow

```
1. Test starts
   └─> ats_test_in_progress = 1
   │
2. Tests execute
   └─> Results collected
   │
3. Test completes
   └─> Metrics updated với:
       - Pass/fail counts
       - Duration
       - Firmware version
       - Timestamp
       - ats_test_in_progress = 0
```

**Benefits:**
- Detect regressions early
- Compare firmware versions objectively
- Provide confidence trước khi release
- Track stability over time

---

## Bảo Mật và Best Practices

### Security Improvements

#### 1. Secrets Management

**Before:**
- Hardcoded Jenkins secret trong `docker-compose.yml`
- Hardcoded Grafana credentials

**After:**
- Environment variables via `.env` file
- `.env.example` template cho documentation
- Secrets excluded từ version control (`.gitignore`)

**Implementation:**
```yaml
# docker-compose.yml
env_file:
  - .env
environment:
  - JENKINS_SECRET=${JENKINS_SECRET}
  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-admin}
```

**Setup:**
1. Copy `.env.example` to `.env`
2. Fill in actual secrets
3. `.env` is automatically ignored by git

#### 2. Container Privileges Reduction

**Before:**
- Containers run với `--privileged` flag (full system access)

**After:**
- Specific capabilities: `--cap-add=SYS_RAWIO`, `--cap-add=SYS_ADMIN`
- Specific device mounts: `--device=/dev/ttyUSB0`
- Read-only mounts where possible: `-v /sys/class/gpio:/sys/class/gpio:ro`

**Security Benefits:**
- Reduced attack surface
- Principle of least privilege
- Better isolation giữa containers

### Best Practices

#### Security
- Never commit `.env` files
- Rotate secrets regularly
- Use least-privilege container capabilities
- Monitor cho security updates

#### Testing
- Run tests trên multiple ATS nodes cho validation
- Monitor success rates trong Grafana
- Investigate flaky tests promptly
- Keep test logs cho debugging

#### Monitoring
- Set up Grafana alerts cho low success rates
- Monitor test duration trends
- Track firmware version correlation với failures
- Review metrics regularly

#### Maintenance
- Update Docker images regularly
- Keep ESP-IDF toolchain updated
- Review và update test scripts
- Document hardware changes

---

## Mở Rộng và Scale

### Multi-ATS Scalability

Hệ thống được thiết kế để scale horizontally:

**Để thêm ATS node mới:**

1. Provision Raspberry Pi / Mini PC
2. Install Docker và dependencies
3. Setup Jenkins agent:
   ```bash
   ./provision/raspi-agent/start-agent.sh \
     https://jenkins.example.com \
     raspi-ats-01 \
     YOUR_SECRET_HERE
   ```
4. Assign `ats-node` label
5. Register metrics endpoint trong Prometheus config
6. **Không cần thay đổi Jenkins pipeline**

**Benefits:**
- Parallel test execution
- Increased test capacity
- Redundancy và fault tolerance
- Natural evolution thành hardware test farm

### Multi-Platform Support

Hệ thống hỗ trợ multiple platforms:

**Current:**
- ESP32 (fully implemented)

**Future:**
- nRF52
- Raspberry Pi (image build)
- Other embedded platforms

**Platform Organization:**
```
platforms/
├── ESP32/
│   ├── Jenkinsfile          # ESP32 build pipeline
│   └── Jenkinsfile.test     # ESP32 test pipeline
├── RaspberryPi/
│   ├── Jenkinsfile          # Raspberry Pi image build
│   └── Jenkinsfile.test     # Raspberry Pi test
└── nRF52/
    ├── Jenkinsfile          # nRF52 build pipeline
    └── Jenkinsfile.test     # nRF52 test pipeline
```

**Để thêm platform mới:**

1. Create platform folder: `platforms/NewPlatform/`
2. Create Jenkinsfiles (copy từ ESP32 template)
3. Update `PLATFORM` environment variable
4. Adjust build commands cho new platform
5. Update JCasC config
6. Create build agent image (nếu cần custom toolchain)

### Future Enhancements

**Planned Improvements:**
- [ ] Parallel test execution across multiple ATS nodes
- [ ] External artifact storage (S3/Artifactory)
- [ ] Webhook integration cho GitHub
- [ ] Visual validation với camera + AI
- [ ] Advanced test parameterization
- [ ] Test suite management
- [ ] Alerting rules cho Prometheus

**Architecture Evolution:**
- [ ] Kubernetes deployment option
- [ ] Multi-region support
- [ ] Cloud-based ATS nodes
- [ ] Enhanced firmware features (GPIO, OLED, OTA demos)

---

## Tóm Tắt

### Hệ Thống ATS Là Gì?

Một hệ thống tự động hóa kiểm thử firmware cho embedded devices, tích hợp trực tiếp vào CI/CD pipeline để:

- **Tự động test firmware trên hardware thật** (không chỉ build/unit test)
- **Loại bỏ kiểm thử thủ công**, tăng tốc độ và giảm lỗi
- **Cung cấp metrics và observability** để theo dõi chất lượng firmware
- **Scale được** với multiple ATS nodes và platforms

### Kiến Trúc Chính

**3 Tầng:**
1. **Control Plane** (Xeon Server): Jenkins, Prometheus, Grafana, Build Agents
2. **Execution Plane** (Raspberry Pi): ATS Nodes với hardware access
3. **Device Under Test** (ESP32): Chạy firmware và expose behavior

### Luồng Hoạt Động

```
Commit → Build (Xeon) → Test (Raspberry Pi) → Report (Jenkins/Grafana)
```

### Các Repository

1. **ats-platform-docs**: Tài liệu và specifications
2. **ats-fw-esp32-demo**: Firmware source code và build pipeline
3. **ats-test-esp32-demo**: Test execution framework
4. **ats-ats-node**: Hardware interaction và test orchestration
5. **ats-ci-infra**: CI/CD infrastructure (Jenkins, Prometheus, Grafana)

### Contracts

- **ATS Manifest (v1)**: Contract giữa build, test, và CI system
- **Test Output Contract (v1)**: Standardized test result format

### Key Principles

- **Separation of concerns**: Mỗi repository có trách nhiệm rõ ràng
- **Jenkins is "dumb"**: Chỉ orchestrate, không biết về hardware
- **Hardware abstraction**: Tất cả hardware logic trong `ats-ats-node`
- **Observability first**: Metrics và logs là first-class citizens
- **Scalable by design**: Dễ thêm nodes và platforms mới

---

## Tài Liệu Tham Khảo

- [System Overview](./README.md)
- [CI Structure](./architecture/ci-structure.md)
- [ATS Manifest Spec v1](./architecture/ats-manifest-spec-v1.md)
- [Test Output Contract v1](./architecture/test-output-contract-v1.md)
- [System Implementation](./architecture/system-implementation.md)

---

**Tác giả:** ATS Platform Team  
**Cập nhật lần cuối:** 2024-12-25  
**Phiên bản:** 1.0
