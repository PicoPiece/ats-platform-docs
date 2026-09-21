# Picopiece Validation Lab — Solo-Founder Business Plan

## Starting point
Official start: **October 1, 2026**

Founder operates:
- engineering
- infrastructure
- test design
- customer discovery
- sales
- demos
- reporting

The business plan must therefore optimize for focus and repeatability.

## Objective for Year 1
The Year-1 goal is **not** 100 customers.

Primary goals:
- prove willingness to pay
- acquire first international customer
- reach 2–5 recurring customers
- build reusable validation IP
- make the system pay for its own infrastructure and AI subscriptions

A small amount of recurring foreign revenue is already a meaningful validation of the idea.

## Problem
Small embedded teams often have:
- capable product engineers
- CI for software
- limited real-hardware automation
- manual firmware flashing
- ad-hoc regression
- weak release evidence
- engineers spending time reproducing infrastructure failures

Hiring a complete in-house HIL / validation function can be expensive and distracting.

## Solution
Picopiece provides managed release validation on real hardware.

Customer keeps:
- product design
- product source code
- feature roadmap
- product decisions

Picopiece provides:
- build/image intake
- hardware deployment
- test execution
- regression test packs
- logs and evidence
- release comparison
- professional validation report
- optional AI-assisted triage

The defensible customer statement is:

> Release X was validated against the declared test scope on Hardware
> Configuration H.

Do not market early results as certification, bug-free software, or verified
software quality outside the declared scope.

## Value proposition
Not “cheap QA labor.”

The value is:
- repeatability
- release confidence
- real-hardware evidence
- less engineering time spent maintaining test infrastructure
- reusable historical results
- validation while the customer's team sleeps
- access to specialized test infrastructure without building an internal lab

## Initial ideal customer profile
Best first prospect:
- startup or small engineering company
- 10–50 employees
- 5–20 engineering staff
- real embedded Linux product
- Yocto / Raspberry Pi / NXP / Jetson / similar
- releasing software repeatedly
- no dedicated HIL team
- CTO or embedded lead accessible

Good verticals:
- industrial IoT
- gateways
- energy
- solar
- EV charging
- robotics
- edge AI
- building automation
- smart devices

Later:
- automotive
- medical devices where appropriate
- heavier regulated products

## Business model

### Evaluation
Purpose: remove friction.

Possible:
- free for highly qualified design partner
- $99–$300 paid evaluation

Includes:
- one image
- reference or customer DUT
- 20–30 standard tests
- report

Before accepting the image, complete the technical intake in
`06_CUSTOMER_ONBOARDING_CONTRACT.md`. Deliver the report according to
`05_VALIDATION_REPORT_SCHEMA_V1.md`.

Artifact mode is the default evaluation workflow. Managed customer Yocto build
is not required for the first evaluation.

### Paid pilot
Suggested range:
**$500–$1,500**

Includes:
- 2–4 weeks
- dedicated target
- 30–60 tests
- initial custom tests
- CI integration
- run history
- report review

If managed build or custom BSP work is requested, scope and price it separately
under `09_CUSTOM_BUILD_POLICY.md`.

### Managed validation subscription
Initial target:
**$800–$1,500/month per customer/station**

Later mature service:
**$1,500–$3,000+/month** depending on:
- number of DUTs
- test frequency
- custom fixtures
- test coverage
- reporting requirements
- support/SLA
- managed Yocto build

### High-touch dedicated lab
Later:
**$3,000–$10,000+/month**
for:
- multiple targets
- dedicated racks
- power measurement
- cellular
- CAN
- custom fixtures
- nightly testing
- long-duration tests
- build services

## Cost philosophy
Keep fixed cost extremely low until demand exists.

Existing strengths:
- build server
- mini PCs
- reference devices
- domain expertise
- home lab
- network and energy infrastructure
- legal entity in Vietnam

Early monthly expenses may include:
- domain / hosting
- cloud/VPS if needed
- OpenAI / model API
- GitHub / CI subscriptions
- monitoring
- replacement parts
- shipping
- payment/accounting fees

The first commercial goal can simply be:
> Recurring revenue > recurring infrastructure + AI cost.

This makes even a $300–$500/month first customer strategically useful.

## Why no office initially
A physical storefront does not increase the value of remote validation.

Spend first on:
- reliability
- spare hardware
- UPS
- network redundancy
- instrumentation
- security
- automation

Move to dedicated commercial space only when:
- customer contracts require it
- station volume justifies it
- security/audit requirements require it

## Sales strategy
No paid advertising initially.

This is a high-trust technical B2B sale. Founder-led outbound is more appropriate.

### Sales funnel
```text
qualified company
   ↓
personalized outreach
   ↓
technical discussion
   ↓
live demo
   ↓
evaluation
   ↓
paid pilot
   ↓
monthly validation
```

### Where to find leads
- LinkedIn
- company engineering blogs
- embedded Linux job listings
- Yocto ecosystem
- GitHub
- Y Combinator startups
- Wellfound
- hardware accelerators
- NXP / Raspberry Pi / Jetson ecosystems
- conference exhibitor lists
- relevant embedded communities

