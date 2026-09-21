# Picopiece Validation Lab — Product & Technical Roadmap

**Official Day 1: 2026-10-01**

The first objective is not maximum revenue. The first objective is to prove that an international customer is willing to pay for repeatable real-hardware release validation.

## North-star milestone
A customer outside Vietnam:
1. gives Picopiece a real software image or build,
2. Picopiece runs it on real hardware,
3. Picopiece produces useful evidence,
4. the customer trusts the result,
5. the customer pays to repeat the service.

## Near-term technical milestones

These milestones are execution gates inside the calendar phases below.

### M0 — Preserve and abstract the ESP32 POC

- complete contracts and offline-safe implementation planning;
- run the current ESP32 flow through narrow station interfaces;
- remove runtime test-script mutation and executor hardcoded pass strings;
- reactivate the station after relocation;
- complete 10 consecutive expected PASS/FAIL runs;
- archive a sanitized golden evidence fixture.

### M1 — Raspberry Pi 4 internal reference validation

- build pinned internal good/regression images using the existing
  `yocto_multi_platform` system;
- provision Pi 4 boot media safely;
- capture UART and wait for SSH;
- execute 20 versioned Linux Core tests;
- compare against a known-good baseline;
- generate JSON/HTML validation report;
- connect one-station public dashboard MVP to real sanitized data.

### M2 — External artifact intake

- pre-signed upload URL;
- size/type/SHA-256 validation;
- quarantine and retention;
- first-class Release, Artifact, ValidationRun, Station, and StationLease;
- capability-aware scheduling;
- external Pi-compatible artifact uses the same provision/test/report path.

### M3 — First customer pilot

- real customer image and real DUT;
- repeatable runs and review meeting;
- convert reusable customer requirements into test packs/adapters;
- managed customer Yocto build only when required by a qualified prospect.

Artifact validation must be proven before adding general build-platform
complexity.

## Temporary station-offline sequence

During relocation:

- continue contracts, report schema, onboarding, BOM, backlog, and sanitized
  dashboard replay;
- do not publish replay data as live;
- defer webcam/live event integration and hardware acceptance.

After station reactivation:

1. stabilize and time-box ESP32;
2. connect dashboard MVP to real ESP32 evidence;
3. execute the Raspberry Pi 4 flagship;
4. open external artifact intake.

---

## Phase 0 — Before Day 1
Until 2026-09-30:
- preserve existing POC
- collect screenshots / architecture notes
- decide public demo target
- prepare backlog
- avoid major new implementation while traveling

---

## Phase 1 — October 2026
### Goal: Productize the existing POC

Do not rebuild the system. Turn the existing end-to-end POC into something another engineer can understand and operate.

Deliverables:
- stable Pi / Yocto reference DUT
- clean Jenkins pipeline
- Dockerized station runner
- artifact metadata model
- 20–30 reliable tests
- standardized PASS / FAIL / SKIP result schema
- evidence directory per run
- HTML report
- first downloadable PDF or static report if useful
- failure log sanitization
- README explaining one complete run

### Dashboard MVP
Implement:
- station health
- webcam
- current DUT / image
- test progress
- recent execution history
- sanitized live log
- PASS / FAIL summary
- report download

Build the dashboard in stages:

1. offline fixture/replay against the report and event contracts;
2. live ESP32 reference data after station soak;
3. Pi 4 baseline comparison and Linux metrics;
4. authentication/private customer features only after a pilot.

### Exit criteria
A person unfamiliar with the project can open one URL and understand:
- what hardware is running,
- what image is under test,
- which test is executing,
- what the final outcome was.

---

## Phase 2 — November 2026
### Goal: Package the proof

Deliverables:
- landing page
- public reference lab
- GitHub public demo or sanitized showcase repo
- sample validation report
- architecture diagram
- flagship 2–5 minute demo video
- first technical YouTube video
- concise service description
- evaluation offer

Suggested homepage message:

> Upload or build your embedded Linux image. We deploy it to real hardware, run regression tests, and return traceable release evidence.

Start building the lead database during this phase.

### Exit criteria
You can send one link to a CTO and that link contains enough proof to justify a technical call.

---

## Phase 3 — December 2026
### Goal: Start sales every week

No more zero-sales weeks.

Target customer:
- embedded startup / small company
- 5–20 technical engineers
- Linux / Yocto
- real hardware product
- no sophisticated HIL team

Lead sources:
- LinkedIn
- job posts
- Wellfound
- Y Combinator company directory
- GitHub
- Crunchbase
- embedded conference exhibitor lists
- Yocto / Raspberry Pi / NXP / Jetson ecosystems
- technical forums where appropriate

Search terms:
- Yocto
- Embedded Linux
- BSP
- Raspberry Pi Compute Module
- NXP i.MX
- Jetson
- IoT gateway
- industrial gateway
- edge device
- OTA
- firmware validation

Weekly activities:
- 10–20 new qualified companies researched
- 5–10 personalized outreach messages
- follow-ups
- 1 technical post or demo update
- 1 improvement driven by prospect feedback

Target by month end:
- 50–100 qualified leads
- 20–40 personalized outreach messages
- 3–5 technical conversations

