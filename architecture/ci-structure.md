# CI/CD Infrastructure Structure

## Overview

The ATS CI/CD infrastructure is built on Jenkins with a multi-platform, scalable architecture. It uses **Configuration as Code (JCasC)** for automated Jenkins setup and supports multiple hardware platforms (ESP32, Raspberry Pi, nRF52, etc.) through a folder-based organization.

## Architecture Components

### 1. Jenkins Infrastructure

The CI infrastructure runs on Docker Compose with the following services:

```
ats-ci-infra/
├── docker-compose.yml          # Main orchestration file
├── jenkins/
│   ├── jenkins_home/           # Jenkins data volume
│   ├── jcasc/                  # Configuration as Code files
│   │   ├── jenkins.yaml        # Main JCasC configuration
│   │   └── setup.sh            # Plugin installation script
│   └── fw-build/
│       └── Dockerfile          # Custom build agent image
├── prometheus/
│   └── prometheus.yml          # Metrics collection config
└── grafana/
    ├── provisioning/          # Grafana provisioning
    └── dashboards/             # Custom dashboards
```

#### Services

- **Jenkins Master** (`jenkins-master`)
  - Orchestrates all CI/CD pipelines
  - Manages build and test agents
  - Exposes UI on port `8080`
  - Agent communication on port `50000`

- **Jenkins Build Agent** (`jenkins-build-agent`)
  - Custom Docker image: `ats-fw-build:esp32`
  - Pre-installed ESP-IDF v5.2 toolchain
  - Label: `fw-build`
  - Handles firmware compilation

- **Prometheus** (`prometheus`)
  - Metrics collection and storage
  - Exposes UI on port `9090`

- **Grafana** (`grafana`)
  - Visualization and dashboards
  - Exposes UI on port `3000`
  - Default credentials: `admin/admin`

### 2. Jenkins Folder Structure

Jobs are organized by platform in a hierarchical folder structure:

```
Jenkins UI
└── platforms/
    ├── ESP32/
    │   ├── ats-fw-esp32-demo              # Build pipeline
    │   └── ats-fw-esp32-demo-ESP32-test   # Test pipeline
    ├── RaspberryPi/
    │   └── (future: image build jobs)
    └── nRF52/
        └── (future: nRF52 build jobs)
```

This structure enables:
- **Scalability**: Easy to add new platforms
- **Organization**: Clear separation of platform-specific jobs
- **Maintainability**: Platform teams can own their folders

### 3. Pipeline Structure

#### Build Pipeline (`Jenkinsfile`)

Location: `platforms/{PLATFORM}/Jenkinsfile`

**Purpose**: Build firmware and create artifacts

**Stages**:
1. **Checkout Source**
   - Agent: `fw-build`
   - Checks out code from specified branch (configurable via `BRANCH_NAME` parameter)

2. **Build ESP32 Firmware**
   - Agent: `fw-build`
   - Sets up ESP-IDF environment
   - Runs `idf.py build`
   - Produces firmware binary: `firmware-{PLATFORM}.bin`

3. **Archive Firmware Artifact**
   - Agent: `fw-build`
   - Archives firmware binary for downstream jobs

4. **Generate ATS Manifest**
   - Agent: `fw-build`
   - Creates `ats-manifest.yaml` with:
     - Build metadata (CI system, job name, build number)
     - Git information (repo, commit, branch)
     - Artifact checksum (SHA256)
     - Device target information
     - Test plan references

5. **Tag Firmware**
   - Agent: `fw-build`
   - Creates Git tag: `{TAG_PREFIX}-{BUILD_NUMBER}-{COMMIT_SHORT}`
   - Attempts to push tag (gracefully handles credential issues)

6. **Trigger Test Pipeline** (optional)
   - Condition: `TRIGGER_TEST == true`
   - Triggers test pipeline with build parameters
   - Handles cases where test job doesn't exist yet

**Parameters**:
- `BRANCH_NAME`: Git branch to build (default: `main`)
- `TRIGGER_TEST`: Auto-trigger test pipeline (default: `true`)
- `TAG_PREFIX`: Prefix for firmware tags (default: `esp32-fw`)

