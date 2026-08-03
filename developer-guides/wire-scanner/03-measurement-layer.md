# Measurement Layer: Wire Scanner Data Collection and Analysis

**Purpose**: Understand the two-stage measurement pipeline — data collection from hardware via timing buffers and post-collection analysis including Gaussian fitting and RMS beam size extraction.

**Audience**: Developers working with wire scan data acquisition, analysis algorithms, or extending scan modes

**Last Updated**: May 28, 2026

---

## Overview

The Measurement Layer sits between Device (Layer 1) and Orchestration (Layer 3), providing a complete pipeline from wire motion through beam size extraction. It is organized as a two-stage architecture:

1. **Collection** — Drives wire motion and acquires synchronized detector data via timing buffers
2. **Analysis** — Organizes raw data by profile, fits curves, and extracts beam sizes

A unified entry point (`WireBeamProfileMeasurement`) composes both stages into a single `measure()` call for typical use. Advanced workflows can invoke collection and analysis independently.

**Location**: `slac-measurements/slac_measurements/wires/`

**Dependencies**:
- `slac_devices` — Wire device abstraction (Layer 1)
- `slac_timing` — Timing buffer creation and management
- `slac_measurements.fitting` — Gaussian and variant curve-fitting modules
- `scipy`, `numpy`, `scikit-image` — Numerical computation and signal processing

---

## Architecture

### Class Hierarchy

```
BeamProfileMeasurement (ABC)
  └── WireBeamProfileMeasurement         [scan.py — unified entry point]

BaseWireMeasurementCollection (ABC)      [collection.py — base collection]
  ├── OTFWireMeasurementCollection       [otf_collection.py — continuous scan]
  └── StepWireMeasurementCollection      [step_collection.py — discrete scan]

BeamProfileAnalysis (ABC)
  └── WireMeasurementAnalysis            [analysis.py — profile fitting]
```

### File Layout

| File | Contents |
|------|----------|
| `scan.py` | `WireBeamProfileMeasurement` — unified measure() composing collection + analysis |
| `collection.py` | `BaseWireMeasurementCollection` (ABC), `create_wire_collection()` factory |
| `otf_collection.py` | `OTFWireMeasurementCollection` — on-the-fly mode |
| `step_collection.py` | `StepWireMeasurementCollection` — step mode |
| `analysis.py` | `WireMeasurementAnalysis` — fitting and RMS extraction |
| `collection_results.py` | `WireMeasurementCollectionResult`, `MeasurementMetadata` |
| `analysis_results.py` | `WireMeasurementAnalysisResult`, `FitResult`, `DetectorFit`, `ProfileMeasurement` |
| `buffer.py` | `reserve_buffer()`, buffer point calculation |
| `__init__.py` | Public API exports |

### Data Flow

```
WireBeamProfileMeasurement.measure(scan_mode, fitting_method, rms_detector)
         │
         ├── create_wire_collection(scan_mode, device, beampath)
         │         │
         │         └── BaseWireMeasurementCollection.measure()
         │                   ├── _reserve_buffer()
         │                   ├── _create_device_dictionary()
         │                   ├── _create_metadata()
         │                   ├── _run_collection_scan()   [mode-specific]
         │                   ├── _get_data_from_buffer()
         │                   └── → WireMeasurementCollectionResult
         │
         └── WireMeasurementAnalysis(collection_result, fitting_method)
                   │
                   └── .analyze(rms_detector)
                             ├── _get_profile_range_indices()
                             ├── _organize_data_by_profile()
                             ├── _fit_data_by_profile()
                             ├── _get_rms_sizes()
                             └── → WireMeasurementAnalysisResult
```

---

## Unified Entry Point

### WireBeamProfileMeasurement (`scan.py`)

The simplest way to run a wire scan — composes collection and analysis into one call.

