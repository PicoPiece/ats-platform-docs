# Release Manifest Specification v2 — Draft

## Status

Draft for M0 design review. It does not replace ATS Manifest v1 and must not be
consumed by production runners until an implementation and migration plan exist.

## Purpose

Manifest v1 describes one firmware artifact produced by one build. Validation
Lab requires a release to contain multiple immutable artifacts and to be tested
multiple times on different hardware configurations.

Version 2 describes:

- release identity and provenance;
- one or more artifacts;
- target and provisioning requirements;
- declared validation scope;
- optional comparison baseline.

Station identity, lease details, resolved secrets, and final run results do not
belong in the release manifest.

## Draft schema

```yaml
manifest_version: 2

release:
  release_id: rel_01JEXAMPLE
  organization_id: org_demo
  project_id: project_gateway
  name: gateway-demo
  version: 1.4.7
  provenance_type: uploaded
  source:
    repository: https://example.invalid/customer/gateway.git
    revision: 32317d2a5b215ae2f49b5a5136ba37e9a552a3c1
  created_at: "2026-10-01T10:30:00Z"

artifacts:
  - artifact_id: art_rootfs
    name: core-image-ats-raspberrypi4.wic.bz2
    role: disk_image
    media_type: application/x-bzip2
    size_bytes: 734003200
    sha256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
    compression: bzip2
    target:
      architecture: aarch64
      machine: raspberrypi4
  - artifact_id: art_bmap
    name: core-image-ats-raspberrypi4.wic.bmap
    role: bmap
    media_type: application/xml
    size_bytes: 8192
    sha256: abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789
  - artifact_id: art_packages
    name: core-image-ats-raspberrypi4.rootfs.manifest
    role: metadata
    media_type: text/plain
    size_bytes: 16384
    sha256: 1111111111111111111111111111111111111111111111111111111111111111

target:
  platform_profile_id: raspberrypi4-sd-v1
  board: raspberry-pi-4-model-b
  hardware_revision_constraints:
    allowed: ["1.4", "1.5"]
  provisioning:
    strategy: sdwire_wic
    artifact_roles:
      image: disk_image
      map: bmap
  runtime:
    required_transports: [ssh]
    readiness_probe: linux.ssh_ready
    network_access_profile_id: dut-offline-v1

validation:
  test_packs:
    - pack_id: linux-core
      version: 0.1.0
  tests:
    include:
      - linux.boot.console
      - linux.boot.ssh_ready
      - linux.service.demo_health
      - linux.network.ethernet
    exclude: []
  thresholds:
    linux.boot.ssh_ready_seconds:
      max: 30
    comparison.boot.ssh_ready_increase_percent:
      max: 10
  baseline:
    run_id: run_01JBASELINE
    compatibility_key:
      platform_profile: rpi4-sd-v1
      hardware_revision: "1.5"
      test_packs:
        - id: linux-core
          version: 0.1.0
      metric_schema: 1
      fixture_revision: fixture-v1
      calibration_profile: none
  policies:
    required_test_failure: fail
    infrastructure_error: error
    skipped_required_test: fail
```

## Required fields

Root:

- `manifest_version`: integer `2`;
- `release`;
- non-empty `artifacts`;
- `target`;
- `validation`.

Release:

- immutable `release_id`;
- organization and project IDs;
- name and version;
- provenance type;
- creation timestamp.

Artifact:

- unique `artifact_id` within the manifest;
- filename without path traversal;
- role and media type;
- positive size;
- lowercase SHA-256 with 64 hexadecimal characters.
- archive intake policy ID when the artifact is an archive.

Target:

- registered platform profile;
- board identifier;
- provisioning strategy;
- mapping from provisioning input names to artifact roles.
- network access profile for DUT execution.

Validation:

- at least one version-pinned test pack;
- declared selection rules;
- resolved or defaultable threshold keys.

## Multi-artifact rules

1. Artifact order has no semantic meaning.
2. Provisioning selects artifacts by role, not filename wildcard.
3. A role may appear more than once only when the platform profile permits it.
4. Every artifact used by a run is copied into the immutable run snapshot.
5. Unreferenced optional artifacts may still be retained as release evidence.

## Provenance

`provenance_type` values:

- `uploaded`: customer or customer CI supplied artifacts;
- `managed_build`: Picopiece produced artifacts from declared source;
- `internal_demo`: Picopiece reference release.

Managed build provenance may include:

```yaml
build:
  build_id: build_01JEXAMPLE
  builder_image_digest: sha256:...
  source_manifest_artifact_id: art_repo_manifest
  layer_revisions_artifact_id: art_layer_revisions
  build_log_artifact_id: art_build_log
  sbom_artifact_id: art_sbom
```

Secrets, temporary signed URLs, Jenkins credentials IDs, and host filesystem
paths are forbidden.

## ValidationRun snapshot

Before acquiring a station, the platform creates a run snapshot containing:

- this manifest checksum;
- exact artifact IDs and storage generations;
- resolved platform profile version;
- resolved test-pack versions;
- resolved thresholds;
- selected baseline run;
- resolved `BaselineCompatibilityKey`;
- hardware compatibility constraints;
- resolved DUT network access profile and firewall policy revision.

Later edits to a draft release cannot affect an existing run.

## Relationship to Manifest v1

Manifest v1 remains supported by the ESP32 POC during M0.

Suggested conversion:

- v1 `build` becomes release provenance metadata;
- v1 single `build.artifact` becomes one v2 artifact;
- v1 `device` maps to v2 `target`;
- v1 `test_plan` maps to v2 validation test selection;
- v1 build timestamp maps to release provenance, not run execution time.

Runners must reject unsupported versions explicitly. They must not guess a
schema version from fields.

## Security validation

Before a release becomes `ready`:

- verify declared file size and SHA-256;
- reject path traversal and unsupported media types;
- keep new uploads in quarantine;
- apply configured malware/content inspection where applicable;
- never execute scripts contained in an uploaded image on the control plane;
- enforce storage retention and tenant namespace;
- record the validation decision in audit history.

Archive artifacts must satisfy
[Artifact Intake Security Policy v1](./artifact-intake-security-policy-v1.md),
including extraction limits, path/link validation, expected-member policy, and
decompression-bomb protection.

Every external image executes under a resolved
[DUT Network Security Policy](./dut-network-security-policy-v1.md) profile.
Default policy denies DUT access to the control plane, home/office LAN, other
DUTs, and the Internet.

Customer cryptographic artifact signatures are optional for MVP. A pre-signed
upload URL authorizes transport; it does not prove artifact publisher identity.

## Open decisions before stabilization

- canonical JSON/YAML serialization for manifest signatures;
- whether one artifact may belong to multiple releases;
- final role vocabulary;
- SBOM format requirements;
- compatibility rules for baseline comparisons;
- migration tooling from v1.
