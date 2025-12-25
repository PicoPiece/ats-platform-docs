# ATS System Implementation & Improvements

> **Comprehensive documentation of ATS Center system implementation and recent improvements**

This document provides a complete overview of the ATS Center system implementation, including architecture, security improvements, observability setup, and test framework enhancements.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Security Improvements](#security-improvements)
3. [Observability & Metrics](#observability--metrics)
4. [Test Framework Enhancements](#test-framework-enhancements)
5. [Repository Structure](#repository-structure)
6. [Deployment Guide](#deployment-guide)
7. [Troubleshooting](#troubleshooting)

---

## System Architecture

### High-Level Architecture

The ATS Center system follows a 3-tier architecture:

```
┌─────────────────────────────────────────────────────────┐
│              Control Plane (Xeon Server)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Jenkins  │  │Prometheus│  │ Grafana  │  │ Build   │ │
│  │ Master   │  │          │  │          │  │ Agent   │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
└─────────────────────────────────────────────────────────┘
                          │
                          │ Network (ats-net)
                          │
┌─────────────────────────────────────────────────────────┐
│         Execution Plane (ATS Nodes - Raspberry Pi)      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  ATS Node 1  │  │  ATS Node 2  │  │  ATS Node N  │ │
│  │  - Jenkins   │  │  - Jenkins   │  │  - Jenkins   │ │
│  │    Agent     │  │    Agent     │  │    Agent     │ │
│  │  - Test      │  │  - Test      │  │  - Test      │ │
│  │    Runner    │  │    Runner    │  │    Runner    │ │
│  │  - Metrics   │  │  - Metrics   │  │  - Metrics   │ │
│  │    Exporter  │  │    Exporter  │  │    Exporter  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                          │
                          │ USB/UART/GPIO
                          │
┌─────────────────────────────────────────────────────────┐
│           Device Under Test (ESP32)                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ ESP32-1  │  │ ESP32-2  │  │ ESP32-N  │             │
│  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────┘
```

### Component Responsibilities

#### Control Plane (Xeon Server)
- **Jenkins Master**: Pipeline orchestration, job scheduling
- **Jenkins Build Agent**: Firmware compilation using ESP-IDF
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards

#### Execution Plane (ATS Nodes)
- **Jenkins Agent**: Receives test jobs from master
- **Test Runner Container**: Executes hardware tests
- **Metrics Exporter**: Exposes Prometheus metrics via HTTP
- **Hardware Access**: Direct access to ESP32 via USB/UART/GPIO

#### Device Under Test
- **ESP32**: Runs firmware under test
- **Observable Behavior**: UART logs, GPIO states, OLED display

---

## Security Improvements

### 1. Secrets Management

**Before:**
- Hardcoded Jenkins secret in `docker-compose.yml`
- Hardcoded Grafana credentials

**After:**
- Environment variables via `.env` file
- `.env.example` template for documentation
- Secrets excluded from version control (`.gitignore`)

**Implementation:**
```yaml
# docker-compose.yml
env_file:
  - .env
environment:
  - JENKINS_SECRET=${JENKINS_SECRET}
  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-admin}
```

**Setup:**
1. Copy `.env.example` to `.env`
2. Fill in actual secrets
3. `.env` is automatically ignored by git

### 2. Container Privileges Reduction

**Before:**
- Containers run with `--privileged` flag (full system access)

**After:**
- Specific capabilities: `--cap-add=SYS_RAWIO`, `--cap-add=SYS_ADMIN`
- Specific device mounts: `--device=/dev/ttyUSB0`
- Read-only mounts where possible: `-v /sys/class/gpio:/sys/class/gpio:ro`

**Security Benefits:**
- Reduced attack surface
- Principle of least privilege
- Better isolation between containers

---

## Observability & Metrics

### Prometheus Metrics Export

**Implementation:**
- Python-based HTTP server (`metrics_exporter.py`)
- Exposes `/metrics` endpoint on port 8080 (configurable)
- Automatically started by test execution scripts
- Metrics persisted to JSON file for reliability

**Available Metrics:**

| Metric | Type | Description |
|--------|------|-------------|
| `ats_test_pass_total` | Counter | Total number of passed tests |
| `ats_test_fail_total` | Counter | Total number of failed tests |
| `ats_test_duration_seconds` | Gauge | Duration of last test run |
| `ats_fw_version` | Gauge | Firmware version under test |
| `ats_test_last_run_timestamp` | Gauge | Unix timestamp of last test |
| `ats_test_in_progress` | Gauge | Whether test is running (1) or idle (0) |

**Metrics Endpoints:**
- `/metrics` - Prometheus format metrics
- `/health` - Health check endpoint

### Prometheus Configuration

**Location:** `ats-ci-infra/prometheus/prometheus.yml`

**Features:**
- Scrapes metrics from all ATS nodes
- Configurable scrape intervals (default: 30s)
- Label-based organization (node, platform)
- External labels for cluster identification

**Example Configuration:**
```yaml
- job_name: 'ats-nodes'
  scrape_interval: 30s
  metrics_path: '/metrics'
  static_configs:
    - targets: ['ats-node-1:8080']
      labels:
        node: 'ats-node-1'
        platform: 'raspberry-pi'
```

### Grafana Dashboards

**Location:** `ats-ci-infra/grafana/dashboards/ats-test-dashboard.json`

**Dashboard Panels:**
1. **Test Pass/Fail Rate** - Rate of test passes and failures over time
2. **Total Tests Passed** - Cumulative count of passed tests
3. **Total Tests Failed** - Cumulative count of failed tests
4. **Test Pass/Fail Trend** - Time series graph of test results
5. **Test Duration** - Duration of test runs over time
6. **Test Success Rate** - Percentage gauge (green >95%, yellow >80%, red <80%)
7. **Last Test Run** - Timestamp of most recent test execution
8. **Test In Progress** - Indicator of current test status

**Provisioning:**
- Automatic dashboard provisioning via JCasC
- Datasource auto-configured to Prometheus
- Dashboards available immediately after Grafana startup

---

## Test Framework Enhancements

### 1. Retry Logic

**UART Reading (`read_uart.sh`):**
- Configurable max retries (default: 3)
- Automatic retry on timeout or empty data
- Stale data clearing before retry
- Success validation based on line count

**GPIO Checking (`gpio_check.sh`):**
- Multiple samples per check (default: 3 samples)
- Majority voting for stable readings
- Retry on direction setting failures
- Configurable retry count

**Benefits:**
- Reduced flakiness from transient hardware issues
- More reliable test results
- Better handling of timing-sensitive operations

### 2. Error Handling Improvements

**Enhanced Error Messages:**
- Detailed troubleshooting hints
- Available device listings
- Permission guidance
- Hardware connection verification

**Example:**
```bash
❌ ESP32 device not found
   Checked ports: /dev/ttyUSB0, /dev/ttyUSB1, /dev/ttyACM0, /dev/ttyACM1
   Available USB devices: [list]
   Troubleshooting:
   - Ensure ESP32 is connected via USB
   - Check USB cable and port
   - Verify device permissions (user should be in dialout group)
```

**Graceful Degradation:**
- Tests continue even if some checks fail
- Partial results reported
- Clear distinction between failures and skips

### 3. Test Validation & Flakiness Detection

**Validation Features:**
- Detection of zero tests executed (all skipped)
- Success rate calculation
- Warning for low success rates (<80%)

**Flakiness Detection:**
- Reboot test failure tracking
- Individual reboot attempt monitoring
- Pattern detection for inconsistent results

**Test Summary:**
```
Test Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Passed: 2
Failed: 1
Total:  3
Success Rate: 66%

⚠️  WARNING: Low success rate (< 80%) - investigate test reliability
```

### 4. Metrics Integration

**Automatic Metrics Collection:**
- Metrics exporter started automatically
- Test results update metrics in real-time
- Firmware version extracted from manifest
- Test duration tracked automatically

**Metrics Update Flow:**
1. Test starts → `ats_test_in_progress = 1`
2. Tests execute → Results collected
3. Test completes → Metrics updated with:
   - Pass/fail counts
   - Duration
   - Firmware version
   - Timestamp
   - `ats_test_in_progress = 0`

---

## Repository Structure

### 5 Repositories

1. **ats-platform-docs** - System documentation
   - Architecture documentation
   - Implementation guides
   - This document

2. **ats-fw-esp32-demo** - Firmware source code
   - ESP32 firmware implementation
   - Multi-platform support structure
   - Jenkins build/test pipelines

3. **ats-test-esp32-demo** - Test execution framework
   - Hardware test scripts
   - Docker test runner
   - Metrics exporter
   - Test reports generation

4. **ats-ci-infra** - CI/CD infrastructure
   - Docker Compose setup
   - Jenkins configuration (JCasC)
   - Prometheus configuration
   - Grafana dashboards and provisioning

5. **ats-ats-node** - Hardware access tools (future)
   - Placeholder for Python-based tools
   - Future: Advanced hardware abstraction
   - Current: Shell scripts in ats-test-esp32-demo

---

## Deployment Guide

### Prerequisites

- Docker and Docker Compose installed
- Xeon server for Control Plane
- Raspberry Pi(s) for ATS nodes
- ESP32 device(s) for testing
- Network connectivity between components

### Control Plane Setup (Xeon Server)

1. **Clone and Configure:**
   ```bash
   cd ats-ci-infra
   cp .env.example .env
   # Edit .env with actual secrets
   ```

2. **Build Build Agent Image:**
   ```bash
   cd jenkins/fw-build
   docker build -t ats-fw-build:esp32 .
   ```

3. **Start Services:**
   ```bash
   docker-compose up -d
   ```

4. **Verify:**
   - Jenkins: http://localhost:8080
   - Prometheus: http://localhost:9090
   - Grafana: http://localhost:3000 (admin/admin)

### ATS Node Setup (Raspberry Pi)

1. **Install Dependencies:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io docker-compose
   sudo usermod -aG docker $USER
   ```

2. **Configure Jenkins Agent:**
   - Connect to Jenkins master
   - Label: `ats-node`
   - Ensure USB/UART access permissions

3. **Test Hardware Access:**
   ```bash
   ls -la /dev/ttyUSB* /dev/ttyACM*
   # Should see ESP32 device
   ```

### First Test Run

1. **Trigger Build Pipeline:**
   - Jenkins UI → `platforms/ESP32/ats-fw-esp32-demo`
   - Build with default parameters

2. **Monitor Test Pipeline:**
   - Auto-triggered after build
   - Runs on ATS node with `ats-node` label
   - Check logs for test execution

3. **View Results:**
   - Jenkins: Test reports and artifacts
   - Grafana: Metrics and trends
   - Prometheus: Raw metrics data

---

## Troubleshooting

### Common Issues

#### 1. Jenkins Agent Not Connecting

**Symptoms:**
- Agent shows as offline in Jenkins
- Build jobs stuck in queue

**Solutions:**
- Check `JENKINS_SECRET` in `.env` matches Jenkins UI
- Verify network connectivity
- Check agent logs: `docker logs jenkins-build-agent`

#### 2. ESP32 Device Not Found

**Symptoms:**
- Test fails with "ESP32 device not found"
- Flash operations fail

**Solutions:**
- Verify USB connection
- Check device permissions: `ls -la /dev/ttyUSB*`
- Add user to dialout group: `sudo usermod -aG dialout $USER`
- Restart container/service

#### 3. Metrics Not Appearing in Prometheus

**Symptoms:**
- No metrics in Prometheus UI
- Grafana dashboards empty

**Solutions:**
- Verify metrics exporter is running: `curl http://ats-node:8080/metrics`
- Check Prometheus targets: http://localhost:9090/targets
- Verify network connectivity between Prometheus and ATS nodes
- Check Prometheus scrape config

#### 4. Test Failures / Flakiness

**Symptoms:**
- Tests fail intermittently
- UART reads timeout
- GPIO checks inconsistent

**Solutions:**
- Retry logic should handle transient issues automatically
- Check hardware connections
- Verify timing (add delays if needed)
- Review test logs for specific error patterns
- Check success rate in Grafana dashboard

#### 5. Container Privilege Issues

**Symptoms:**
- GPIO access denied
- Device access errors

**Solutions:**
- Verify container has required capabilities
- Check device mounts in Jenkinsfile
- Ensure `/dev/gpiomem` is accessible
- Review container security settings

---

## Best Practices

### Security
- Never commit `.env` files
- Rotate secrets regularly
- Use least-privilege container capabilities
- Monitor for security updates

### Testing
- Run tests on multiple ATS nodes for validation
- Monitor success rates in Grafana
- Investigate flaky tests promptly
- Keep test logs for debugging

### Monitoring
- Set up Grafana alerts for low success rates
- Monitor test duration trends
- Track firmware version correlation with failures
- Review metrics regularly

### Maintenance
- Update Docker images regularly
- Keep ESP-IDF toolchain updated
- Review and update test scripts
- Document hardware changes

---

## Future Enhancements

### Planned Improvements
- [ ] Parallel test execution across multiple ATS nodes
- [ ] External artifact storage (S3/Artifactory)
- [ ] Webhook integration for GitHub
- [ ] Visual validation with camera + AI
- [ ] Advanced test parameterization
- [ ] Test suite management
- [ ] Alerting rules for Prometheus

### Architecture Evolution
- [ ] Kubernetes deployment option
- [ ] Multi-region support
- [ ] Cloud-based ATS nodes
- [ ] Enhanced firmware features (GPIO, OLED, OTA demos)

---

## References

- [System Overview](../README.md)
- [CI Structure](./ci-structure.md)
- [Repository READMEs](../README.md)

---

**Last Updated:** $(date -u +%Y-%m-%d)  
**Version:** 1.0  
**Author:** ATS Platform Team
