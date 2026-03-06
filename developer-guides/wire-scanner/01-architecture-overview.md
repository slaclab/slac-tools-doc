# Wire Scanner System Architecture Overview

**Purpose**: Understand the layered architecture of the Wire Scanner system from low-level EPICS device control through high-level orchestration and GUI presentation.

**Audience**: Developers extending or maintaining wire scanner software

**Last Updated**: March 6, 2026

---

## Architectural Layers

The Wire Scanner system is organized in four distinct layers, each with clear responsibilities and well-defined interfaces:

```
┌─────────────────────────────────────────────────────────────┐
│                      GUI Layer (Level 4)                    │
│                      ws_gui.py (slacwire)                   │
│        PyQt5/PyDM interface, threading, plotting, eLog      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Orchestration Layer (Level 3)              │
│                   ws_suite.py (slacwire)                    │
│      WireScanSuite: run tracking, data mgmt, automation     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Measurement Layer (Level 2)                │
│         ws_collection.py, ws_analysis.py (lcls-tools)       │
│    Data collection, Gaussian fitting, RMS extraction        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     Device Layer (Level 1)                  │
│                  wire.py (lcls-tools)                       │
│         EPICS PV control, motion management, state          │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Device Layer

### Location
`lcls_tools/common/devices/wire.py`

### Responsibility
Low-level control of wire scanner hardware via EPICS Process Variables (PVs). Provides abstractions for:
- Motor position control and readback
- Scan parameter configuration (speed, pulse count, ranges)
- State management (initialization, homing, enabled status)
- Safety interlocks (speed limits, range validation)

### Key Classes

**`WirePVSet`**
- Maps EPICS PV names to logical controls
- Motor control: `motor`, `motor_rbv`, `start_scan`, `abort_scan`, `retract`
- Configuration: `speed`, `scan_pulses`, `beam_rate`
- Per-plane settings: `{x,y,u}_wire_inner`, `{x,y,u}_wire_outer`, `use_{x,y,u}_wire`
- State monitoring: `initialize_status`, `enabled`, `homed`

**`Wire`**
- High-level device interface built on `WirePVSet`
- Methods: `initialize()`, `start_scan()`, `abort_scan()`, `retract()`
- Properties: `speed`, `enabled`, `homed`, `initialize_status`
- Configuration: `set_range(plane, [inner, outer])`, `use(plane, bool)`

### Example Usage

```python
from lcls_tools.common.devices.reader import create_wire

# Create wire device
wire = create_wire(area="LI28", name="WS28144")

# Configure scan
wire.use_x_wire = True
wire.x_range = [37000, 45000]  # [inner, outer] in motor units (microns)
wire.speed = 10000  # µm/s

# Initialize and check state
wire.initialize()
while not wire.initialize_status:
    time.sleep(0.1)

# Ready to scan at higher layers
```

### Interface to Layer 2
Exposes:
- Device configuration (ranges, speeds, active planes)
- Motion control commands
- State queries
- Metadata (detectors, BPMs, install angle)

---

## Layer 2: Measurement Layer

### Location
- `lcls_tools/common/measurements/ws_collection.py`
- `lcls_tools/common/measurements/ws_analysis.py`

### Responsibility
Orchestrates wire scanning and extracts beam profile information. Two sub-layers:

#### 2a. Data Collection (`ws_collection.py`)
- Synchronizes wire motion with detector data acquisition
- Manages BSA/EDEF buffer reservations
- Returns raw position and detector arrays
- Supports two scan modes: **on-the-fly** and **step**

#### 2b. Data Analysis (`ws_analysis.py`)
- Organizes raw data by profile (X, Y, U planes)
- Fits Gaussian curves to detector signals
- Extracts RMS beam sizes
- Transforms coordinates (stage → beam)

### Key Classes

**`WireMeasurementCollection`** (data collection)
- Constructor: `WireMeasurementCollection(beam_profile_device=wire, beampath="CU_HXR")`
- Main method: `measure(scan_type="step") → WireMeasurementCollectionResult`
- Workflow:
  1. Reserve BSA buffer
  2. Execute wire motion (OTF or step)
  3. Synchronize data acquisition
  4. Extract position + detector arrays
  5. Release buffer
  6. Return raw data + metadata

**`WireMeasurementAnalysis`** (post-measurement)
- Constructor: `WireMeasurementAnalysis(collection_result=raw_result)`
- Main method: `analyze() → WireMeasurementAnalysisResult`
- Workflow:
  1. `get_profile_range_indices()` - Identify data for each profile
  2. `organize_data_by_profile()` - Separate by X/Y/U
  3. `fit_data_by_profile()` - Gaussian fitting per detector
  4. `get_rms_sizes()` - Extract (x_rms, y_rms)

### Data Structures

**Collection Results** (raw data):
```python
WireMeasurementCollectionResult:
    raw_data: dict          # {device_name: numpy array}
    metadata: MeasurementMetadata  # Wire name, detectors, ranges, etc.
