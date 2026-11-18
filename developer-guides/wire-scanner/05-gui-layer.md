# Wire Scanner GUI Layer

**Purpose**: Understand the PyDM/Qt GUI that provides the operator interface for wire scanner beam profile measurements at LCLS.

**Audience**: Developers adding GUI features, modifying widget behavior, or debugging operator-facing issues

**Last Updated**: November 12, 2026

---

## Role in the Architecture

```
Layer 4 (GUI)          ← THIS LAYER: operator interface, threading, PyDM/EPICS binding
       ↓
Layer 3 (Suite)        ← WireScanSuite.run_single(), WireScanView.draw_*()
       ↓
Layer 2 (Measurement)  ← WireBeamProfileMeasurement.measure()
       ↓
Layer 1 (Device)       ← Wire EPICS control via LazyPV
```

The GUI layer translates operator interactions (select wire, click scan, view results) into suite-level method calls and renders results back to the user. It adds:

- **Navigation** — beampath/area/wire hierarchical selection
- **Live EPICS binding** — PyDM widgets connected to scan parameter PVs
- **Threading** — background scan execution to keep the UI responsive
- **Real-time plotting** — matplotlib canvases updated via Qt signals
- **Logging** — status messages displayed in-app and persisted to file
- **Physics eLog integration** — one-click posting of profile plots to the logbook

---

## File Layout

```
slacwire/
├── ws_gui.py                  # Main display: WireScanSuiteGUI, WireScanSuiteThread
├── wire_scan_gui.ui           # Qt Designer layout (XML)
├── wire_scan_gui.yaml         # Beampath → area → wire configuration
├── __main__.py                # CLI entry point
├── _constants.py              # VALID_BEAMPATHS, VALID_FIT_METHODS, paths
├── speed_status.py            # Qt-free display/logging rules for wire speed status
├── plot_navigation.py         # Qt-free pan/zoom axis-limit math for MplCanvas
├── config/
│   ├── __init__.py
│   └── wire_config.py         # WireConfig, get_wire_config, set_wire_config (SQLite)
├── registry/
│   ├── registry.py            # RunRegistry (JSON audit trail)
│   ├── registry_sqlite.py     # SQLite-backed registry
│   ├── kpi_cli.py             # KPI command-line interface
│   ├── kpi_daily_summary.py   # Daily KPI aggregation
│   ├── kpi_queries.py         # KPI database queries
│   └── log_run_cli.py         # CLI for logging runs
└── widgets/
    ├── __init__.py            # Public exports
    ├── navigation.py          # NavigationWidget (beampath + area combos)
    ├── measurement.py         # MeasurementWidget (wire, detector, BPMs, toroids, options)
    ├── plots.py               # PlotWidget, MplCanvas, ProfileControl, FitControl
    ├── text_logger.py         # QTextEditLogger, attach_logger_to_widget
    ├── speed_status.py        # SpeedStatusMonitor, build_speed_labels
    └── error_dialog.py        # ErrorDialog (resizable scan-failure popup)
```

---

## WireScanSuiteGUI

The main application class. Inherits from `pydm.Display` which provides the PyDM framework (EPICS channel integration, stylesheet support, command-line argument handling).

### Construction Flow

```python
WireScanSuiteGUI.__init__(parent, args, macros)
    │
    ├── Load wire_scan_gui.yaml → beampath/area/wire hierarchy
    ├── Create NavigationWidget(yaml_data)
    ├── Create MeasurementWidget(area_to_wires, create_wire_fn)
    ├── Create PlotWidget()
    ├── Build WireScanSuite(wires=[], beampath, detector)
    ├── Connect signals between widgets
    └── init_ui() → wire up .ui file, buttons, logger, plot layout
```

### Key Attributes

