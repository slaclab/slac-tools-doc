# Wire Scanner GUI Layer

**Purpose**: Understand the PyDM/Qt GUI that provides the operator interface for wire scanner beam profile measurements at LCLS.

**Audience**: Developers adding GUI features, modifying widget behavior, or debugging operator-facing issues

**Last Updated**: June 18, 2026

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
└── widgets/
    ├── __init__.py            # Public exports
    ├── navigation.py          # NavigationWidget (beampath + area combos)
    ├── measurement.py         # MeasurementWidget (wire, detector, BPMs, options)
    ├── plots.py               # PlotWidget, MplCanvas, ProfileControl
    └── text_logger.py         # QTextEditLogger, attach_logger_to_widget
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
NavigationWidget.areaChanged ──────► MeasurementWidget.update_area()
NavigationWidget.beampathChanged ──► GUI._on_beampath_changed()
MeasurementWidget.wireChanged ─────► GUI.update_parameters()
MeasurementWidget.wireChanged ─────► GUI.update_plots()
MeasurementWidget.detectorChanged ─► GUI._on_detector_changed()
MeasurementWidget.detectorChanged ─► GUI.update_plots()
GUI.dataChanged ───────────────────► GUI.update_plots()
ProfileControl.profileChanged ─────► GUI.update_profile_plot()
```

### Button Callbacks

| Button | Callback | Behavior |
|--------|----------|----------|
| Start Scan | `start_scan_callback` | Spawns `WireScanSuiteThread`, disables button |
| Save Data | `save_callback` | Saves latest result to HDF5 via file dialog |
| Load Data | `load_callback` | Opens HDF5 file and loads result for display |
| Post to Logbook | `logbook_callback` | Saves profile plot PNG, submits to physics elog |
| Save/Load Config | *(disabled)* | Placeholder for future config persistence |

---

## WireScanSuiteThread

A `QThread` subclass that runs the scan off the main thread to keep the GUI responsive.

### Signals

| Signal | Payload | When Emitted |
|--------|---------|--------------|
| `scan_complete` | `(wire_name, method, data, entry)` | Scan succeeded |
| `scan_failed` | `(wire_identifier, error_message)` | Exception during scan |

### Execution Flow

```python
WireScanSuiteThread.run()
    │
    ├── Extract wire_name from "WIRE:AREA" identifier
    ├── Set beampath and detector on shared suite
    ├── suite.run_single(wire=wire_name, scan_mode="otf")
    ├── data = suite.latest_run(wire_name)
    ├── Find matching registry entry
    └── emit scan_complete(wire_name, method, data, entry)
         │
         └── on exception → emit scan_failed(wire_identifier, str(exc))
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
- BPM list (`QListWidget`) — populated from `wire.metadata.tmitloss.upstream`
- Jitter correction checkbox *(currently disabled)*
- Charge normalization checkbox *(currently disabled)*

**Device Creation**: On wire selection, calls `create_wire_fn(area, name)` (injected at construction) to instantiate the `Wire` device object. This connects to EPICS and loads metadata.

### PlotWidget

**File**: `widgets/plots.py`

Container widget holding the trajectory canvas, profile canvas, and profile radio buttons.

**Components**:
- `trajectory_plot` — `MplCanvas` for wire position vs detector plot
- `profile_plot` — `MplCanvas` for beam profile with fitted curve
- `profile_control` — `ProfileControl` with X/Y/U radio buttons

#### MplCanvas

A `FigureCanvasQTAgg` wrapper that creates a matplotlib `Figure` with one subplot. The figure is rendered into by `WireScanView.draw_trajectory()` and `draw_profile()`.

#### ProfileControl

Radio button group (X, Y, U) for selecting which beam profile to display. Emits `profileChanged(str)` on toggle.

### QTextEditLogger

**File**: `widgets/text_logger.py`

Bridges Python's `logging` module to a `QTextEdit` widget. Log records are formatted and delivered to the widget via a `pyqtSignal` to ensure thread-safe updates.

**Format**: `HH:MM:SS - LEVEL - message`

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

Widget object names in the `.ui` file must match the PV attribute names on the device (e.g., `x_wire_inner`, `use_y_wire`, `scan_pulses`).

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
   ├── WireScanSuiteThread spawned
   └── Thread calls suite.run_single(wire, scan_mode="otf")
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

1. Determines logbook name from beampath prefix: `SC_*` → `"lcls2"`, `CU_*` → `"lcls"`
2. Constructs a title: `"{wire} Scan v. {detector} - {profile} Profile"`
3. Saves the current profile plot figure to a temporary PNG
4. Calls `physicselog.submit_entry()` to post to the elog

The `physicselog` module is imported dynamically — the GUI degrades gracefully if it's unavailable.

---

## Logging

Two output targets for the `"wire_scan_logger"`:

1. **File** — written to `{outdir}/WireScanLog-YYYY-MM-DD.txt`
2. **Status widget** — the `QTextEdit` in the Scan group, updated in real-time via `QTextEditLogger`

The logger is shared with the suite layer — the `suite/` modules and `registry.py` log to the same `"wire_scan_logger"` name, so their messages appear in the GUI status area.

Attempts to use `lcls_tools.common.logger.file_logger.custom_logger` first; falls back to a basic `logging.FileHandler` if that package isn't available.

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
