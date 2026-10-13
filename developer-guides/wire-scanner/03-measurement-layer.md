# Measurement Layer: Wire Scanner Data Collection and Analysis

**Purpose**: Understand the two-stage measurement pipeline — data collection from hardware via timing buffers and post-collection analysis including Gaussian fitting and RMS beam size extraction.

**Audience**: Developers working with wire scan data acquisition, analysis algorithms, or extending scan modes

**Last Updated**: October 13, 2026

---

## Overview

The Measurement Layer sits between Device (Layer 1) and Orchestration (Layer 3), providing a complete pipeline from wire motion through beam size extraction. It is organized as a two-stage architecture:

1. **Collection** — Drives wire motion and acquires synchronized detector data via timing buffers
2. **Analysis** — Organizes raw data by profile, fits curves, and extracts beam sizes

A unified entry point (`WireBeamProfileMeasurement`) composes both stages into a single `measure()` call for typical use. Advanced workflows can invoke collection and analysis independently.

The layer also provides:
- **Jitter correction** — BPM-based orbit-fit correction of pulse-to-pulse beam jitter
- **Charge normalization** — Toroid-based pulse charge normalization
- **Beamless scan modes** — Buffer-less motion-only scans for commissioning and mechanical validation
- **MATLAB export** — `.mat` file output compatible with the legacy MATLAB `wirescan_gui`

**Location**: `slac-measurements/slac_measurements/wires/`

**Dependencies**:
- `slac_devices` — Wire device abstraction (Layer 1)
- `slac_timing` — Timing buffer creation and management
- `slac_measurements.fitting` — Gaussian and variant curve-fitting modules
- `meme` — R-matrix retrieval for jitter correction
- `scipy`, `numpy`, `scikit-image` — Numerical computation and signal processing

---

## Architecture

### Class Hierarchy

```
BeamProfileMeasurement (ABC)
  └── WireBeamProfileMeasurement         [scan.py — unified entry point]

BaseWireMeasurementCollection (ABC)      [collection/base.py — base collection]
  ├── OTFWireMeasurementCollection       [collection/otf.py — continuous scan]
  └── StepWireMeasurementCollection      [collection/step.py — discrete scan]

BeamProfileAnalysis (ABC)
  └── WireMeasurementAnalysis            [analysis/analysis.py — profile fitting]
```

### File Layout

The measurement layer is organized into three subpackages plus supporting modules:

| File | Contents |
|------|----------|
| **Top-level** | |
| `scan.py` | `WireBeamProfileMeasurement` — unified measure() composing collection + analysis |
| `detector_util.py` | `DetectorConfig`, `resolve_detectors()`, beampath-keyed detector resolution |
| **`collection/`** | |
| `collection/base.py` | `BaseWireMeasurementCollection` (ABC), `create_wire_collection()` factory |
| `collection/otf.py` | `OTFWireMeasurementCollection` — on-the-fly mode |
| `collection/step.py` | `StepWireMeasurementCollection` — step mode |
| `collection/buffer.py` | `reserve_buffer()`, buffer point calculation |
| `collection/results.py` | `WireMeasurementCollectionResult`, `MeasurementMetadata` |
| `collection/beamless_otf.py` | `run_beamless_otf_scan()` — buffer-less OTF motion |
| `collection/beamless_step.py` | `run_beamless_step_scan()` — buffer-less step motion |
| **`analysis/`** | |
| `analysis/analysis.py` | `WireMeasurementAnalysis` — fitting and RMS extraction |
| `analysis/results.py` | `WireMeasurementAnalysisResult`, `FitResult`, `DetectorFit`, `ProfileMeasurement` |
| `analysis/jitter_correction.py` | `compute_jitter()` — BPM orbit-fit jitter correction |
| `analysis/charge_normalization.py` | `compute_charge_normalization()` — toroid charge normalization |
| `analysis/coordinates.py` | `stage_to_beam()`, `beam_to_stage()` — coordinate transforms |
| `analysis/mat_export.py` | `analysis_result_to_mat()` — MATLAB `.mat` export |
| **`motion/`** | |
| `motion/otf.py` | `initialize_otf_with_retry()` — OTF initialization with retry |
| `motion/step.py` | `initialize_step_with_retry()`, `get_step_positions()`, `move_to_step_position()` |
| `motion/utils.py` | `poll_motor_rbv()` — motor position polling with settle detection |

### Data Flow