```

**Analysis Results** (fitted data):
```python
WireMeasurementAnalysisResult:
    rms_sizes: tuple        # (x_rms, y_rms) in microns
    fit_result: dict        # {profile: FitResult}
    profiles: dict          # {profile: ProfileMeasurement}
    collection_result: WireMeasurementCollectionResult  # Embedded raw
```

### Scan Type Selection Logic

**On-The-Fly (OTF)**:
- Wire moves continuously
- Buffer acquires during motion
- Optimal: 120 Hz < beam_rate ≤ 16 kHz

**Step Scan**:
- Wire moves to discrete positions
- Buffer acquires during motion
- Required: beam_rate ≤ 120 Hz

### Example Usage

```python
from lcls_tools.common.measurements.ws_collection import WireMeasurementCollection
from lcls_tools.common.measurements.ws_analysis import WireMeasurementAnalysis

# Collection
collection = WireMeasurementCollection(
    beam_profile_device=wire,
    beampath="CU_HXR"
)
raw_result = collection.measure(scan_type="step")

# Analysis
analyzer = WireMeasurementAnalysis(collection_result=raw_result)
analysis_result = analyzer.analyze()

# Access results
x_rms, y_rms = analysis_result.rms_sizes  # Beam sizes in microns
print(f"X: {x_rms:.1f} µm, Y: {y_rms:.1f} µm")

# Access fits
x_fits = analysis_result.fit_result['x']  # FitResult for X profile
for detector, fit in x_fits.detectors.items():
    print(f"{detector}: mean={fit.mean}, sigma={fit.sigma}")
```

### Interface to Layer 3
Exposes:
- `measure()` method → results with raw/analyzed data
- Result objects serializable to HDF5
- Metadata for tracking and plotting
- RMS sizes, fit parameters, profiles

---

## Layer 3: Orchestration Layer

### Location
`slacwire/ws_suite.py`

### Responsibility
Provides programmatic API for batch/automated scanning with:
- Run tracking and versioning
- Automated data persistence (HDF5)
- Plot generation and saving
- Scan workflow management
- Result caching and retrieval

### Key Class

**`WireScanSuite`** (dataclass)
- Constructor: `WireScanSuite(wires=["WS28144:L3"], beampath="CU_HXR")`
- Main method: `run(do_otf=False, do_step=True, save=True, save_plots=True)`

### Internal Architecture

**State Management**:
```python
run_counter: int                    # Incremental run ID
run_registry: List[dict]            # Metadata per run
results: Dict[str, List[Result]]    # Keyed by wire name
devices: Dict[str, Wire]            # Cached device objects
```

**Workflow Methods**:
- `_run_otf()` - Execute single OTF scan
- `_run_step()` - Execute single step scan
- `_latest_run(wire_name)` - Get most recent result for wire

**Persistence Methods**:
- `save_run(data, filename)` - HDF5 serialization
- `save_fig(fig, name)` - PNG export
- Uses dated directory structure: `/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/`

**Plotting Methods**:
- `plot_trajectory(result)` - Position vs time
- `plot_profile(result, detector, profile)` - Beam profile with Gaussian fit

### Example Usage

```python
from slacwire.ws_suite import WireScanSuite

# Single wire
suite = WireScanSuite(
    wires=["WS28144:L3"],
    beampath="CU_HXR"
)
suite.run(do_otf=True, save=True, save_plots=True)

# Access results
latest = suite._latest_run("WS28144")  # Most recent
x_rms, y_rms = latest.rms_sizes  # Beam sizes in microns
```

### Run Registry

Each `run()` invocation creates a registry entry:
```python
{
    'run_id': 1,
    'timestamp': '2026-03-06 14:30:00',
    'method': 'step',
    'wire': 'WS28144',
    'filepath': '/u1/.../Step_WS28144_20260306_143000.h5',
    'plots': ['/u1/.../Step_Profile_x_WS28144_20260306_143000.png', ...],
    'status': 'ok'
}
```

### Interface to Layer 4
Exposes:
- High-level `run()` method with multiple scan modes
- Result caching for GUI display
- Plot generation for GUI integration
- Registry for run history tracking

---

## Layer 4: GUI Layer

### Location
`slacwire/ws_gui.py`

### Responsibility
Provides operator interface with:
- Beampath/area/wire navigation
- Real-time parameter controls with PyDM (EPICS integration)
- Threaded scanning to prevent UI blocking
- Live plotting with matplotlib
- Physics eLog submission
- Text logging and status updates

### Key Components

**Main Window** (`WireScanSuiteGUI`)
- Organizes widget layout
- Connects signals between widgets
- Manages application lifecycle

**Widget Organization**:
- `NavigationWidget` - Beampath/area/wire selection
- `MeasurementWidget` - Scan controls, start/stop buttons
- `PlotWidget` - Real-time matplotlib canvas
- `TextLoggerWidget` - Status messages and errors

**Threading** (`WireScanSuiteThread`)
- Inherits `QThread`
- Runs measurement collection in background
- Signals:
  - `scan_complete(wire_name, method, data, entry)` - Success
  - `scan_failed(wire_identifier, error_message)` - Error

### Automatic Scan Type Selection

GUI automatically chooses scan type based on beam rate:
```python
if beam_rate <= 120:
    scan_type = "step"
