# Device Layer: Wire Scanner Hardware Control

**Purpose**: Understand low-level wire scanner device control, EPICS PV management, and hardware abstraction in the Wire Scanner system.

**Audience**: Developers working with wire scanner hardware interfaces, EPICS integration, or device-level functionality

**Last Updated**: March 6, 2026

---

## Overview

The Device Layer provides the foundation for all wire scanner operations by managing direct communication with wire scanner hardware through EPICS Process Variables (PVs). This layer abstracts hardware complexity and provides type-safe, validated interfaces for:

- Motor position control and readback
- Scan parameter configuration (speed, pulse count, scan ranges)
- Device state management (initialization, homing, enabled status)
- Safety interlocks and validation (speed limits, range constraints)
- Profile plane selection (X, Y, U planes)

**Location**: `lcls_tools/common/devices/wire.py`

**Dependencies**: 
- `epics` - EPICS Channel Access client
- `pydantic` - Data validation and type safety
- `lcls_tools.common.devices.device` - Base device abstraction

---

## Architecture

### Class Hierarchy

```
Device (base class)
  └── Wire
        └── WireControlInformation
              └── WirePVSet (EPICS PV mapping)
        └── WireMetadata (detector/BPM associations)

WireCollection (container)
  └── Dict[str, Wire]
```

### Responsibility Boundaries

**Device Layer DOES**:
- Read/write EPICS PVs with type validation
- Enforce safety constraints (speed limits, range validation)
- Manage device state transitions (initialization, homing)
- Provide property-based access to hardware parameters
- Abstract wire-specific PV naming conventions

**Device Layer DOES NOT**:
- Perform data collection or analysis
- Manage scan workflows or orchestration
- Handle data persistence or plotting
- Implement GUI logic or threading

---

## Wire Class

### Core Responsibilities

The `Wire` class represents a single wire scanner device with three possible simultaneous scan planes (X, Y, U). Each wire has:

1. **Control Information**: EPICS PV mappings and hardware parameters
2. **Metadata**: Associated detectors and beam position monitors (BPMs)
3. **Configuration State**: Current scan ranges, enabled planes, speed settings
4. **Validation Logic**: Constraints ensuring safe and valid operations

### Initialization

```python
from lcls_tools.common.devices.reader import create_wire

# Create wire device from YAML configuration
wire = create_wire(name="WS28144", area="L3")

# Wire object includes:
# - controls_information: WireControlInformation with PV connections
# - metadata: WireMetadata with detector/BPM lists
# - name: "WS28144"
# - area: "L3"
```

**Creation Pattern**: Use `create_wire()` factory function rather than direct instantiation. This ensures proper YAML configuration loading and PV connection establishment.

---

## WirePVSet: EPICS PV Mapping

### PV Categories

**Motion Control**:
- `motor` - Motor position setpoint (write)
- `motor_rbv` - Motor readback value (read-only)
- `start_scan` - Initiate scan sequence (write, value=1)
- `abort_scan` - Emergency stop (write, value=1)
- `retract` - Move wire to home/safe position (write, value=1)
- `speed` - Current calculated speed (read/write, µm/s)
- `speed_max` - Maximum allowable speed (read-only, µm/s)
- `speed_min` - Minimum allowable speed (read-only, µm/s)

**Scan Configuration**:
- `scan_pulses` - Number of beam pulses to acquire (read/write)
- `beam_rate` - Current beam repetition rate (read-only, Hz)
- `timeout` - Enable/disable operation timeout (read/write, bool)

**Profile Planes** (per-plane settings for X, Y, U):
- `use_{x,y,u}_wire` - Enable/disable plane for scanning (bool)
- `{x,y,u}_wire_inner` - Inner scan range endpoint (int, motor units)
- `{x,y,u}_wire_outer` - Outer scan range endpoint (int, motor units)
- `{x,y,u}_size` - Wire thickness in microns (read-only)

**Device State**:
- `initialize` - Trigger initialization sequence (write, value=1)
- `initialize_status` - Initialization complete flag (read-only, bool)
- `enabled` - Device enabled status (read-only, bool)
- `homed` - Home position reached flag (read-only, bool)
- `torque_enable` - Motor torque status (read/write, bool)
- `temperature` - RTD temperature reading (read-only, optional)
- `install_angle` - Wire installation angle (read-only, degrees)

### Optional PVs