```python
from slac_measurements.wires import WireBeamProfileMeasurement

measurement = WireBeamProfileMeasurement(
    beam_profile_device=wire,
    beampath="CU_HXR",
)
result = measurement.measure(
    scan_mode="otf",
    fitting_method="gaussian",
    rms_detector=None,  # uses default from wire metadata
)
```

**Parameters for `measure()`**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `scan_mode` | `"otf"` \| `"step"` | `"otf"` | Wire motion mode |
| `fitting_method` | `"gaussian"` \| `"asymmetric_gaussian"` \| `"super_gaussian"` | `"gaussian"` | Curve fitting algorithm |
| `rms_detector` | `str \| None` | `None` | Detector for RMS; defaults to `metadata.default_detector` |

**Internal flow**:
1. Creates appropriate collection class via `create_wire_collection()`
2. Calls `collection.measure()` → `WireMeasurementCollectionResult`
3. Creates `WireMeasurementAnalysis(collection_result=..., fitting_method=...)`
4. Calls `analysis.analyze(rms_detector=...)` → `WireMeasurementAnalysisResult`

The intermediate `collection_result` is also stored on the instance for inspection:
```python
measurement.collection_result  # accessible after measure() completes
```

---

## Stage 1: Data Collection

### BaseWireMeasurementCollection (`collection.py`)

Abstract base class defining the collection contract. Concrete subclasses implement `_run_collection_scan()` for mode-specific wire motion and buffer timing.

**Constructor fields** (Pydantic model):
```python
BaseWireMeasurementCollection(
    beam_profile_device: Wire,   # Wire device instance
    beampath: str,               # Beamline identifier (e.g., "CU_HXR")
)
```

Additional state initialized by `_run_setup()` model validator:
- `logger` — File-based logger writing to `/u1/lcls/physics/data/wire_scan/logs/`
- `detectors` — Detector names extracted from wire metadata
- `buffer` — Timing buffer (reserved during `measure()`)
- `devices` — Dictionary of all device objects (wire + detectors)
- `metadata` — Per-run metadata

### Factory: `create_wire_collection()`

```python
from slac_measurements.wires.collection import create_wire_collection

collection = create_wire_collection(
    scan_mode="otf",              # or "step"
    beam_profile_device=wire,
    beampath="CU_HXR",
)
```

Returns an `OTFWireMeasurementCollection` or `StepWireMeasurementCollection` based on `scan_mode`.

### Collection `measure()` Workflow

```python
result = collection.measure()  # → WireMeasurementCollectionResult
```

1. **Reserve buffer** — `_reserve_buffer()` allocates a timing buffer with calculated point count
2. **Create device dictionary** — `_create_device_dictionary()` instantiates detector objects (LBLM, PMT, TMITLOSS) from wire metadata
3. **Create metadata** — `_create_metadata()` captures wire name, area, beampath, detectors, scan ranges, install angle
4. **Run scan** — `_run_collection_scan()` (abstract, mode-specific)
5. **Get data** — `_get_data_from_buffer()` retrieves position and detector arrays
6. **Release buffer** — always releases in `finally` block

**Error handling**: Buffer is always released even if the scan fails (try/finally pattern).

---

### On-The-Fly Mode: OTFWireMeasurementCollection (`otf_collection.py`)

Wire moves continuously through all profile ranges. Buffer acquires data during motion.

**`_run_collection_scan()` steps**:
1. `_initialize_otf_with_retry(max_attempts=3)`:
   - Calls `wire.start_scan()` to arm OTF motion
   - Waits for `wire.homed` and `wire.on_status` to become True
   - Retries up to 3 times with timeout
   - Raises `RuntimeError` on failure
2. Start timing buffer and poll for completion:
   - Calls `buffer.start()`
   - Polls `buffer.is_complete()` every 0.1s
   - Logs wire position every second
   - Raises `TimeoutError` if acquisition exceeds calculated timeout

**When to use**: Standard beam rates where the wire can traverse all profiles in a single continuous pass.

