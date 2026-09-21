# Reference Demo Runbook

## Goal

Demonstrate in 2–5 minutes that Picopiece can take an immutable release artifact,
deploy it to real hardware, execute a declared regression scope, and return
traceable evidence.

The demo has two stages:

1. ESP32 proof: validate the existing end-to-end station after relocation.
2. Raspberry Pi 4 flagship: demonstrate Embedded Linux release regression.

The Pi 4 demo is the primary customer-facing demo for the target market. ESP32
is the reliability proof and fallback.

## Current offline mode

While the station is offline:

- use archived ESP32 runs only as report/dashboard fixtures;
- mark every replay screen as `HISTORICAL DEMO DATA`;
- do not show fabricated station uptime or live status;
- build report schema, dashboard data contract, and presentation script;
- do not claim a new validation run occurred.

## Demo A — ESP32 station reactivation

### Purpose

Prove the existing hardware pipeline still works after relocation before adding
Raspberry Pi complexity.

### Preconditions

- station controller online;
- Jenkins agent connected;
- ESP32 detected by stable serial identity;
- container image pinned by digest or known tag;
- known-good firmware and known-fail firmware archived;
- public-safe evidence policy enabled.

### Sequence

1. Run station preflight.
2. Acquire exclusive station lease.
3. Execute known-good firmware:
   - flash;
   - reset;
   - capture UART;
   - run deterministic tests;
   - archive result.
4. Execute known-fail firmware:
   - use an immutable artifact with an intentionally incorrect application
     response;
   - do not patch the test script at runtime;
   - collect deterministic failure and evidence.
5. Restore known-good firmware.
6. Release lease.

### Reactivation gate

- 10 consecutive end-to-end runs without infrastructure error;
- expected PASS and FAIL classification every time;
- no stale serial logs;
- evidence and checksums present;
- recovery returns station to idle;
- one golden evidence set archived for dashboard/report fixtures.

Time-box ESP32 work. Do not expand its feature scope after this gate.

## Demo B — Raspberry Pi 4 flagship

## Demo story

```text
Known-good release
  -> provision real Pi 4
  -> boot and SSH
  -> 20 Linux Core tests PASS
  -> mark run as baseline

Regression release
  -> provision same Pi 4 profile
  -> run same suite/version
  -> boot readiness exceeds threshold
  -> demo service health fails
  -> report shows deterministic regression and evidence
```

## Demo artifacts

Build two immutable internal Yocto releases from the same pinned layer manifest.

### Good release

Suggested profile:

- `rpi4/ats-reference-good`;
- systemd;
- OpenSSH;
- controlled demo user/credential bootstrap;
- `picopiece-demo-health.service` active;
- predictable network configuration;
- test metadata file containing image version;
- no customer data.

### Regression release

Suggested profile:

- `rpi4/ats-reference-regression`;
- same base layer revisions and package set where practical;
- add a deliberate startup delay, for example an image-owned systemd drop-in;
- make `picopiece-demo-health.service` deterministically fail or become
  inactive;
- include release metadata that identifies this as an intentional demo fault.

The fault belongs in the immutable image recipe/profile. The runner and test
pack remain unchanged between good and regression runs.

## Initial Linux Core suite

Target 20 deterministic tests:

1. `linux.boot.console_seen`
2. `linux.boot.kernel_started`
3. `linux.boot.no_kernel_panic`
4. `linux.boot.rootfs_mounted`
5. `linux.boot.systemd_reached_target`
6. `linux.boot.ssh_ready`
7. `linux.identity.expected_machine`
8. `linux.identity.expected_release`
9. `linux.service.ssh_active`
10. `linux.service.demo_health`
11. `linux.service.failed_units`
12. `linux.network.ethernet_link`
13. `linux.network.ip_assigned`
14. `linux.network.gateway_reachable`
15. `linux.time.clock_sane`
16. `linux.time.ntp_state`
17. `linux.storage.rootfs_read_write`
18. `linux.storage.free_space`
19. `linux.memory.minimum_available`
20. `linux.logs.no_critical_signatures`

Tests requiring unavailable Internet or external services must be scoped
explicitly. Do not silently skip them.

## Baseline thresholds

Initial demo thresholds:

```yaml
linux.boot.ssh_ready_seconds:
  absolute_max: 30
  max_increase_percent_from_baseline: 10

linux.storage.free_space_percent:
  minimum: 20

linux.memory.available_mb:
  minimum: 256

linux.service.failed_units:
  maximum: 0
```

Example comparison:

```text
Metric: SSH readiness
Previous release: 16.8 s
Current release: 19.2 s
Delta: +2.4 s / +14.3%
Threshold: maximum +10%
Result: FAIL
```

## Live execution sequence

### Preflight

- dashboard shows station as idle;
- station capabilities match Pi profile;
- artifact checksums verified;
- selected baseline is compatible;
- known-good recovery artifact available;
- test network and redaction policy active.

### Run known-good release

1. Register release and artifacts.
2. Create validation run.
3. Acquire station lease and fencing token.
4. Power off DUT.
5. Switch media to controller.
6. Write `.wic.bz2` using `.wic.bmap`.
7. Verify and switch media to DUT.
8. Start UART capture.
9. Power on.
10. Wait for SSH.
11. Execute `linux-core` pack.
12. Collect journal, metrics, and report.
13. Mark eligible PASS run as demo baseline.
14. Restore station to idle.

### Run regression release

Repeat the exact same process with:

- same hardware configuration;
- same platform profile;
- same test-pack version;
- same thresholds.

Expected deterministic failures:

- SSH readiness baseline increase exceeds 10%;
- demo health service is not active;
- optional failed-unit count is greater than zero.

### Recovery

- collect final failure evidence;
- power off DUT;
- re-provision known-good release if configured;
- boot and run a small station health subset;
- return station to idle or quarantine on recovery failure.

## Evidence expected

- release and run snapshots;
- artifact checksums;
- provisioning log;
- timestamped UART log;
- SSH readiness timestamp;
- `systemctl` status and failed-units output;
- relevant journal excerpts;
- JUnit and structured summary;
- baseline comparison;
- validation report;
- optional public-safe webcam frame.

## Dashboard presentation script

### Opening

Show:

- physical Pi 4 on webcam;
- station state;
- release and artifact checksum;
- selected suite and baseline.

Say:

> This is a release validation run on real hardware. PASS and FAIL are produced
> by declared deterministic assertions.

### During run

Show:

- lease/run ID;
- provisioning progress;
- UART and SSH readiness events;
- test progress;
- sanitized logs.

### Final

Show:

- total PASS/FAIL/SKIP;
- new failures;
- boot-time delta and threshold;
- deterministic service failure;
- evidence links;
- downloadable sample report.

Say:

> This release was validated against the declared suite and hardware
> configuration. The result is not product certification.

## Success criteria

A technical buyer can answer within five minutes:

- what release was tested;
- which exact artifacts were used;
- what physical hardware ran it;
- what suite and thresholds applied;
- why the regression failed;
- which evidence supports the result;
- how to submit their own release.

## Failure contingencies

If station infrastructure fails:

- display `ERROR`, not DUT `FAIL`;
- preserve partial evidence;
- explain station recovery state;
- switch to the most recent clearly labeled historical replay;
- never manually edit a failed result to make the demo pass.

## Recording checklist

- no customer hardware or data visible;
- no home/private area visible;
- no credentials or private IPs in logs;
- camera and timestamps synchronized;
- both artifacts and report are immutable;
- intentional fault disclosed as demo-only;
- report wording uses declared-scope validation, not certification.
