# -ATS_Center-Ats-platform-docs

# File structure
ats-platform-docs/
├── README.md  
├── architecture/
│   ├── system-overview.md
│   ├── ci-flow.md
│   └── ats-node-design.md
├── roadmap.md
└── use-cases.md

# Introduce
Embedded Firmware Automation Test System (ATS)

Overview

This repository documents a production-style Embedded Firmware Automation Test System (ATS) designed to integrate directly with a Jenkins-based CI/CD pipeline and validate firmware on real hardware.
The system targets embedded and IoT teams who struggle with manual hardware testing, firmware regressions, and unsafe OTA releases, by introducing hardware-in-the-loop (HIL) testing that is automatically triggered on every firmware change.
This project is intentionally structured as a real internal platform, not a demo toy.

1. What Problem Does This System Solve?
In many embedded and IoT product teams:
CI pipelines stop at build or unit tests
Firmware validation on real devices is manual and slow
Hardware-dependent bugs are discovered late
OTA updates are risky and hard to reproduce
QA effort does not scale with firmware development speed
As a result:
Regressions reach field devices
Release confidence is low
Debugging hardware issues is expensive
This ATS addresses those pain points by:
Automatically flashing firmware to real ESP32 hardware
Running power, IO, connectivity, and OTA tests
Auditing results via UART, GPIO, and visual inspection
Feeding results back into CI and observability systems

2. High-Level System Architecture
The ATS is split into three clearly separated planes:
Control & Build Plane (Xeon Server)
Jenkins Master (pipeline orchestration)
Jenkins Build Agents (firmware build & artifact release)
Centralized metrics collection (Prometheus)
Visualization (Grafana)
Execution Plane (ATS Nodes)
Raspberry Pi / Mini PC
Jenkins ATS Agent
Direct access to hardware:
UART
GPIO
Relay / power control
Camera
Device Under Test (DUT)
ESP32 running test firmware
GPIO outputs / inputs
I2C OLED display
This separation ensures:
CI orchestration is stable
Hardware access is isolated
The system can scale to multiple ATS nodes

3. CI Flow: Commit → Build → Test → Report
The end-to-end workflow is intentionally Jenkins-native:
A developer creates a commit or PR in GitHub
GitHub triggers Jenkins via webhook
Jenkins Build Node (Xeon):
Builds ESP32 firmware
Produces a versioned firmware artifact
Jenkins dispatches the artifact to an ATS Node
ATS Node executes:
Firmware flashing
Power/reset control
UART log capture
GPIO and OLED validation
Optional visual audit via camera + AI
Test results and logs are:
Reported back to Jenkins
Exposed as metrics to Prometheus
Grafana dashboards provide:
Regression trends
Pass/fail history
Firmware version correlation
All steps run without manual intervention.

4. Role of Each Node
Jenkins Master (Xeon)
Pipeline orchestration only
No hardware access
Schedules jobs across build and ATS agents
Jenkins Build Node (Xeon)
Builds firmware and images
Produces artifacts
Does not run hardware tests
ATS Node (Raspberry Pi / Mini PC)
Dedicated hardware test executor
Flashing firmware to ESP32
Power cycling and fault injection
UART/GPIO inspection
Camera capture and visual validation
Exposes test metrics via HTTP for Prometheus
ESP32 (DUT)
Runs test firmware
Exposes observable behavior:
GPIO state
OLED output
Boot logs
OTA behavior

5. Observability and Auditability
The ATS is designed to be observable by default.
Metrics (Prometheus)
Examples:
Firmware test duration
Pass/fail counts
OTA failure rate
Reboot and crash counts
Firmware version under test
Visualization (Grafana)
Firmware regression trends
Stability over time
Confidence before release
This enables objective, data-driven release decisions, not gut feeling.

6. Multi-ATS Scalability
The architecture supports horizontal scaling by design.
Each ATS node is a self-contained test cell
Jenkins schedules jobs using agent labels
Prometheus scrapes metrics from all ATS nodes
Adding capacity means:
Plug in another Pi / Mini PC
Register a new Jenkins agent
No pipeline changes required
This model naturally evolves into a hardware test farm.

7. Repository Structure
This project is intentionally split into multiple repositories:
ats-platform-docs (this repo)
System overview, architecture, and roadmap

ats-fw-esp32-demo
ESP32 firmware designed for automated testing

ats-ats-node
ATS agent scripts for flashing, testing, audit, and metrics

ats-ci-infra
Jenkins, Prometheus, and Grafana infrastructure (Docker-based)

Each repository has a single, well-defined responsibility.

8. Project Status & Intent
This system is:
A reference architecture
A foundation for real product adaptation
Designed to mirror real embedded teams’ workflows
It is not a generic testing framework or a UI-driven product.
The intent is to demonstrate:
Embedded systems expertise
Hardware-aware CI/CD design
Test automation ownership
Production-grade thinking

Author
Hai Dang Son
Senior Embedded / Embedded Linux / IoT Engineer

Specialized in:
Firmware CI/CD
BSP and hardware bring-up
Hardware-in-the-loop testing
Automation and observability

