# Picopiece Validation Lab — Concept & System Overview

## Mission
Build a Vietnam-based managed embedded validation lab that helps small international embedded teams validate software releases on real hardware.

The service does **not** compete with the customer's product engineering team. The customer builds the product; Picopiece independently validates release candidates against a repeatable regression suite and returns traceable evidence.

> Customer image/release candidate → deploy to real DUT → run regression → collect evidence → analyze failures → issue validation report.

## Core positioning
**Managed Embedded Release Validation Lab**
**Hardware CI & Regression Validation for Embedded Linux**

Suggested message:

> You build the product. We continuously prove that your software still works on the real hardware.

The system never claims that an image is “bug free.” The defensible statement is:

> Image X passed Test Suite Y, version Z, on Hardware Configuration H, at Time T.

## Initial target customer
Small international embedded teams, ideally:
- 5–20 embedded / firmware / software engineers
- roughly 10–50 total employees
- Linux / Yocto / Raspberry Pi Compute Module / NXP / Jetson / similar devices
- no dedicated hardware-CI / validation infrastructure team
- frequent firmware or image releases
- willingness to use an external managed validation lab

Initial geographic priority:
1. United States
2. Europe
3. Canada / Australia
4. Japan

## What the customer sends
Two supported workflows:

### Artifact mode
Customer CI uploads:
- `.wic`, `.img`, `.bin`, `.tar`, OTA package, or other release artifact
- metadata: build ID, git SHA, release name, target HW revision

### Managed build mode
Customer supplies:
- Git repository or manifest
- Yocto layers / config
- build instructions
- target BSP definition

Picopiece build server produces the release image, archives it, and sends it to the validation flow.

Artifact mode is the default evaluation path. Managed build mode is conditional
and separately scoped. Internal Raspberry Pi reference images may be built with
the existing `yocto_multi_platform` system without making customer managed
build a generally available service.

## End-to-end architecture

```text
Customer Git / Image
        |
        v
Artifact Ingestion / Yocto Build Farm
        |
        v
Release Registry
        |
        v
Jenkins / Scheduler
        |
        v
Station Controller (Docker)
        |
        +------> Flash / Provision DUT
        |
        +------> Power / Reset / Recovery
        |
        +------> Test Execution
        |
        +------> Serial / SSH / Network / Peripheral Evidence
        |
        v
Evidence Store
        |
        +------> Deterministic PASS / FAIL
        |
        +------> AI-assisted classification / comparison / summary
        |
        v
Validation Report + Dashboard + API
```

## Core domain model

The platform separates three customer-facing execution entities:

- **Release**: one immutable software release candidate and its declared
  metadata;
- **Artifact**: one immutable file belonging to a release, such as
  `rootfs.wic`, `bootloader.bin`, `ota.tar`, or `manifest.json`;
- **ValidationRun**: one execution of an exact artifact set, test scope, and
  hardware configuration.

One release can own multiple artifacts and can be validated by multiple runs on
different hardware revisions, stations, schedules, or retry attempts.

`ValidationRun.status` and `ValidationRun.result` are separate:

- status tracks lifecycle (`queued` through `completed` or `cancelled`);
- result is `PASS`, `FAIL`, `ERROR`, or `INCOMPLETE`;
- cancellation maps to `cancelled` / `INCOMPLETE`;
- infrastructure failure maps to `completed` / `ERROR`.

Physical execution additionally uses:

- **Station**: one controller, DUT, fixture, and capability set;
- **StationLease**: exclusive time-bound authority for one run to control one
  station, including heartbeat, expiry, and fencing token;
- **HardwareConfig**: immutable board, revision, media, fixture, and peripheral
  description;
- **Baseline**: a reference to a compatible completed run used for regression
  comparison.

Baseline comparison is machine-gated by `BaselineCompatibilityKey`, covering
platform profile, hardware revision, test-pack versions, metric schema, fixture
revision, and calibration profile.

Jenkins build numbers may be retained as external references, but they are not
the primary identity for releases or runs.

## Station runtime boundaries

Station behavior is split into narrow responsibilities:

