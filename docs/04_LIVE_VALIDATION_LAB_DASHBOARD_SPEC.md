# Picopiece Live Validation Lab — Dashboard Specification

## Purpose
The dashboard is a live technical proof that the validation lab is real.

It must help a technical buyer understand the service within five minutes without a sales presentation.

The dashboard is a view over immutable `Release`, `Artifact`, and
`ValidationRun` data. It is not the system of record and it does not determine
PASS or FAIL.

## Delivery sequence

### Offline / contract stage

While the physical station is offline:

- build the page structure and data contract using sanitized archived ESP32
  runs;
- label replay data clearly as **HISTORICAL DEMO DATA**;
- do not display fabricated live status, uptime, or SLA;
- implement report download against fixture data;
- defer webcam streaming and live event integration.

### ESP32 station reactivation

After the station passes its reactivation soak:

- connect station health and current run;
- stream sanitized execution events/logs;
- display deterministic PASS/FAIL and finalized report;
- retain replay mode as a clearly labeled fallback.

### Raspberry Pi 4 M1

Add:

- provisioning state;
- UART and SSH readiness;
- Linux Core test progress;
- known-good baseline comparison;
- boot-time and service-health regressions.

### Customer pilot

Only after public reference flow is stable, add authentication, tenant
isolation, private artifact upload, API keys, and customer-specific views.

## Public demo dashboard

### Header
- Picopiece Validation Lab
- station status: ONLINE / RUNNING / IDLE / OFFLINE
- public demo disclaimer
- CTA: **Run a release through the lab**

### Panel 1 — Live hardware
- webcam feed
- DUT name
- board model
- hardware revision
- station ID
- power state

Public camera view should show only:
- rack/station
- DUT
- indicators / fixture

Never show:
- home interior
- personal information
- customer hardware under NDA
- screens containing secrets

### Panel 2 — Current release
- release name
- immutable release ID
- image filename
- artifact ID and role
- git SHA
- build ID
- image checksum
- build time
- test suite version

### Panel 3 — Execution
- validation run ID
- station lease state
- current test
- progress bar
- elapsed time
- total
- passed
- failed
- skipped
- retry count

### Panel 4 — Live log
Read-only, sanitized stream:
- serial console
- selected Jenkins/test events
- timestamps

Apply:
- secret redaction
- token redaction
- network credential redaction
- customer-data isolation

### Panel 5 — Evidence
Depending on pack:
- CPU / memory
- boot time
- network
- temperature
- power
- BLE
- cellular
- CAN signals

### Panel 6 — Release comparison
Compare against previous validated version:
- baseline run ID
- new failures
- resolved failures
- boot delta
- power delta
- performance delta
- threshold and deterministic comparison result

### Panel 7 — Final result
Show:
- completed timestamp
- test counts
- list of failures
- deterministic reason
- AI-assisted summary clearly marked as analysis
- validation scope
- report download

Example:

```text
Release: demo-1.4.7
Hardware: Raspberry Pi CM4
Suite: linux-core v0.3.0

42 tests
40 PASS
2 FAIL

New regressions:
- NET-014 Ethernet reconnect
- BOOT-007 boot time threshold

Validation statement:
This release was validated against the declared suite and hardware configuration.
```

## Customer private dashboard
Later add:
- authentication
- organization isolation
- project selection
- private DUT webcam (optional)
- private artifact upload
- API key
- release history
- report history
- test-suite selection
- thresholds
- customer-specific tests
- comments
- webhooks
- CI integration

## Backend APIs

Possible endpoints:

```text
POST /api/releases
POST /api/releases/{id}/run
GET  /api/runs/{id}
GET  /api/runs/{id}/events
GET  /api/runs/{id}/report
GET  /api/stations
GET  /api/stations/{id}
```

Recommended event model:
- Server-Sent Events or WebSocket for live execution
- immutable run metadata
- append-only evidence/events where practical

Minimum event fields:

```text
event_id
run_id
station_id
sequence
timestamp
event_type
visibility
payload
```

The UI must order by run-local sequence and tolerate reconnect/replay without
duplicating events.

## Data sources

MVP sources:

- Release/Artifact/ValidationRun snapshot;
- station and lease status;
- append-only run events;
- finalized `validation-report.json`;
- evidence index;
- existing JUnit/summary/metadata during migration.

The browser must not read Jenkins workspaces or private artifact storage
directly. Backend endpoints apply visibility, tenant, and redaction policy.

## Security
Public and private data must be separated from the beginning.

Minimum:
- no public access to customer artifacts
- secrets only in server-side secret storage
- pre-signed, time-limited artifact/report access where appropriate
- log sanitizer
- per-customer directory / namespace
- audit logs
- explicit retention rules

## Reliability indicators
Consider showing public reference lab uptime later:
- station uptime
- successful run percentage
- latest validation timestamp

Do not publish fabricated SLA metrics.

## Design principle
The dashboard should communicate:

> “This is not a slide deck. The hardware exists and the validation is running now.”

But the final output remains the report and evidence, not the webcam itself.

The public dashboard must use only the dedicated reference station. Customer
hardware, releases, logs, and run existence remain private unless explicit
written publication approval exists.
