# Picopiece Validation Report Schema v1

## Purpose

The validation report is the customer-facing summary of one immutable
`ValidationRun`. It explains what was tested, on which hardware, with which
artifacts and test scope, and what deterministic evidence was produced.

The report does not certify the product and does not claim that software is
defect-free.

Approved statement:

> Release X was validated against Test Suite Y, version Z, on Hardware
> Configuration H at Time T.

## Report outputs

Each terminal run (`completed` or `cancelled`) should produce:

- `validation-report.json`: machine-readable canonical report;
- `validation-report.html`: customer-readable report;
- optional PDF rendered from the same canonical data;
- `evidence-index.json`;
- existing low-level outputs such as JUnit, summary, metadata, and logs.

HTML and PDF are views. `validation-report.json` is the source of truth.

## Required report sections

### 1. Validation statement

- report ID and run ID;
- release name/version;
- test scope and version;
- hardware configuration;
- execution timestamp;
- final result;
- explicit scope limitation.

### 2. Release and artifacts

- immutable release ID;
- provenance: upload, managed build, or internal demo;
- source revision when declared;
- exact artifact IDs, roles, filenames, sizes, and SHA-256 values.

### 3. Hardware configuration

- station ID or customer-safe alias;
- board model and hardware revision;
- boot media;
- fixture revision;
- material peripherals;
- platform profile version.

### 4. Execution summary

- start/end time and duration;
- final lifecycle status;
- final result;
- PASS/FAIL/SKIP counts;
- infrastructure error count;
- retry lineage;
- station recovery actions;
- deterministic final result.

### 5. Test results

For every test:

- stable test ID;
- human-readable name;
- test-pack ID and version;
- status;
- duration;
- deterministic assertion details;
- failure reason;
- evidence references.

### 6. Baseline comparison

When a compatible baseline exists:

- baseline run identity;
- current and baseline `BaselineCompatibilityKey` values/hashes;
- compatibility decision and optional policy ID;
- new failures;
- resolved failures;
- unchanged failures;
- metric baseline/current/delta/threshold/result;
- compatibility notes.

When comparison is unavailable, the report states why. It must not silently
compare incompatible hardware, test versions, or units.

### 7. Evidence index

List evidence by:

- evidence ID;
- type;
- description;
- file or object reference;
- SHA-256;
- size;
- visibility;
- redaction status;
- collection timestamp.
- finalization state.

### 8. Scope and limitations

- included and excluded tests;
- skipped tests and reasons;
- known issues supplied by customer;
- unavailable instruments or capabilities;
- environmental assumptions;
- retention date.

### 9. Assisted analysis

Optional AI-assisted summaries are visually and structurally marked:

> Assisted analysis — not used to determine PASS or FAIL.

The report records provider/model metadata only according to privacy policy. It
must not expose private prompts, credentials, or unrestricted raw customer data.

## Canonical JSON shape

