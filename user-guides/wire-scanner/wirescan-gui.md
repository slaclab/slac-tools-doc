# Wire Scan GUI User Guide

**Purpose**: Operator guide for running wire scanner beam profile measurements using the Wire Scan GUI.

**Audience**: LCLS operators and physicists

**Last Updated**: October 13, 2026

---

## Launching the GUI

The Wire Scan GUI is a PyDM application. Launch it with:

```bash
pydm slacwire/ws_gui.py
```

The window opens with the title **"Wire Scan GUI"** and displays the LCLS header with live accelerator time-of-day and machine mode readbacks.

### Launching with Pre-Selected State

Append `--beampath`, `--area`, `--wire`, and/or `--detector` after the display file to open the GUI with specific selections already made. All arguments are optional and applied in order — later arguments are ignored if an earlier one doesn't match an available option.

```bash
# Open on SC_HXR beampath, COL1 area, wire WSC106
pydm slacwire/ws_gui.py --beampath SC_HXR --area COL1 --wire WSC106

# Full pre-configuration including detector
pydm slacwire/ws_gui.py --beampath CU_HXR --area L3 --wire WS28144 --detector PMT29150

# Just pre-select the beampath (area/wire/detector default to first available)
pydm slacwire/ws_gui.py --beampath SC_DIAG0
```

| Argument | Description |
|----------|-------------|
| `--beampath` | Accelerator beamline (`CU_HXR`, `CU_SXR`, `SC_HXR`, `SC_SXR`, `SC_BSYD`, `SC_DIAG0`) |
| `--area` | Machine area (must exist under the selected beampath) |
| `--wire` | Wire scanner name (must exist in the selected area) |
| `--detector` | Detector name (must be available for the selected wire) |

