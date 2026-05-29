# WireScanSuite User Guide

**Purpose**: Run wire scanner beam profile measurements at LCLS using the WireScanSuite programmatic interface or the Wire Scan GUI.

**Audience**: Accelerator operators and beam physicists

**Last Updated**: May 29, 2026

---

## What is WireScanSuite?

WireScanSuite is the primary tool for measuring transverse beam profiles using wire scanners at LCLS. It handles the full workflow from device setup through data acquisition, Gaussian fitting, result storage, and plot generation.

You can use WireScanSuite in two ways:

1. **GUI** — A graphical application for interactive scanning with live plots and physics eLog posting
2. **Scripting** — A Python API for automated or batch scanning from notebooks and scripts

---

## Quick Start (GUI)

### Launching

From a terminal on an LCLS physics workstation:

```bash
pydm slacwire/ws_gui.py
```

### Running a Scan

1. **Select beampath** — Choose the active beampath (e.g., `CU_HXR`, `SC_BSYD`) from the top-left dropdown
2. **Select area** — Choose the accelerator section (e.g., `LI28`, `LTUH`)
3. **Select wire** — Choose the wire scanner from the list (e.g., `WS28144`)
4. **Verify parameters** — Check that the scan ranges, active planes (X/Y/U), and pulse count are appropriate. These fields are live-connected to EPICS and can be edited directly.
5. **Click "Start Scan"** — The scan runs in the background; the UI remains responsive
6. **View results** — Trajectory and profile plots update automatically when the scan completes. Use the X/Y/U radio buttons to switch between profile views.

### Saving and Sharing Results

- **HDF5 data** — Saved automatically to `/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/`
- **Post to eLog** — Click "Post to Logbook" to submit the current profile plot to the physics elog
- **Load previous data** — Click "Load Data" to open and display a saved HDF5 file

---

## Quick Start (Scripting)

```python
from slacwire import WireScanSuite

# Create suite — the wires list defines the default set for batch scans (run_all)
suite = WireScanSuite(
    wires=["WS28144"],
    beampath="CU_HXR",
)

# Run a scan
suite.run_single("WS28144")

# View results
result = suite.latest_run("WS28144")
x_rms, y_rms = result.rms_sizes
print(f"X: {x_rms:.1f} µm, Y: {y_rms:.1f} µm")
```

> **Note**: The `wires` list defines which wires `run_all()` will scan. However, you can pass *any* valid wire name to `run_single()` regardless of whether it appears in the `wires` list. The list is a convenience for batch operations, not a restriction on what the suite can scan.

---

## Scan Modes

WireScanSuite supports two scan modes, selected by the `scan_mode` parameter:

| Mode | Description | When to Use |
|------|-------------|-------------|
| `"otf"` (default) | On-the-fly — wire moves continuously through the beam | Standard beam rates (≥ 60 Hz) |
| `"step"` | Step scan — wire moves to discrete positions | Low beam rates or special measurements |

### Selecting Scan Mode

**GUI**: The GUI currently uses on-the-fly (`otf`) mode.

**Scripting**:
```python
suite.run_single("WS28144", scan_mode="otf")   # continuous motion
suite.run_single("WS28144", scan_mode="step")  # discrete steps
```

---

## Configuring a Scan

### Scan Ranges

Each wire scanner has up to three profile planes: **X** (horizontal), **Y** (vertical), and **U** (longitudinal). Each plane has an inner and outer endpoint that defines the region the wire sweeps through.

- **In the GUI**: Edit the Inner/Outer fields for each plane in the Parameters panel. Values are in motor units (microns).
- **In scripts**: Ranges are configured on the `Wire` device before scanning (the suite handles this internally based on current EPICS settings).

### Active Planes

Enable or disable individual planes to control which profiles are measured:

- **In the GUI**: Check/uncheck the "Use X Wire", "Use Y Wire", "Use U Wire" checkboxes
- **In scripts**: Set `wire.use_x_wire = True`, etc. before running

### Scan Pulses

The number of beam pulses acquired during the scan. Higher values improve statistics but increase scan duration.

- Typical value: **350 pulses**
- **In the GUI**: Edit the "Scan Pulses" field
- The IOC automatically calculates the required wire speed from the pulse count, beam rate, and scan range

### Detector Selection

Wire scanners have one or more detectors that measure the beam-induced signal:

- **PMT** — Photo-multiplier tube (most common)
- **LBLM** — Beam loss monitor
- **TMIT Loss** — Transmitted intensity loss via BPMs

**In the GUI**: Select from the detector dropdown (populated from device configuration).

**In scripts**:
```python
suite = WireScanSuite(
    wires=["WS28144"],
    beampath="CU_HXR",
    detector="PMT29150",  # override default detector
)
```