---

## Phase 4 — January 2027
### Goal: Real external evaluation

Offer a low-friction evaluation:
- one board / one image
- 20–30 generic tests
- report
- short review meeting

Possible commercial formats:
- free for a highly qualified design partner
- $99–$300 evaluation
- credit evaluation fee toward paid pilot

Success criteria:
- at least one real external artifact / device goes through the pipeline
- feedback is collected on the report and value

---

## Phase 5 — February 2027
### Goal: First paid pilot

Suggested pilot:
- 2–4 weeks
- dedicated DUT
- 30–60 tests
- CI integration
- run history
- evidence
- failure summaries
- custom test additions

Target price:
- roughly $500–$1,500 depending on scope

The important metric is willingness to pay, not optimization of price.

Artifact mode remains the default pilot path. Managed Yocto build is a
separately scoped add-on, not a prerequisite for the first paid pilot.

---

## Phase 6 — March 2027
### Goal: First recurring customer

Convert successful pilot into:
- monthly validation
- dedicated station
- agreed release cadence
- test pack maintenance
- historical comparison
- report retention

Target recurring range initially:
- approximately $800–$1,500/month
- increase later as value and coverage increase

---

## Phase 7 — April to September 2027
### Goal: 2–5 recurring customers and reusable product packs

Do not expand into every platform.

Build only capabilities requested by real prospects/customers.

Candidate packs:
1. Linux Core
2. BSP
3. OTA / Recovery
4. Power
5. BLE / Wi-Fi
6. Cellular / SIM
7. CAN

Standardize station onboarding:
```text
platform adapter
+ station config
+ test pack
+ customer overrides
```

Add:
- scheduler
- station leases
- automatic recovery
- artifact retention policies
- trend database
- release comparison
- secrets management

---

## Phase 8 — 5 to 10 customers
### Goal: Stop behaving like a bespoke consulting shop

Requirements:
- standard onboarding checklist
- station template
- standard contracts / SOW
- customer isolation
- authentication
- consistent reports
- defined support hours
- test pack versioning
- measurement calibration process where applicable
- backup / restore
- automated station health checks

Founder should spend less time executing tests manually and more time on:
- customer relationships
- framework architecture
- high-value test design
- sales

---

## Phase 9 — 10 to 30 customers
### Goal: Productize the service

Build:
- multi-tenant customer portal
- image upload API
- release API
- scheduler
- station pools
- private station support
- automated invoicing integration
- role-based access
- audit history

At this point consider the first operations / test engineer.

---

## Phase 10 — 30 to 100 customers
### Goal: Operate as a validation platform

This scale is not realistic as a one-person bespoke service.

Needed:
- multiple operations engineers
- sales function
- more formal security program
- capacity planning
- regional station options where needed
- stronger customer isolation
- SLA tiers
- automated provisioning

The path to 100 customers is not “work harder.” It is “make each new customer mostly configuration.”

---

# AI roadmap

## V0
OpenAI or another strong LLM:
- summarize failures
- compare logs
- draft customer report text

## V1
Provider abstraction:
- OpenAI
- DeepSeek
- other providers

## V2
Failure memory:
- embeddings or structured incident store
- previous similar failures
- prior resolution suggestions

Do not add a dedicated memory product until normal SQL / artifact history becomes limiting.

## V3
Fast decision model / System-One layer
Possible use cases:
- infra versus DUT classification
- retry recommendation
- ownership routing
- severity

## V4
Guarded agent
Agent can invoke only approved tools:
- collect dmesg
- collect journal
- reboot DUT
- power cycle
- rerun test
- compare release
- create a diagnostic bundle

Never expose arbitrary shell execution without policy and isolation.

---

# Media roadmap

## Flagship content
1. **From Yocto image to real hardware validation report**
2. **Detecting a power regression automatically**
3. **OTA failure and automatic recovery**
4. **BLE/Wi-Fi reconnect regression**
5. **CAN signal validation on a real gateway**

Recommended cadence after launch:
- 1–2 high-quality videos/month
- 1 technical LinkedIn post/week when possible

Media is proof and education, not entertainment.

---

# Automotive roadmap

Do not lead with automotive in the first six months.

### Preparation now
Keep interfaces modular:
```text
SerialTransport
SSHTransport
PowerController
BLEAdapter
CellularAdapter
CANAdapter
```

### Generic CAN demo later
Build:
- reference Linux gateway
- USB-CAN
- simulated CAN peer
- DBC
- known signal regression
- intentional timing / value fault
- captured trace
- report

### Automotive-specific expansion later
Potential:
- CAN-FD
- UDS
- diagnostic sessions
- gateway validation
- update validation
- long-duration tests

Only market automotive-specific validation when operational maturity and compliance expectations are understood.

---

# Canada roadmap

If relocation occurs:
- retain Vietnam lab
- use Canada for sales and relationships
- attend industry events
- build one local demo station only if required
- offer Canada-hosted customer hardware as a premium option later

The hybrid model can become a differentiator:
**customer-facing presence in North America + cost-efficient execution lab in Vietnam.**