**Environment Variables**:
- `PLATFORM`: Platform identifier (e.g., `ESP32`)
- `FW_ARTIFACT`: Artifact filename (e.g., `firmware-esp32.bin`)

#### Test Pipeline (`Jenkinsfile.test`)

Location: `platforms/{PLATFORM}/Jenkinsfile.test`

**Purpose**: Test firmware on ATS hardware nodes

**Stages**:
1. **Copy Firmware Artifact**
   - Agent: `ats-node` (or custom label)
   - Copies firmware and manifest from build job
   - Validates artifact presence

2. **Flash Firmware**
   - Agent: `ats-node`
   - Executes `flash_fw.sh` script
   - Flashes firmware to ESP32 device

3. **Run Tests**
   - Agent: `ats-node`
   - Executes `run_tests.sh` script
   - Captures test results and logs

4. **Archive Test Reports**
   - Agent: `ats-node`
   - Archives test reports
   - Publishes test results (JUnit XML format)

**Parameters**:
- `BUILD_JOB_NAME`: Full path to build job (e.g., `platforms/ESP32/ats-fw-esp32-demo`)
- `BUILD_NUMBER`: Build number to test
- `FW_ARTIFACT`: Firmware artifact name
- `PLATFORM`: Platform name
- `ATS_NODE_LABEL`: Label for ATS node pool (default: `ats-node`)

### 4. Build Agent Setup

#### Custom Docker Image: `ats-fw-build:esp32`

**Base**: `jenkins/inbound-agent:latest`

**Key Components**:
- **ESP-IDF v5.2**: Pre-installed in `/opt/esp/idf`
- **Python Environment**: ESP-IDF Python tools in `/home/jenkins/.espressif`
- **Build Tools**: CMake, Ninja, GCC, Python 3, Git
- **User**: Runs as `jenkins` user (not root)

**Build Process**:
```bash
cd ats-ci-infra/jenkins/fw-build
docker build -t ats-fw-build:esp32 .
```

**Features**:
- ESP-IDF environment automatically sourced in `.bashrc`
- Git safe directory configured for ESP-IDF repo
- All tools accessible to Jenkins agent

### 5. Configuration as Code (JCasC)

#### Overview

Jenkins configuration is managed via YAML files, enabling:
- **Version Control**: All Jenkins config in Git
- **Automated Setup**: Jobs and folders created on Jenkins restart
- **Reproducibility**: Same config across environments

#### Configuration File

Location: `jenkins/jcasc/jenkins.yaml`

**Structure**:
```yaml
jenkins:
  systemMessage: "ATS CI/CD Infrastructure - Multi-Platform Firmware Build System"

unclassified:
  location:
    adminAddress: "admin@ats-ci.local"
    url: "http://localhost:8080/"

tool:
  git:
    installations:
      - name: "Default"
        home: "git"

jobs:
  - script: |
      // Job DSL script to create folders and jobs
      folder('platforms') { ... }
      folder('platforms/ESP32') { ... }
      pipelineJob('platforms/ESP32/ats-fw-esp32-demo') { ... }
      pipelineJob('platforms/ESP32/ats-fw-esp32-demo-ESP32-test') { ... }
```

#### Setup Process

1. **Install Plugins**:
   ```bash
   cd ats-ci-infra
   ./jenkins/jcasc/setup.sh
   ```
   Installs: `configuration-as-code`, `job-dsl`, `workflow-aggregator`, `cloudbees-folder`

2. **Mount Configuration**:
   - JCasC config mounted at `/var/jenkins_home/casc/jenkins.yaml`
   - Environment variable: `CASC_JENKINS_CONFIG=/var/jenkins_home/casc/jenkins.yaml`

3. **Auto-Load on Restart**:
   - Jenkins automatically loads config on startup
   - Jobs and folders are created/updated automatically

#### Updating Configuration

1. Edit `jenkins/jcasc/jenkins.yaml`
2. Restart Jenkins: `docker compose restart jenkins`
3. Or reload via UI: **Manage Jenkins → Configuration as Code → Reload existing configuration**