| Attribute | Type | Purpose |
|-----------|------|---------|
| `nav` | `NavigationWidget` | Beampath and area selection |
| `measurement` | `MeasurementWidget` | Wire, detector, BPM selection |
| `plots` | `PlotWidget` | Trajectory and profile canvases |
| `suite` | `WireScanSuite` | Orchestration engine (shared with thread) |
| `current_runs` | `dict` | Metadata of completed runs this session |
| `loaded_results` | `dict` | Results loaded from HDF5 files |
| `logger` | `Logger` | Named `"wire_scan_logger"`, writes to file + status widget |

### Signal Wiring

```
NavigationWidget.areaChanged ──────────► MeasurementWidget.update_area()
NavigationWidget.beampathChanged ──────► GUI._on_beampath_changed()
NavigationWidget.beampathChanged ──────► MeasurementWidget.set_beampath()
MeasurementWidget.wireChanged ─────────► GUI.update_parameters()
MeasurementWidget.wireChanged ─────────► GUI.update_plots()
MeasurementWidget.detectorChanged ─────► GUI._on_detector_changed()
MeasurementWidget.detectorChanged ─────► GUI.update_plots()
MeasurementWidget.jitter_checkbox ─────► GUI._on_jitter_toggled()
GUI.dataChanged ───────────────────────► GUI.update_plots()
ProfileControl.profileChanged ─────────► GUI.update_profile_plot()
FitControl.fitMethodChanged ───────────► GUI._on_fit_method_changed()
```

Additionally, `MeasurementWidget` internally connects `wireChanged` to its own `update_detectors()`, `update_toroids()`, `update_bpms()`, and `_apply_config()` methods.

### Button Callbacks

| Button | Callback | Behavior |
|--------|----------|----------|
| Start Scan | `start_scan_callback` | Spawns `WireScanSuiteThread`, disables button |
| Save Data | `save_callback` | Saves latest result to HDF5 via file dialog |
| Load Data | `load_callback` | Opens HDF5 file and loads result for display |
| Post to Logbook | `logbook_callback` | Saves profile plot PNG, delegates to `suite.post_to_logbook()` |
| Save Config | `save_config_callback` | Persists current scan settings to SQLite via `measurement.save_config()` |
| Load Config | `load_config_callback` | Loads saved settings via `measurement._apply_config()` |
| Export to .mat | `export_mat_callback` | Writes displayed result to `suite.outdir/Export_<wire>_<timestamp>.mat` via `result.to_mat()` (MATLAB `wirescan_gui` format); logs the directory and filename |

---

## WireScanSuiteThread

A `QThread` subclass that runs the scan off the main thread to keep the GUI responsive.

### Signals

| Signal | Payload | When Emitted |
|--------|---------|--------------|
| `scan_complete` | `(wire_name, method, data, entry)` | Scan succeeded |
| `scan_failed` | `(wire_identifier, error_message, traceback)` | Exception during scan |

### Constructor

```python
WireScanSuiteThread(suite, wire_identifier, beampath, detector,
                    jitter_correction=False, charge_normalization=False,
                    scan_mode="otf")
```

`scan_mode` is resolved per wire by `start_scan_callback()` via
`slacwire._constants.default_scan_mode()`. Wires listed in
`SCAN_MODE_OVERRIDES` use the mapped mode; all others default to `"otf"`.
Currently `WS02` is overridden to `"step"` (OTF unsupported due to lagging hardware).
The same defaults apply in the suite and CLI — see the Suite Layer guide.

### Execution Flow

```python
WireScanSuiteThread.run()
    │
    ├── Extract wire_name from "WIRE:AREA" identifier
    ├── Set beampath and detector on shared suite
    ├── suite.run_single(wire=wire_name, scan_mode=self.scan_mode,
    │                     jitter_correction=..., charge_normalization=...)
    ├── data = suite.latest_run(wire_name)
    ├── Find matching registry entry
    └── emit scan_complete(wire_name, method, data, entry)
         │
         └── on exception → emit scan_failed(wire_identifier, str(exc), traceback)
```

### Thread Safety

- The suite instance is shared between the main thread and the worker thread
- `suite.show = False` and `suite.save_plots = False` during threaded scans — plotting happens on the main thread after `scan_complete` fires
- All GUI updates (plot rendering, button state, log messages) happen on the main thread via signal/slot connections