```json
{
  "schema_version": 1,
  "report_id": "report_01JEXAMPLE",
  "run_id": "run_01JEXAMPLE",
  "generated_at": "2026-10-01T10:40:00Z",
  "statement": {
    "text": "Release demo-1.0.1 was validated against linux-core 0.1.0 on rpi4-reference-v1.",
    "scope_claim": "validated_against_declared_test_scope",
    "result": "FAIL"
  },
  "release": {
    "release_id": "rel_demo_101",
    "name": "demo",
    "version": "1.0.1",
    "provenance_type": "internal_demo",
    "source_revision": "32317d2a5b215ae2f49b5a5136ba37e9a552a3c1",
    "artifacts": [
      {
        "artifact_id": "art_demo_wic",
        "role": "disk_image",
        "name": "picopiece-demo-rpi4.wic.bz2",
        "size_bytes": 734003200,
        "sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
      }
    ]
  },
  "hardware": {
    "hardware_config_id": "hw_rpi4_reference_v1",
    "station_alias": "reference-pi-01",
    "board": "Raspberry Pi 4 Model B",
    "hardware_revision": "1.5",
    "boot_media": "microSD via SD multiplexer",
    "fixture_revision": "fixture-v1",
    "platform_profile_id": "raspberrypi4-sd-v1"
  },
  "execution": {
    "started_at": "2026-10-01T10:30:00Z",
    "completed_at": "2026-10-01T10:39:12Z",
    "duration_seconds": 552,
    "status": "completed",
    "result": "FAIL",
    "counts": {
      "total": 20,
      "passed": 18,
      "failed": 2,
      "skipped": 0,
      "errors": 0
    },
    "parent_run_id": null,
    "recovery_actions": []
  },
  "test_scope": {
    "packs": [
      {
        "pack_id": "linux-core",
        "version": "0.1.0"
      }
    ],
    "included_test_ids": [
      "linux.boot.console",
      "linux.boot.ssh_ready",
      "linux.service.demo_health"
    ],
    "excluded_test_ids": []
  },
  "tests": [
    {
      "test_id": "linux.boot.ssh_ready",
      "name": "SSH readiness time",
      "pack_id": "linux-core",
      "pack_version": "0.1.0",
      "status": "FAIL",
      "duration_seconds": 19.2,
      "failure": "Boot time increased 14.3%, exceeding the allowed 10%.",
      "evidence_ids": ["ev_serial", "ev_boot_metric"]
    },
    {
      "test_id": "linux.service.demo_health",
      "name": "Demo service health",
      "pack_id": "linux-core",
      "pack_version": "0.1.0",
      "status": "FAIL",
      "duration_seconds": 0.42,
      "failure": "Expected systemd state active; observed failed.",
      "evidence_ids": ["ev_service_status", "ev_journal"]
    }
  ],
  "comparison": {
    "baseline_run_id": "run_demo_100_baseline",
    "compatible": true,
    "compatibility": {
      "current_key_hash": "sha256:current-key",
      "baseline_key_hash": "sha256:current-key",
      "policy_id": "strict-equality-v1",
      "reason": "Compatibility keys are identical."
    },
    "new_failures": [
      "linux.boot.ssh_ready",
      "linux.service.demo_health"
    ],
    "resolved_failures": [],
    "metrics": [
      {
        "metric_id": "boot.ssh_ready_seconds",
        "unit": "seconds",
        "baseline": 16.8,
        "current": 19.2,
        "delta": 2.4,
        "delta_percent": 14.3,
        "threshold": {
          "operator": "max_increase_percent",
          "value": 10
        },
        "result": "FAIL"
      }
    ]
  },
  "evidence": [
    {
      "evidence_id": "ev_serial",
      "type": "serial_log",
      "description": "Timestamped boot console",
      "uri": "evidence/serial.log",
      "sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "size_bytes": 12480,
      "visibility": "customer_private",
      "redaction_status": "completed",
      "collected_at": "2026-10-01T10:31:00Z",
      "finalized": true
    }
  ],
  "scope_and_limitations": {
    "known_issues": [],
    "skipped": [],
    "limitations": [
      "No power measurement was included.",
      "Result applies only to the declared hardware and suite versions."
    ],
    "retention_until": "2027-01-01T00:00:00Z"
  },
  "assisted_analysis": {
    "enabled": false,
    "authoritative": false,
    "summary": null
  }
}
```

## Result consistency rules

Allowed final status values are `completed` and `cancelled`.

Allowed final result values are `PASS`, `FAIL`, `ERROR`, and `INCOMPLETE`.

1. Required test `FAIL` produces `completed` / `FAIL`.
2. Infrastructure failure that prevents a valid assertion produces
   `completed` / `ERROR`.
3. Customer/operator/scheduler cancellation produces
   `cancelled` / `INCOMPLETE`.
4. Station loss produces `completed` / `ERROR` after the control plane
   finalizes the run.
5. A declared test-owned timeout may produce `completed` / `FAIL`; an
   orchestration timeout produces `completed` / `ERROR`.
6. Required test `SKIP` follows the resolved skip policy.
7. Counts must match test records.
8. Baseline comparison is forbidden unless compatibility keys match or an
   explicit versioned compatibility policy allows it.
9. Baseline metric result must match its declared operator and threshold.
10. Report result must match canonical run result and low-level result outputs.
11. Assisted analysis cannot modify deterministic result fields.

## Evidence directory

Recommended immutable layout:

```text
runs/<run_id>/
├── run-snapshot.yaml
├── validation-report.json
├── validation-report.html
├── evidence-index.json
├── junit.xml
├── ats-summary.json
├── meta.yaml
└── evidence/
    ├── serial.log
    ├── journal/
    ├── commands/
    ├── metrics/
    └── images/
```

## Evidence finalization

`evidence-index.json` contains:

```yaml
schema_version: 1
run_id: run_01JEXAMPLE
finalized: true
finalized_at: "2026-10-01T10:40:00Z"
objects:
  - path: evidence/serial.log
    size_bytes: 12480
    sha256: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

Before finalization, the collector may append run-scoped evidence. Finalization
calculates the index and object hashes, sets `finalized: true`, and makes the
run tree non-writable through both application and filesystem/storage policy.

Any post-finalization correction creates a new version/object and audit event;
it never silently overwrites evidence referenced by an existing report.

The MVP may use local filesystem permissions and application guards. A later
S3-compatible backend may use content-addressed object keys, bucket versioning,
and Object Lock.

## Public demo redaction

Public reports use a generated sanitized view:

- remove customer organization and project identifiers;
- remove private storage URIs;
- remove credentials, hostnames, MAC/IP addresses when policy requires;
- remove customer source repository URLs;
- include only evidence marked `public_demo`;
- preserve checksums only when they do not expose confidential artifact identity.

## Validation checklist

- report references one immutable run;
- exact artifacts and test-pack versions are present;
- hardware configuration is unambiguous;
- result counts are consistent;
- every failure has deterministic reason and evidence;
- baseline compatibility is explicit;
- limitations are visible;
- AI text is clearly non-authoritative;
- no certification claim appears.
