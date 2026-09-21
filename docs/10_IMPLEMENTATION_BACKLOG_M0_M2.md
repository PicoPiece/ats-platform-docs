# Implementation Backlog M0–M2

## Usage

Each item below is intended to become one GitHub issue. Keep issue IDs stable in
design discussion. Add repository labels and assignees when issues are created.

Priority:

- P0: blocks milestone;
- P1: required for milestone exit;
- P2: useful but may move later.

Current constraint: the physical station is offline during relocation.
Offline-safe documentation and fixture work can proceed; hardware acceptance
issues remain blocked until station reactivation.

## M0 — Contracts and ESP32 abstraction

### M0-01 — Freeze current ESP32 golden run

- Priority: P0
- Repository: `ats-ci-infra` / evidence archive
- Dependencies: station reactivation
- Scope: execute and archive known PASS and known FAIL artifacts with complete
  checksums, manifest, logs, JUnit, summary, and metadata.
- Acceptance:
  - both artifacts are immutable and identified by SHA-256;
  - PASS/FAIL result is deterministic;
  - evidence contains no stale data from previous run;
  - run references are documented.

### M0-02 — Define station adapter protocols

- Priority: P0
- Repository: `ats-ats-node`
- Dependencies: station runtime interface contract
- Scope: add typed protocols/interfaces for `Provisioner`,
  `PowerController`, `ConsoleTransport`, `RuntimeTransport`,
  `RecoveryStrategy`, and `EvidenceCollector`.
- Acceptance:
  - interfaces have structured inputs/results/errors;
  - no ESP32-specific type appears in generic orchestrator contract;
  - unit tests cover adapter result/error serialization.

### M0-03 — Add PlatformProfile model and registry

- Priority: P0
- Repository: `ats-ats-node`
- Dependencies: M0-02
- Scope: load versioned profile data and resolve adapter IDs/capabilities.
- Acceptance:
  - ESP32 profile loads from configuration;
  - unknown adapter/profile is rejected;
  - secrets and host `/dev` paths are rejected from profile data.

### M0-04 — Extract ESP32 Provisioner

- Priority: P0
- Repository: `ats-ats-node`
- Dependencies: M0-02, M0-03
- Scope: move esptool logic out of monolithic executor.
- Acceptance:
  - current flash flow runs through `EspToolProvisioner`;
  - port is resolved by stable station mapping;
  - flash result includes duration, tool version, and evidence;
  - generic runner contains no esptool call.

### M0-05 — Extract ESP32 console and reset adapters

- Priority: P0
- Repository: `ats-ats-node`
- Dependencies: M0-02
- Scope: separate USB UART capture and DTR/RTS reset.
- Acceptance:
  - console capture starts before reset;
  - logs are run-scoped and timestamped;
  - reset and console failures use structured error categories;
  - no hardcoded `/dev/ttyUSB0` remains in generic runner.

### M0-06 — Replace runtime test-script mutation

- Priority: P0
- Repository: `ats-ats-node`, `ats-test-esp32-demo`
- Dependencies: M0-01
- Scope: remove regex rewriting of `run_tests.sh`.
- Acceptance:
  - checked-out test-pack files remain unchanged;
  - test input/evidence paths are passed through declared interfaces;
  - test pack checksum before/after execution is identical.

### M0-07 — Remove hardcoded `Hello RKTech` from executor

- Priority: P0
- Repository: `ats-ats-node`, `ats-test-esp32-demo`
- Dependencies: M0-06
- Scope: move all test assertions into versioned test cases.
- Acceptance:
  - executor does not contain application pass strings;
  - manifest test selection maps to executed stable test IDs;
  - summary/JUnit test names match declared tests.

### M0-08 — Extract EvidenceCollector

- Priority: P1
- Repository: `ats-ats-node`
- Dependencies: M0-02
- Scope: centralize evidence paths, checksums, indexing, and finalization.
- Acceptance:
  - every run has isolated evidence root;
  - boot/serial logs are declared in evidence index;
  - index records size/SHA-256 and `finalized: true`;
  - application/storage rejects in-place mutation after finalization;
  - required low-level result files remain compatible with v1;
  - evidence finalization prevents later accidental overwrite.

### M0-09 — Implement generic orchestrator state machine