- `PlatformProfile`: declarative target requirements and adapter selection;
- `Provisioner`: writes/flashes the selected artifact set;
- `PowerController`: power, reset, and power-cycle operations;
- `ConsoleTransport`: early boot/out-of-band console such as UART;
- `RuntimeTransport`: post-boot SSH, API, MQTT, or another runtime protocol;
- `RecoveryStrategy`: bounded reset/reflash/quarantine behavior;
- `TestPack`: versioned deterministic assertions;
- `EvidenceCollector`: capture, checksum, redact, index, and finalize evidence.

The orchestrator coordinates these interfaces. Platform-specific adapters must
not become one broad `TargetAdapter` god-object.

## Existing / planned physical infrastructure
- Xeon-class build server for Yocto / image builds
- Mini PC for station orchestration
- Docker-based station workers
- Jenkins pipeline
- Raspberry Pi reference DUT
- capability to add customer DUTs
- stable network
- low-cost / self-managed energy infrastructure
- UPS / backup power to be treated as a reliability requirement
- webcam for live public demo station
- remote power control
- optional sensors for temperature / environment
- future test instruments such as Joulescope, USB-CAN, cellular modems, BLE radios

## Untrusted artifact and DUT boundaries

Customer archives remain in quarantine until bounded inspection validates
member paths, links, file count, extracted size, expansion ratio, and explicit
expected members. Uploaded archive contents are never executed on the control
plane.

Customer images run on a dedicated DUT VLAN/interface with default-deny policy:

- no DUT access to Jenkins/control plane;
- no DUT access to station management or home/office LAN;
- no cross-DUT/customer traffic;
- no Internet by default;
- station runner may access only declared DUT ports;
- Internet/DNS/NTP requires an explicit restricted network profile.

## Test taxonomy
Tests should be reusable by design.

### L0 — Generic Linux validation
Reusable across almost every Linux target:
- boot / reboot
- systemd service health
- CPU / memory
- filesystem
- network
- RTC / NTP
- log rotation
- storage
- SSH
- watchdog
- kernel panic detection

### L1 — BSP / platform validation
Hardware-specific:
- GPIO
- I2C
- SPI
- UART
- USB
- Ethernet PHY
- Wi-Fi
- Bluetooth
- camera
- storage controllers
- power rails
- thermal behavior

### L2 — Product feature / regression
Product-independent framework, product-specific assertions:
- OTA
- rollback
- daemon behavior
- application health
- peripheral state
- API behavior
- network services

### L3 — Customer-specific validation
Private customer requirements:
- proprietary features
- customer fixtures
- product-specific workflows
- private acceptance criteria

The long-term IP is the reusable L0/L1/L2 test library and the framework that lets L3 tests plug in cleanly.

## Suggested GitHub structure

```text
picopiece-validation/
├── validation-core/
│   ├── runner/
│   ├── evidence/
│   ├── reporting/
│   ├── station/
│   └── providers/
├── test-packs/
│   ├── linux-core/
│   ├── bsp/
│   ├── power/
│   ├── ota/
│   ├── connectivity/
│   ├── ble/
│   ├── cellular/
│   └── can/
├── platform-adapters/
│   ├── raspberry-pi/
│   ├── yocto-generic/
│   └── customer-template/
├── customer-config/
│   └── examples/
├── dashboard/
├── docs/
└── examples/
```

Customers should not fork the entire framework. They should consume the core and supply:
- target config
- platform adapter
- private tests
- thresholds
- release metadata

The release registry and validation runner consume artifacts the same way
whether they were uploaded by customer CI or produced by a managed build.

## AI role
AI is not the source of truth for PASS / FAIL.

Deterministic assertions stay deterministic:
- exact return code
- service active/inactive
- power threshold
- packet loss threshold
- timeout
- log signature
- expected protocol response

AI is used after evidence exists:
- failure classification
- compare current release with prior releases
- summarize logs
- suggest likely subsystem
- propose next debugging step
- draft new tests
- create customer-facing incident summaries
- detect similar historical failures