---

### Step Mode: StepWireMeasurementCollection (`step_collection.py`)

Wire moves to discrete inner/outer positions sequentially. Buffer runs concurrently.

**`_run_collection_scan()` steps**:
1. `_initialize_step_with_retry(max_attempts=3)`:
   - Calls `wire.initialize()` if not already enabled
   - Waits for `wire.enabled` to become True
   - Retries up to 3 times
2. Start timing buffer (`buffer.start()`)
3. Build sorted position list from active profiles (inner/outer for each)
4. Move through each position sequentially:
   - Even indices (inner positions): move at `speed_max`
   - Odd indices (outer positions): calculate speed from `(range / scan_pulses) * beam_rate`
   - Wait for arrival within 250 µm tolerance
5. Retract wire after all positions
6. Wait for buffer to complete (with timeout)

**Speed calculation**:
```python
# Inner positions: fast traverse
speed = wire.speed_max

# Outer positions: data-taking speed
speed = (outer - inner) / wire.scan_pulses * wire.beam_rate
```

**When to use**: Low beam rates or scenarios requiring discrete position measurements.

---

### Timing Buffer Management (`buffer.py`)

**`reserve_buffer(beampath, pulses, beam_rate)`** allocates a timing buffer sized for the scan:

```python
from slac_measurements.wires.buffer import reserve_buffer

buf = reserve_buffer(
    beampath="CU_HXR",
    pulses=350,
    beam_rate=120,
    logger=logger,
)
```

**Buffer point calculation**: Uses a logarithmic fudge factor to account for rate-dependent overhead:
- Low rates (~10 Hz): fudge factor ≈ 1.5
- High rates (~16 kHz): fudge factor ≈ 1.1
- Formula: `points = pulses × 3 × fudge(rate) + rate / 6`

**Constants**:
- `_MAX_BEAM_RATE = 16000` Hz
- `_MIN_BEAM_RATE = 10` Hz
- Buffer name: `"SLAC Tools Wire Scan"`

**Acquisition timeout** (in collection base class):
```python
timeout = max(
    (n_points / beam_rate) * 1.25,      # 25% margin
    (n_points / beam_rate) + 10.0,      # minimum 10s extra
)
```

---

### Device Dictionary

The collection layer auto-discovers detectors from wire metadata and creates device objects:

| Detector Prefix | Device Type | Buffer Method |
|-----------------|-------------|---------------|
| `LBLM` | Beam loss monitor | `"fast_buffer"` |
| `PMT` | Photo-multiplier tube | `"qdcraw_buffer"` |
| `TMITLOSS` | Transmitted intensity loss | Direct `measure()` call |
| *(wire itself)* | Wire position | `"position_buffer"` |

```python
# Internally creates:
devices = {
    "WS28144": wire,          # position data
    "PMT": pmt_device,        # PMT detector
    "LBLM": lblm_device,      # loss monitor
}
```

---

### Collection Result

```python
class WireMeasurementCollectionResult:
    raw_data: dict[str, Any]        # {device_name: numpy array}
    metadata: MeasurementMetadata
```

**`MeasurementMetadata`**:
```python
class MeasurementMetadata:
    wire_name: str
    buffer_number: int | None
    area: str
    beampath: str
    detectors: list[str]
    default_detector: str
    rms_detector: str | None
    scan_ranges: dict[str, tuple[int, int]]  # {"x": (inner, outer), ...}
    timestamp: datetime | None
    active_profiles: list[str]               # ["x", "y"] or ["x", "y", "u"]
    install_angle: float                     # Wire angle in degrees
    notes: str | None
```

**Persistence**: Collection results can be saved/loaded independently:
```python
result.save_to_h5("/path/to/collection.h5")

from slac_measurements.wires import load_collection_from_h5
result = load_collection_from_h5("/path/to/collection.h5")
```

---

