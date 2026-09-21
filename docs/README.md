# Picopiece Validation Lab Planning Pack

Official project start: **2026-10-01**

## Current operating context

- The existing ESP32 hardware-in-the-loop flow is the working POC.
- The physical station is temporarily offline during relocation.
- Offline-safe documentation, contracts, report/dashboard fixtures, and backlog
  work can continue.
- After reactivation, ESP32 is stabilized first and time-boxed.
- Raspberry Pi 4 is the flagship Embedded Linux validation demo.
- The existing `yocto_multi_platform` system is an available
  internal build asset; general managed customer builds remain deferred until
  customer demand.

## Product and business documents

1. [`01_CONCEPT_AND_SYSTEM_OVERVIEW.md`](01_CONCEPT_AND_SYSTEM_OVERVIEW.md) —
   mission, architecture, test taxonomy, and product boundaries.
2. [`02_PRODUCT_TECHNICAL_ROADMAP.md`](02_PRODUCT_TECHNICAL_ROADMAP.md) —
   product, technical, sales, AI, and expansion roadmap.
3. [`03_SOLO_FOUNDER_BUSINESS_PLAN.md`](03_SOLO_FOUNDER_BUSINESS_PLAN.md) —
   ideal customer, commercial model, sales process, and founder priorities.
4. [`04_LIVE_VALIDATION_LAB_DASHBOARD_SPEC.md`](04_LIVE_VALIDATION_LAB_DASHBOARD_SPEC.md) —
   public and future private dashboard behavior.

## Delivery contracts and execution guides

5. [`05_VALIDATION_REPORT_SCHEMA_V1.md`](05_VALIDATION_REPORT_SCHEMA_V1.md) —
   customer-facing report contract and canonical JSON example.
6. [`06_CUSTOMER_ONBOARDING_CONTRACT.md`](06_CUSTOMER_ONBOARDING_CONTRACT.md) —
   required customer, artifact, hardware, recovery, test, and security inputs.
7. [`07_REFERENCE_STATION_BOM.md`](07_REFERENCE_STATION_BOM.md) —
   Raspberry Pi 4 reference station hardware and provisioning safety.
8. [`08_REFERENCE_DEMO_RUNBOOK.md`](08_REFERENCE_DEMO_RUNBOOK.md) —
   ESP32 reactivation and Pi 4 known-good/regression demonstrations.
9. [`09_CUSTOM_BUILD_POLICY.md`](09_CUSTOM_BUILD_POLICY.md) —
   artifact-first policy and boundaries for internal/customer Yocto builds.
10. [`10_IMPLEMENTATION_BACKLOG_M0_M2.md`](10_IMPLEMENTATION_BACKLOG_M0_M2.md) —
    issue-ready M0–M2 backlog and milestone exit criteria.

## Platform architecture contracts

Located under [`../architecture/`](../architecture/):

- `ats-manifest-spec-v1.md` — current ESP32 input contract;
- `test-output-contract-v1.md` — current low-level result contract;
- `validation-domain-model-v1.md` — Release, Artifact, ValidationRun, Station,
  StationLease, and baseline model;
- `station-runtime-interfaces-v1.md` — platform/runtime adapter boundaries;
- `release-manifest-spec-v2-draft.md` — multi-artifact release draft;
- `station-lease-contract-v1.md` — exclusive station ownership and fencing;
- `artifact-intake-security-policy-v1.md` — untrusted upload/archive controls;
- `dut-network-security-policy-v1.md` — default-deny customer DUT network.

## Near-term milestone order

```text
Offline work
  -> contracts, sample report, onboarding, BOM, backlog, dashboard replay

M0
  -> ESP32 through new abstractions
  -> station reactivation and 10-run soak

M1
  -> Pi 4 internal image
  -> provision, boot, SSH, 20 Linux tests
  -> baseline regression, evidence, report, dashboard MVP

M2
  -> external artifact intake
  -> Release Registry, StationLease, scheduler
  -> same Pi validation and report pipeline

M3
  -> first real customer image/DUT and repeatable pilot
  -> managed customer Yocto build only when justified
```

## Language policy

Use:

- “validated against the declared test scope”;
- “regression evidence generated for release X”.

Do not claim:

- certification;
- bug-free software;
- verified software quality outside the declared scope.
