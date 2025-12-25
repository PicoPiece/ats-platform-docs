# ATS Test Output Contract (v1)

> **Standardized test result format for CI/CD consumption**

## Overview

The ATS test execution produces a **standardized output structure** in the `/results` directory. This contract ensures that:
- CI systems can reliably parse test results
- Test reports are consistent across platforms
- Exit codes correctly reflect pass/fail status
- All artifacts are CI-consumable

**This contract is the ONLY way test results are communicated to CI systems.**

---

## Directory Structure

All test results MUST be written to a single directory (typically `/workspace/results` or `/results`):

```
/results/
├── ats-summary.json    # Human-readable test summary (JSON)
├── junit.xml          # JUnit XML for CI test reporting
├── serial.log         # Serial/UART output log (text)
└── meta.yaml          # Execution metadata (YAML)
```

---

## File Specifications

### 1. `ats-summary.json` (Required)

**Purpose**: Human-readable test summary in JSON format.

**Schema**:
```json
{
  "status": "PASS" | "FAIL",
  "tests": [
    {
      "name": "string",        // Test case name
      "status": "PASS" | "FAIL" | "SKIP",
      "failure": "string"      // Optional: failure message (only if status is FAIL)
    }
  ],
  "manifest": {
    "build_number": "string",
    "device_target": "string"
  }
}
```

**Example**:
```json
{
  "status": "PASS",
  "tests": [
    {
      "name": "uart_boot_test",
      "status": "PASS"
    },
    {
      "name": "gpio_toggle_test",
      "status": "PASS"
    }
  ],
  "manifest": {
    "build_number": "42",
    "device_target": "esp32"
  }
}
```

**Validation**:
- `status` must be either `"PASS"` or `"FAIL"`
- `tests` must be a non-empty array
- Each test must have `name` and `status` fields
- `failure` field is optional and only present when `status` is `"FAIL"`

---

### 2. `junit.xml` (Required)

**Purpose**: Standard JUnit XML format for CI test reporting (Jenkins, GitLab CI, GitHub Actions, etc.).

**Schema**: Standard JUnit XML 1.0 format.

**Example**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="ATS Hardware Tests" tests="2" failures="0">
    <testcase name="uart_boot_test" classname="HardwareTest"/>
    <testcase name="gpio_toggle_test" classname="HardwareTest">
      <failure>GPIO pin 2 did not toggle as expected</failure>
    </testcase>
  </testsuite>
</testsuites>
```

**Validation**:
- Must be valid XML
- Root element: `<testsuites>`
- At least one `<testsuite>` element
- `tests` attribute = total number of test cases
- `failures` attribute = number of failed tests
- Each `<testcase>` must have `name` attribute
- Failed tests must include `<failure>` element with error message

---

### 3. `serial.log` (Optional but Recommended)

**Purpose**: Raw serial/UART output from the device during test execution.

**Format**: Plain text file with device output.

**Example**:
```
ets Jun  8 2016 00:22:57

rst:0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs_drv:0x00,hd_drv:0x00,wp_drv:0x00
mode:DIO, clock div:2
load:0x3fff0030,len:4
load:0x3fff0034,len:7128
...
Hello from ESP32!
ATS ESP32 Demo Firmware
Build #42
```

**Validation**:
- Plain text file (UTF-8 encoding)
- May be empty if serial logging is not available
- Should contain device boot messages and test output

---

### 4. `meta.yaml` (Required)

**Purpose**: Execution metadata for traceability and debugging.

**Schema**:
```yaml
execution:
  timestamp: string    # ISO 8601 UTC timestamp
  exit_code: integer  # Process exit code (0 = PASS, non-zero = FAIL)
  status: "PASS" | "FAIL"

manifest:
  version: integer           # Manifest schema version
  build_number: string       # Build number from manifest
  device_target: string      # Device target from manifest
