# ATS Manifest Specification (v1)

> **Contract between firmware build, ATS node, test runner, and CI system**

## Overview

The `ats-manifest.yaml` is the **single source of truth** for firmware test execution. It contains all information needed to:
- Identify the firmware artifact
- Understand build context
- Execute hardware tests
- Report results

**This manifest is the ONLY contract between components. No hardcoded assumptions allowed.**

---

## Schema Definition

### Root Structure

```yaml
manifest_version: 1  # Schema version (integer)

build:
  # Build system metadata
  ci_system: string          # CI system name (e.g., "jenkins", "gitlab-ci")
  job_name: string          # CI job name (full path if in folders)
  build_number: string      # CI build number
  git:
    repo: string            # Git repository URL
    commit: string          # Full commit SHA
    branch: string          # Git branch name
  artifact:
    name: string            # Firmware artifact filename
    checksum: string        # Checksum in format "algo:value" (e.g., "sha256:abc123...")
    build_node: string      # Node/host where build occurred

device:
  target: string            # Device target (e.g., "esp32", "nrf52", "raspberrypi")
  board: string             # Board identifier (e.g., "esp32-devkit", "nrf52-dk")

test_plan:
  - string                  # List of test identifiers to execute

timestamps:
  build_time: string        # ISO 8601 timestamp (UTC): "YYYY-MM-DDTHH:MM:SSZ"
```

---

## Field Specifications

### `manifest_version` (required)

- **Type**: Integer
- **Value**: `1` (for this specification)
- **Purpose**: Schema version for forward compatibility

### `build` (required)

Build system metadata. **CI-agnostic** - no Jenkins-specific fields.

#### `build.ci_system` (required)
- **Type**: String
- **Examples**: `"jenkins"`, `"gitlab-ci"`, `"github-actions"`
- **Purpose**: Identify CI system for reporting

#### `build.job_name` (required)
- **Type**: String
- **Examples**: `"platforms/ESP32/ats-fw-esp32-demo"`, `"firmware-build"`
- **Purpose**: Unique job identifier

#### `build.build_number` (required)
- **Type**: String
- **Examples**: `"42"`, `"123"`
- **Purpose**: Build sequence number

#### `build.git` (required)

Git source information.

- **`build.git.repo`** (required): Repository URL
  - **Examples**: `"https://github.com/PicoPiece/ats-fw-esp32-demo.git"`
- **`build.git.commit`** (required): Full commit SHA
  - **Examples**: `"32317d2a5b215ae2f49b5a5136ba37e9a552a3c1"`
- **`build.git.branch`** (required): Branch name
  - **Examples**: `"origin/main"`, `"develop"`

#### `build.artifact` (required)

Firmware artifact metadata.

- **`build.artifact.name`** (required): Filename
  - **Examples**: `"firmware-esp32.bin"`, `"firmware.bin"`
- **`build.artifact.checksum`** (required): Checksum in format `"algorithm:value"`
  - **Examples**: `"sha256:1deed34cb306a96dba6cb07d4f930a6c6f4a3fd8399881511ea9b92ee434700e"`
  - **Supported algorithms**: `sha256`, `md5`
- **`build.artifact.build_node`** (required): Build host identifier
  - **Examples**: `"xeon-fw-build"`, `"build-agent-01"`

### `device` (required)

Target device information.

- **`device.target`** (required): Device target identifier
  - **Examples**: `"esp32"`, `"nrf52"`, `"raspberrypi"`
- **`device.board`** (required): Board identifier
  - **Examples**: `"esp32-devkit"`, `"nrf52-dk"`, `"raspberrypi-4"`

### `test_plan` (required)

List of test identifiers to execute.

- **Type**: Array of strings
- **Examples**: 
  ```yaml
  test_plan:
    - gpio_toggle_test
    - oled_display_test
    - uart_boot_test
  ```
- **Purpose**: Test runner uses this to determine which tests to run
- **Note**: Test identifiers are implementation-specific to test runner