- Priority: P0
- Repository: `ats-ats-node`
- Dependencies: M0-03 through M0-08
- Scope: replace ESP32-specific executor sequencing with generic run states.
- Acceptance:
  - states and errors match domain contract;
  - adapter calls are bounded by timeouts;
  - infrastructure `ERROR` is distinct from test `FAIL`;
  - current ESP32 external result behavior remains compatible.

### M0-10 — Add unit tests for M0 core

- Priority: P1
- Repository: `ats-ats-node`
- Dependencies: M0-09
- Scope: fake adapters and test orchestrator state/error behavior offline.
- Acceptance:
  - PASS, FAIL, provision error, boot timeout, cancellation, and recovery paths
    are tested;
  - tests require no physical hardware;
  - CI can execute them in a normal container.

### M0-11 — Define run fixture from archived ESP32 evidence

- Priority: P1
- Repository: dashboard/reporting workspace
- Dependencies: archived evidence; can begin offline
- Scope: create sanitized historical fixture for report/dashboard development.
- Acceptance:
  - fixture is marked historical;
  - all secrets/private identifiers are removed;
  - schema validation passes;
  - it cannot be confused with live station state.

### M0-12 — Station reactivation and soak

- Priority: P0
- Repository: `ats-ci-infra`
- Dependencies: relocation complete, M0-01
- Scope: verify power, network, Jenkins agent, Docker, serial, flash, and
  recovery.
- Acceptance:
  - 10 consecutive expected PASS/FAIL executions;
  - zero infrastructure errors;
  - station returns to idle after each run;
  - golden evidence is archived.

## M1 — Raspberry Pi 4 reference validation

### M1-01 — Procure and document Pi reference station

- Priority: P0
- Repository: infrastructure documentation
- Dependencies: BOM approval
- Scope: Pi 4 DUT, controller, media switch, UART, relay, network, UPS, spares.
- Acceptance:
  - serial numbers and stable mappings recorded privately;
  - wiring diagram and safety review complete;
  - controller and DUT are separate;
  - customer image network is isolated.

### M1-02 — Build Pi media safety preflight

- Priority: P0
- Repository: future Raspberry Pi platform adapter
- Dependencies: M1-01
- Scope: block writes unless media matches explicit allow-list.
- Acceptance:
  - controller root/system disks are always rejected;
  - stable identity and capacity are checked;
  - mounted target media is handled safely;
  - destructive test uses disposable media only.

### M1-03 — Create pinned RPi4 Yocto layer manifest

- Priority: P0
- Repository: `yocto_multi_platform`
- Dependencies: none; offline-safe
- Scope: pin exact SHAs using the RK3588 `navonz_v1` pattern.
- Acceptance:
  - clean sync resolves identical revisions twice;
  - manifest artifact is archived;
  - branch-only references are absent from the release build.

### M1-04 — Create ATS reference Yocto layer

- Priority: P0
- Repository: `yocto_multi_platform`
- Dependencies: M1-03
- Scope: add systemd, OpenSSH, demo health service, and release metadata.
- Acceptance:
  - layer is isolated from generic platform configuration;
  - image contains required packages/services;
  - customer-inappropriate `debug-tweaks` is explicitly controlled;
  - image version is queryable at runtime.

### M1-05 — Build immutable good and regression image profiles

- Priority: P0
- Repository: `yocto_multi_platform`
- Dependencies: M1-04
- Scope: create good and intentionally faulty Pi 4 demo images.
- Acceptance:
  - both use the same pinned base revisions;
  - good service is active;
  - regression image introduces declared startup delay/service failure;
  - no runtime mutation creates the fault;
  - WIC, bmap, package manifest, log, and checksums are retained.

### M1-06 — Implement WIC/SDWire Provisioner

- Priority: P0
- Repository: Raspberry Pi platform adapter
- Dependencies: M1-01, M1-02, M1-05
- Scope: write `.wic.bz2` with `.wic.bmap` to controlled media.
- Acceptance:
  - lease token is checked before switching/writing;
  - artifact checksum and media allow-list are enforced;
  - write/flush/verify evidence is captured;
  - interrupted write produces infrastructure error and safe state.

### M1-07 — Implement relay PowerController

- Priority: P0
- Repository: Raspberry Pi platform adapter
- Dependencies: M1-01
- Scope: power state, on/off/cycle with fencing.
- Acceptance:
  - stale token is rejected;
  - operations are bounded and logged;
  - physical state is verified where possible;
  - repeated power cycle does not corrupt controller storage.

### M1-08 — Implement UART ConsoleTransport