If an argument value is not found in its dropdown (e.g., a wire that doesn't exist in the selected area), a warning is logged and subsequent arguments are skipped. The GUI remains usable with whatever selections were successfully applied.

---

## GUI Layout

The interface is divided into three areas:

| Area | Location | Contains |
|------|----------|---------|
| **Controls Panel** | Left column (fixed width) | Navigation, measurement settings, EPICS parameters, scan controls |
| **Plot Area** | Right column (expanding) | Trajectory plot, profile plot, profile/fit selectors, logbook button |
| **Footer** | Bottom bar | Save Data, Load Data, Save Config, Load Config buttons |

---

## Step-by-Step: Running a Wire Scan

### 1. Select Beampath and Area

The **Navigation Control** group at the top of the controls panel has two dropdown menus:

- **Beampath** — Select the accelerator beamline. Available beampaths:
  - `CU_HXR` — Copper, Hard X-Ray
  - `CU_SXR` — Copper, Soft X-Ray
  - `SC_HXR` — Superconducting, Hard X-Ray
  - `SC_SXR` — Superconducting, Soft X-Ray
  - `SC_BSYD` — Superconducting, Beam Switchyard
  - `SC_DIAG0` — Superconducting, Diagnostic Line

- **Area** — Select the machine area. This dropdown automatically repopulates when the beampath changes. The first area is selected by default.

Changing the beampath updates the area list, which in turn updates the wire list below.

### 2. Select a Wire

The **Measurement Settings** group contains the **Wire** dropdown. When the area changes, this dropdown is repopulated with the wire scanners available in that area.

When a wire is selected:
- A device connection to EPICS is established
- The detector, toroid, and BPM lists are populated from the device metadata
- Saved configuration preferences for this wire/beampath pair are loaded automatically (fitting method, detector, jitter/charge settings)
- The **Parameters** group below updates with live EPICS readbacks for the selected wire

### 3. Choose a Detector

The **Detector** dropdown is populated from the selected wire's metadata. The default detector is pre-selected based on the device configuration.

For beamlines that serve both CU and SC timing, the detector list is automatically filtered to show only detectors relevant to the active beampath.

### 4. Configure Analysis Options (Optional)

#### Jitter Correction

Check **"Apply Jitter Correction"** to enable BPM-based orbit-fit jitter subtraction. This corrects pulse-to-pulse beam position jitter using the wire's configured BPMs.

The **Jitter BPMs** list shows available BPMs. By default, all BPMs are used. To use a subset, select specific BPMs from the list (multi-select is supported).

#### Charge Normalization

Check **"Normalize by Charge"** to normalize detector signals by bunch charge. Select the toroid to use from the **Toroid** dropdown (populated from wire metadata).

Both options can be toggled after a scan completes to see their effect on the existing data without re-scanning (see "Post-Scan Analysis" below).

### 5. Verify Scan Parameters

The **Parameters** group shows live EPICS readbacks for the selected wire:

| Parameter | Description |
|-----------|-------------|
| **Use X/Y/U Wire** | Checkboxes to enable/disable each scan plane |
| **Inner / Outer** | Scan range endpoints for each plane (motor units, microns) |
| **Scan Pulses** | Number of beam pulses to acquire during the scan |

These values are read from and written to the wire scanner's EPICS PVs in real time. Adjust them as needed before starting a scan.

### 6. Start the Scan

Click **Start Scan** to begin an on-the-fly (OTF) wire scan. The button disables while the scan is in progress.

The **Status Update** text area below the button displays log messages in real time, including:
- Scan initialization progress
- Wire position updates during the scan
- Completion or failure notifications

When the scan completes:
- The Start Scan button re-enables
- The trajectory and profile plots update automatically
- An HDF5 data file is saved to the dated output directory (`/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/`)
- The scan is logged in the run registry

If the scan fails, an error dialog appears with the failure message and a "Show Details" button for the full traceback. The Start Scan button re-enables after you dismiss the dialog.

### 7. Abort a Scan

Click **Abort Scan** to stop a scan in progress. This sends an abort command to the wire scanner hardware.

---

## Viewing Results

### Trajectory Plot

The upper plot shows the wire scan trajectory:
- **Left axis** (blue): Wire motor position in microns vs. scan point
- **Right axis** (orange): Detector signal counts vs. scan point

This plot shows the raw data from the scan — the wire's motion through the beam and the detector response.

### Profile Plot

The lower plot shows the fitted beam profile for the selected plane:
- **Dotted line**: Measured detector signal vs. position
- **Solid line**: Fitted curve (Gaussian or selected method)
- **Bottom x-axis**: Stage coordinates (motor units)
- **Top x-axis**: Beam coordinates (transverse position)
- **Annotation box**: Fit parameters — mean, sigma, amplitude, offset

If the selected plane was not measured (e.g., the U wire was not enabled), the plot displays "Profile not measured" in gray text.

### Switching Profiles

Use the **X**, **Y**, **U** radio buttons below the profile plot to switch between scan planes. The default is X. The profile plot updates immediately when a different plane is selected.

### Changing Fit Method

Use the **Fitting Method** dropdown (next to the profile radio buttons) to change the curve fitting algorithm:

| Method | Use Case |
|--------|----------|
| **Gaussian** (default) | Standard symmetric beam profiles |
| **Asymmetric Gaussian** | Profiles with asymmetric tails |
| **Super Gaussian** | Flat-top or non-Gaussian profiles |

Additional methods (RMS Raw, RMS Cut Peak, RMS Cut Area, RMS Floor) appear in the list but are grayed out — they are not yet implemented.

Changing the fit method **immediately reanalyzes** the current scan data and updates the profile plot. No new scan is needed.

---

## Post-Scan Analysis

After a scan completes, you can adjust analysis settings without re-scanning:

- **Toggle jitter correction** — Check or uncheck "Apply Jitter Correction". The data is reanalyzed and plots update immediately.
- **Change fit method** — Select a different method from the dropdown. The profile plot updates immediately with the new fit.
- **Switch profiles** — Click X, Y, or U to view different planes.
- **Change detector** — Select a different detector from the dropdown. The trajectory and profile plots update.

These operations call `reanalyze()` on the existing collection data, so they are fast and do not require beam.

---

## Saving and Loading Data

### Save Data

Click **Save Data** in the footer to export the current scan result to an HDF5 file. The file is saved automatically to the suite output directory with a timestamped name:

```
/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/Manual_{wire}_{timestamp}.h5
```

A confirmation message appears in the status log.

### Load Data

Click **Load Data** to open a file browser starting at `/u1/lcls/physics/data/wire_scan/`. Select an HDF5 file to load a previously saved scan.

When a file is loaded:
- The GUI automatically navigates to the wire that was scanned (beampath, area, and wire dropdowns update)
- The trajectory and profile plots display the loaded data
- A confirmation message appears in the status log

---

## Saving and Loading Configuration

### Save Config

Click **Save Config** in the footer to persist your current measurement settings for this wire/beampath combination:
- Selected detector
- Selected toroid
- Fitting method
- Jitter correction on/off
- Selected BPMs
- Charge normalization on/off

Settings are stored in a shared SQLite database. The next time you (or another operator) selects this wire on this beampath, the saved settings load automatically.

### Load Config

Click **Load Config** to reload the saved configuration from the database, overwriting any manual changes you made to the measurement settings.

---

## Posting to the Physics eLog

Click **Post to Logbook...** (cyan button above the plot area) to post the current profile plot to the LCLS physics logbook.

The entry is posted with:
- **Title**: `"{wire} Scan v. {detector} - {profile} Profile"` (e.g., "WS28144 Scan v. PMT29150 - X Profile")
- **Image**: The current profile plot as a PNG attachment
- **Logbook**: `lcls2` for SC beampaths, `lcls` for CU beampaths

A confirmation or error message appears in the status log.

---

## Available Wire Scanners

### CU_HXR / CU_SXR (Copper Beamlines)

| Area | Wires |
|------|-------|
| IN20 | WS02, WS03 |
| LI21 | WS11, WS12, WS13 |
| LI28 | WS27644, WS28144, WS28444, WS28744 |
| LTUH (CU_HXR) | WS31, WS32, WS33, WS34 |
| LTUS (CU_SXR) | WS31B, WS32B, WS33B, WS34B |

### SC_DIAG0 (Superconducting Diagnostic)

| Area | Wires |
|------|-------|
| HTR | WS0H04 |
| DIAG0 | WSDG01 |

### SC_BSYD / SC_HXR / SC_SXR (Superconducting Beamlines)

| Area | Wires |
|------|-------|
| HTR | WS0H04 |
| COL1 | WSC104, WSC106, WSC108, WSC110 |
| EMIT2 | WSEMIT2 |
| BYP | WSBP1, WSBP2, WSBP3, WSBP4 |
| SPD (SC_BSYD only) | WSSP1D |
| LTUH (SC_HXR only) | WS31, WS32, WS33, WS34 |
| LTUS (SC_SXR only) | WS31B, WS32B, WS33B, WS34B |

---

## Typical Workflow

1. Select your beampath and area
2. Select the wire scanner
3. Verify scan parameters (ranges, pulses, active planes) in the Parameters group
4. Enable jitter correction and/or charge normalization if desired
5. Click **Start Scan** and wait for completion
6. Review the trajectory plot and profile plots
7. Switch between X/Y/U profiles to inspect each plane
8. Try different fit methods if the default Gaussian fit doesn't match the profile shape
9. Click **Post to Logbook...** to record results
10. Click **Save Config** to remember your settings for next time

---

## Troubleshooting

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| Start Scan button stays grayed out | Scan is still running | Wait for completion or click Abort Scan |
| "Profile not measured" in profile plot | That plane's wire was not enabled | Enable the plane in the Parameters group and re-scan |
| Scan fails with initialization error | Wire scanner hardware not responding | Check the wire scanner IOC and try `wire.initialize()` manually |
| Scan fails with timeout | Beam not delivered during scan window | Verify beam is present and beam rate is correct |
| No detectors in dropdown | Wire metadata missing detectors | Check the device YAML configuration |
| Plots don't update after toggling jitter | No scan data available for this wire | Run a scan first, then toggle jitter correction |
| Save Config has no effect | Database not accessible | Check write access to `/u1/lcls/physics/data/wire_scan/wire_config.db` |

---

## Output Files

All output is saved to `/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/`:

| File | Contents |
|------|---------|
| `OTF_{wire}_{timestamp}.h5` | Full scan data (raw + analyzed) |
| `plots/OTF_Trajectory_{wire}_{timestamp}.png` | Trajectory plot |
| `plots/OTF_Profile_{plane}_{wire}_{timestamp}.png` | Profile plot per plane |
| `WireScanLog-{date}.txt` | Session log file |

The run registry at `/u1/lcls/physics/data/wire_scan/ws_run_registry.json` tracks every scan with its method, wire, beampath, detector, file paths, and status.
