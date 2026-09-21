# Customer Onboarding Contract

## Purpose

This document defines the technical information and responsibilities required
before Picopiece accepts a customer release for evaluation or pilot validation.
It is an engineering intake contract, not a replacement for an NDA, MSA, SOW,
privacy agreement, or legal advice.

## Engagement identity

Customer provides:

- legal organization name;
- project/product name;
- primary technical contact;
- escalation contact;
- timezone and working hours;
- billing contact when engagement is paid;
- NDA requirement and applicable agreement reference;
- requested engagement: evaluation, pilot, or recurring validation.

Picopiece assigns:

- organization ID;
- project ID;
- engagement ID;
- data classification;
- default retention policy;
- approved communication channel.

## Workflow mode

### Artifact mode — default

Customer supplies one or more immutable release artifacts.

Required:

- artifact filename and role;
- format: `.wic`, `.wic.bz2`, `.img`, `.bin`, `.tar`, OTA package, or agreed type;
- expected size;
- SHA-256;
- target architecture and machine;
- release name/version;
- source Git SHA or customer build ID when available;
- required relationship between multiple artifacts.

MVP transport:

- pre-signed upload URL;
- size/type validation;
- SHA-256 verification;
- quarantine before registration;
- explicit retention.

A customer cryptographic artifact signature is optional unless the engagement
requires it.

Archive artifacts such as `.tar` require an approved archive profile defining
expected members, extracted-size/file-count limits, path/link rules, and nested
archive behavior. Archives are inspected in quarantine and never executed.
See `../architecture/artifact-intake-security-policy-v1.md`.

### Managed build mode — conditional

Managed build is not the default first evaluation. It requires paid discovery
or explicit approval.

Customer additionally supplies:

- source repository or repo manifest;
- pinned revisions for all required layers;
- Yocto release and distribution;
- `MACHINE`, image target, and build command;
- custom layers and BSP ownership;
- license acceptance requirements;
- private dependency access method;
- expected output artifacts;
- known build duration and storage requirements;
- reference successful build log;
- expected SBOM/provenance outputs.

Secrets are provided through approved secret storage, never in manifests,
tickets, chat logs, or release metadata.

## Target hardware

Required:

- board manufacturer and exact model/SKU;
- hardware revision;
- CPU architecture and SoC;
- boot media type and capacity;
- bootloader and expected boot flow;
- serial console voltage, connector, pinout, and baud rate;
- power input specification;
- reset/recovery controls;
- network interfaces required during validation;
- required peripherals and fixture wiring;
- unique device restrictions or licenses;
- whether Picopiece or customer supplies the DUT;
- number of DUTs and spare availability.

Customer-supplied hardware must include:

- safe power supply;
- required cables/adapters;
- recovery media and procedure;
- shipping inventory;
- replacement value;
- known physical defects.

## Provisioning and recovery

Customer declares:

- normal provisioning method;
- expected artifact-to-media mapping;
- destructive erase constraints;
- known-good recovery artifact;
- recovery mode entry procedure;
- maximum acceptable recovery attempts;
- operations that are prohibited;
- whether device identity/calibration partitions must be preserved;
- expected result after power interruption.

Picopiece confirms:

- supported `PlatformProfile`;
- `Provisioner`;
- power/reset capability;
- console and runtime transports;
- station recovery strategy;
- media allow-list and write safety controls.

No run starts until recovery has been demonstrated or explicitly accepted as a
known limitation.

## Runtime access

Customer provides only what the declared test scope requires:

- SSH user and authentication method;
- host-key/bootstrap policy;
- API endpoint and authentication reference;
- MQTT broker/topic policy;
- static/DHCP network expectations;
- DNS/NTP/Internet requirements;
- outbound destinations that must be allowed;
- inbound connectivity requirements;
- certificate and clock requirements.

Every engagement selects a `NetworkAccessProfile`. The default denies DUT
access to the control plane, home/office LAN, other DUTs, and the Internet.
Required Internet/DNS/NTP access must be explicit, destination-restricted,
logged, and approved. See
`../architecture/dut-network-security-policy-v1.md`.

