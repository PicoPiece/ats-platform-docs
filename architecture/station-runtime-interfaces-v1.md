# Station Runtime Interfaces v1

## Purpose

This document defines the boundaries used by a station runner. The goal is to
avoid embedding ESP32, Raspberry Pi, SSH, UART, relay, or test-pack behavior in
one platform-specific executor.

The station orchestrator owns sequencing and state transitions. Adapters own
hardware or protocol behavior.

## Design rules

1. `PlatformProfile` is configuration data, not executable orchestration.
2. Each interface has one narrow responsibility.
3. Adapters return structured results and evidence references.
4. No adapter may silently select arbitrary host devices.
5. Test packs do not flash hardware or own station recovery.
6. Evidence collection does not decide PASS or FAIL.
7. Jenkins invokes a run; it does not implement station behavior.
8. AI analysis is downstream of deterministic results.

## PlatformProfile

Describes how a release maps to a target platform.

Example fields:

```yaml
profile_id: raspberrypi4-sd-v1
platform: raspberrypi4
architecture: aarch64
artifact_roles:
  required: [disk_image]
  optional: [bmap, sbom]
provisioner: sdwire_wic
power_controller: usb_relay
console_transport: uart
runtime_transport: ssh
recovery_strategy: reflash_known_good
timeouts:
  provision_seconds: 600
  boot_seconds: 180
  runtime_connect_seconds: 120
capability_requirements:
  - sdwire
  - uart
  - ssh_network
  - usb_relay
```

The profile must not contain customer secrets or host-specific `/dev/sdX`
paths. Station-local device mapping resolves stable identities at runtime.

## Provisioner

Responsible for deploying the selected artifact set to the DUT.

Inputs:

- immutable run snapshot;
- artifact files and checksums;
- platform profile;
- station-local device mapping;
- active station lease and fencing token.

Responsibilities:

- validate required artifact roles;
- validate target media against an allow-list;
- prepare/decompress artifacts;
- write or flash;
- verify written data or tool-reported checksum;
- emit timing and provisioning evidence;
- leave the DUT in a defined pre-boot state.

Non-responsibilities:

- power sequencing beyond calls delegated to `PowerController`;
- runtime health checks;
- test execution;
- report generation.

Example implementations:

- `EspToolProvisioner`;
- `WicSdWireProvisioner`;
- `RockchipMaskromProvisioner`;
- future OTA provisioner.

## PowerController

Controls DUT power and reset lines.

Operations:

- `power_on`;
- `power_off`;
- `power_cycle`;
- `reset`;
- `get_power_state`.

Every operation must:

- validate the active lease fencing token;
- be idempotent where practical;
- record timestamps and controller response;
- use bounded timeouts.

Example implementations:

- USB relay;
- GPIO relay;
- managed PDU;
- ESP32 DTR/RTS reset adapter.

## ConsoleTransport

Captures early boot and out-of-band console data.

Operations:

- `open`;
- `start_capture`;
- `write` when explicitly allowed;
- `stop_capture`;
- `close`.

Typical protocols:

- UART;
- USB serial;
- future IPMI serial-over-LAN.

Output is timestamped evidence. Console text must pass through redaction before
public display.

## RuntimeTransport

Communicates with the target after the operating system or firmware runtime is
available.

Typical implementations:

- SSH;
- HTTP/HTTPS API;
- MQTT;
- serial command protocol;
- ADB or another platform-specific runtime protocol.

Operations:

- `wait_ready`;
- `execute` through a restricted command specification;
- `upload`;
- `download`;
- `request`;
- `close`.

The MVP SSH transport must use explicit host-key policy, bounded commands,
timeouts, output limits, and secret references. Customer credentials are never
stored in release metadata or evidence.

## RecoveryStrategy

Attempts to return a station and DUT to a known state after infrastructure or
DUT failure.

Inputs:

- run state;
- failure category;
- remaining recovery budget;
- platform profile;
- active lease.

Possible actions:

- reconnect transport;
- reset;
- power cycle;
- re-provision current release;
- restore known-good artifact;
- quarantine station.

Recovery produces a structured action history. It does not erase the failed
attempt. A rerun creates a new `ValidationRun`.

## TestPack

Defines deterministic test cases independently from provisioning and station
control.

Metadata:

- pack ID and semantic version;
- supported profiles/capabilities;
- test IDs;
- required runtime transports;
- required fixtures;
- default thresholds;
- evidence declarations.

Execution inputs:

- immutable run context;
- approved transport handles;
- resolved thresholds;
- output/evidence directory.

Test result:

```yaml
test_id: linux.service.demo_health
status: PASS
duration_seconds: 0.42
assertions:
  - actual: active
    expected: active
evidence:
  - evidence/service-demo-health.txt
```

Test packs may request approved recovery or diagnostic actions through the
orchestrator. They must not access arbitrary host devices or modify runner code.

## EvidenceCollector

Collects, indexes, redacts, and finalizes evidence.

Responsibilities:

- allocate a run evidence directory;
- capture metadata and timestamps;
- receive serial logs, command results, metrics, screenshots, and traces;
- calculate checksums;
- apply visibility and redaction policy;
- write an evidence index;
- finalize evidence as read-only.

Evidence visibility:

- `public_demo`;
- `customer_private`;
- `internal_sensitive`;
- `secret_rejected`.

The collector does not determine test results. It preserves the evidence used by
deterministic assertions and later analysis.

## Station orchestrator

The orchestrator coordinates interfaces in this order:

```text
resolve run snapshot
  -> acquire station lease
  -> preflight station capabilities
  -> start evidence collection
  -> provision
  -> start console capture
  -> power/reset DUT
  -> wait for runtime transport
  -> execute test packs
  -> collect final diagnostics
  -> generate deterministic result
  -> finalize evidence and report
  -> recover or return station to idle
  -> release lease
```

It must revalidate the fencing token before every destructive hardware action.

## Error categories

Adapters return one of:

- `configuration_error`;
- `artifact_error`;
- `station_error`;
- `provision_error`;
- `boot_error`;
- `transport_error`;
- `test_failure`;
- `collection_error`;
- `timeout`;
- `cancelled`.

`test_failure` contributes to result `FAIL`. Infrastructure categories normally
produce `ERROR`, not `FAIL`, unless a declared test explicitly evaluates that
condition.

## ESP32 compatibility

The current ESP32 flow maps as follows:

- `PlatformProfile`: ESP32 DevKit metadata;
- `Provisioner`: existing esptool behavior;
- `PowerController`: DTR/RTS reset;
- `ConsoleTransport`: USB UART;
- `RuntimeTransport`: optional serial command protocol;
- `RecoveryStrategy`: reset/reflash;
- `TestPack`: ESP32 demo tests;
- `EvidenceCollector`: current summary, JUnit, metadata, metrics, and boot logs.

M0 is complete when the current ESP32 PASS/FAIL flow runs through these
boundaries without changing its externally visible result contract.
