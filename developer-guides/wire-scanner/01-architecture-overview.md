# Wire Scanner System Architecture Overview

**Purpose**: Understand the layered architecture of the Wire Scanner system from low-level EPICS device control through high-level orchestration and GUI presentation.

**Audience**: Developers extending or maintaining wire scanner software

**Last Updated**: June 18, 2026

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
│              suite/, view.py (slacwire)                      │
│      WireScanSuite: run tracking, data mgmt, automation     │
│      WireScanView: plotting and figure management           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Measurement Layer (Level 2)                │
│        collection.py, analysis.py (slac-measurements)       │
│  otf_collection.py, step_collection.py, scan.py            │
│    Data collection, Gaussian fitting, RMS extraction        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     Device Layer (Level 1)                  │
│                  wire.py (slac-devices)                     │
│         EPICS PV control, motion management, state          │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Device Layer

### Location
`slac-devices/slac_devices/wire.py`

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
from slac_devices.reader import create_wire

# Create wire device
wire = create_wire(area="L3", name="WS28144")

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
- `slac-measurements/slac_measurements/wires/scan.py` (unified entry point)
- `slac-measurements/slac_measurements/wires/collection.py` (base collection + factory)
- `slac-measurements/slac_measurements/wires/otf_collection.py` (on-the-fly scan)
- `slac-measurements/slac_measurements/wires/step_collection.py` (step scan)
- `slac-measurements/slac_measurements/wires/analysis.py` (profile fitting and RMS extraction)
- `slac-measurements/slac_measurements/wires/collection_results.py` (collection result types)
- `slac-measurements/slac_measurements/wires/analysis_results.py` (analysis result types)

### Responsibility
Orchestrates wire scanning and extracts beam profile information via a two-stage pipeline:

#### 2a. Data Collection (`collection.py`, `otf_collection.py`, `step_collection.py`)
- Synchronizes wire motion with detector data acquisition
- Manages timing buffer reservations
- Returns raw position and detector arrays
- Supports two scan modes: **on-the-fly** and **step**

#### 2b. Data Analysis (`analysis.py`)
- Organizes raw data by profile (X, Y, U planes)
- Fits Gaussian curves (standard, asymmetric, or super-Gaussian)
- Extracts RMS beam sizes
- Transforms coordinates (stage → beam)

### Key Classes

**`WireBeamProfileMeasurement`** (unified entry point in `scan.py`)
- Constructor: `WireBeamProfileMeasurement(beam_profile_device=wire, beampath="CU_HXR")`
- Main method: `measure(scan_mode="otf", fitting_method="gaussian", rms_detector=None) → WireMeasurementAnalysisResult`
- Composes collection + analysis in a single call

**`BaseWireMeasurementCollection`** (abstract base in `collection.py`)
- Constructor: created via `create_wire_collection(scan_mode, beam_profile_device, beampath)`
- Main method: `measure() → WireMeasurementCollectionResult`
- Workflow:
  1. Reserve timing buffer
  2. Create device dictionary (wire + detectors)
  3. Create metadata
  4. `_run_collection_scan()` - Mode-specific wire motion + buffer acquisition
  5. `_get_data_from_buffer()` - Extract position + detector arrays
  6. Release buffer
  7. Return raw data + metadata

**`OTFWireMeasurementCollection`** (on-the-fly scan)
- Inherits from `BaseWireMeasurementCollection`
- Starts wire scan via `start_scan()`, then acquires buffer while wire moves continuously

**`StepWireMeasurementCollection`** (step scan)
- Inherits from `BaseWireMeasurementCollection`
- Initializes wire, starts buffer, moves to discrete positions sequentially, retracts, waits for buffer completion

**`WireMeasurementAnalysis`** (post-collection analysis)
- Constructor: `WireMeasurementAnalysis(collection_result=raw_result, fitting_method="gaussian")`
- Main method: `analyze(rms_detector=None) → WireMeasurementAnalysisResult`
- Workflow:
  1. `_get_profile_range_indices()` - Identify data for each profile
  2. `_organize_data_by_profile()` - Separate by X/Y/U
  3. `_fit_data_by_profile()` - Gaussian fitting per detector
  4. `_get_rms_sizes()` - Extract (x_rms, y_rms)

### Data Structures

**Collection Result** (raw data):
```python
WireMeasurementCollectionResult:
    raw_data: dict[str, Any]        # {device_name: numpy array}
    metadata: MeasurementMetadata   # Wire name, detectors, ranges, etc.
```

**Analysis Result** (fitted data):
```python
WireMeasurementAnalysisResult:
    fit_result: dict[str, FitResult]                    # {profile: FitResult}
    rms_sizes: Optional[NDArrayAnnotatedType]           # (x_rms, y_rms) in microns
    collection_result: WireMeasurementCollectionResult   # Embedded raw data
    profiles: dict[str, ProfileMeasurement]             # Organized profile data
    metadata: MeasurementMetadata
```