```
WireBeamProfileMeasurement.measure(scan_mode, fitting_method, rms_detector,
                                    jitter_correction, charge_normalization, charge_toroid)
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
                   └── .analyze(rms_detector, jitter_correction,
                   │            charge_normalization, charge_toroid, physics_model)
                   │
                   ├── [optional] compute_charge_normalization()
                   ├── [optional] compute_jitter()
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
| `jitter_correction` | `bool` | `False` | Apply BPM orbit-fit jitter correction |
| `charge_normalization` | `bool` | `False` | Normalize detector signals by toroid charge |
| `charge_toroid` | `str \| None` | `None` | Toroid for normalization; auto-resolved if `None` |

**Internal flow**:
1. Creates appropriate collection class via `create_wire_collection()`
2. Calls `collection.measure()` → `WireMeasurementCollectionResult`
3. Creates `WireMeasurementAnalysis(collection_result=..., fitting_method=...)`
4. Calls `analysis.analyze(rms_detector=..., jitter_correction=..., charge_normalization=..., charge_toroid=...)` → `WireMeasurementAnalysisResult`

The intermediate `collection_result` is also stored on the instance for inspection:
```python
measurement.collection_result  # accessible after measure() completes
```

---

## Stage 1: Data Collection

### BaseWireMeasurementCollection (`collection/base.py`)

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

### On-The-Fly Mode: OTFWireMeasurementCollection (`collection/otf.py`)

Wire moves continuously through all profile ranges. Buffer acquires data during motion.

**`_run_collection_scan()` steps**:
1. `initialize_otf_with_retry(device, logger, max_attempts=3)`:
   - Standalone function in `motion/otf.py`
   - Calls `wire.start_scan()` to arm OTF motion
   - Waits for `wire.homed` and `wire.on_status` to become True
   - Retries up to 3 times with timeout
   - Raises `RuntimeError` on failure
2. Start timing buffer and poll for completion:
   - Calls `buffer.start()`
   - Polls `buffer.is_complete()` every 0.1s
   - Logs wire position periodically
   - Raises `TimeoutError` if acquisition exceeds calculated timeout

**When to use**: Standard beam rates where the wire can traverse all profiles in a single continuous pass.

---

### Step Mode: StepWireMeasurementCollection (`collection/step.py`)

Wire moves to discrete positions sequentially. Buffer runs concurrently.

**`_run_collection_scan()` steps**:
1. `initialize_step_with_retry(device, logger, max_attempts=3)`:
   - Standalone function in `motion/step.py`
   - Calls `wire.initialize()` if not already enabled
   - Waits for `wire.enabled` to become True
   - Retries up to 3 times
2. Start timing buffer (`buffer.start()`)
3. Build sorted position list via `get_step_positions(device)` from active profiles (inner/outer for each)
4. Move through each position sequentially via `move_to_step_position()`:
   - Even-indexed positions (fast travel): move at `speed_max`
   - Odd-indexed positions (data-taking traverse): calculate speed from `(range / scan_pulses) * beam_rate`
   - Wait for arrival within 250 µm tolerance
5. Retract wire after all positions (with 2s wait before and after retract)
6. Wait for buffer to complete (with timeout)

**Speed calculation**:
```python
# Even indices: fast traverse to next starting position
speed = device.speed_max

# Odd indices: data-taking speed across a profile range
position_delta = positions[i] - positions[i - 1]
speed = (position_delta / device.scan_pulses) * device.beam_rate
```

**When to use**: Low beam rates or scenarios requiring discrete position measurements.

---

### Motion Subpackage (`motion/`)

Wire motion primitives used by both collection classes and beamless scan modes. These are standalone functions, not methods on the collection classes.

**`motion/otf.py`**:
- `initialize_otf_with_retry(device, logger, max_attempts=3)` — Initialize wire for OTF scan with retry logic

**`motion/step.py`**:
- `initialize_step_with_retry(device, logger, max_attempts=3)` — Initialize wire for step scan with retry
- `get_step_positions(device) → list` — Build sorted position list from active profiles
- `move_to_step_position(device, logger, *, position, position_index, total_positions, positions)` — Move wire to a single step position with appropriate speed

**`motion/utils.py`**:
- `poll_motor_rbv(device, poll_interval_s=0.05) → list[float]` — Poll motor readback until settled
- Constants: `POLL_INTERVAL_S = 0.05`, `SETTLE_THRESHOLD_UM = 50`, `SETTLE_COUNT = 20`

The settle detection considers the wire stopped when `SETTLE_COUNT` (20) consecutive readings differ by less than `SETTLE_THRESHOLD_UM` (50 µm).

---

### Beamless Scan Modes

Buffer-less motion-only scans for commissioning, mechanical validation, and motion testing. These bypass the timing buffer system entirely, collecting only motor position data by polling the readback PV.

**`collection/beamless_otf.py`**:
```python
from slac_measurements.wires.collection import run_beamless_otf_scan