- Priority: P0
- Repository: Raspberry Pi platform adapter
- Dependencies: M1-01
- Scope: timestamped Pi boot console capture.
- Acceptance:
  - capture starts before power-on;
  - stable serial mapping is used;
  - log is run-scoped;
  - redaction and size limits apply.

### M1-09 — Implement SSH RuntimeTransport

- Priority: P0
- Repository: Raspberry Pi platform adapter
- Dependencies: M1-04
- Scope: readiness, restricted commands, upload/download, close.
- Acceptance:
  - explicit host-key/bootstrap policy;
  - credentials use secret references;
  - timeout/output limits enforced;
  - command/evidence metadata contains no secret.

### M1-10 — Implement Linux Core test pack

- Priority: P0
- Repository: new `ats-test-linux-core`
- Dependencies: M1-09
- Scope: implement the 20 tests in the reference demo runbook.
- Acceptance:
  - stable IDs and versioned thresholds;
  - tests run unchanged on good/regression images;
  - JUnit and structured results agree;
  - offline unit tests cover parsers and threshold logic.

### M1-11 — Implement known-good baseline comparison

- Priority: P0
- Repository: validation core/reporting
- Dependencies: M1-10
- Scope: compare eligible run metrics and test outcomes.
- Acceptance:
  - canonical `BaselineCompatibilityKey` is stored and hashed;
  - strict key equality is the default machine rule;
  - any compatibility exception has an explicit versioned policy ID;
  - new/resolved/unchanged failures are classified;
  - absolute and percentage deltas are correct;
  - incompatible baseline reports a reason instead of comparing.

### M1-12 — Implement Pi recovery strategy

- Priority: P1
- Repository: Raspberry Pi platform adapter
- Dependencies: M1-06 through M1-09
- Scope: power cycle, retry readiness, restore known-good, quarantine.
- Acceptance:
  - recovery budget is bounded;
  - every action is recorded;
  - failed recovery quarantines station;
  - stale run cannot control station after fencing.

### M1-13 — Generate canonical validation report

- Priority: P0
- Repository: reporting component
- Dependencies: M0-08, M1-11
- Scope: generate JSON and HTML from immutable run/evidence data.
- Acceptance:
  - conforms to report schema v1;
  - results/counts/checksums are consistent;
  - scope limitations are visible;
  - no certification language;
  - sanitized public sample is generated.

### M1-14 — Build dashboard offline/replay prototype

- Priority: P1
- Repository: dashboard
- Dependencies: M0-11, report/event contracts
- Scope: station page, run progress, logs, result, history, report link using
  historical fixture.
- Acceptance:
  - replay is visibly labeled;
  - no fabricated live state or SLA;
  - public/private visibility is enforced in fixture rendering;
  - layout works without station connectivity.

### M1-15 — Connect dashboard MVP to live reference station

- Priority: P1
- Repository: dashboard/backend
- Dependencies: M0-12, M1-13, M1-14
- Scope: consume real events/evidence for one public reference station.
- Acceptance:
  - offline/idle/running/recovery states are accurate;
  - log stream is sanitized;
  - report download uses finalized run;
  - customer data cannot appear in public view.

### M1-16 — Execute and record flagship demo

- Priority: P0
- Repository: operations/content
- Dependencies: M1-05 through M1-15
- Scope: good baseline then regression release on real Pi 4.
- Acceptance:
  - good run passes declared scope;
  - regression produces expected deterministic failures;
  - boot delta exceeds declared threshold;
  - recovery returns station to known-good;
  - 2–5 minute video and downloadable report are produced.

## M2 — External artifact intake and scheduling

### M2-01 — Implement Release/Artifact/ValidationRun persistence

- Priority: P0
- Repository: validation API
- Dependencies: domain model contract
- Scope: persist immutable IDs and relationships.
- Acceptance:
  - one release supports multiple artifacts and runs;
  - run pins exact artifact set and versions;
  - retry creates a new run;
  - tenant/project boundaries are enforced.

### M2-02 — Implement pre-signed artifact intake

- Priority: P0
- Repository: validation API/storage
- Dependencies: M2-01
- Scope: pre-signed upload, quarantine, size/type/SHA-256, retention.
- Acceptance:
  - mismatched size/hash is rejected;
  - unsupported media type is rejected;
  - archive absolute paths, traversal, escaping links, and special files are
    rejected;
  - extracted size, file count, nesting, and expansion ratio limits prevent
    decompression bombs;
  - archive members must match an explicit expected-member policy;
  - upload authorization is distinct from artifact signature;
  - artifact is immutable after acceptance.