elif 120 < beam_rate <= 16000:
    scan_type = "on_the_fly"
else:
    raise ValueError("Beam rate too high")
```

### Signal Flow Example

```
User selects wire in NavigationWidget
         ↓
NavigationWidget.wireChanged signal
         ↓
MeasurementWidget.update_parameters()
         ↓
PyDM channels connected to EPICS PVs
         ↓
User clicks "Start Scan"
         ↓
WireScanSuiteThread created and started
         ↓
Thread runs WireMeasurementCollection internally
         ↓
scan_complete signal emitted
         ↓
PlotWidget updated with new data
         ↓
TextLoggerWidget shows "Scan complete"
```

### Integration with Layer 3

GUI uses `WireScanSuite` for:
- Automated run tracking
- Consistent file naming
- Plot generation
- eLog submission preparation

## Data Flow Through All Layers

### Complete Single Wire Scan

```
Layer 1 (Device):
  Wire device initialized, ranges set, wire profiles selected
         ↓
Layer 2a (Collection):
  WireMeasurementCollection.measure()
    - Reserve buffer
    -- wire.start_scan() for on-the-fly scans [call to Layer 1]
    -- wire.motor setter for step scans [call to Layer 1]
    - Synchronize acquisition
    - Extract raw data
    → WireMeasurementCollectionResult
         ↓
Layer 2b (Analysis):
  WireMeasurementAnalysis.analyze()
    - Organize by profile
    - Fit Gaussians
    - Extract RMS
    → WireMeasurementAnalysisResult
         ↓
Layer 3 (Orchestration):
  WireScanSuite.run()
    - Call Layer 2
    - Save to HDF5
    - Generate plots
    - Update run registry
    → Persisted data + cached results
         ↓
Layer 4 (GUI):
  Display results
    - Update plot canvas
    - Show RMS values
    - Log to text widget
    - Optional eLog submission
```

## Shared System Concerns

### Error Handling

**Layer 1**: Decorator guards return `None` on validation failure
```python
@check_state
def start_scan(self):
    # Only runs if initialize_status == True
```

**Layer 2**: Raises explicit exceptions
```python
if not self._validate_position_data(positions):
    raise RuntimeError("Position data invalid")
```

**Layer 3**: Catches and logs, continues with other wires
```python
try:
    result = collection.measure()
except Exception as e:
    self.logger.error(f"Scan failed: {e}")
    continue  # Process other wires
```

**Layer 4**: Displays user-friendly messages
```python
except Exception as e:
    self.scan_failed.emit(wire_identifier, str(e))
    # TextLoggerWidget shows: "Scan failed: Check wire initialization"
```

### Thread Safety

**Layers 1-3**: Not thread-safe (designed for sequential use)

**Layer 4**: Qt signals/slots ensure thread-safe GUI updates from worker threads

---

## Design Principles

### Separation of Concerns
Each layer has a single, well-defined responsibility and communicates through explicit interfaces.

### Dependency Direction
Dependencies flow downward only:
- Layer 4 → Layer 3 → Layer 2 → Layer 1
- No upward dependencies
- Lower layers have no knowledge of higher layers


## Key Takeaways

1. **Wire Scanner is a 4-layer system** - Device → Measurement → Orchestration → GUI
2. **Each layer is independently testable** 
3. **Data flows downward through calls, upward through return values** - No callbacks
4. **Results are immutable** - Once created, never modified (enables caching)

---

## Related Documentation

- [Device Layer Deep Dive](02-device-layer.md) *(coming soon)*
- [Measurement Layer Deep Dive](03-measurement-layer.md) *(coming soon)*
- [Suite Layer Deep Dive](04-suite-layer.md) *(coming soon)*
- [GUI Development Guide](05-gui-layer.md) *(coming soon)*
- [Testing Strategy](../08-testing-strategy.md) *(coming soon)*
