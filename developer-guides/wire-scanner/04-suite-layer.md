# Wire Scanner Suite Layer (Orchestration)

**Purpose**: Understand the `WireScanSuite` and `WireScanView` classes that compose measurement primitives into a complete data acquisition workflow with persistence, plotting, and run tracking.

**Audience**: Developers extending scan workflows, adding new output formats, or integrating suite functionality into higher-level tools

**Last Updated**: October 13, 2026

---

## Role in the Architecture

```
Layer 4 (GUI)          ← calls suite.run_single(), view.draw_*()
       ↓
Layer 3 (Suite)        ← THIS LAYER: orchestration, persistence, plotting, diagnostics
       ↓
Layer 2 (Measurement)  ← WireBeamProfileMeasurement.measure()
       ↓
Layer 1 (Device)       ← Wire EPICS control via LazyPV
```

The Suite layer transforms raw measurement primitives into a production data acquisition system by adding:

- **Batch processing** — run multiple wires in sequence
- **Data provenance** — persistent JSON registry of every run
- **Automated visualization** — trajectory and profile plots
- **Structured file management** — timestamped HDF5 and PNG outputs
- **Simplified API** — single method calls replace multi-step workflows

---

## Components

| File | Class / Role | Responsibility |
|------|-------|---------------|
| `slacwire/suite/__init__.py` | `WireScanSuite` | Composed dataclass assembling all mixins |
| `slacwire/suite/_base.py` | `WireScanSuiteBase` | Dataclass fields, `__post_init__`, device management |
| `slacwire/suite/_run.py` | `RunMixin` | `run_single`, `run_all` (collection + analysis + persist) |
| `slacwire/suite/_collect.py` | `CollectMixin` | `collect_single` (raw data without analysis) |
| `slacwire/suite/_motion.py` | `MotionTestMixin` | `motion_test` (beam-less motion validation) |
| `slacwire/suite/_results.py` | `ResultsMixin` | `latest_run`, `replot`, `summary` |
| `slacwire/suite/_compare.py` | `CompareMixin` | `compare_detectors` (multi-detector overlay) |
| `slacwire/suite/_diagnostics.py` | `DiagnosticsMixin` | `cache_info`, `cache_pvs`, `cache_summary` |
| `slacwire/suite/_logbook.py` | `LogbookMixin` | `post_to_logbook`, `post_last_to_logbook` |
| `slacwire/suite/_jitter_compare.py` | `JitterCompareMixin` | `jitter_compare`, `residual_compare`, FFT analysis |
| `slacwire/suite/_loader.py` | `LoaderMixin` | `load_scan`, `discover_scans` (HDF5 file discovery) |
| `slacwire/suite/_beamsizes.py` | `BeamsizesMixin` | `get_beamsizes` (multi-wire emittance extraction) |
| `slacwire/_constants.py` | — | `WIRE_AREA_LOOKUP`, `VALID_BEAMPATHS`, path utilities (package root) |
| `slacwire/view.py` | `WireScanView` | Plot generation (standalone figures and in-place GUI rendering) |
| `slacwire/registry/registry.py` | `RunRegistry` | Persistent JSON audit trail of all scan runs |

---

## WireScanSuite

### Composition Architecture

`WireScanSuite` is a `@dataclass` composed from separate mixin classes via multiple inheritance:

```python
@dataclass
class WireScanSuite(
    LogbookMixin, JitterCompareMixin, CompareMixin, DiagnosticsMixin,
    MotionTestMixin, CollectMixin, RunMixin, LoaderMixin, BeamsizesMixin,
    ResultsMixin, WireScanSuiteBase
):
    pass
```

Each mixin adds a focused set of methods. The base class (`WireScanSuiteBase`) holds all dataclass fields and infrastructure. MRO ensures `__post_init__` runs from the base.

### Construction

```python
from slacwire import WireScanSuite

suite = WireScanSuite(
    wires=["WS28144", "WS27644"],
    beampath="CU_HXR",
    detector="PMT29150",       # optional: overrides device default
    save=True,                 # write HDF5 after each scan
    save_plots=True,           # write PNG plot files
    show=True,                 # call fig.show() on generated plots
    dev_mode=False,            # if True, skip registry logging on errors
)
```