### Best signal
A job posting for:
- Yocto
- Embedded Linux
- BSP
- firmware QA
- CI / HIL

means the company likely has a relevant pain.

## Outreach message
Do not start with AI.

Example:

> Hi [Name],
> I noticed your team is building [device/product] on Embedded Linux.
> I run a managed real-hardware validation lab for embedded releases. A team can send an image or connect its CI, and the lab flashes a real target, runs regression tests, captures evidence, and returns a release validation report.
> I have a live Yocto/Raspberry Pi reference lab available to demonstrate the workflow.
> If useful, I can run a small evaluation against one of your releases.

## Landing page
Landing page is required even if GitHub already contains the full POC.

GitHub proves engineering.
The landing page explains the business.

Must answer:
1. What problem does this solve?
2. How does it work?
3. Is the hardware real?
4. What does the customer receive?
5. How can I try it?
6. What exact scope and limitations apply to a result?

### Hero
**Real-hardware release validation for Embedded Linux.**

Subtext:
> Send an image or connect your CI. We deploy it to real hardware, execute regression tests, collect evidence, and return a validation report.

CTA:
**Run a release through the lab**

### Live proof
Show:
- webcam
- station status
- live sanitized test run
- current image
- result counters
- downloadable sample report

## YouTube strategy
Use YouTube as technical trust, not as a creator business.

### Flagship video
2–5 minutes:
**From Yocto Image to Real Hardware Validation Report**

Flow:
- release appears
- image built/uploaded
- board flashes
- webcam shows board
- tests execute
- deliberate regression fails
- report generated

Embed it on the landing page.

### Ongoing technical proof
Examples:
- power regression
- OTA recovery
- BLE reconnect
- cellular test
- CAN validation
- remote recovery
- custom Yocto BSP validation

Frequency:
1–2 strong videos/month is enough.

## Weekly founder operating system

### Time split
At the start:
- 40% productization / engineering
- 30% sales / customer discovery
- 20% content / demo assets
- 10% operations / review

After demo is complete:
- reduce engineering
- increase sales

### Non-negotiable rule
Every week must contain at least one activity that can directly lead to a customer.

Do not spend four weeks only improving infrastructure.

### Weekly dashboard
Track:
- qualified leads added
- outreach sent
- follow-ups sent
- replies
- technical conversations
- demos
- evaluations
- paid pilots
- recurring customers
- MRR
- reusable tests added
- station uptime

## 90-day targets from Day 1

### October
- polished end-to-end demo
- dashboard
- report
- test pack
- live public station

### November
- landing page
- flagship video
- GitHub proof
- sample report
- 20+ initial leads

### December
- total 50–100 leads
- 20–40 personalized outreach
- 3–5 technical conversations
- at least one serious evaluation prospect

Do not judge the business by revenue in the first 90 days.
Judge it by whether strangers engage with the offer.

## Six-month target
By end of March 2027:
- several real customer conversations
- 1+ external evaluation
- 1 paid pilot
- ideally 1 recurring customer

## Twelve-month target
By October 2027:
- 2–5 recurring customers
- reusable Linux/BSP packs
- at least one differentiated pack such as Power, OTA, Connectivity, or CAN
- monthly revenue covering the project’s own running cost
- clear evidence whether to scale further

## Scaling rule
### 1–5 customers
Founder-led, highly technical.

### 5–10 customers
Standardized station + test packs.

### 10–30 customers
Add portal, scheduler, automation, first operations engineer.

### 30–100 customers
Requires organization, not a solo founder:
- operations
- sales
- security
- SLAs
- formal capacity management

## Automotive strategy
Automotive is an expansion market, not the starting market.

First create a generic CAN pack and sell it where compliance barriers are lower:
- industrial gateways
- EV charging
- BMS
- robotics
- heavy equipment

Later, automotive-specific work can include:
- CAN-FD
- DBC validation
- UDS
- gateway / ECU validation
- OTA regression

Avoid implying certified automotive validation until the relevant processes and standards are actually in place.

## Canada strategy
If founder moves to Canada:

### Keep Vietnam
- main lab
- build servers
- long-run execution
- lower-cost hardware scaling

### Use Canada for
- customer acquisition
- networking
- face-to-face meetings
- conferences
- North American trust
- optional local edge station

This can become a strong operating model rather than a disruption.

## Founder risk management
The biggest risk is not technical failure.
It is spending all available time improving the platform without asking customers to pay.

The project should be managed as:
**engineering system + sales experiment.**

Technical progress is necessary, but the first paid foreign customer is the real proof of the business.

Near-term engineering must therefore follow this order:

1. contracts and report while the station is offline;
2. time-boxed ESP32 station reactivation and golden evidence;
3. Raspberry Pi 4 flagship validation;
4. external artifact evaluation;
5. managed customer build only when a qualified prospect needs it.