Provider abstraction should allow:
- OpenAI
- DeepSeek
- future decision models such as Jev / System-One style models
- other providers later

Suggested interface:

```text
AIProvider
  classify_failure()
  compare_release()
  summarize_evidence()
  propose_next_steps()
  draft_test()
```

Do not let model providers directly execute arbitrary shell commands. Expose narrow validated tools only.

## Live Validation Lab dashboard
The dashboard is part of sales infrastructure, not just an engineering UI.

### Public demo page
Use a safe reference station only, never customer confidential data.

Show:
- live webcam of the reference DUT
- station online/offline state
- current image / build ID
- test currently executing
- total / pass / fail / skipped
- sanitized serial / test log stream
- last release comparison
- power graph when applicable
- recent run history
- downloadable sample report
- CTA: **Run a release through this lab**

The goal is that a CTO or engineering lead can understand the service in less than five minutes.

While the physical station is offline, archived runs may be used only as
clearly labeled historical replay fixtures. The dashboard must never fabricate
live station state or uptime.

### Private customer dashboard
Later:
- authentication
- customer-specific stations
- artifact upload
- private logs
- report history
- release comparisons
- traceability
- API token management

## Demonstration flow
The flagship demo should be:

```text
Git commit / upload image
        ↓
Yocto build
        ↓
Artifact registered
        ↓
Real Raspberry Pi flashed
        ↓
30+ regression tests
        ↓
One intentional regression detected
        ↓
Evidence captured
        ↓
AI-assisted explanation
        ↓
Professional validation report
```

Create:
- landing page
- 2–5 minute flagship YouTube video
- downloadable sample report
- live public station
- GitHub reference project

Near-term order:

1. reactivate and soak the existing ESP32 flow;
2. use real ESP32 evidence for the first live dashboard MVP;
3. build the Raspberry Pi 4 known-good/regression flagship;
4. accept an external artifact only after the same evidence/report pipeline is
   stable.

## Expansion test packs

### Power validation
High-value because results are quantitative and repeatable:
- idle current
- boot energy
- workload energy
- suspend / sleep
- regression versus baseline
- battery / power state transition

### Connectivity
- Ethernet
- Wi-Fi
- BLE
- cellular / SIM
- reconnect behavior
- signal-loss recovery
- roaming / interface switching where feasible
- network throughput / latency

### OTA / recovery
- upgrade
- rollback
- corrupted image
- interrupted update
- boot-loop recovery
- watchdog recovery
- power interruption

### CAN / vehicle-bus capability
Architect CAN as a generic interface, but do not make automotive the first sales focus.

Possible future pack:
- CAN / CAN-FD capture
- DBC signal decoding
- stimulus / replay
- timeout / timing validation
- bus-off / reconnect scenarios
- gateway validation
- trace evidence
- UDS later

CAN also opens markets beyond automotive:
- EV charging
- battery systems
- industrial vehicles
- agriculture
- heavy equipment
- robotics

Automotive-specific validation should come later because ISO 26262, ASPICE, cybersecurity, OEM procurement, and certification expectations increase sales and delivery friction.

## Home lab versus commercial premises
A home lab is sufficient for the initial business.

Prioritize:
- stable rack
- UPS
- backup network
- remote power switching
- fixed cameras
- clean wiring
- environmental monitoring
- fire / electrical safety
- spare station controller
- spare DUT
- access control
- no personal/private area visible on webcam

A dedicated facility becomes relevant when:
- customer security requirements demand it
- the station count becomes large
- physical access audits are required
- SLA requirements justify the expense

## Canada strategy
If the founder relocates to Canada, keep the model hybrid:

**Canada**
- sales
- customer meetings
- industry networking
- conferences
- local demo / edge station if required

**Vietnam**
- main hardware lab
- build farm
- long-running regression
- operations
- lower-cost scaling

Do not duplicate the entire farm until customer requirements justify it.

## Strategic rule
Every customer request should create at least one of:
1. reusable test IP,
2. reusable infrastructure capability,
3. recurring revenue.

Otherwise the company risks becoming pure custom consulting.