---

## Widgets

### NavigationWidget

**File**: `widgets/navigation.py`

Provides hierarchical selection of beampath and area. Populated from `wire_scan_gui.yaml`.

| Signal | Payload | Trigger |
|--------|---------|---------|
| `beampathChanged` | `str` | User selects a beampath |
| `areaChanged` | `str` | User selects an area (or area list refreshes) |

**Properties**: `beampath`, `area`, `get_wires()`

**UI**: Two `QComboBox` widgets in a horizontal layout. Changing beampath repopulates the area combo (with signal blocking to prevent spurious emissions).

### MeasurementWidget

**File**: `widgets/measurement.py`

Manages wire device selection and scan parameters. Creates `Wire` device instances on selection to populate detector and BPM lists from device metadata.

| Signal | Payload | Trigger |
|--------|---------|---------|
| `wireChanged` | `str` | User selects a wire |
| `detectorChanged` | `str` | User selects a detector |

**Properties**: `wire`, `active_wire`, `detector`, `selected_bpms`, `jitter_enabled`, `charge_normalized`

**UI Components**:
- Wire combo (`QComboBox`) — wire names, with area stored as item data
- Detector combo (`QComboBox`) — populated from `wire.metadata.detectors`
- Toroid combo (`QComboBox`) — populated from `wire.metadata.charge_toroids`
- BPM list (`QListWidget`) — populated from `wire.metadata.tmitloss.upstream`
- Jitter correction checkbox
- Charge normalization checkbox

**Additional Methods**:
- `update_toroids()` — populates the toroid combo from wire metadata
- `set_beampath(beampath)` — updates internal beampath and refreshes detector/config state
- `set_fit_control(fit_control)` — cross-widget reference to `FitControl` for config integration
- `_apply_config()` — loads per-wire config from SQLite and applies to UI widgets
- `save_config()` — persists current UI state to SQLite

**Device Creation**: On wire selection, calls `create_wire_fn(area, name)` (injected at construction) to instantiate the `Wire` device object. This connects to EPICS and loads metadata. Wire selection also triggers `_apply_config()` to load saved preferences.

### PlotWidget

**File**: `widgets/plots.py`

Container widget holding the trajectory canvas, profile canvas, profile radio buttons, and fit method selector.

**Components**:
- `trajectory_plot` — `MplCanvas` for wire position vs detector plot
- `profile_plot` — `MplCanvas` for beam profile with fitted curve
- `profile_control` — `ProfileControl` with X/Y/U radio buttons
- `fit_control` — `FitControl` for fitting method selection

#### MplCanvas

A `FigureCanvasQTAgg` wrapper that creates a matplotlib `Figure` with one subplot. The figure is rendered into by `WireScanView.draw_trajectory()` and `draw_profile()`.

**X-axis navigation** (lets operators move a peak out from behind the legend):

| Input | Action |
|-------|--------|
| Scroll wheel / horizontal swipe | Pan X by 10% of the visible span per notch |
| Ctrl + scroll (Cmd on macOS) | Zoom X by 0.8× per notch, centered on the cursor |
| Double-click | Reset X to autoscale |

`MplCanvas.wheelEvent` handles scrolling directly. Scrolls outside the axes are ignored and passed to the parent widget. The limit math lives in the Qt-free `slacwire/plot_navigation.py` (`pan_limits`, `zoom_limits`), covered by `tests/test_plot_navigation.py`. On the trajectory plot, the twin axes share X, so both traces move together. A redraw (new scan, profile or fit change) resets the view, because `draw_*` calls `fig.clf()`.

#### ProfileControl

Radio button group (X, Y, U) for selecting which beam profile to display. Emits `profileChanged(str)` on toggle.

#### FitControl

A `QGroupBox` containing a `QComboBox` for selecting the fitting method used during analysis.

| Signal | Payload | Trigger |
|--------|---------|---------|
| `fitMethodChanged` | `str` | User selects a fitting method |