---

## Understanding Results

### Beam Profile Plots

After a scan completes, two plot types are displayed:

**Trajectory Plot** (left canvas):
- Blue line: wire position (µm) vs scan point number
- Orange line: detector signal vs scan point number
- Shows the full wire motion and where signal was detected

**Profile Plot** (right canvas):
- Blue dots: measured beam profile (detector signal vs wire position)
- Solid line: fitted Gaussian curve
- Bottom axis: stage coordinates (motor position in µm)
- Top axis: beam coordinates (transverse position in µm)
- Annotation box: fit parameters (mean, sigma, amplitude, offset)

### RMS Beam Size

The primary output is the **RMS beam size** (sigma) for each measured plane:
- Extracted from Gaussian fits to the detector signal
- Reported in **microns** (µm)
- Represents the beam width in beam coordinates (corrected for wire installation angle)

### Switching the RMS Detector

A single scan acquires data from all configured detectors simultaneously. You can change which detector provides the reported RMS beam sizes without re-scanning or re-fitting:

```python
result = suite.latest_run("WS28144")

# Default detector (set by device configuration or suite.detector)
print(f"RMS (default): {result.rms_sizes}")

# Switch to a different detector
result.set_rms_detector("TMITLOSS")
print(f"RMS (TMIT): {result.rms_sizes}")

# Switch back
result.set_rms_detector("PMT:LI29:150")
print(f"RMS (PMT): {result.rms_sizes}")
```

This updates `rms_sizes` in place using the existing fit results — no re-analysis required. The available detectors for a given result are listed in `result.metadata.detectors`.

### Accessing Results in Scripts

```python
result = suite.latest_run("WS28144")

# RMS beam sizes
x_rms, y_rms = result.rms_sizes  # in microns

# Fit details for a specific profile and detector
x_fit = result.fit_result["x"].detectors["PMT:LI29:150"]
print(f"Centroid: {x_fit.mean:.1f} µm")
print(f"Sigma:    {x_fit.sigma:.1f} µm")
print(f"Amplitude: {x_fit.amplitude:.1f}")

# Summary of all wires
suite.summary()
```

---

## Running Multiple Wires

### Batch Scanning

Scan all configured wires sequentially:

```python
suite = WireScanSuite(
    wires=["WS27644", "WS28144", "WS28444", "WS28744"],
    beampath="CU_HXR",
)
suite.run_all(scan_mode="otf")
suite.summary()
```

### Selective Re-plotting

Regenerate plots from cached results without re-scanning:

```python
suite.replot("WS28144")
```

---

## Fitting Methods

Three curve-fitting algorithms are available for extracting beam sizes:

| Method | Description | Use Case |
|--------|-------------|----------|
| `"gaussian"` (default) | Standard symmetric Gaussian | Most beam profiles |
| `"asymmetric_gaussian"` | Gaussian with different left/right widths | Asymmetric tails |
| `"super_gaussian"` | Generalized Gaussian (variable exponent) | Flat-top or non-Gaussian beams |

**In scripts** (re-analyze existing data with a different fit):

```python
from slac_measurements.wires import WireMeasurementAnalysis, load_collection_from_h5

raw = load_collection_from_h5("/u1/lcls/physics/data/wire_scan/2026/05/29/OTF_WS28144.h5")
analyzer = WireMeasurementAnalysis(
    collection_result=raw,
    fitting_method="asymmetric_gaussian",
)
result = analyzer.analyze(rms_detector="PMT:LI29:150")
print(f"X RMS: {result.rms_sizes[0]:.1f} µm")
```

---

## Data Storage

### Automatic File Output

Every scan produces files in a dated directory structure:

```
/u1/lcls/physics/data/wire_scan/
└── 2026/
    └── 05/
        └── 29/
            ├── OTF_WS28144_20260529_140000.h5       ← HDF5 data
            ├── plots/
            │   ├── OTF_Trajectory_WS28144_20260529_140000.png
            │   ├── OTF_Profile_x_WS28144_20260529_140000.png
            │   └── OTF_Profile_y_WS28144_20260529_140000.png
            └── WireScanLog-2026-05-29.txt           ← session log
```

**File naming convention**: `{Method}_{Type}_{Wire}_{Timestamp}.{ext}`

### Loading Saved Data

```python
from slac_measurements.wires import load_analysis_from_h5

result = load_analysis_from_h5("/u1/lcls/physics/data/wire_scan/2026/05/29/OTF_WS28144_20260529_140000.h5")
x_rms, y_rms = result.rms_sizes
```

### Run Registry

All scans are logged to a persistent registry at:
`/u1/lcls/physics/data/wire_scan/ws_run_registry.json`

Each entry records: wire name, beampath, scan mode, detector, file paths, timestamp, and status.