Some PVs are optional and may be `None` depending on wire scanner installation and configuration:
- `beam_rate` - Substituted with global beam rate PVs in some areas
- `temperature`, `torque_enable`, `homed` - Hardware-dependent features
- `retract`, `initialize` - Available only on newer scanner models

**Always check for `None`** before accessing optional PVs:

```python
if wire.controls_information.PVs.temperature is not None:
    temp = wire.controls_information.PVs.temperature.get()
```

---

## Wire Properties and Methods

### State Properties

**Read-Only State Indicators**:

```python
# Initialization status
wire.initialize_status  # bool: True if initialized
wire.initialize()       # Trigger initialization

# Position and homing
wire.motor              # int: Current motor position
wire.motor_rbv          # int: Motor readback (may differ during motion)
wire.homed              # bool: True if at home position

# Device status
wire.enabled            # bool: Device enabled for operations
wire.beam_rate          # float: Current beam rate (Hz)
wire.install_angle      # float: Wire installation angle (degrees)
wire.temperature        # float: RTD temperature (if available)
```

**Design Note**: State properties are read-only because they reflect hardware status. Use methods like `initialize()` or `retract()` to change state.

### Speed Properties

```python
# Current speed (read/write)
wire.speed = 10000      # Manual override (rarely needed)
current = wire.speed    # Read current/calculated speed

# Speed limits (read-only, hardware-defined)
wire.speed_min          # Minimum safe speed (µm/s)
wire.speed_max          # Maximum safe speed (µm/s)
```

**Speed Calculation**: The EPICS IOC automatically calculates required speed based on:
- `beam_rate` - Beam repetition rate (Hz)
- `scan_pulses` - Desired sample count
- Scan range - Distance to traverse (`outer - inner`)

Formula: `speed = beam_rate × (range / scan_pulses)`

**Note**: In typical usage, you configure `scan_pulses` and scan ranges, then the IOC calculates and validates the required speed. Manual speed setting is rarely needed.

### Scan Pulses Configuration

```python
# Number of beam pulses to acquire during scan
wire.scan_pulses = 350          # Set pulse count
current = wire.scan_pulses      # Read current setting

# Typical values:
# - 350 for standard profile measurements
# - Higher values improve statistics but increase scan time
# - Must balance with speed constraints
```

### Profile Plane Configuration

**Plane Activation**:

```python
# Enable/disable planes individually
wire.use_x_wire = True          # Enable X (horizontal) profile
wire.use_y_wire = True          # Enable Y (vertical) profile
wire.use_u_wire = False         # Disable U (longitudinal) profile

# Check current settings
x_enabled = wire.use_x_wire     # bool

# Generic interface
wire.use(plane="X", val=True)   # Enable plane by name (X/Y/U)
```

**Range Configuration**:

```python
# Set scan range for a plane (two-endpoint format)
wire.x_range = [37000, 45000]   # [inner, outer] in motor units
current = wire.x_range          # Returns [inner, outer] list

# Individual endpoint control
wire.x_wire_inner = 37000       # Set inner endpoint
wire.x_wire_outer = 45000       # Set outer endpoint

# Read individual endpoints
inner = wire.x_wire_inner
outer = wire.x_wire_outer

# Generic interface for all planes
wire.set_range(plane="Y", val=[20000, 28000])
wire.set_inner_range(plane="U", val=15000)
wire.set_outer_range(plane="U", val=23000)
```

**Range Validation**:
- Inner < Outer (enforced by `RangeModel`)
- Both values must be integers
- Values typically in micron units (area-dependent)

**X, Y, U Properties** (similar patterns for all planes):

```python
# Y plane
wire.use_y_wire = True
wire.y_range = [20000, 28000]
wire.y_wire_inner, wire.y_wire_outer  # Individual access
wire.y_size  # Wire thickness (µm, read-only)
```

### Scan Control Methods

```python
# Start scan with current parameters
wire.start_scan()

# Emergency stop during scan
wire.abort_scan()

# Return wire to home position
wire.retract()
```

**Thread Safety**: These methods write to EPICS PVs asynchronously. The caller is responsible for:
- Monitoring scan completion (via GUI signals or polling)
- Handling scan failures or timeouts
- Coordinating multi-wire operations

---

## Validation and Safety

### Pydantic Validation Models

The Device Layer uses Pydantic models to enforce type safety and business logic constraints:

**`RangeModel`** - Validates scan ranges:
```python
# Enforces:
# 1. List length == 2
# 2. First element < second element

wire.x_range = [45000, 37000]  # Raises ValidationError
wire.x_range = [37000, 45000]  # Valid
```

**`PlaneModel`** - Validates plane identifiers:
```python
# Enforces: plane ∈ {"X", "Y", "U"} (case-insensitive)

wire.use(plane="Z", val=True)  # Raises ValidationError
wire.use(plane="x", val=True)  # Valid (case-insensitive)
```

**`IntegerModel`** - Validates integer parameters:
```python
# Enforces strict integer types (no floats coerced)

wire.motor = 42000      # Valid
wire.motor = 42000.5    # Raises ValidationError
```

**`BooleanModel`** - Validates boolean parameters:
```python
wire.use_x_wire = True  # Valid
wire.use_x_wire = 1     # Raises ValidationError (strict bool)
```

### State Checking

**Initialization Check**: Always verify initialization status before operations:
```python
# Check initialization before scanning
if not wire.initialize_status:
    wire.initialize()
    # Wait for initialization to complete (polling/callback in higher layers)

# Now safe to configure and start scan
wire.start_scan()
```

**Speed Constraints**: Ensure speed satisfies hardware limits and beam rate requirements:
```python
# Optional pre-validation: check if IOC-calculated speed will be within limits
wire_range = wire.x_range[1] - wire.x_range[0]
expected_speed = wire.beam_rate * (wire_range / wire.scan_pulses)

# Verify against hardware limits
if wire.speed_min < expected_speed < wire.speed_max:
    wire.start_scan()  # IOC will calculate and apply speed
else:
    print(f"Expected speed {expected_speed} outside limits [{wire.speed_min}, {wire.speed_max}]")
    # Adjust scan_pulses or range before starting
```

### Safety Interlocks

**Hardware Limits** (enforced by EPICS IOC):
- Physical range limits prevent motor damage
- Speed limits prevent wire breakage
- Timeout protection for stuck scans

**Software Validation** (enforced by Device Layer):
- Range ordering (inner < outer)
- Type safety for all parameters
- State preconditions for operations

---

## WireMetadata

### Purpose

Associates wire scanner with downstream measurement devices:

```python
class WireMetadata(Metadata):
    detectors: List[str]                # Required: data acquisition devices
    bpms_before_wire: Optional[List[str]]  # Upstream BPMs
    bpms_after_wire: Optional[List[str]]   # Downstream BPMs
```

### Detector Configuration

**Detectors**: Devices that measure beam-induced signal as wire crosses beam:
- PMT (Photo-Multiplier Tube) - measures wire-generated light
- LBLM (Beam Loss Monitor) - measures scattered particles
- TMIT Loss - measures beam loss as function of transmitted intensity using BPMs

**Configuration Example** (from YAML):
```yaml
WS28144:
  controls_information:
    PVs:
      # ... PV mappings ...
  metadata:
    detectors:
      - "PMT:LI29:150"
    bpms_before_wire:
      - "BPMS:LI28:401"
    bpms_after_wire:
      - "BPMS:LI28:501"
```

**Usage in Measurement Layer**:
```python
# Measurement layer automatically queries all configured detectors
for detector in wire.metadata.detectors:
    # Acquire data from detector during scan
    pass
```

---

## WireCollection

### Purpose

Manages multiple wire scanner devices as a logical group. Enables:
- Batch initialization of related wires
- Convenient access to wire groups by beampath or area
- Configuration validation across multiple devices

### Structure

```python
class WireCollection(BaseModel):
    wires: Dict[str, Wire]
    
# Key format: wire name (string)
# Value: Wire instance
```

### Creation and Access

```python
from lcls_tools.common.devices.reader import create_wire_collection

# Load all wires from configuration
collection = create_wire_collection()

# Access individual wire by name
wire = collection.wires["WS28144"]

# Iterate over all wires
for name, wire in collection.wires.items():
    print(f"{name}: Area {wire.area}, Enabled: {wire.enabled}")
```

### Validation

**Name Injection**: Validator automatically sets wire name from dictionary key:
```python
@field_validator("wires", mode="before")
def validate_wires(cls, v) -> Dict[str, Wire]:
    for name, wire in v.items():
        wire = dict(wire)
        wire.update({"name": name})  # Inject name field
        v.update({name: wire})
    return v
```