Credentials:

- are stored server-side in approved secret storage;
- are referenced by secret ID;
- are never copied into artifacts, reports, or public logs;
- have minimum required privileges;
- are rotated or revoked at engagement end.

## Validation scope

Customer and Picopiece agree:

- test-pack IDs and exact versions;
- required and optional test IDs;
- L0 Linux, L1 BSP, L2 product, and L3 customer-specific boundaries;
- deterministic acceptance thresholds;
- timeout values;
- allowed retry count;
- SKIP policy;
- baseline release/run;
- supported hardware revisions;
- environmental assumptions;
- excluded capabilities.

Each test must define:

- stable ID;
- preconditions;
- action;
- assertion;
- threshold and unit where applicable;
- evidence produced;
- cleanup behavior;
- failure ownership when known.

## Known issues

Customer lists:

- accepted defects;
- flaky services or peripherals;
- expected warning/error log signatures;
- unsupported features;
- temporary workarounds;
- issue references and expiry/review date.

Known issues do not automatically become PASS. The agreed test scope defines
whether they are excluded, expected failures, or reported limitations.

## Baseline and comparison

Customer chooses:

- no comparison;
- compare with latest eligible PASS run;
- compare with a pinned run ID;
- compare with a named release.

Comparison requires compatible:

- hardware configuration;
- test semantics/version;
- metric units;
- station calibration where relevant.

Compatibility is enforced with a machine-readable
`BaselineCompatibilityKey`, including platform profile, hardware revision,
test-pack versions, metric schema, fixture revision, and calibration profile.
No automatic comparison occurs when keys differ unless an explicit versioned
compatibility policy allows it.

Threshold examples:

- SSH readiness increase no more than 10%;
- idle memory increase no more than 15%;
- required systemd services must remain active;
- packet loss no more than 1%.

## Evidence and reporting

Agree before the first run:

- HTML/JSON/PDF output requirements;
- raw serial/journal/command evidence;
- screenshots or webcam use;
- public/private evidence classification;
- fields requiring redaction;
- retention period;
- report recipients;
- review meeting requirement;
- customer download method.

Approved result language:

> Validated against the declared test scope.

Disallowed unless separately contracted and qualified:

- certified;
- bug-free;
- verified software quality;
- compliant with a regulated standard.

## Data and security

Customer declares:

- confidentiality classification;
- export-control or data-residency constraints;
- personal data presence;
- credentials or device identity embedded in images;
- whether Internet access is prohibited;
- artifact deletion deadline.

Picopiece declares:

- storage location and tenant namespace;
- access control;
- quarantine behavior;
- log sanitization;
- backup scope;
- retention and deletion process;
- public-demo exclusion.

Customer artifacts and hardware never appear on the public dashboard.

## Responsibility boundary

Customer is responsible for:

- legal right to provide software and hardware;
- accurate target and recovery information;
- artifact integrity checksum;
- licensing and third-party terms;
- acceptance thresholds and product expectations;
- timely replacement of failed customer hardware.

Picopiece is responsible for:

- executing the agreed scope on declared hardware;
- preserving run traceability;
- separating infrastructure errors from validation failures;
- collecting and retaining agreed evidence;
- protecting credentials and private artifacts according to policy;
- reporting limitations and deviations.

## Evaluation acceptance checklist

- engagement and contacts recorded;
- workflow mode approved;
- board model and HW revision confirmed;
- artifacts and checksums received;
- provisioning method reviewed;
- console/runtime access tested or documented;
- recovery method demonstrated;
- test scope and thresholds signed off;
- baseline selected;
- known issues recorded;
- evidence/retention/redaction agreed;
- success and stop conditions agreed.

## Evaluation exit criteria

An evaluation is complete when:

1. one declared release is registered;
2. one eligible DUT is configured;
3. the agreed suite executes or blocked tests are reported explicitly;
4. evidence and report are delivered;
5. result limitations are reviewed with the customer;
6. both parties decide whether to stop, extend, or convert to a paid pilot.
