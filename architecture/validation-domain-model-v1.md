# Validation Domain Model v1

## Purpose

This document defines the minimum domain model for Picopiece Validation Lab.
It separates customer releases, uploaded or built artifacts, hardware executions,
and station ownership so that build, validation, reporting, and scheduling can
evolve independently.

The model supports both:

- artifact mode: a customer uploads an existing image;
- managed build mode: Picopiece builds one or more artifacts from source.

Both modes must converge at the same `Release` and `Artifact` boundary.

## Core entities

### Release

A customer-visible software release candidate.

Required fields:

- `release_id`: immutable globally unique identifier;
- `organization_id`;
- `project_id`;
- `name`;
- `version`;
- `source_revision`: optional Git SHA or source manifest revision;
- `created_at`;
- `created_by`;
- `provenance_type`: `uploaded`, `managed_build`, or `internal_demo`;
- `status`: `draft`, `ready`, `superseded`, or `withdrawn`;
- `metadata`: customer-supplied non-secret labels.

A release does not contain binary data directly. It owns one or more artifacts.
Creating another validation run does not create another release.

### Artifact

An immutable file associated with a release.

Examples:

- `rootfs.wic.bz2`;
- `rootfs.wic.bmap`;
- `bootloader.bin`;
- `ota.tar`;
- `manifest.json`;
- `firmware-esp32.bin`.

Required fields:

- `artifact_id`;
- `release_id`;
- `name`;
- `role`: `boot`, `rootfs`, `disk_image`, `ota`, `metadata`, `sbom`, or `other`;
- `media_type`;
- `size_bytes`;
- `sha256`;
- `storage_uri`;
- `created_at`;
- `retention_until`;
- `quarantine_status`: `pending`, `accepted`, or `rejected`.

Optional fields:

- compression;
- architecture;
- target machine;
- build provenance reference;
- customer-provided signature metadata.

Artifact bytes are immutable. Replacing bytes requires a new `artifact_id`.
Uploaded artifacts remain in quarantine until they satisfy
[Artifact Intake Security Policy v1](./artifact-intake-security-policy-v1.md).

### ValidationRun

One attempt to execute a declared test scope against one release on one hardware
configuration.

Required fields:

- `run_id`;
- `release_id`;
- exact `artifact_ids` used by the run;
- `station_id`;
- `hardware_config_id`;
- `platform_profile_id`;
- `test_pack_versions`;
- `baseline_compatibility_key`;
- `network_access_profile_id` and firewall policy revision;
- resolved thresholds;
- `created_at`;
- nullable `started_at` and `completed_at`, populated by lifecycle;
- `status`;
- `result`;
- `trigger_type`: `manual`, `ci`, `scheduled`, or `retry`;
- `evidence_root`;
- evidence index finalization state and checksum;
- `report_uri`.

Optional fields:

- `baseline_run_id`;
- `parent_run_id` for retry lineage;
- customer reference;
- build-system reference;
- failure classification.

Run status:

```text
queued
  -> leased
  -> provisioning
  -> booting
  -> testing
  -> completed
```

Recovery and cancellation transitions:

```text
leased | provisioning | booting | testing -> recovering
recovering -> provisioning | booting | testing | completed
any active state -> cancelled
```

### ValidationRun status and result

`status` records lifecycle progress. `result` records the validation outcome.
They are separate fields and must never be inferred from one combined enum.

Allowed status values:

- `queued`;
- `leased`;
- `provisioning`;
- `booting`;
- `testing`;
- `recovering`;
- `completed`;
- `cancelled`.

`result` is `null` while a run is active. Terminal result values:

- `PASS`: all required deterministic assertions passed;
- `FAIL`: at least one required assertion failed;
- `ERROR`: infrastructure or orchestration failure prevented a valid result;
- `INCOMPLETE`: execution intentionally stopped before a valid result.

Required terminal mappings:

| Condition | Final status | Final result |
|---|---|---|
| All required assertions pass | `completed` | `PASS` |
| A required deterministic assertion fails | `completed` | `FAIL` |
| Provision, station, transport, or orchestration timeout | `completed` | `ERROR` |
| Station lost and run cannot safely resume | `completed` | `ERROR` |
| Customer abort | `cancelled` | `INCOMPLETE` |
| Operator cancellation | `cancelled` | `INCOMPLETE` |
| Scheduler cancellation before execution | `cancelled` | `INCOMPLETE` |
| Test assertion intentionally checks a timeout and fails | `completed` | `FAIL` |

An infrastructure timeout is not automatically a test failure. A test timeout
is `FAIL` only when the declared deterministic assertion owns that timeout.