### M2-03 — Implement StationLease service

- Priority: P0
- Repository: validation API/station service
- Dependencies: lease contract, M2-01
- Scope: atomic acquire/renew/release/expire with fencing.
- Acceptance:
  - concurrent acquire has one winner;
  - stale renew/release is rejected;
  - fencing token increases monotonically;
  - expiry triggers recovery gate before reuse.

### M2-04 — Implement capability-aware scheduler

- Priority: P0
- Repository: validation API/scheduler
- Dependencies: M2-03
- Scope: match run constraints to healthy station capabilities.
- Acceptance:
  - incompatible station is never selected;
  - queued run remains explainable;
  - lease is acquired before dispatch;
  - cancellation and timeout release or recover safely.

### M2-05 — Emit ATS release metadata from Yocto builds

- Priority: P1
- Repository: `yocto_multi_platform`
- Dependencies: manifest v2 draft, M1-05
- Scope: post-build collect artifacts, revisions, config, logs, and checksums.
- Acceptance:
  - WIC, bmap, package manifest, layer revisions, and build log are described;
  - no secret/host path leaks;
  - output validates against accepted manifest version;
  - normalized names do not rely on timestamp glob ambiguity.

### M2-06 — Replace Raspberry Pi Jenkins stub

- Priority: P1
- Repository: `ats-fw-esp32-demo` or successor build repo, `ats-ci-infra`
- Dependencies: M2-05
- Scope: invoke real Yocto build/archive and trigger validation job.
- Acceptance:
  - no empty `touch` image;
  - build runs on build agent, not station;
  - artifacts and metadata enter registry/archive;
  - validation consumes artifact IDs/roles.

### M2-07 — Add customer evaluation intake workflow

- Priority: P1
- Repository: operations/validation API
- Dependencies: onboarding contract, M2-02
- Scope: capture board, artifacts, recovery, access, scope, thresholds, known
  issues, retention, and contacts.
- Acceptance:
  - required fields block incomplete onboarding;
  - secrets are references, not form/report values;
  - accepted scope produces a reviewable run configuration;
  - customer acknowledgment is retained.

### M2-08 — Run first external artifact evaluation

- Priority: P0
- Repository: operations
- Dependencies: M2-01 through M2-07, M2-09
- Scope: one external image, one declared Pi-compatible DUT/profile, one report.
- Acceptance:
  - artifact follows upload/quarantine path;
  - run uses station lease and immutable snapshot;
  - report and evidence are delivered;
  - customer feedback and next-step decision are recorded.

### M2-09 — Enforce DUT network security profile

- Priority: P0
- Repository: station/network infrastructure
- Dependencies: DUT network security contract, M1-01
- Scope: place uploaded-image DUTs on a dedicated VLAN/interface with
  default-deny firewall and immutable `NetworkAccessProfile`.
- Acceptance:
  - DUT cannot reach Jenkins/control plane, station management, home/office
    LAN, other DUTs, or Internet under the default profile;
  - station runner reaches only declared DUT ports;
  - IPv4 and IPv6 policy are equivalent;
  - restricted DNS/NTP/Internet profiles allow only approved destinations;
  - bandwidth/connection limits and policy-violation evidence work;
  - network profile ID and firewall revision are pinned in the run snapshot.

## Explicitly deferred after M2

- general managed customer Yocto build;
- arbitrary custom BSP engineering;
- RK3588/S905x3 validation adapters without customer demand;
- multi-tenant customer portal;
- automated invoicing;
- CAN/cellular/power packs;
- regulated certification claims;
- formal SLA tiers.

## Milestone exit summary

### M0 exit

- ESP32 uses generic interfaces;
- no runtime test mutation or executor pass-string hardcode;
- 10-run station soak passes;
- offline fixtures and contracts are stable.

### M1 exit

- real Pi 4 good/regression images execute unchanged Linux Core suite;
- report shows baseline regression and evidence;
- station automatically recovers;
- public dashboard MVP uses real sanitized data;
- flagship demo is recorded.

### M2 exit

- external artifact upload reaches the same validation pipeline;
- domain records and station leases are first-class;
- one real external evaluation is completed;
- customer feedback determines M3 priorities.