---

## Controlling Output

```python
suite = WireScanSuite(
    wires=["WS28144"],
    beampath="CU_HXR",
    save=True,          # write HDF5 files (default: True)
    save_plots=True,    # write PNG plots (default: True)
    show=True,          # display plots interactively (default: True)
)
```

### Custom Output Directory

```python
from pathlib import Path

suite = WireScanSuite(
    wires=["WS28144"],
    beampath="CU_HXR",
    outdir=Path("/tmp/my_scans"),
)
```

---

## Supported Beampaths

| Beampath | Description |
|----------|-------------|
| `CU_HXR` | Copper linac, hard X-ray |
| `CU_SXR` | Copper linac, soft X-ray |
| `SC_HXR` | Superconducting linac, hard X-ray |
| `SC_SXR` | Superconducting linac, soft X-ray |
| `SC_BSYD` | Superconducting, beam switchyard |
| `SC_DIAG0` | Superconducting, diagnostic line |

---

## Available Wire Scanners

Wire scanners are organized by accelerator area:

| Area | Wires | Beampath |
|------|-------|----------|
| DL1 | WS01, WS02, WS03, WS04 | CU_HXR, CU_SXR |
| BC1 | WS11, WS12, WS13 | CU_HXR, CU_SXR |
| L3 | WS27644, WS28144, WS28444, WS28744 | CU_HXR |
| LTUH | WS31, WS32, WS33, WS34 | CU_HXR |
| LTUS | WS31B, WS32B, WS33B, WS34B | CU_SXR |
| HTR | WS0H04 | SC_BSYD |
| COL1 | WSC104, WSC106, WSC108, WSC110 | SC_BSYD |
| EMIT2 | WSEMIT2 | SC_BSYD |
| BYP | WSBP2, WSBP3, WSBP4 | SC_BSYD |
| SPD | WSSP1D | SC_BSYD |
| DIAG0 | WSDG01 | SC_DIAG0 |

---

## Troubleshooting

### Scan fails with "Wire initialization failed"

The wire scanner is not initialized. In the GUI, this should be handled automatically. If the problem persists:
1. Check that the wire IOC is responsive
2. Try reinitializing from the device controls panel

### Scan times out

The timing buffer did not complete within the expected window. Common causes:
- **No beam** — verify beam is being delivered at the expected rate
- **Beam rate mismatch** — the configured beam rate does not match the actual rate, leading to incorrect timeout calculation
- **Wire did not move** — check initialization and motor status

### Profile fit looks wrong

- Try switching detectors — some detectors have better signal-to-noise for specific wires
- Verify the scan range encompasses the full beam profile (signal should rise from and return to baseline)
- Consider using `"asymmetric_gaussian"` fitting if the profile has visible asymmetry

### "No signal above threshold" warning

The analysis could not identify beam signal above background noise. This typically means:
- Beam is not present at the wire location
- The detector is not responding
- The scan range does not overlap with the beam

### GUI is unresponsive during scan

This should not happen — scans run on a background thread. If the GUI freezes:
- The scan is likely stuck during wire initialization
- Use Ctrl+C in the terminal or close the window to abort

---

## Complete Scripting Example

```python
from slacwire import WireScanSuite

# Set up a multi-wire scan session
suite = WireScanSuite(
    wires=["WS27644", "WS28144", "WS28444", "WS28744"],
    beampath="CU_HXR",
    detector="PMT29150",
)

# Run all wires
suite.run_all(scan_mode="otf")

# Print summary table
suite.summary()

# Access individual results
for wire in suite.wires:
    result = suite.latest_run(wire)
    if result.rms_sizes is not None:
        x, y = result.rms_sizes
        print(f"{wire}: X={x:.1f} µm, Y={y:.1f} µm")

# Re-analyze a specific wire with different fitting
from slac_measurements.wires import WireMeasurementAnalysis

raw = suite.latest_run("WS28144").collection_result
analyzer = WireMeasurementAnalysis(
    collection_result=raw,
    fitting_method="super_gaussian",
)
refit = analyzer.analyze()
print(f"Super-Gaussian fit: X={refit.rms_sizes[0]:.1f} µm")
```

---

## Tips

- **Check ranges before scanning** — Ensure the scan range fully contains the beam. A truncated profile gives unreliable RMS values.
- **Use the default detector** unless you have a reason to override. The default is chosen for best signal quality at each wire location.
- **350 scan pulses** is a good default for 120 Hz scans. Increase for better statistics at low beam rates; decrease if scan time is a concern.
- **Re-analyze rather than re-scan** — If you only need a different fit method or detector, load the saved HDF5 and re-analyze without moving the wire again.
- **The registry is your audit trail** — Check `ws_run_registry.json` to find file paths for past scans.