This ensures `wire.name` always matches its collection key.

### Use Cases

**1. Multi-Wire Initialization**:
```python
# Initialize all wires in a beampath
for name, wire in collection.wires.items():
    if wire.area == "LI28" and not wire.initialize_status:
        wire.initialize()
```

**2. Configuration Audits**:
```python
# Check which wires are enabled
enabled_wires = [
    name for name, wire in collection.wires.items()
    if wire.enabled
]
```

**3. Batch Operations**:
```python
# Set common scan parameters across multiple wires
for wire in collection.wires.values():
    wire.scan_pulses = 350
    wire.use_x_wire = True
    wire.use_y_wire = True
```

---

## EPICS Integration

### Beam Rate Handling

Some wire scanners lack dedicated beam rate PVs. The `Wire` class implements fallback logic:

```python
@property
def beam_rate(self):
    nc_areas = ["L3", "LI20", "LI24", "LI28", "LTUH", "DL1", "BC1", "BC2", "LTU"]
    
    if self.controls_information.PVs.beam_rate is None:
        if self.area in nc_areas:
            # Use global LCLS beam rate
            return PV("EVNT:SYS0:1:LCLSBEAMRATE").get()
        elif self.area == "DIAG0":
            # Use timing system rate
            return PV("TPG:SYS0:1:DST01:RATE").get()
    else:
        # Use wire-specific beam rate PV
        return self.controls_information.PVs.beam_rate.get()
```

**Implication**: Developers working with beam rate should always use `wire.beam_rate` property rather than directly accessing the PV, as the property handles fallback logic transparently.


## Complete Usage Example

### Single Wire Scan Configuration

```python
from lcls_tools.common.devices.reader import create_wire

# 1. Create wire device
wire = create_wire(area="LI28", name="WS28144")

# 2. Check and initialize if needed
if not wire.initialize_status:
    print("Initializing wire scanner...")
    wire.initialize()
    # Wait for initialization (implementation-dependent)
    # In practice, higher layers handle polling/waiting

# 3. Configure scan parameters
wire.scan_pulses = 350

# 4. Configure profile planes
wire.use_x_wire = True
wire.use_y_wire = True
wire.use_u_wire = True

# 5. Set scan ranges (motor units of microns)
wire.x_range = [37000, 45000]
wire.y_range = [20000, 28000]

# 6. Verify configuration
print(f"Beam rate: {wire.beam_rate} Hz")
print(f"Speed limits: {wire.speed_min}-{wire.speed_max} µm/s")
print(f"X range: {wire.x_range}")
print(f"Y range: {wire.y_range}")
print(f"U range: {wire.u_range}")
print(f"Detectors: {wire.metadata.detectors}")

# 7. Start scan (if state checks pass)
wire.start_scan()

# 8. Monitor scan progress (handled by higher layers)
# Device layer does not provide progress monitoring or data collection
# See Measurement Layer documentation for data collection
```

### Error Handling

```python
from pydantic import ValidationError

try:
    # Attempt invalid range
    wire.x_range = [45000, 37000]  # Inner > Outer
except ValidationError as e:
    print(f"Configuration error: {e}")

try:
    # Attempt invalid plane
    wire.use(plane="Z", val=True)
except ValidationError as e:
    print(f"Invalid plane: {e}")

# Check state before operations
if not wire.enabled:
    print("Wire scanner not enabled, cannot scan")
elif not wire.initialize_status:
    print("Wire scanner not initialized, calling initialize()")
    wire.initialize()
else:
    wire.start_scan()
```

---

## Design Principles

### 1. Property-Based Interface

**Rationale**: EPICS PVs are state-based values, not operations. Properties provide natural syntax for reading and writing state.

```python
# Property access (recommended)
wire.speed = 10000
current = wire.speed

# Method-based alternative (verbose, less intuitive)
wire.set_speed(10000)
current = wire.get_speed()
```

### 2. Validation at Boundaries

**Rationale**: Catch invalid configurations before EPICS communication, providing immediate feedback with clear error messages.

```python
# Validation happens at assignment
wire.x_range = [45000, 37000]  # Immediate ValidationError

# Without validation, error would occur at:
# - EPICS IOC (cryptic error)
# - Scan runtime (wasted time)
# - Data analysis (corrupted data)
```