On construction (`__post_init__` in `_base.py`):
1. `_build_devices()` — creates `Wire` device instances via `create_wire(area, name)` for every wire in the list
2. `_build_view()` — instantiates a `WireScanView` for plotting
3. Output directories are created: `outdir` (dated) and `plotdir` (outdir/plots)

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `wires` | `list[str]` | Wire names to operate on (e.g. `["WS28144"]`) |
| `devices` | `dict[str, Wire]` | Cached device instances keyed by wire name |
| `beampath` | `Beampath` | Literal: `"CU_HXR"`, `"CU_SXR"`, `"SC_HXR"`, `"SC_SXR"`, `"SC_BSYD"`, `"SC_DIAG0"` |
| `detector` | `str \| None` | Override detector; falls back to device default |
| `outdir` | `Path` | HDF5 output directory (default: `/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/`) |
| `plotdir` | `Path` | PNG output directory (`outdir/plots/`) |
| `results` | `dict[str, list]` | Cached results keyed by wire name, ordered by run |
| `registry` | `RunRegistry` | Persistent run audit trail |
| `view` | `WireScanView` | Plot rendering engine |
| `save` | `bool` | Write HDF5 files |
| `show` | `bool` | Display plots interactively |
| `save_plots` | `bool` | Write PNG plot files |
| `dev_mode` | `bool` | Skip registry error logging (development use) |

### Public Methods

#### `run_single(wire, scan_mode="otf", rms_detector=None, jitter_correction=False, charge_normalization=False)` — `_run.py`

Execute a complete scan for one wire: collection → analysis → save → plot → registry.

```python
suite.run_single("WS28144", scan_mode="otf")
suite.run_single("WS27644", scan_mode="step", rms_detector="PMT27650")
suite.run_single("WS28144", jitter_correction=True, charge_normalization=True)
```

#### `run_all(scan_mode="otf", rms_detector=None)` — `_run.py`

Run all configured wires sequentially in the given scan mode.

```python
suite.run_all(scan_mode="otf")
```

#### `collect_single(wire, scan_mode="otf")` — `_collect.py`

Collect raw data without analysis. Useful for debugging or when custom post-processing is needed.

```python
suite.collect_single("WS28144", scan_mode="step")
raw_data = suite.latest_run("WS28144")  # collection-only result
```

#### `motion_test(wire, scan_mode="otf", plot=True)` — `_motion.py`

Run beam-less motion validation for a single wire. Exercises the wire motion without requiring beam to verify mechanical behavior.

```python
result = suite.motion_test("WS28144", scan_mode="otf")
result = suite.motion_test("WS28144", scan_mode="step", plot=False)
```

#### `latest_run(wire) → result` — `_results.py`

Retrieve the most recent result for a wire. Raises `KeyError` if no results exist.

```python
result = suite.latest_run("WS28144")
x_rms, y_rms = result.rms_sizes
```

#### `replot(wire, detector=None) → list[Path]` — `_results.py`

Regenerate plots from cached results without re-scanning.

```python
paths = suite.replot("WS28144")
```

#### `summary()` — `_results.py`

Print a formatted table of latest results for all wires with sigma values.

```python
suite.summary()
# Wire Scan Suite  ·  beampath: CU_HXR
# ─────────────────────────────────────────────────────────────────────────────
# WS28144  (3 runs)
#   Latest  |  20260529_140000  |  otf  |  detector: PMT29150
#     x :  σ =    42.3 µm
#     y :  σ =    38.7 µm
```

#### `compare_detectors(wire, detectors=None) → list[Path]` — `_compare.py`

Render multi-detector overlay plots for the latest scan of a wire. Each detector's profile is normalized (offset subtracted, scaled by fit amplitude) so shapes are directly comparable regardless of signal magnitude. Defaults to all detectors with fit results.