Cancellation still triggers bounded cleanup/recovery. If cleanup also fails,
the run remains `cancelled` / `INCOMPLETE`; the cleanup failure is recorded as
an infrastructure incident and may quarantine the station.

### Station

A physical validation cell containing a controller, connected DUT, and test
capabilities.

Required fields:

- `station_id`;
- `state`: `offline`, `idle`, `leased`, `running`, `recovery`, or `quarantine`;
- `controller_identity`;
- capabilities;
- connected hardware configuration;
- health timestamp;
- current lease reference;
- location and visibility classification.

Capabilities are explicit, for example:

```yaml
platforms: [esp32, raspberrypi4]
provisioning: [esptool, sdwire]
console: [uart]
runtime: [ssh]
power_control: [usb_relay]
evidence: [serial, metrics, webcam]
```

### StationLease

Exclusive, time-bound authority for one validation run to control one station.

Required fields:

- `lease_id`;
- `station_id`;
- `run_id`;
- `owner_id`;
- `acquired_at`;
- `expires_at`;
- `heartbeat_at`;
- monotonically increasing `fencing_token`;
- `status`: `active`, `released`, `expired`, or `revoked`.

Lease behavior is defined in `station-lease-contract-v1.md`.

### HardwareConfig

An immutable description of the hardware used for a run.

Required fields:

- `hardware_config_id`;
- board model;
- hardware revision;
- boot media;
- fixture revision;
- connected peripherals;
- network topology reference;
- serial numbers stored according to visibility policy.

Changing hardware revision, fixture wiring, or material peripherals creates a
new hardware configuration.

### Baseline

A baseline is a reference to a completed, eligible `ValidationRun`; it is not a
copy of test results.

Every completed run stores a `BaselineCompatibilityKey`:

```yaml
platform_profile: rpi4-sd-v1
hardware_revision: "1.5"
test_packs:
  - id: linux-core
    version: 0.1.0
metric_schema: 1
fixture_revision: fixture-v1
calibration_profile: none
```

The canonical key sorts test packs by ID and is stored with a deterministic
hash. Automatic comparison is allowed only when baseline and current keys are
identical.

An explicitly versioned compatibility policy may allow a controlled exception,
for example two hardware revisions proven equivalent for a specific metric
schema. The policy ID and decision reason become report data. The runner/report
must never silently compare incompatible keys.

At minimum the key covers:

- platform profile and hardware revision;
- complete test-pack ID/version set;
- metric schema;
- fixture revision;
- relevant calibration profile.

## Relationships

```text
Organization
  -> Project
      -> Release
          -> Artifact [one or more]
          -> ValidationRun [zero or more]

ValidationRun
  -> Artifact [exact pinned set]
  -> StationLease
  -> HardwareConfig
  -> TestPackVersion [one or more]
  -> Evidence [one or more]
  -> ValidationReport [one]
  -> Baseline ValidationRun [optional]
```

## Identity and immutability rules

1. IDs are generated by the platform and never reused.
2. Release metadata may be completed while `draft`; artifact bytes never change.
3. A run snapshots all resolved inputs before provisioning starts.
4. A report generated after completion references the immutable run snapshot.
5. Retrying creates a new run with `parent_run_id`; it never overwrites history.
6. Evidence may be appended while a run is active. Finalization writes an
   evidence index containing each object SHA-256, size, and path, then sets
   `finalized: true`. Application and filesystem policy must reject mutation
   after finalization.
7. Customer deletion and retention workflows remove storage according to policy
   while preserving only legally permitted audit metadata.

The MVP may enforce evidence immutability with permissions and application
guards. A later object-store backend may add content-addressed keys, versioning,
and Object Lock without changing report semantics.

## Baseline comparison model

Each metric comparison records:

- baseline and current `BaselineCompatibilityKey` hashes;
- compatibility decision and optional policy ID;
- metric ID and unit;
- baseline value;
- current value;
- absolute delta;
- percentage delta when valid;
- threshold and comparison operator;
- deterministic result;
- reason when comparison is unavailable.

Example:

```yaml
metric_id: boot.ssh_ready_seconds
unit: seconds
baseline: 16.8
current: 19.2
delta: 2.4
delta_percent: 14.3
threshold:
  operator: max_increase_percent
  value: 10
result: FAIL
```

## Validation statement

The platform may state:

> Release X was validated against Test Suite Y, version Z, on Hardware
> Configuration H at Time T.

The platform must not represent this result as product certification, proof that
software is defect-free, or verification outside the declared test scope.

## MVP persistence

The MVP may use filesystem metadata or a small relational database, but it must
preserve the entity boundaries and immutable IDs above. Jenkins build numbers
may be stored as external references; they are not primary domain identities.