**Properties**: `fitting_method` (returns current slug), `set_fitting_method(method)`

The combo is populated from `VALID_FIT_METHODS` (defined in `slacwire/_constants.py`). Methods not yet implemented are disabled (greyed out). Currently implemented: `"gaussian"`, `"asymmetric_gaussian"`, `"super_gaussian"`.

### QTextEditLogger

**File**: `widgets/text_logger.py`

Bridges Python's `logging` module to a `QTextEdit` widget. Log records are formatted and delivered to the widget via a `pyqtSignal` to ensure thread-safe updates.

**Format**: `HH:MM:SS - LEVEL - message`

### ErrorDialog

**File**: `widgets/error_dialog.py`

Resizable `QDialog` shown by `on_scan_failure()`. Replaces `QMessageBox`, which has a fixed size and makes long tracebacks hard to read. Shows a critical icon, the error summary, and the full traceback in a read-only, scrollable, monospace text area. A size grip is enabled.

---

## Configuration

### wire_scan_gui.yaml

Defines the beampath → area → wire hierarchy. Each wire entry is formatted as `"WIRE_NAME:AREA"`:

```yaml
CU_HXR:
    IN20: ['WS01:DL1', 'WS02:DL1', 'WS03:DL1', 'WS04:DL1']
    LI28: ['WS27644:L3', 'WS28144:L3', 'WS28444:L3', 'WS28744:L3']
    LTUH: ['WS31:LTUH', 'WS32:LTUH', 'WS33:LTUH', 'WS34:LTUH']
SC_BSYD:
    HTR: ['WS0H04:HTR']
    COL1: ['WSC104:COL1', 'WSC106:COL1', ...]
```

To add a wire to the GUI, add its `"NAME:AREA"` entry under the appropriate beampath and area. Also ensure the wire exists in `WIRE_AREA_LOOKUP` in `suite.py`.

### wire_scan_gui.ui

Qt Designer XML file defining the static layout:

- **Header frame** — "LCLS Physics Applications" branding + PyDM labels for system time and machine mode
- **Controls panel** (left side):
  - `ParametersGroupBox` — PyDM checkboxes (use x/y/u wire) and line edits (inner/outer ranges, scan pulses) bound to EPICS PVs
  - `ScanGroupBox` — Start/Abort buttons + status text area
- **Plot frame** (right side) — populated programmatically with `PlotWidget` and logbook button
- **Footer** — Save/Load Data and Save/Load Config buttons

### PyDM Channel Binding

When a wire is selected, `update_parameters()` iterates over all widgets in `ParametersGroupBox` and sets their `.channel` property to the corresponding PV name from the wire's `controls_information.PVs`:

```python
pv_obj = getattr(wire.controls_information.PVs, widget_name, None)
if pv_obj is not None:
    child.channel = pv_obj.pvname
```

Widget object names in the `.ui` file must match the PV attribute names on the device (e.g., `x_wire_inner`, `use_y_wire`, `scan_pulses`). Widgets whose names end in `Label` are skipped.

### Wire Speed Status

Because the range fields write straight to the PVs, the device-layer setters never run, so the GUI checks speed itself. `SpeedStatusMonitor` (`widgets/speed_status.py`) polls `wire.range_status(profile)` once a second on a worker thread (a disconnected PV would otherwise block the GUI) and drives a **Speed** column (column 3 of the Parameters grid).

The column is **hidden while no active plane is in alarm**. As soon as any active plane is too fast or too slow, the whole column appears:

| Plane state | Status label | Range fields | Tooltip |
|---|---|---|---|
| Active, too fast | `Too Fast`, red bold | red border | target width, e.g. `Narrow the range to less than 800 um` |
| Active, too slow | `Too Slow`, red bold | red border | target width, e.g. `Widen the range to more than 2917 um` |
| Active, OK | `OK`, green | normal | — |
| Inactive or unreadable | blank | normal | — |