## Stage 2: Data Analysis

### WireMeasurementAnalysis (`analysis.py`)

Post-collection analysis: organizes data by profile, fits curves, extracts beam sizes.

```python
from slac_measurements.wires import WireMeasurementAnalysis

analyzer = WireMeasurementAnalysis(
    collection_result=raw_result,
    fitting_method="gaussian",  # or "asymmetric_gaussian", "super_gaussian"
)
result = analyzer.analyze(rms_detector="PMT")
```

**Constructor fields**:
- `collection_result: WireMeasurementCollectionResult` — Raw scan data
- `fitting_method: FittingMethod` — Algorithm choice (default: `"gaussian"`)

---

### Analysis Workflow: `analyze()`

#### Step 1: `_get_profile_range_indices()`

Identifies which buffer indices correspond to each active profile:

1. Retrieves wire position data from `raw_data[wire_name]`
2. Validates position data (min ≠ max)
3. For each active profile (x, y, u):
   - Gets range bounds from `metadata.scan_ranges[profile]`
   - Finds indices where position is within `[inner, outer]`
   - Filters to the **longest monotonically non-decreasing segment** (excludes retraction data)

**Monotonic filtering** ensures only forward-motion data is used for fitting, excluding return passes and overshoot regions.

#### Step 2: `_organize_data_by_profile()`

Separates raw detector arrays into per-profile measurements:

```python
# Result structure:
{
    "x": ProfileMeasurement(
        positions=wire_positions[x_indices],
        detectors={
            "PMT": DetectorProfileMeasurement(values=pmt_data[x_indices], ...),
            "LBLM": DetectorProfileMeasurement(values=lblm_data[x_indices], ...),
        },
        profile_indices=x_indices,
    ),
    "y": ProfileMeasurement(...),
}
```

#### Step 3: `_fit_data_by_profile()`

Fits a curve to each detector signal in each profile:

1. **Coordinate conversion**: Stage positions → beam coordinates using wire install angle
   ```python
   scale = {"x": sin(angle_rad), "y": cos(angle_rad), "u": 1.0}
   x_beam = x_stage * abs(scale[profile])
   ```

2. **Peak windowing** (`_peak_window()`): Extracts signal region around the beam:
   - Applies median filter (size=5) to smooth noise
   - Applies triangle threshold (Otsu-like) to identify signal above background
   - Computes weighted centroid and RMS of thresholded signal
   - Extracts window: `centroid ± 8σ`
   - Fallback: if no signal above threshold, uses peak position with quarter-range width

3. **Curve fitting**: Dynamically imports fitting module based on `fitting_method`:
   ```python
   fitting_module = importlib.import_module(f"slac_measurements.fitting.{self.fitting_method}")
   fp = fitting_module.fit(pos=window_x, data=window_y)
   curve = fitting_module.curve(x=window_x, **fp)
   ```

4. **Result**: `DetectorFit` with `mean` (stage coords), `sigma`, `amplitude`, `offset`, `curve`, `positions` (beam coords)

#### Step 4: `_get_rms_sizes()`

Extracts beam sizes from the selected detector's fit results:

```python
x_rms = fit_result["x"].detectors[detector].sigma  # microns
y_rms = fit_result["y"].detectors[detector].sigma  # microns
return (x_rms, y_rms)
```

Returns `(None, None)` if x or y profiles are not available.

---

### Fitting Methods

Three fitting algorithms are supported, selected via `fitting_method`:

| Method | Module | Use Case |
|--------|--------|----------|
| `"gaussian"` | `slac_measurements.fitting.gaussian` | Standard symmetric beam profiles |
| `"asymmetric_gaussian"` | `slac_measurements.fitting.asymmetric_gaussian` | Profiles with asymmetric tails |
| `"super_gaussian"` | `slac_measurements.fitting.super_gaussian` | Flat-top or non-Gaussian profiles |