### `timestamps` (required)

Timing metadata.

- **`timestamps.build_time`** (required): ISO 8601 timestamp (UTC)
  - **Format**: `"YYYY-MM-DDTHH:MM:SSZ"`
  - **Examples**: `"2024-12-25T10:30:00Z"`

---

## Example Manifest

```yaml
manifest_version: 1

build:
  ci_system: jenkins
  job_name: platforms/ESP32/ats-fw-esp32-demo
  build_number: 42
  git:
    repo: https://github.com/PicoPiece/ats-fw-esp32-demo.git
    commit: 32317d2a5b215ae2f49b5a5136ba37e9a552a3c1
    branch: origin/main
  artifact:
    name: firmware-esp32.bin
    checksum: sha256:1deed34cb306a96dba6cb07d4f930a6c6f4a3fd8399881511ea9b92ee434700e
    build_node: xeon-fw-build

device:
  target: esp32
  board: esp32-devkit

test_plan:
  - gpio_toggle_test
  - oled_display_test
  - uart_boot_test

timestamps:
  build_time: "2024-12-25T10:30:00Z"
```

---

## Validation Rules

### Required Fields

All fields marked as `(required)` must be present. Missing required fields cause manifest validation to fail.

### Type Validation

- `manifest_version`: Must be integer `1`
- `build.build_number`: Must be non-empty string
- `build.artifact.checksum`: Must match pattern `"algorithm:hexvalue"` where algorithm is `sha256` or `md5`
- `timestamps.build_time`: Must match ISO 8601 format

### Semantic Validation

- `build.git.commit`: Should be valid SHA (40 hex characters for full SHA)
- `build.git.repo`: Should be valid URL
- `device.target`: Should match known device targets
- `test_plan`: Should be non-empty array

---

## Usage by Components

### Firmware Build (ats-fw-esp32-demo)

**Responsibility**: Generate `ats-manifest.yaml` with all required fields.

**Output**: 
- `firmware-esp32.bin` (or platform-specific name)
- `ats-manifest.yaml` (compliant with v1 schema)

**Location**: Both artifacts must be archived together.

### ATS Node (ats-ats-node)

**Responsibility**: 
- Read `ats-manifest.yaml` from `/workspace/ats-manifest.yaml`
- Extract `device.target` to determine hardware type
- Extract `build.artifact.name` to locate firmware file
- Extract `test_plan` to pass to test runner

**Input**: `/workspace/ats-manifest.yaml` + `/workspace/{artifact.name}`

### Test Runner (ats-test-esp32-demo)

**Responsibility**:
- Read `ats-manifest.yaml` for test parameters
- Execute tests listed in `test_plan`
- Use `device.target` and `device.board` for test configuration

**Input**: `ats-manifest.yaml` (read from workspace)

### CI System (Jenkins)

**Responsibility**:
- Archive `ats-manifest.yaml` alongside firmware artifact
- Pass both to ATS node via artifact copy
- Do NOT parse or modify manifest

---

## Forward Compatibility

### Versioning Strategy

- **v1**: Current stable schema
- **Future versions**: Will increment `manifest_version`
- **Backward compatibility**: Test runners should support at least v1

### Extension Points

If new fields are needed:
1. Add to schema as optional fields
2. Increment `manifest_version` only for breaking changes
3. Document in new specification version

---

## Implementation Notes

### No CI-Specific Logic

The manifest is **CI-agnostic**. Do not include:
- Jenkins-specific job URLs
- CI-specific environment variables
- Build system internals

### No Hardware-Specific Logic

The manifest describes **what** to test, not **how**:
- Do not include USB port paths
- Do not include GPIO pin numbers
- Do not include test execution details

### Single Source of Truth

All components read from the same manifest. No duplication or transformation.

---

## References

- **Schema Version**: 1
- **Last Updated**: 2024-12-25
- **Maintained By**: ATS Platform Team