Labels are created at runtime by `build_speed_labels()` and named `{x,y,u}_speed_statusLabel` / `SpeedStatusHeaderLabel` so `update_parameters()` skips them.

A short red state while editing one end of a range is expected; the 1 s poll also acts as a debounce. The activity log gets one line per state change only: a warning when an active plane becomes invalid or unreadable (e.g. `X Profile too slow: range too small. Widen the range to more than 2917 um`), and `X Profile speed OK` when it recovers. The text comes from `speed_summary` / `speed_advice` in `speed_status.py`; the device layer returns only data. The advice never suggests changing scan pulses.

**Start gate**: `start_scan_callback()` refuses to start if `active_wire.invalid_profiles()` is non-empty and logs an error with the fix for each plane. The Start button stays enabled because the remote protocol uses `startButton.isEnabled()` to mean "not scanning". Remote `SCAN` replies `ERROR:<advice>` then `READY` instead of `SCANNING`. Polling is paused while a scan thread runs. The suite's pre-scan check is the final safeguard.

The display and logging rules (`describe_plane`, `transition_level`) live in the Qt-free `slacwire/speed_status.py` so they can be unit-tested without Qt.

---

## Data Flow: Complete Scan Cycle

```
1. User selects beampath/area/wire
   ├── NavigationWidget → areaChanged → MeasurementWidget.update_area()
   ├── MeasurementWidget creates Wire device
   ├── Detector/BPM combos populated from device metadata
   └── PyDM parameter widgets bound to wire EPICS PVs

2. User adjusts parameters via PyDM widgets
   └── Values written directly to EPICS PVs (PyDM handles this)

3. User clicks "Start Scan"
   ├── startButton disabled
   ├── suite.show = False, suite.save_plots = False
   ├── scan_mode = default_scan_mode(wire)  ("step" for WS02, else "otf")
   ├── WireScanSuiteThread spawned
   └── Thread calls suite.run_single(wire, scan_mode=scan_mode)
       └── (Layer 4 → Layer 3 → Layer 2 execution)

4. Scan completes
   ├── Thread emits scan_complete(wire, method, data, entry)
   ├── on_scan_complete():
   │   ├── startButton re-enabled
   │   ├── Run metadata cached in current_runs
   │   ├── Logger records completion
   │   └── dataChanged emitted
   └── dataChanged → update_plots()
       ├── view.draw_trajectory(trajectory_canvas.figure, data, wire, detector)
       ├── trajectory_canvas.draw()
       ├── view.draw_profile(profile_canvas.figure, data, wire, detector, profile)
       └── profile_canvas.draw()

5. User changes profile radio button (X/Y/U)
   └── profileChanged → update_profile_plot() → redraw profile canvas

6. User clicks "Post to Logbook"
   ├── Profile plot saved to plotdir/profile_plot.png
   └── physicselog.submit_entry(logbook, "Wire Scan GUI", title, "", image_path)
```

---

## Physics eLog Integration

The logbook callback:

1. Saves the current profile plot figure to a PNG via `suite.view.save_fig_to_path()`
2. Constructs a title: `"{wire} Scan v. {detector} - {profile} Profile"`
3. Delegates to `suite.post_to_logbook(title, body, attachment)` (defined in `LogbookMixin`)

The `LogbookMixin` handles logbook resolution (SC → `"lcls2"`, CU → `"lcls"`) and calls `physicselog.submit_entry()`. The `physicselog` module is imported dynamically — the GUI degrades gracefully if it's unavailable.

---

## Logging

Two output targets for the `"wire_scan_logger"`:

1. **File** — written to `{outdir}/WireScanLog-YYYY-MM-DD.txt`
2. **Status widget** — the `QTextEdit` in the Scan group, updated in real-time via `QTextEditLogger`

The logger is shared with the suite layer — the `suite/` modules and `registry.py` log to the same `"wire_scan_logger"` name, so their messages appear in the GUI status area.

Uses `slac_measurements.logger.file_logger.custom_logger` for file logging; falls back to a basic `logging.FileHandler` if that module isn't available.