```python
# Compare all detectors
suite.compare_detectors("WS28144")

# Compare a specific subset
suite.compare_detectors("WS28144", detectors=["PMT29150", "LBLM32A"])
```

#### `cache_info()` / `cache_pvs()` / `cache_summary()` — `_diagnostics.py`

EPICS Channel Access cache inspection for debugging PV connection issues.

```python
suite.cache_summary()    # Print formatted summary
info = suite.cache_info()  # Get dict with context, counts, per-device breakdown
pvs = suite.cache_pvs()    # Get actual PV names grouped by device
```

#### `post_to_logbook(title, body="", attachment=None)` — `_logbook.py`

Post an entry to the SLAC physics logbook. Logbook is auto-resolved from beampath: `SC_*` → `"lcls2"`, otherwise → `"lcls"`.

```python
suite.post_to_logbook("Wire scan complete", body="WS28144 σ_x=42µm")
```

#### `post_last_to_logbook(wire, ...)` — `_logbook.py`

Post the latest scan result for a wire to the logbook, including profile plot images.

#### `load_scan(path)` — `_loader.py`

Load a single `.h5` scan file from disk and return a `WireMeasurementAnalysisResult`.

```python
result = suite.load_scan("/u1/lcls/physics/data/wire_scan/2026/10/13/OTF_WS28144.h5")
```

#### `discover_scans(wire, date, limit=10, subdir=None)` — `_loader.py`

Discover and load `.h5` scans for a wire on a given date. Returns results sorted most-recent first.

```python
results = suite.discover_scans("WS28144", date="2026/10/13")
results = suite.discover_scans("WS28144", date="2026/10/13", limit=None)  # all
```

#### `get_beamsizes(**run_kwargs)` — `_beamsizes.py`

Extract beam sizes from all configured wires. Returns an `(n_wires, 2)` NumPy array with columns `[sigma_x, sigma_y]` in microns. Runs scans for any wires missing results.

```python
sizes = suite.get_beamsizes(scan_mode="otf")
# sizes[0] → [sigma_x, sigma_y] for first wire
```

#### `jitter_compare(wire, ...)` — `_jitter_compare.py`

Compare jitter-corrected vs uncorrected profiles for a wire.

#### `residual_compare(wire, ...)` — `_jitter_compare.py`

Compare fit residuals between jitter-corrected and uncorrected analyses.

#### `fft_spectrum(wire, ...)` / `fft_overlay(wire, ...)` — `_jitter_compare.py`

Compute and plot FFT spectra of fit residuals for vibration analysis.

#### `vibration_summary(wire, ...)` — `_jitter_compare.py`

Generate a comprehensive vibration diagnostic report: profiles, residuals, FFT, and overlay.

#### `set_wires(wires)` — `_base.py`

Replace the wire list and rebuild device instances.

```python
suite.set_wires(["WS28144", "WS28444", "WS28744"])
```

### Internal Scan Flow (`_run_device_scan`)

Every public scan method delegates to `_run_device_scan`, which implements the common workflow:

```
_run_device_scan(device, method, scan_fn, rms_detector, file_prefix)
    │
    ├── Record scan_started timestamp
    ├── Resolve detector (rms_detector or device default)
    │
    ├── try:
    │   ├── scan_fn(device, rms_detector=...) → data
    │   │     └── calls _measure() → WireBeamProfileMeasurement.measure()
    │   ├── Append data to self.results[wire]
    │   ├── if save: data.save_to_h5(outdir/{prefix}_{wire}_{stamp}.h5)
    │   ├── if save_plots or show: view.render(...) → plot_paths
    │   ├── Resolve scope_data CSV (OTF only)
    │   └── registry.log(method, wire, beampath, detector, filepath, plots, ...)
    │
    └── except Exception:
        ├── registry.log(..., status="error", error=str(e))
        └── re-raise
```

### Wire-to-Area Resolution

`WireScanSuite` maps wire names to accelerator areas via the `WIRE_AREA_LOOKUP` dict in `_constants.py`:

| Wire Pattern | Area |
|-------------|------|
| `WS01`–`WS04` | DL1 |
| `WS11`–`WS13` | BC1 |
| `WS27644`–`WS28744` | L3 |
| `WS0H04` | HTR |
| `WSDG01` | DIAG0 |
| `WSC104`–`WSC110` | COL1 |
| `WSEMIT2` | EMIT2 |
| `WSBP2`–`WSBP4` | BYP |
| `WSSP1D` | SPD |
| `WS31`–`WS34` | LTUH |
| `WS31B`–`WS34B` | LTUS |

To add a new wire, add an entry to `WIRE_AREA_LOOKUP` in `suite/_constants.py`.

### File Output Structure

```
/u1/lcls/physics/data/wire_scan/
└── 2026/
    └── 05/
        └── 29/
            ├── OTF_WS28144_20260529_140000.h5
            ├── Step_WS27644_20260529_141500.h5
            └── plots/
                ├── OTF_Trajectory_WS28144_20260529_140000.png
                ├── OTF_Profile_x_WS28144_20260529_140000.png
                ├── OTF_Profile_y_WS28144_20260529_140000.png
                └── Step_Trajectory_WS27644_20260529_141500.png
```

Naming conventions:
- HDF5: `{Method}_{Wire}_{Timestamp}.h5` (e.g., `OTF_WS28144_20260529_140000.h5`)
- Trajectory plots: `{Method}_Trajectory_{Wire}_{Timestamp}.png`
- Profile plots: `{Method}_Profile_{profile}_{Wire}_{Timestamp}.png`
- Multi-panel: `{Method}_Multi_{Wire}_{Timestamp}.png`
- Detector comparison: `{Method}_DetectorCompare_{Wire}_{Timestamp}.png`
- Motion test: `MotionTest_Trajectory_{Wire}_{Timestamp}.png`
- Jitter/FFT/Vibration plots use their own prefixes (e.g., `JitterCompare_`, `FFT_`, `Vibration_`)

### Scope Data Resolution

For OTF scans, `_resolve_scope_data_path` looks for the most recent CSV in `/u1/lcls/physics/genMotion/wirescanners/scope_data/{wire}/` that was modified after the scan started. This links external motion controller scope captures to the run registry entry.

---

## WireScanView

The view component provides two rendering modes:

### In-Place Drawing (GUI)

Methods that render into an existing `matplotlib.Figure` — used by the GUI's embedded canvas widgets. The caller owns the figure lifecycle and must call `canvas.draw()` after.

#### `draw_trajectory(fig, data, wire, detector)`

Clears and redraws a dual-axis trajectory plot:
- Left axis (blue): wire position in µm vs scan point
- Right axis (orange): detector counts vs scan point

#### `draw_profile(fig, data, wire, detector, profile)`

Clears and redraws a beam profile plot:
- Measured data (dotted blue)
- Fitted Gaussian curve (solid line)
- Dual x-axis: stage coordinates (bottom) and beam coordinates (top)
- Parameter annotation box: mean, sigma, amplitude, offset

The stage-to-beam coordinate transform uses `cos(45°)` for x and y profiles (wires mounted at 45°), and identity for u profiles.

#### `draw_detector_comparison(fig, data, wire, profile, detectors=None)`

Clears and redraws a multi-detector overlay into an existing figure. Each detector's measured data (dotted) and fitted curve (solid) are normalized for shape comparison. Defaults to all detectors with fit results for the given profile.

### Standalone Figure Creation (Suite)

Methods that create new `plt.figure()` instances — used by the suite for saving to disk.

#### `plot_trajectory(data, wire, detector) → Figure`

Same visualization as `draw_trajectory` but returns a new figure.

#### `plot_profile(data, profile, wire, detector) → Figure`

Same visualization as `draw_profile` but returns a new figure.

#### `plot_detector_comparison(data, profile, wire, detectors=None) → Figure`

Creates a new figure overlaying normalized profiles from multiple detectors. Legend shows each detector's sigma. Defaults to all detectors with fit results.