result = run_beamless_otf_scan(device=wire, poll_interval_s=0.05)
```
Returns a `WireMeasurementCollectionResult` with position-only data (no detector signals). Initializes OTF motion, then polls `motor_rbv` at 20 Hz until the wire settles.

**`collection/beamless_step.py`**:
```python
from slac_measurements.wires.collection import run_beamless_step_scan

result = run_beamless_step_scan(device=wire)
```
Steps through all positions without a buffer. Runs the motor polling in a background thread with a 60-second timeout.

**When to use**: Verifying mechanical behavior without beam delivery — e.g., during wire scanner commissioning or troubleshooting motion issues. These are called by the suite layer's `motion_test()` method.

---

### Timing Buffer Management (`collection/buffer.py`)

**`reserve_buffer(beampath, pulses, beam_rate)`** allocates a timing buffer sized for the scan:

```python
from slac_measurements.wires.collection.buffer import reserve_buffer

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

### Detector Resolution (`detector_util.py`)

Wire metadata can store detectors and default_detector as either simple strings/lists or timing-keyed dicts (CU vs SC). The `detector_util` module handles resolution:

```python
from slac_measurements.wires.detector_util import resolve_detectors, DetectorConfig

config: DetectorConfig = resolve_detectors(
    detectors=wire.metadata.detectors,
    default_detector=wire.metadata.default_detector,
    beampath="CU_HXR",
)
# config.names     → ["PMT29150", "LBLM32A"]
# config.default   → "PMT29150"
# config.raw_strings → ["PMT29150:LTUH", "LBLM32A:LTUH"]
```

Key functions:
- `resolve_detectors(detectors, default_detector, beampath) → DetectorConfig`
- `pick_by_beampath(container, beampath)` — Resolve a CU/SC-keyed dict to the active timing entry
- `strip_colon_suffix(name)` — Remove `:AREA` suffix (e.g., `"PMT29150:LTUH"` → `"PMT29150"`)

---

### Device Dictionary

The collection layer auto-discovers detectors from wire metadata and creates device objects:

| Detector Prefix | Device Type | Buffer Method |
|-----------------|-------------|---------------|
| `LBLM` | Beam loss monitor | `"fast_buffer"` |
| `PMT` | Photo-multiplier tube | `"qdcraw_buffer"` |
| `TMITLOSS` | Transmitted intensity loss | Direct `measure()` call |
| *(wire itself)* | Wire position | `"position_buffer"` |

BPM devices (for jitter correction) collect both x and y position buffers, and optionally TMIT if the BPM is listed as a charge toroid.

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
    charge_toroids: list[str] | None         # Toroid devices for charge normalization
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

### WireMeasurementAnalysis (`analysis/analysis.py`)

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

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `rms_detector` | `str \| None` | `None` | Detector for RMS extraction |
| `jitter_correction` | `bool` | `False` | Apply BPM orbit-fit correction |
| `charge_normalization` | `bool` | `False` | Normalize by toroid charge |
| `charge_toroid` | `str \| None` | `None` | Toroid device; auto-resolved if `None` |
| `physics_model` | `str` | `"BLEM"` | R-matrix source for jitter correction |

#### Pre-processing (optional)

Before the core pipeline, two optional pre-processing steps run if enabled:

1. **Charge normalization** (`compute_charge_normalization()`): Computes per-pulse scale factors from toroid TMIT data and applies them to all detector signals in the raw data.
2. **Jitter correction** (`compute_jitter()`): Computes per-pulse beam position jitter at the wire using BPM orbit fits and R-matrices, stores the jitter arrays for later subtraction during fitting.

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

### Jitter Correction

BPM-based orbit-fit correction that removes pulse-to-pulse beam position jitter from wire scan profiles.

**Location**: `analysis/jitter_correction.py`

**Algorithm**:
1. Extract BPM x/y buffer data from the collection result (keys starting with `"BPM"`)
2. Fetch 6×6 R-matrices from each BPM to the wire via `meme.model.Model(beampath, model_source=physics_model)`
3. Build transport matrices per plane: `[R11, R12, R16]` for x, `[R33, R34, R36]` for y
4. Subtract mean BPM positions, convert mm → m
5. Solve per-pulse least-squares orbit fit: the first element of the orbit vector is the beam position at the wire
6. Convert m → µm to produce per-pulse jitter arrays

If the dispersion column (R16 or R36) is below `_DISPERSION_THRESHOLD = 1e-4`, it is dropped from the fit.