### 6. Multi-Platform Support

#### Platform Organization

Each platform has its own folder with dedicated pipelines:

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

#### Adding a New Platform

1. **Create Platform Folder**:
   ```bash
   mkdir -p platforms/NewPlatform
   ```

2. **Create Jenkinsfiles**:
   - Copy `platforms/ESP32/Jenkinsfile` as template
   - Update `PLATFORM` environment variable
   - Adjust build commands for new platform

3. **Update JCasC Config**:
   - Add folder definition in `jenkins/jcasc/jenkins.yaml`
   - Add build and test job definitions
   - Restart Jenkins

4. **Create Build Agent** (if needed):
   - Create new Dockerfile for platform-specific toolchain
   - Build custom image: `ats-fw-build:{platform}`
   - Update `docker-compose.yml` with new agent service

### 7. Artifact Management

#### Build Artifacts

- **Firmware Binary**: `firmware-{PLATFORM}.bin`
- **ATS Manifest**: `ats-manifest.yaml`
- **Build Logs**: Stored in Jenkins build history

#### Artifact Flow

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
Flashes to hardware
    ↓
Runs tests
    ↓
Archives test reports
```

### 8. Network Architecture

All services run on Docker network `ats-net`:

```
ats-net (bridge network)
├── jenkins-master (8080, 50000)
├── jenkins-build-agent (fw-build label)
├── prometheus (9090)
└── grafana (3000)
```

**External Access**:
- Jenkins: `http://localhost:8080`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000`

**Internal Communication**:
- Agents connect to Jenkins via `http://jenkins:8080`
- Prometheus scrapes metrics from ATS nodes
- Grafana queries Prometheus

### 9. Key Design Decisions

#### Separation of Build and Test

- **Build Pipeline**: Runs on `fw-build` agent (Xeon server)
- **Test Pipeline**: Runs on `ats-node` agents (Raspberry Pi)
- **Rationale**: Build requires heavy compute, test requires hardware access

#### Platform-Aware Structure

- **Folders**: Organize jobs by platform
- **Naming**: `{job-name}-{platform}-test` for test jobs
- **Rationale**: Scales to multiple platforms without job name conflicts

#### Configuration as Code

- **JCasC**: All Jenkins config in YAML
- **Job DSL**: Programmatic job creation
- **Rationale**: Version control, reproducibility, automated setup

#### Custom Build Agents

- **Pre-installed Toolchains**: ESP-IDF ready in container
- **Dedicated Images**: One image per platform toolchain
- **Rationale**: Faster builds, consistent environments

### 10. Troubleshooting

#### Common Issues

1. **Build Agent Not Connected**
   - Check `JENKINS_SECRET` in `docker-compose.yml`
   - Verify agent can reach Jenkins: `docker exec jenkins-build-agent curl http://jenkins:8080`

2. **JCasC Config Not Loading**
   - Verify mount: `docker exec jenkins-master ls -la /var/jenkins_home/casc/`
   - Check environment variable: `docker exec jenkins-master env | grep CASC`

3. **Pipeline Fails with "idf.py: command not found"**
   - Verify ESP-IDF in build agent: `docker exec jenkins-build-agent idf.py --version`
   - Check environment sourcing in Jenkinsfile

4. **Test Pipeline Not Triggered**
   - Check `TRIGGER_TEST` parameter
   - Verify test job exists: `platforms/ESP32/ats-fw-esp32-demo-ESP32-test`
   - Check build logs for trigger errors

### 11. Future Enhancements

- **Multi-Agent Pools**: Support multiple build agents per platform
- **Parallel Testing**: Run tests on multiple ATS nodes simultaneously
- **Artifact Storage**: External artifact storage (S3, Artifactory)
- **Pipeline Templates**: Shared library for common pipeline stages
- **Webhook Integration**: Auto-trigger on Git push/PR

---

## Related Documentation

- [System Overview](../README.md)
- [CI Flow](./ci-flow.md)
- [ATS Node Design](./ats-node-design.md)