#### `render_detector_comparison(data, wire, profiles, detectors, file_prefix, plotdir, stamp, show, save) → list[Path]`

Renders multi-detector comparison plots for all available profiles. Saves as `{prefix}_DetectorCompare_{profile}_{wire}_{stamp}.png`.

#### `render(data, wire, detector, profiles, file_prefix, plotdir, stamp, show, save) → list[Path]`

Primary entry point for the suite. Generates all plots for a completed scan:

1. Creates and optionally shows/saves a trajectory plot
2. Iterates over active profiles (x, y, u) and creates/shows/saves each profile plot
3. Returns list of saved PNG paths

#### `render_multi(data, wire, detector, profiles, file_prefix, plotdir, stamp, show, save) → list[Path]`

Renders a multi-panel plot combining trajectory and all profiles in a single figure.

#### `render_motion_test(data, wire, file_prefix, plotdir, stamp, show, save) → list[Path]`

Renders trajectory plots from beamless motion test results (no detector data).

#### `save_fig(fig, name, plotdir, stamp) → Path`

Saves a figure to `{plotdir}/{name}_{stamp}.png` at 150 DPI with opaque white background (required for `img2pdf` used by the physics elog poster).

#### `save_fig_to_path(fig, path) → Path`

Saves a figure to an explicit file path (used by the GUI for logbook image export).

#### Jitter and Vibration Analysis Plots

These methods support the `JitterCompareMixin`:

- `plot_jitter_compare(...)` — Overlay jitter-corrected vs uncorrected profiles
- `plot_vibration_residuals(...)` — Plot fit residuals for vibration analysis
- `plot_residual_compare(...)` — Compare residual distributions
- `plot_fft_spectrum(...)` — FFT spectrum of fit residuals
- `plot_fft_overlay(...)` — Overlay FFT spectra across profiles

---

## RunRegistry

### Purpose

Persistent, append-only audit trail of every scan run. Enables:
- Traceability: which wire was scanned, when, by what method
- Error tracking: failed scans are logged with error context
- Data discovery: file paths for HDF5, plots, and scope data

### Storage

JSON file at `/u1/lcls/physics/data/wire_scan/ws_run_registry.json`.

### Concurrency Safety

Multiple `WireScanSuite` instances may share the same registry file. Before each write, `RunRegistry` re-reads the file to find the current max `run_id`, ensuring monotonically increasing IDs even under concurrent use. Writes use atomic temp-file replacement.

### Entry Schema

```python
{
    "run_id": 42,                                    # Auto-incrementing integer
    "timestamp": "20260529_140000",                  # Scan completion time
    "method": "otf",                                 # "otf" | "step" | "otf_collection" | "step_collection"
    "wire": "WS28144",                               # Device name
    "beampath": "CU_HXR",                            # Accelerator beampath
    "detector": "PMT29150",                          # Detector used for RMS
    "filepath": "/u1/.../OTF_WS28144_....h5",        # HDF5 data file (null if save=False)
    "scope_data": "/u1/.../scope_data/WS28144/...",  # Scope CSV (OTF only, null otherwise)
    "plots": ["/u1/.../plots/OTF_Trajectory_....png", ...],  # Saved plot paths
    "status": "ok",                                  # "ok" | "error"
    "error": null                                    # Error message string on failure
}
```

### Error Normalization

When a scan fails, the error message is normalized to include the wire name if not already present, ensuring that error entries are self-describing when viewed in the registry file.

### API

```python
from slacwire.registry import RunRegistry

registry = RunRegistry()                         # loads from default path
registry = RunRegistry(path=Path("custom.json")) # custom path

entry = registry.log(
    method="otf",
    wire="WS28144",
    beampath="CU_HXR",
    detector="PMT29150",
    filepath=Path("..."),
    plots=[Path("...")],
)
```

---

## Integration with Layer 4 (GUI)

The GUI layer uses the suite through these integration points:

| GUI Need | Suite Method |
|----------|-------------|
| Execute a scan | `suite.run_single(wire, scan_mode, jitter_correction, charge_normalization)` |
| Get latest result for display | `suite.latest_run(wire)` |
| Render trajectory into canvas | `suite.view.draw_trajectory(fig, data, wire, detector)` |
| Render profile into canvas | `suite.view.draw_profile(fig, data, wire, detector, profile)` |
| Render detector comparison into canvas | `suite.view.draw_detector_comparison(fig, data, wire, profile, detectors)` |
| Clear canvas | `suite.view.clear_figure(fig)` |
| Access run history | `suite.registry.entries` |
| Post to logbook | `suite.post_to_logbook(title, body, attachment)` |
| Save figure for logbook | `suite.view.save_fig_to_path(fig, path)` |

The GUI runs scans on a `QThread` and receives results via Qt signals. The view's `draw_*` methods are called on the main thread after signal delivery, ensuring thread-safe GUI updates.

---

## Integration with Layer 2 (Measurement)

The suite delegates all scan execution to Layer 2 via a single integration point:

```python
def _measure(self, device, scan_mode, rms_detector=None,
             jitter_correction=False, charge_normalization=False,
             charge_toroid=None):
    measurement = WireBeamProfileMeasurement(
        beam_profile_device=device, beampath=self.beampath
    )
    return measurement.measure(
        scan_mode=scan_mode,
        rms_detector=rms_detector,
        jitter_correction=jitter_correction,
    )
```

The suite delegates scan execution to `WireBeamProfileMeasurement.measure()`. Collection-only workflows (`collect_single`) use a separate `_collect` method that calls the collection factory directly.

---

## Extending the Suite

### Adding a New Wire

Add an entry to `WIRE_AREA_LOOKUP` in `suite/_constants.py`:

```python
WIRE_AREA_LOOKUP = {
    ...
    "WS_NEW": "NEW_AREA",
}
```

### Adding a New Mixin (New Feature Area)

1. Create `suite/_feature.py` with a mixin class (e.g., `FeatureMixin`)
2. Methods can access `self.devices`, `self.view`, `self.results`, etc. from the base
3. Import and add the mixin to the `WireScanSuite` class in `suite/__init__.py`
4. Export any new public names in `__all__`

### Adding a New Scan Mode

1. Implement the mode in Layer 2 (`slac_measurements/wires/`)
2. Add it to the `mode not in (...)` validation in `_run.py` and `_collect.py`
3. Choose a `file_prefix` for the new mode

### Adding a New Plot Type

1. Add a `plot_*` method to `WireScanView` that returns a `Figure`
2. Add a corresponding `draw_*` method for GUI in-place rendering
3. Call the new plot method from `render()` to include it in automated output

### Custom Output Directory

```python
suite = WireScanSuite(
    wires=["WS28144"],
    beampath="CU_HXR",
    outdir=Path("/tmp/test_scans"),
)
```

---

## Design Decisions

### Why a dataclass composed from mixins?

`WireScanSuite` holds mutable state (results, devices) that accumulates over a session. A dataclass with `field(default_factory=...)` provides clean initialization with sensible defaults while remaining lightweight. The mixin decomposition keeps each concern in its own file, making it easy to locate and extend behavior without navigating a large monolithic module.

### Why separate View from Suite?

The view must support two rendering modes — standalone figures for batch/automated use and in-place rendering for GUI canvases. Separating it allows the GUI to call `draw_*` methods directly on its own figures without going through the suite's save/show logic.

### Why re-raise after logging errors?

The registry captures error context for auditability, but the caller (especially the GUI thread) needs the exception to update UI state and notify the operator. Swallowing exceptions would make failures invisible.

### Why atomic registry writes?

Multiple operators may run wire scans concurrently on the same NFS-mounted data directory. Atomic temp-file replacement prevents partial writes from corrupting the shared registry.

---

## Related Documentation

- [Architecture Overview](01-architecture-overview.md)
- [Device Layer Deep Dive](02-device-layer.md)
- [Measurement Layer Deep Dive](03-measurement-layer.md)
- [GUI Development Guide](05-gui-layer.md)