**Public functions**:
```python
def compute_jitter(
    collection_result: WireMeasurementCollectionResult,
    beampath: str,
    physics_model: str = "BLEM",
) -> tuple[np.ndarray, np.ndarray]:
    # Returns (jitter_x, jitter_y) in µm, one value per buffer pulse

def get_jitter_rmat(
    wire_name: str, bpm_names: list[str],
    beampath: str, physics_model: str = "BLEM",
) -> tuple[np.ndarray, np.ndarray]:
    # Returns (rmat_x, rmat_y) transport matrices
```

**Requirements**: At least 2 BPMs with valid data must be present in the collection result.

---

### Charge Normalization

Per-pulse charge normalization using toroid TMIT data to correct for charge fluctuations during a scan.

**Location**: `analysis/charge_normalization.py`

**Algorithm**:
1. Resolve which toroid to use (explicit or auto from `metadata.charge_toroids`)
2. Extract 1D TMIT charge array from the collection result
3. Mask pulses below `_LOW_CHARGE_THRESHOLD = 1e7`
4. Compute mean charge over valid pulses
5. Scale factor for each pulse: `mean_charge / charge[i]` (1.0 for invalid pulses)
6. Apply normalization factors to all detector signals

**Public function**:
```python
def compute_charge_normalization(
    collection_result: WireMeasurementCollectionResult,
    toroid: str | None = None,
) -> tuple[np.ndarray, np.ndarray, str]:
    # Returns (normalization_factors, valid_mask, resolved_toroid)
```

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
    fitting_method: str                                 # Fitting algorithm used

    # Jitter correction state
    jitter_corrected: bool                              # Whether jitter correction was applied
    jitter_rms: tuple[float, float] | None              # (jitter_rms_x, jitter_rms_y) in µm

    # Charge normalization state
    charge_normalized: bool                             # Whether charge normalization was applied
    charge_toroid: str | None                           # Toroid device used

    # Inherited from BeamProfileMeasurementResult:
    rms_sizes: Optional[NDArrayAnnotatedType]           # (x_rms, y_rms)
    centroids: Optional[NDArrayAnnotatedType]
    total_intensities: Optional[NDArrayAnnotatedType]
    signal_to_noise_ratios: Optional[NDArrayAnnotatedType]
    metadata: MeasurementMetadata
```

**Re-analysis**: Change analysis parameters without re-scanning:
```python
new_result = result.reanalyze(
    fitting_method="asymmetric_gaussian",
    rms_detector="LBLM",
    jitter_correction=True,
    charge_normalization=True,
)
```

**MATLAB export**: Save to `.mat` format compatible with the legacy MATLAB `wirescan_gui`:
```python
result.to_mat("/path/to/output.mat")
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

### Supporting Result Types

**`FitResult`**:
```python
class FitResult:
    detectors: dict[str, DetectorFit]  # {detector_name: fit parameters}
```

**`DetectorFit`**:
```python
class DetectorFit:
    mean: float        # Centroid position (stage coords)
    sigma: float       # RMS beam size (microns, beam coords)
    amplitude: float   # Peak amplitude
    offset: float      # Baseline offset
    curve: ndarray     # Fitted curve array
    positions: ndarray # Position array (beam coords)
```

**`ProfileMeasurement`**:
```python
class ProfileMeasurement:
    positions: ndarray                              # Wire positions for this profile
    detectors: dict[str, DetectorProfileMeasurement]  # Detector data arrays
    profile_indices: ndarray                        # Index array within full scan
```

**`DetectorProfileMeasurement`**:
```python
class DetectorProfileMeasurement:
    values: ndarray       # Detector signal values
    units: str | None     # Signal units
    label: str | None     # Display label
```

---

## Coordinate Systems

Wire scanners measure beam profiles by sweeping a wire through the beam at an angle. Two coordinate systems are relevant:

**Stage coordinates** (motor units, microns): The raw motor position as the wire moves. Used for scan ranges, position control, and stored in `DetectorFit.mean`.

**Beam coordinates** (microns): Physical transverse position perpendicular to the beam axis. Used for fitting and stored in `DetectorFit.positions` and `DetectorFit.curve`.

**Conversion** (via `analysis/coordinates.py`):
```python
from slac_measurements.wires.analysis.coordinates import (
    wire_angle_scales, stage_to_beam, beam_to_stage, xy_to_stage
)

scales = wire_angle_scales(install_angle_deg=45.0)
# {"x": sin(45°), "y": cos(45°), "u": 1.0}

beam_pos = stage_to_beam(stage_positions, profile="x", install_angle_deg=45.0)
stage_pos = beam_to_stage(beam_positions, profile="x", install_angle_deg=45.0)

# Project x/y jitter onto wire stage direction
stage_jitter = xy_to_stage(jitter_x, jitter_y, install_angle_deg=45.0)
```

