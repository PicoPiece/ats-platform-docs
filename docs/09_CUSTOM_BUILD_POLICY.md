# Managed Yocto and Custom Build Policy

## Decision

Artifact validation is the default customer workflow.

Picopiece may use the existing `yocto_multi_platform` system to build internal
Raspberry Pi reference images for M1. This does not mean managed customer builds
are generally available.

Customer managed Yocto/custom BSP work begins only when:

- a qualified prospect requires it;
- paid discovery or an approved design-partner scope exists;
- source, licensing, secrets, build time, storage, and support boundaries are
  understood;
- artifact validation flow is already stable.

## Service modes

### Mode A — Artifact validation

Customer supplies immutable artifacts and metadata.

Picopiece:

- verifies size/type/SHA-256;
- inspects archives under bounded expected-member policy;
- registers release and artifacts;
- provisions declared hardware;
- executes the DUT under an explicit default-deny network profile;
- runs agreed tests;
- returns evidence and report.

This is the preferred evaluation and first-pilot mode.

### Mode B — Internal reference build

Picopiece builds its own demo images with `yocto_multi_platform`.

Purpose:

- create known-good and regression Pi 4 images;
- demonstrate provenance and reproducibility;
- develop build-to-validation integration;
- avoid customer source/secrets during M1.

This mode is internal product development, not a customer deliverable.

### Mode C — Managed customer build

Picopiece builds customer software and then validates the produced release.

This mode adds:

- source access;
- private layers;
- license obligations;
- secret handling;
- reproducibility requirements;
- cache isolation;
- build failure support;
- larger compute/storage cost.

It requires a separate scope and price.

### Mode D — Custom BSP engineering

Porting layers, fixing recipes, kernel/device-tree work, or creating a BSP is
engineering consulting. It is not included automatically in managed validation.

Custom BSP work must have:

- explicit deliverables;
- ownership and licensing terms;
- separate acceptance criteria;
- separate estimate;
- clear transition into a repeatable validation pack.

## Existing build-system assets

The current `yocto_multi_platform` system provides:

- Docker-based Ubuntu builder;
- `setup_env.sh` and `build.sh` entrypoints;
- shared downloads and sstate cache;
- Raspberry Pi 4, Pi Zero W, S905x3, and RK3588 configurations;
- `.wic` outputs for Raspberry Pi;
- a SHA-pinned RK3588 `navonz_v1` pattern;
- build logs and per-platform build directories.

Known gaps before managed customer use:

- no ATS Release/Artifact manifest emitter;
- no Jenkins pipeline in this repository;
- Pi default manifest pins `kirkstone` branches rather than exact SHAs;
- current Pi `core-image-minimal` does not guarantee systemd/OpenSSH/demo
  services needed by M1;
- `debug-tweaks` is enabled in development configuration;
- some setup is interactive;
- persistent build directories are not customer-isolated ephemeral workers;
- SBOM/provenance export is not standardized.

## M1 internal reference build

Allowed M1 work:

- add internal Pi 4 good/regression image profiles;
- add an ATS reference layer for systemd, SSH, health service, and release
  metadata;
- pin all layer revisions for the demo;
- capture WIC, bmap, rootfs package manifest, build log, and checksums;
- manually register artifacts until M2 integration exists.

Not M1:

- arbitrary customer repositories;
- private customer layers;
- multiple customer BSPs;
- production SLA;
- automated billing or customer portal.

## M2 integration

After the Pi validation path is stable:

- Jenkins build agent invokes non-interactive Yocto entrypoints;
- build produces normalized artifact names and SHA-256;
- build emits Release/Artifact metadata;
- Jenkins archives or uploads artifacts to registry storage;
- external uploads and managed outputs enter the same registry;
- validation consumes artifacts by role, not build-directory path;
- build failure is reported separately from validation failure.

## M3 managed build entry criteria

All are required:

- customer passed technical onboarding;
- pinned source/layer manifest exists;
- reference build succeeds twice from a clean TMPDIR;
- expected artifacts and checksums are defined;
- license requirements reviewed;
- secret injection method approved;
- compute/storage estimate approved;
- support boundary and build-failure ownership documented;
- output can enter the normal Release Registry;
- validation target and recovery path are available.

## Reproducibility requirements

Managed builds must record:

- source repository revisions;
- repo/layer manifest;
- Yocto release;
- `MACHINE`, `DISTRO`, and image target;
- relevant configuration hashes;
- builder image digest;
- build start/end time;
- build host class, not sensitive hostname;
- output artifact checksums;
- package manifest;
- build log;
- SBOM when required.

Branch-only layer references are insufficient for a claimed reproducible build.

## Builder isolation

Recommended model:

- immutable builder container image;
- per-build or per-customer TMPDIR;
- shared download mirror/cache with controlled permissions;
- sstate sharing only under an explicit trust policy;
- no customer code executed on Jenkins master;
- resource quotas and timeouts;
- network egress policy;
- workspace cleanup after retention boundary.

Shared caches improve performance but are not evidence of build reproducibility.

## Secret policy

Secrets:

- live in approved secret storage;
- are injected at job runtime;
- are scoped per organization/project;
- are redacted from logs;
- are never copied into release manifests;
- are revoked when engagement ends.

If a source fetch requires long-lived personal credentials, onboarding is
blocked until a service credential or safer mechanism exists.

## Cache policy

Downloads:

- may be shared when licensing permits;
- use integrity verification;
- are not customer evidence by default.

Sstate:

- may be shared for internal/reference builds;
- customer sharing requires policy review;
- cache hit/miss must not change declared artifact identity;
- clean rebuild is required for reproducibility qualification.

## SBOM and licensing

When required, produce:

- SPDX or agreed SBOM format;
- package/rootfs manifest;
- license manifest;
- source offer/material according to applicable licenses.

Picopiece does not assume responsibility for customer product licensing without
an explicit contracted review.

## Commercial boundary

Managed build pricing accounts for:

- initial discovery;
- build environment setup;
- recurring compute and storage;
- cache maintenance;
- source credential management;
- build failure investigation;
- Yocto upgrade/layer drift;
- SBOM/provenance retention.

A request that does not create reusable build capability, reusable validation
IP, or recurring revenue should normally be declined or priced as consulting.

## Stop conditions

Stop or rescope managed build when:

- source cannot be pinned;
- licensing prevents required handling;
- secrets cannot be managed safely;
- build requires unsupported proprietary tools;
- clean builds are not reproducible;
- customer expects BSP development inside validation pricing;
- hardware recovery is unavailable;
- required data residency cannot be met.

## Roadmap rule

Internal Yocto reference images are an M1 enabler.

General customer managed build is an M3 option, activated by customer demand.
It must not delay the customer-facing artifact validation pipeline.