```

**Example**:
```yaml
execution:
  timestamp: "2024-12-25T10:30:00Z"
  exit_code: 0
  status: "PASS"

manifest:
  version: 1
  build_number: "42"
  device_target: "esp32"
```

**Validation**:
- `execution.timestamp` must be ISO 8601 format (UTC)
- `execution.exit_code` must be integer (0 for success, non-zero for failure)
- `execution.status` must match exit code (0 = "PASS", non-zero = "FAIL")
- `manifest.version` must match manifest schema version

---

## Exit Code Contract

The test execution process MUST exit with the following codes:

- **`0`**: All tests passed (or skipped)
- **`1`**: One or more tests failed
- **`2`**: Execution error (e.g., manifest not found, hardware not available)

**CI systems MUST interpret exit code `0` as success and any non-zero as failure.**

---

## Result Generation Responsibility

### ATS Node Container (`ats-node-test`)

**Responsibility**: Generate all result files in `/results` directory.

**Process**:
1. Execute test runner
2. Collect test results
3. Write `ats-summary.json`
4. Write `junit.xml`
5. Write `serial.log` (if available)
6. Write `meta.yaml`
7. Exit with appropriate exit code

**Location**: `/workspace/results/` (mounted by Jenkins)

### Test Runner (`ats-test-esp32-demo`)

**Responsibility**: Execute tests and output structured results.

**Output**: May write intermediate results, but final aggregation is done by ATS node container.

---

## CI System Consumption

### Jenkins

**Consumption**:
- `junit.xml` → `publishTestResults` step
- `/results/**` → `archiveArtifacts` step
- Exit code → Pipeline pass/fail status

**Example**:
```groovy
stage('Archive Results') {
    steps {
        archiveArtifacts artifacts: "results/**", allowEmptyArchive: true
        publishTestResults testResultsPattern: "results/**/*.xml", allowEmptyResults: true
    }
}
```

### GitLab CI

**Consumption**:
- `junit.xml` → `junit` artifact report
- Exit code → Job pass/fail status

**Example**:
```yaml
artifacts:
  reports:
    junit: results/junit.xml
  paths:
    - results/
```

### GitHub Actions

**Consumption**:
- `junit.xml` → Test summary action
- Exit code → Step pass/fail status

**Example**:
```yaml
- name: Publish Test Results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()
  with:
    files: results/junit.xml
```

---

## Validation Checklist

Before marking test execution as complete, verify:

- [ ] `/results/ats-summary.json` exists and is valid JSON
- [ ] `/results/junit.xml` exists and is valid XML
- [ ] `/results/meta.yaml` exists and is valid YAML
- [ ] Exit code matches `meta.yaml` execution status
- [ ] `ats-summary.json.status` matches exit code
- [ ] `junit.xml` test counts match `ats-summary.json` test counts
- [ ] All failed tests have failure messages

---

## Forward Compatibility

### Versioning Strategy

- **v1**: Current stable contract
- **Future versions**: Will increment version number
- **Backward compatibility**: CI systems should support at least v1

### Extension Points

If new result files are needed:
1. Add as optional files (do not break existing CI consumers)
2. Document in new contract version
3. Maintain backward compatibility

---

## Implementation Notes

### File Encoding

- All text files: UTF-8
- JSON files: UTF-8 with no BOM
- XML files: UTF-8 with XML declaration

### Timestamps

- All timestamps: ISO 8601 format, UTC timezone
- Format: `"YYYY-MM-DDTHH:MM:SSZ"`

### Error Handling

- If result generation fails, exit with code `2`
- Partial results are acceptable (e.g., `serial.log` may be missing)
- Required files (`ats-summary.json`, `junit.xml`, `meta.yaml`) must always be generated

---

## References

- **Contract Version**: 1
- **Last Updated**: 2024-12-25
- **Maintained By**: ATS Platform Team
- **Related**: [ATS Manifest Specification v1](./ats-manifest-spec-v1.md)