### 3. Explicit State Validation

**Rationale**: State preconditions must be checked explicitly before operations. The Device Layer provides properties to query state; calling code is responsible for validation.

```python
# Check state before operations
if not wire.initialize_status:
    raise RuntimeError(f"Wire {wire.name} not initialized")

if not (wire.speed_min < wire.speed < wire.speed_max):
    raise ValueError(f"Speed {wire.speed} outside hardware limits")

wire.start_scan()
```

### 4. Optional PV Handling

**Rationale**: Different wire scanner models have varying PV availability. Optional PVs enable unified interface across hardware generations.

```python
# Graceful degradation for missing PVs
if wire.controls_information.PVs.temperature is not None:
    temp = wire.temperature
else:
    temp = None  # Handle absence appropriately
```

### 5. Separation of Concerns

**Rationale**: Device Layer focuses exclusively on hardware abstraction. Measurement logic, data processing, and orchestration belong in higher layers.

**Device Layer**:
- Configure scan parameters
- Start/abort scans
- Read device state

**Higher Layers**:
- Data acquisition (Measurement Layer)
- Gaussian fitting (Measurement Layer)
- Multi-wire sequencing (Orchestration Layer)
- GUI updates (GUI Layer)

---

## Common Pitfalls

### 1. Ignoring Initialization State

**Problem**: Calling operations before wire is initialized.

```python
# May fail if not initialized
wire.start_scan()

# Check state first
if not wire.initialize_status:
    wire.initialize()
    # Wait for initialization to complete
wire.start_scan()
```

### 2. Direct PV Access

**Problem**: Bypassing property interface loses validation.

```python
# Bypasses validation
wire.controls_information.PVs.x_wire_inner.put(45000)
wire.controls_information.PVs.x_wire_outer.put(37000)  # Invalid: outer < inner

# Use properties
wire.x_range = [37000, 45000]  # Validates inner < outer
```

### 3. Assuming Synchronous Operations

**Problem**: EPICS PV writes are asynchronous.

```python
# Assumes immediate completion
wire.start_scan()
data = wire.motor_rbv  # May not reflect scan motion yet

# Higher layers handle polling/callbacks
# Device layer provides immediate commands, not completion promises
```

### 4. Ignoring Optional PVs

**Problem**: Accessing `None` PVs raises `AttributeError`.

```python
# May fail if PV not available
temp = wire.controls_information.PVs.temperature.get()

# Check for None first
pv = wire.controls_information.PVs.temperature
if pv is not None:
    temp = pv.get()
else:
    temp = None
```

### 5. Incorrect Speed Assumptions

**Problem**: Manually setting speed when it should be auto-calculated by IOC.

```python
# Generally unnecessary - IOC calculates speed automatically
wire.speed = 10000  # Manual override (rarely needed)

# Typical workflow - let IOC calculate speed
wire.scan_pulses = 350
wire.x_range = [37000, 45000]
# IOC automatically calculates: speed = beam_rate × (range / pulses)
wire.start_scan()  # IOC validates speed is within limits

# Manual calculation only needed for pre-validation
wire_range = wire.x_range[1] - wire.x_range[0]
expected_speed = wire.beam_rate * (wire_range / wire.scan_pulses)
if not (wire.speed_min < expected_speed < wire.speed_max):
    print(f"Warning: Expected speed {expected_speed} outside limits")
```

---

## Key Takeaways

1. **Wire Scanner Abstraction**: The `Wire` class provides a unified, type-safe interface to heterogeneous wire scanner hardware via EPICS PVs.

2. **Safety Through Validation**: Pydantic models prevent invalid configurations from reaching hardware with immediate type and constraint checking.

3. **Property-Based Design**: State-based properties align naturally with EPICS PV semantics.

4. **Optional PV Support**: Graceful handling of hardware variations through optional PVs and fallback logic.

5. **Layer Boundaries**: Device Layer provides hardware control; data acquisition and analysis belong in higher layers.

6. **Configuration Management**: Use factory functions (`create_wire()`) and YAML configuration rather than manual instantiation.

---

## Next Steps

- **[03-measurement-layer.md](03-measurement-layer.md)**: Data collection, Gaussian fitting, and analysis
- **[01-architecture-overview.md](01-architecture-overview.md)**: System-wide architecture context
- **API Reference**: Complete Wire class API documentation (future)