All fitting modules expose a common interface:
```python
fp = fitting_module.fit(pos=positions, data=signal)  # Returns dict with mean, sigma, amp, off
curve = fitting_module.curve(x=positions, **fp)      # Returns fitted curve array
```

---

### Analysis Result

```python
class WireMeasurementAnalysisResult(BeamProfileMeasurementResult):
    fit_result: dict[str, FitResult]                    # {profile: FitResult}
    collection_result: WireMeasurementCollectionResult   # Embedded raw data
    profiles: dict[str, ProfileMeasurement]             # Organized profile data

    # Inherited from BeamProfileMeasurementResult:
    rms_sizes: Optional[NDArrayAnnotatedType]           # (x_rms, y_rms)
    centroids: Optional[NDArrayAnnotatedType]
    total_intensities: Optional[NDArrayAnnotatedType]
    signal_to_noise_ratios: Optional[NDArrayAnnotatedType]
    metadata: MeasurementMetadata
```

**Post-hoc detector selection**: Change which detector provides RMS sizes without re-analyzing:
```python
result.set_rms_detector("LBLM")  # Mutates rms_sizes in place
```

**Persistence**:
```python
result.save_to_h5("/path/to/analysis.h5")

from slac_measurements.wires import load_analysis_from_h5
result = load_analysis_from_h5("/path/to/analysis.h5")
```

**HDF5 structure**:
```
/collection_result/metadata/   — MeasurementMetadata attributes
/collection_result/raw_data/   — Raw device arrays
/rms_sizes                     — (x_rms, y_rms) array
/analysis/fit_result/{profile}/{detector}/  — Fit parameters + curve
/analysis/profiles/{profile}/  — Positions, indices, detector values
```

---

## Coordinate Systems

Wire scanners measure beam profiles by sweeping a wire through the beam at an angle. Two coordinate systems are relevant:

**Stage coordinates** (motor units, microns): The raw motor position as the wire moves. Used for scan ranges, position control, and stored in `DetectorFit.mean`.

**Beam coordinates** (microns): Physical transverse position perpendicular to the beam axis. Used for fitting and stored in `DetectorFit.positions` and `DetectorFit.curve`.

**Conversion**:
```python
install_angle_rad = deg2rad(metadata.install_angle)

# Scale factors per profile
scale = {
    "x": sin(install_angle_rad),   # Horizontal
    "y": cos(install_angle_rad),   # Vertical
    "u": 1.0,                      # Longitudinal (no projection)
}

beam_position = stage_position * abs(scale[profile])
```

The `sigma` in `DetectorFit` is in beam coordinates (microns) and directly represents the RMS beam size.

---

## Public API (`__init__.py`)

```python
# Primary entry point
from slac_measurements.wires import WireBeamProfileMeasurement

# Analysis (for re-processing saved data without hardware)
from slac_measurements.wires import FittingMethod, WireMeasurementAnalysis

# Result types (for type hints and loading saved files)
from slac_measurements.wires import (
    DetectorFit,
    DetectorProfileMeasurement,
    FitResult,
    ProfileMeasurement,
    WireMeasurementAnalysisResult,
    MeasurementMetadata,
    WireMeasurementCollectionResult,
    load_analysis_from_h5,
    load_collection_from_h5,
)

# Collection factory (for advanced users)
from slac_measurements.wires import ScanMode, create_wire_collection
```

---

## Common Usage Patterns

### Standard scan (GUI/suite workflow)

```python
from slac_devices.reader import create_wire
from slac_measurements.wires import WireBeamProfileMeasurement

wire = create_wire(area="L3", name="WS28144")
measurement = WireBeamProfileMeasurement(
    beam_profile_device=wire, beampath="CU_HXR"
)
result = measurement.measure(scan_mode="otf")
print(f"RMS: x={result.rms_sizes[0]:.1f} µm, y={result.rms_sizes[1]:.1f} µm")
```