**Supporting Types**:
```python
FitResult:
    detectors: dict[str, DetectorFit]  # {detector_name: fit parameters}

DetectorFit:
    mean: float        # Centroid position
    sigma: float       # RMS beam size (microns)
    amplitude: float   # Peak amplitude
    offset: float      # Baseline offset
    curve: ndarray     # Fitted Gaussian curve
    positions: ndarray # Position array

ProfileMeasurement:
    positions: ndarray                              # Wire positions for this profile
    detectors: dict[str, DetectorProfileMeasurement]  # Detector data arrays
    profile_indices: ndarray                        # Index array within full scan
```

### Scan Mode Selection

Scan mode is selected via the `scan_mode` parameter, which dispatches to the appropriate collection class via `create_wire_collection()`:

| `scan_mode` | Collection Class | Motion | Use Case |
|-------------|-----------------|--------|----------|
| `"otf"` | `OTFWireMeasurementCollection` | Continuous | Standard beam rates |
| `"step"` | `StepWireMeasurementCollection` | Step-and-settle | Low-rate / discrete scans |

### Example Usage

```python
from slac_measurements.wires import WireBeamProfileMeasurement

# Unified measurement (collection + analysis)
measurement = WireBeamProfileMeasurement(
    beam_profile_device=wire,
    beampath="CU_HXR"
)
result = measurement.measure(scan_mode="otf")

# Access results
x_rms, y_rms = result.rms_sizes  # Beam sizes in microns
print(f"X: {x_rms:.1f} µm, Y: {y_rms:.1f} µm")

# Access fits
x_fits = result.fit_result['x']  # FitResult for X profile
for detector, fit in x_fits.detectors.items():
    print(f"{detector}: mean={fit.mean}, sigma={fit.sigma}")

# Save to HDF5
result.save_to_h5("/path/to/output.h5")

# --- Or use collection + analysis separately ---
from slac_measurements.wires.collection import create_wire_collection
from slac_measurements.wires.analysis import WireMeasurementAnalysis

collection = create_wire_collection(
    scan_mode="step",
    beam_profile_device=wire,
    beampath="CU_HXR",
)
raw_result = collection.measure()

analyzer = WireMeasurementAnalysis(
    collection_result=raw_result,
    fitting_method="gaussian",
)
analysis_result = analyzer.analyze(rms_detector="PMT")
```

### Interface to Layer 3
Exposes:
- `WireBeamProfileMeasurement.measure()` → unified result
- Separate `create_wire_collection()` + `WireMeasurementAnalysis` for advanced workflows
- Result objects serializable to HDF5 via `save_to_h5()`
- Metadata for tracking and plotting
- RMS sizes, fit parameters, profiles

---

## Layer 3: Orchestration Layer

### Location
- `slacwire/suite/` (scan orchestration — mixin-based sub-package)
- `slacwire/view.py` (plotting and figure management)
- `slacwire/registry/registry.py` (run tracking)

### Responsibility
Provides programmatic API for batch/automated scanning with:
- Run tracking and versioning
- Automated data persistence (HDF5)
- Plot generation and saving (delegated to `WireScanView`)
- Scan workflow management
- Result caching and retrieval
- Beam-less motion validation tests
- EPICS CA cache diagnostics

### Key Classes

**`WireScanSuite`** (dataclass, composed from mixins)
- Constructor: `WireScanSuite(wires=["WS28144"], beampath="CU_HXR")`
- Main methods: `run_single(wire, scan_mode="otf")`, `run_all(scan_mode="otf")`
- Composed from: `WireScanSuiteBase`, `RunMixin`, `CollectMixin`, `ResultsMixin`, `MotionTestMixin`, `DiagnosticsMixin`

**`WireScanView`** (plotting)
- Provides `render()`, `draw_trajectory()`, `draw_profile()`, `plot_trajectory()`, `plot_profile()`
- Handles both standalone figure creation and in-place GUI canvas rendering

**`RunRegistry`** (dataclass)
- Persistent JSON-backed run log
- Provides `log()` method to record each scan

### Suite Sub-Package Structure

```
suite/
├── __init__.py        # Composes WireScanSuite from all mixins
├── _base.py           # Dataclass fields, __post_init__, device management
├── _run.py            # run_single, run_all (collection + analysis)
├── _collect.py        # collect_single (raw data, no analysis)
├── _motion.py         # motion_test (beam-less validation)
├── _results.py        # latest_run, replot, summary
├── _diagnostics.py    # cache_info, cache_pvs, cache_summary
└── _constants.py      # WIRE_AREA_LOOKUP, Beampath, paths
```

### Internal Architecture

**State Management** (in `_base.py`):
```python
results: dict                       # {wire_name: List[Result]}
devices: dict                       # {wire_name: Wire} cached device objects
registry: RunRegistry               # Persistent run tracking
view: WireScanView                  # Plotting (created in __post_init__)
```