The `sigma` in `DetectorFit` is in beam coordinates (microns) and directly represents the RMS beam size.

---

## MATLAB Export

The `mat_export` module produces `.mat` files compatible with the MATLAB `wirescan_gui` (File > Open). This enables cross-validation with the legacy analysis tool.

```python
from slac_measurements.wires.analysis.mat_export import analysis_result_to_mat

analysis_result_to_mat(result, "/path/to/output.mat")
# Or via the convenience method:
result.to_mat("/path/to/output.mat")
```

The `.mat` file contains a `data` struct matching the MATLAB `dataSave` layout: wire positions, PMT/detector data, BPM data, toroid data, R-matrices, calibrated profiles, beam statistics (mean, sigma, XY correlation, profile intensity), and metadata (wire angle, scan ranges, timestamp, beampath, device names in EPICS format).

---

## Public API (`__init__.py`)

```python
# Primary entry point
from slac_measurements.wires import WireBeamProfileMeasurement

# Analysis (for re-processing saved data without hardware)
from slac_measurements.wires import FittingMethod, WireMeasurementAnalysis

# Jitter correction
from slac_measurements.wires import compute_jitter

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

# Collection factory and beamless scans (for advanced users)
from slac_measurements.wires import ScanMode, create_wire_collection
from slac_measurements.wires.collection import run_beamless_otf_scan, run_beamless_step_scan
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

### Scan with jitter correction and charge normalization

```python
result = measurement.measure(
    scan_mode="otf",
    jitter_correction=True,
    charge_normalization=True,
)
print(f"Jitter-corrected: {result.jitter_corrected}")
print(f"Jitter RMS: x={result.jitter_rms[0]:.1f}, y={result.jitter_rms[1]:.1f} µm")
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

### Re-analyze from an existing result

```python
from slac_measurements.wires import load_analysis_from_h5

result = load_analysis_from_h5("/path/to/analysis.h5")
new_result = result.reanalyze(
    fitting_method="super_gaussian",
    jitter_correction=True,
)
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

### Export to MATLAB

```python
result.to_mat("/path/to/output.mat")
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
| `ValueError` (jitter) | Fewer than 2 BPMs with valid data | Ensure BPMs are configured in wire metadata |
| `ValueError` (charge) | Toroid not found in collection result | Check `metadata.charge_toroids` or specify toroid explicitly |
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

Result objects (`WireMeasurementCollectionResult`, `WireMeasurementAnalysisResult`) are Pydantic models. Once created, they represent a complete snapshot of the measurement that can be persisted and loaded. The `reanalyze()` method returns a new result rather than mutating.

### 5. Monotonic Filtering

Only the longest forward-motion segment is used for fitting, making the analysis robust to:
- Wire retraction data captured in the buffer
- Position overshoot at range endpoints
- Multi-pass scans where only one direction is meaningful

### 6. Motion Separation

Wire motion primitives (initialization, stepping, polling) are standalone functions in the `motion/` subpackage, shared between buffered collection classes and beamless scan modes. This avoids duplicating initialization and positioning logic.

---

## Key Takeaways

1. **Unified entry point**: `WireBeamProfileMeasurement.measure()` handles collection + analysis in one call
2. **Two scan modes**: OTF (continuous) and Step (discrete), selected by `scan_mode` parameter
3. **Separation of collection and analysis**: Raw data persists independently; analysis can be re-run via `reanalyze()`
4. **Three fitting methods**: gaussian, asymmetric_gaussian, super_gaussian — pluggable via string parameter
5. **Jitter correction and charge normalization**: Optional pre-processing stages enabled by boolean flags
6. **Beamless scan modes**: Motion-only scans for commissioning without beam delivery
7. **Automatic coordinate conversion**: Stage → beam coordinates using wire install angle
8. **Buffer is always released**: try/finally ensures no leaked timing buffers on failure
9. **MATLAB compatibility**: `.mat` export for cross-validation with legacy tools

---

## Related Documentation

- [Architecture Overview](01-architecture-overview.md) — System-wide context
- [Device Layer](02-device-layer.md) — Wire hardware control (Layer 1)
- [Suite Layer Deep Dive](04-suite-layer.md) — Orchestration and run tracking
- [GUI Development Guide](05-gui-layer.md) — Operator interface