### Re-analyze saved data with different detector or fitting method

```python
from slac_measurements.wires import (
    WireMeasurementAnalysis,
    load_collection_from_h5,
)

raw = load_collection_from_h5("/u1/lcls/physics/data/wire_scan/2026/05/28/OTF_WS28144.h5")
analyzer = WireMeasurementAnalysis(
    collection_result=raw,
    fitting_method="asymmetric_gaussian",
)
result = analyzer.analyze(rms_detector="LBLM")
```

### Collection-only (no analysis)

```python
from slac_measurements.wires.collection import create_wire_collection

collection = create_wire_collection(
    scan_mode="step",
    beam_profile_device=wire,
    beampath="CU_HXR",
)
raw_result = collection.measure()
raw_result.save_to_h5("/path/to/raw.h5")
```

---

## Error Handling

### Collection Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| `RuntimeError` | Wire failed to initialize/home after 3 attempts | Check hardware, try `wire.initialize()` manually |
| `TimeoutError` | Buffer acquisition did not complete within expected time | Check beam delivery, verify beam_rate is correct |
| `FileNotFoundError` | Log directory `/u1/lcls/physics/data/wire_scan/logs/` missing | Create directory or run from appropriate host |
| `BufferError` | Cannot determine username for buffer reservation | Check system authentication |

### Analysis Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| `RuntimeError("Min and max position are the same")` | Wire did not move during scan | Verify collection succeeded |
| `RuntimeError("Scan did not reach expected profile range")` | Wire didn't traverse far enough | Check scan ranges vs actual motion |
| `ValueError("Detector not available")` | Requested `rms_detector` not in metadata | Use a detector from `metadata.detectors` |
| `UserWarning("No signal above threshold")` | Beam may not be present | Check beam delivery; peak window falls back to simple peak finding |

**Buffer release guarantee**: The collection `measure()` method uses try/finally to ensure the timing buffer is always released, even on scan failure.

---

## Design Principles

### 1. Two-Stage Pipeline

Collection and analysis are separated so that:
- Raw data can be saved and re-analyzed with different parameters
- Analysis algorithms can evolve without touching hardware code
- Failed analyses don't require re-scanning

### 2. Factory Pattern for Scan Modes

`create_wire_collection()` encapsulates mode selection, allowing new scan modes to be added as subclasses without modifying calling code.

### 3. Pluggable Fitting

Fitting methods are loaded dynamically via `importlib`, allowing new algorithms to be added as modules in `slac_measurements.fitting/` without modifying analysis code.

### 4. Immutable Results

Result objects (`WireMeasurementCollectionResult`, `WireMeasurementAnalysisResult`) are Pydantic models. Once created, they represent a complete snapshot of the measurement that can be persisted and loaded.

### 5. Monotonic Filtering

Only the longest forward-motion segment is used for fitting, making the analysis robust to:
- Wire retraction data captured in the buffer
- Position overshoot at range endpoints
- Multi-pass scans where only one direction is meaningful

---

## Key Takeaways

1. **Unified entry point**: `WireBeamProfileMeasurement.measure()` handles collection + analysis in one call
2. **Two scan modes**: OTF (continuous) and Step (discrete), selected by `scan_mode` parameter
3. **Separation of collection and analysis**: Raw data persists independently; analysis can be re-run
4. **Three fitting methods**: gaussian, asymmetric_gaussian, super_gaussian — pluggable via string parameter
5. **Automatic coordinate conversion**: Stage → beam coordinates using wire install angle
6. **Buffer is always released**: try/finally ensures no leaked timing buffers on failure

---

## Related Documentation

- [Architecture Overview](01-architecture-overview.md) — System-wide context
- [Device Layer](02-device-layer.md) — Wire hardware control (Layer 1)
- [Suite Layer Deep Dive](04-suite-layer.md) — Orchestration and run tracking *(coming soon)*