---

## Per-Wire Config Persistence

The GUI persists operator scan preferences per wire/beampath pair in a shared SQLite database.

**Location**: `slacwire/config/wire_config.py`

**Schema**: Table `wire_config` with columns:
- `wire` (TEXT, PK) + `beampath` (TEXT, PK) — composite primary key
- `fitting_method` (TEXT, default `"gaussian"`)
- `detector`, `toroid` (TEXT, nullable)
- `charge_normalization`, `jitter_correction` (INTEGER, boolean)
- `jitter_bpms` (TEXT, nullable)
- `updated_at`, `updated_by` (audit fields)

**API**:
```python
from slacwire.config import WireConfig, get_wire_config, set_wire_config

config = get_wire_config(wire="WS28144", beampath="CU_HXR")
set_wire_config(wire="WS28144", beampath="CU_HXR", fitting_method="asymmetric_gaussian")
```

**Workflow**: When a wire is selected in the GUI, `MeasurementWidget._apply_config()` loads the saved config and applies it to the detector combo, toroid combo, jitter checkbox, charge normalization checkbox, and fit method selector. When the user clicks "Save Config", `MeasurementWidget.save_config()` writes current widget states back to the database.

---

## Launching the GUI

The application is a PyDM Display. Launch with:

```bash
pydm slacwire/ws_gui.py
```

Or from Python:

```python
from pydm import Display
from slacwire.ws_gui import WireScanSuiteGUI
```

---

## Extending the GUI

### Adding a New Widget

1. Create `widgets/my_widget.py` with a `QWidget` or `QGroupBox` subclass
2. Define signals for state changes
3. Export from `widgets/__init__.py`
4. Instantiate in `WireScanSuiteGUI.__init__`
5. Wire signals in the constructor
6. Add to layout in `init_ui()`

### Adding a New Button

1. Add the button in `wire_scan_gui.ui` using Qt Designer (or edit XML directly)
2. Connect in `init_ui()`: `self.ui.myButton.clicked.connect(self.my_callback)`
3. Implement the callback method on `WireScanSuiteGUI`

### Adding a New PyDM Parameter

1. Add a `PyDMLineEdit` or `PyDMCheckbox` to `ParametersGroupBox` in the `.ui` file
2. Set the widget's `objectName` to match the PV attribute name on `Wire.controls_information.PVs`
3. `update_parameters()` will auto-bind the channel on wire selection

### Adding a New Beampath/Wire

1. Add entries to `wire_scan_gui.yaml`
2. Ensure the wire name exists in `WIRE_AREA_LOOKUP` in `suite/_constants.py`
3. No code changes needed — navigation is data-driven

---

## Design Decisions

### Why PyDM?

PyDM is the standard GUI framework for LCLS high-level applications. It provides:
- Automatic EPICS channel connections via widget properties
- Stylesheet and macro support
- Standard look-and-feel matching other LCLS tools
- Built-in alarm handling and display archiving

### Why a shared suite instance?

The GUI needs access to suite state (results, registry, view) after the scan thread completes. Sharing the instance avoids serializing large numpy arrays across thread boundaries. Thread safety is maintained by restricting plotting to the main thread.

### Why disable save_plots during threaded scans?

Matplotlib is not thread-safe. Creating and saving figures on the worker thread while the main thread renders to canvases causes segfaults. Instead, the GUI saves plots explicitly via `view.save_fig_to_path()` when the user clicks "Post to Logbook", and the suite handles automated saves only when running headless (non-GUI).

### Why dynamic imports for physicselog and lcls_tools?

These packages may not be installed in all environments (development, testing, CI). Dynamic imports with fallbacks allow the GUI to degrade gracefully rather than crash on import.

---

## Related Documentation

- [Architecture Overview](01-architecture-overview.md)
- [Device Layer Deep Dive](02-device-layer.md)
- [Measurement Layer Deep Dive](03-measurement-layer.md)
- [Suite Layer Deep Dive](04-suite-layer.md)