**Workflow Methods**:
- `run_single(wire, scan_mode="otf", rms_detector=None)` - Execute single wire scan (`_run.py`)
- `run_all(scan_mode="otf", rms_detector=None)` - Run all configured wires (`_run.py`)
- `collect_single(wire, scan_mode="otf")` - Collect raw data without analysis (`_collect.py`)
- `motion_test(wire, scan_mode="otf", plot=True)` - Beam-less motion validation (`_motion.py`)
- `latest_run(wire)`, `replot(wire)`, `summary()` - Result access (`_results.py`)
- `cache_info()`, `cache_pvs()`, `cache_summary()` - EPICS diagnostics (`_diagnostics.py`)

**Persistence**:
- Uses dated directory structure: `/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/`
- HDF5 serialization via `result.save_to_h5()`
- Plot export via `view.render()`

**Plotting** (via `self.view`):
- `view.render(data, wire, detector, profiles, ...)` - Generate trajectory + profile plots
- `view.draw_trajectory(fig, data, wire, detector)` - Render into existing figure (GUI)
- `view.draw_profile(fig, data, wire, detector, profile)` - Render into existing figure (GUI)

### Example Usage

```python
from slacwire import WireScanSuite

# Single wire
suite = WireScanSuite(
    wires=["WS28144"],
    beampath="CU_HXR"
)
suite.run_single("WS28144", scan_mode="otf")

# Access results
latest = suite.latest_run("WS28144")  # Most recent
x_rms, y_rms = latest.rms_sizes  # Beam sizes in microns

# Run all configured wires
suite.run_all(scan_mode="otf")

# Regenerate plots without re-scanning
suite.replot("WS28144")

# Print summary of all latest results
suite.summary()
```

### Run Registry

Each scan invocation creates a registry entry (persisted to JSON):
```python
{
    'run_id': 1,
    'timestamp': '20260306_143000',
    'method': 'otf',
    'wire': 'WS28144',
    'beampath': 'CU_HXR',
    'detector': 'PMT:LI29:150',
    'filepath': '/u1/.../OTF_WS28144_20260306_143000.h5',
    'scope_data': '/u1/.../scope_WS28144_20260306_143000.csv',
    'plots': ['/u1/.../OTF_Profile_x_WS28144_20260306_143000.png', ...],
    'status': 'ok',
    'error': None
}
```

### Interface to Layer 4
Exposes:
- `run_single()` / `run_all()` methods with scan mode selection
- Result caching for GUI display via `latest_run()`
- In-place plot rendering for GUI canvases via `view.draw_trajectory()` / `view.draw_profile()`
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

### Scan Mode

The GUI currently uses a fixed scan mode (`"otf"`) passed to `WireScanSuite.run_single()`. Scan mode selection is not yet automated based on beam rate.

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
Thread runs WireScanSuite.run_single() internally
         ↓
scan_complete signal emitted (wire_name, method, data, entry)
         ↓
PlotWidget updated with new data
         ↓
TextLoggerWidget shows "Scan complete"
```

### Integration with Layer 3

GUI uses `WireScanSuite` for:
- Automated run tracking via `run_single()`
- Consistent file naming and dated output directories
- In-place plot rendering via `view.draw_trajectory()` / `view.draw_profile()`
- Registry-based run history
- eLog submission preparation

## Data Flow Through All Layers

### Complete Single Wire Scan

```
Layer 1 (Device):
  Wire device initialized, ranges set, wire profiles selected
         ↓
Layer 2a (Collection):
  create_wire_collection(scan_mode, ...).measure()
    - Reserve timing buffer
    - _run_collection_scan() → wire.start_scan() [call to Layer 1]
    - _get_data_from_buffer() → extract raw data
    - Release buffer
    → WireMeasurementCollectionResult
         ↓
Layer 2b (Analysis):
  WireMeasurementAnalysis(collection_result=...).analyze()
    - _get_profile_range_indices() → identify profiles
    - _organize_data_by_profile() → separate by X/Y/U
    - _fit_data_by_profile() → Gaussian fitting
    - _get_rms_sizes() → extract (x_rms, y_rms)
    → WireMeasurementAnalysisResult
         ↓
Layer 3 (Orchestration):
  WireScanSuite.run_single()
    - _run_device_scan() calls Layer 2 via WireBeamProfileMeasurement
    - Save to HDF5 via result.save_to_h5()
    - Generate plots via view.render()
    - Update run registry via registry.log()
    → Persisted data + cached results
         ↓
Layer 4 (GUI):
  Display results
    - Update plot canvas via view.draw_trajectory() / draw_profile()
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
if not self._wait_until(lambda: wire.initialize_status, timeout=10):
    raise RuntimeError("Wire initialization failed")
```

**Layer 3**: Catches and logs, records error in registry
```python
try:
    result = measurement.measure()
except Exception as e:
    self.registry.log(..., status="error", error=str(e))
    logger.error(f"Scan failed: {e}")
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

- [Device Layer Deep Dive](02-device-layer.md)
- [Measurement Layer Deep Dive](03-measurement-layer.md)
- [Suite Layer Deep Dive](04-suite-layer.md)
- [GUI Development Guide](05-gui-layer.md)
