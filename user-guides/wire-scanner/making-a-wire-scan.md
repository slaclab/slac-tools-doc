# Making a Wire Scan

**Purpose**: Measure the transverse beam size at a wire scanner using the Wire Scan GUI, from opening the GUI to posting the result in the logbook.

**Audience**: LCLS operators

**Last Updated**: November 18, 2026

---

## Before You Start

You need:

- **Beam** at the wire, on the beampath you plan to scan. The scan uses beam pulses to measure the profile, so it cannot run without beam.
- **The beam rate you expect.** The wire speed comes from the beam rate. If the rate changes during setup, the speed check changes with it.
- **For CU wires in LTUH or LTUS:** the BYKIK (HXR) or BYKIKS (SXR) rate. The scan records only the shots that pass the kicker. If the kicker rate is 0 Hz, no beam reaches the LTU and the scan will not start.

---

## Quick Version

1. Open the GUI and select **Beampath → Area → Wire**.
2. Check **Use X / Y / U Wire**, the **Inner / Outer** range for each plane, and **Scan Pulses**.
3. Make sure no plane shows **Too Fast** or **Too Slow**.
4. Click **Start Scan** and watch the status log.
5. Check the profile plots for X, Y and U, and read the sigma from each.
6. Click **Post to Logbook...** for each plane you want to record.

The rest of this guide explains each step.

---

## Step 1: Open the GUI

From a terminal on a physics workstation:

```bash
pydm slacwire/ws_gui.py
```

The window is titled **Wire Scan GUI**. The header shows the accelerator time and machine mode.

To open the GUI with a wire already selected, add the beampath, area and wire:

```bash
pydm slacwire/ws_gui.py --beampath CU_HXR --area LI28 --wire WS28144
```

You can also add `--detector PMT29150` to select a detector. If a value isn't found, the GUI logs a warning, skips the remaining arguments, and opens normally.

### Window layout

| Area | Location | Contains |
|------|----------|----------|
| Controls | Left column | Beampath/area/wire selection, detector, analysis options, scan parameters, Start/Abort, status log |
| Plots | Right column | Trajectory plot, profile plot, X/Y/U selector, fit method, **Post to Logbook...** |
| Footer | Bottom bar | Save Data, Load Data, Save Config, Load Config, Export to .mat |

---

## Step 2: Select the Wire

Select from the dropdowns in this order. Each one fills in the next.

1. **Beampath** — the beamline the beam is on.

   | Beampath | Beamline |
   |----------|----------|
   | `CU_HXR` | Copper linac, hard X-ray |
   | `CU_SXR` | Copper linac, soft X-ray |
   | `SC_HXR` | Superconducting linac, hard X-ray |
   | `SC_SXR` | Superconducting linac, soft X-ray |
   | `SC_BSYD` | Superconducting, BSY dump |
   | `SC_DIAG0` | Superconducting, diagnostic line |

2. **Area** — the machine area.
3. **Wire** — the wire scanner. The wire list for each area is in [Available Wires](#available-wires).

When you select a wire, the GUI:

- connects to the wire scanner
- fills in the **Detector**, **Toroid** and **Jitter BPMs** lists for that wire
- loads the saved settings for this wire and beampath, if anyone has saved them (see [Saving Your Settings](#saving-your-settings))
- shows live values for the wire in the **Parameters** box

Wire selection can take a few seconds while the GUI connects.

---

## Step 3: Choose a Detector

The **Detector** dropdown lists the detectors for the selected wire. The default detector is already selected.

| Detector type | What it measures |
|---------------|------------------|
| **PMT** (photomultiplier tube) | Light from the wire as it crosses the beam. The most common choice. |
| **LBLM** (beam loss monitor) | Particles scattered by the wire |
| **TMIT loss** | Drop in beam intensity between upstream and downstream BPMs |

Use the default unless you have a reason to change it. You don't have to choose before scanning: every detector records during the scan, and you can switch detectors afterwards without scanning again.

---

## Step 4: Set Analysis Options (Optional)

Both options can be turned on or off after the scan. The data is re-analyzed straight away, without scanning again.

**Apply Jitter Correction** — Uses BPMs to estimate the beam position on each pulse and removes the position jitter from the profile. The **Jitter BPMs** list selects which BPMs are used. All are used by default, and you can multi-select a subset. At least 2 BPMs need good data. If they don't, the analysis fails with an error.

**Normalize by Charge** — Scales each pulse by its bunch charge from the selected **Toroid**, which removes charge fluctuation from the profile. Pulses with very low charge are excluded. If no toroid data is available, the scan still completes without normalization.

---

## Step 5: Check the Scan Parameters

The **Parameters** box shows the wire scanner's live settings. **Changes take effect on the wire scanner as soon as you enter them.**

| Parameter | Meaning |
|-----------|---------|
| **Use X / Y / U Wire** | Which planes to measure. Uncheck a plane to skip it. |
| **Inner / Outer** | Start and end of the wire travel for each plane, in motor microns. Inner must be less than Outer. |
| **Scan Pulses** | Number of beam pulses across each plane. 350 is the usual value. |

### Choosing the range

The range must cover the **whole beam profile**. The signal should start at baseline, rise through the peak, and return to baseline inside the range. If the profile is cut off at either end, the fitted beam size is not reliable.

If you don't know where the beam is, start from the existing range. After the first scan, check the trajectory and profile plots and adjust the range.

### The speed check

The wire moves at a speed set by the wire scanner IOC:

```
speed = beam rate × (Outer − Inner) / Scan Pulses
```

The speed must stay between a minimum (the motor minimum, or the machine-protection limit if that is higher) and the motor maximum. The GUI checks this once a second for every active plane:

- If all active planes are OK, the **Speed** column stays hidden.
- If any active plane is out of limits, the **Speed** column appears:

| Shows | Meaning | Fix |
|-------|---------|-----|
| **Too Fast** (red) | The range is too wide | Narrow the range. Hover over the range fields to see the width needed, for example *"Narrow the range to less than 800 um"*. |
| **Too Slow** (red) | The range is too narrow | Widen the range, for example *"Widen the range to more than 2917 um"*. |
| **OK** (green) | The plane is within limits | — |

Fix the speed by changing the **range**, not the scan pulses.

A plane can briefly show red while you edit one end of its range. This is normal and clears when both ends are set.

**Start Scan refuses to run while any active plane is Too Fast or Too Slow.** The status log shows the fix for each plane. If the beam rate is 0, every active plane is invalid.

---

## Step 6: Run the Scan

Click **Start Scan**.

- The status log shows the scan type (on-the-fly or step) and its progress, including wire position updates.
- **Start Scan** is greyed out and **Abort Scan** is enabled while the scan runs.
- The GUI stays responsive during the scan.

Most wires use an **on-the-fly** scan: the wire moves through all active planes in one continuous pass. **WS02** uses a **step** scan instead, because its hardware lags in on-the-fly mode. The GUI chooses the scan type automatically.

How long the scan takes depends on scan pulses, beam rate and the number of planes. With BYKIK limiting the LTU rate, the scan takes the same time but records fewer shots.

### Aborting

Click **Abort Scan** to stop. This stops the scan and **retracts the wire**. The log shows *"Scan aborted for {wire}"* and **Start Scan** becomes available again. The GUI does not plot an aborted scan.

### When the scan finishes

- The trajectory and profile plots update.
- The data is saved automatically (see [Where Your Data Goes](#where-your-data-goes)).
- The scan is recorded in the run registry.

If the scan fails, an error window opens with a short message and the full details underneath. You can resize the window. See [If Something Goes Wrong](#if-something-goes-wrong).

---

## Step 7: Read the Results

### Trajectory plot (top)

Shows the whole scan against scan point number:

- **Blue, left axis**: wire position (µm)
- **Orange, right axis**: detector signal

Use this plot to confirm the wire moved through each range and to see where the signal peaks fall. Each active plane should show one clear peak.

### Profile plot (bottom)

Shows one plane's beam profile:

- **Dotted line**: measured detector signal against wire position
- **Solid line**: fitted curve
- **Bottom axis**: wire stage position (motor µm)
- **Top axis**: position in beam coordinates (µm), corrected for the wire's mounting angle
- **Box**: fit results: **mean**, **sigma**, **amplitude** and **offset**

**Sigma is the RMS beam size** for that plane, in µm.

Select **X**, **Y** or **U** under the plot to switch planes. A plane that wasn't scanned shows *"Profile not measured"*.

### Moving around the plot

| Action | Effect |
|--------|--------|
| Scroll wheel or horizontal swipe | Pan left/right |
| Ctrl + scroll (Cmd + scroll on Mac) | Zoom in/out around the cursor |
| Double-click | Reset the view |

Use these to move a peak out from behind the legend. The view resets whenever the plot redraws.

### Is the result good?

Check each plane for these signs of a good result:

- The profile starts and ends at baseline inside the range.
- The fitted curve follows the measured data through the peak and the tails.
- The peak is clearly above the noise.

If the fit doesn't match the profile shape, try another **Fitting Method**:

| Method | Use for |
|--------|---------|
| **Gaussian** (default) | Normal, symmetric profiles |
| **Asymmetric Gaussian** | Profiles with one tail longer than the other |
| **Super Gaussian** | Flat-topped or otherwise non-Gaussian profiles |

Other methods in the list are greyed out because they are not available yet.

You can change these settings after a scan, and each change re-analyzes the existing data straight away, without scanning again:

- fitting method
- detector
- jitter correction on/off
- charge normalization on/off

---

## Step 8: Record the Result

### Post to the logbook

Click **Post to Logbook...** (cyan button above the plots). This posts the **profile plot currently shown** to the physics logbook:

- **Title**: `{wire} Scan v. {detector} - {plane} Profile`, for example *WS28144 Scan v. PMT29150 - X Profile*
- **Logbook**: `lcls2` for SC beampaths, `lcls` for CU beampaths

Each post contains only the plane on screen. To record X and Y, select X and post, then select Y and post again.

The status log shows whether the post succeeded.

### Where your data goes

Every completed scan is saved automatically to the day's folder:

```
/u1/lcls/physics/data/wire_scan/YYYY/MM/DD/
```

| File | Created by |
|------|-----------|
| `OTF_{wire}_{time}.h5` or `Step_{wire}_{time}.h5` | Every completed scan (automatic) |
| `Manual_{wire}_{time}.h5` | **Save Data** button: saves a copy of the result currently shown |
| `Export_{wire}_{time}.mat` | **Export to .mat** button: a file in the MATLAB wirescan GUI format |
| `WireScanLog-{date}.txt` | The day's status log |

Every scan, including failed ones, is listed in the run registry at `/u1/lcls/physics/data/wire_scan/ws_run_registry.json` with its wire, beampath, detector, file path and status.

### Looking at an old scan

Click **Load Data** and select an `.h5` file. The GUI switches to that wire and shows the saved plots. You can change fit method, detector and plane on loaded data the same way as on a new scan.

---

## Saving Your Settings

**Save Config** stores your current choices for **this wire on this beampath**:

- detector
- toroid
- fitting method
- jitter correction on/off, and the selected BPMs
- charge normalization on/off

The settings are stored in a **shared database**. They load automatically the next time anyone selects this wire on this beampath, so saving changes the defaults for every operator.

**Load Config** restores the saved settings and undoes any changes you made since selecting the wire.

Scan ranges and scan pulses are wire scanner settings, not GUI settings. They stay set on the IOC and are not stored by Save Config.

---

## If Something Goes Wrong

| What you see | Likely cause | What to do |
|--------------|--------------|------------|
| *"Scan not started, invalid wire speed"* in the log | An active plane is Too Fast or Too Slow, or there is no beam rate | Adjust the range as the log says ([the speed check](#the-speed-check)). Check the beam rate. |
| Scan fails at the start, mentioning initialization | The wire didn't home or turn on after 3 tries | Check the wire scanner status and IOC. Call the wire scanner expert if it keeps failing. |
| Scan fails with a timeout | The scan didn't collect its pulses in time. Usually no beam, or the beam rate changed. | Confirm beam is at the wire at the expected rate, then retry. |
| Scan fails with *NoBeamError* (CU LTUH/LTUS) | BYKIK/BYKIKS is at 0 Hz, so no beam reaches the LTU | Wait until beam is passing the kicker. |
| Scan fails reserving a buffer (*ReservationError*) | No timing buffers are free, or a timing PV didn't respond | Wait and retry. Other applications may be using all the buffers. |
| *"Insufficient scan range"* | The wire didn't reach the start of a plane's range | Check the trajectory plot and the range settings. Retry. |
| *"Positions all identical"* | The wire didn't move | Check the wire motor and IOC. |
| Fit is poor, or the log warns *"No signal above threshold"* | Beam isn't crossing the range, the detector isn't responding, or the range is wrong | Check the trajectory plot for a peak. Try another detector. Check the range covers the beam. |
| Jitter correction fails (*too few BPMs*) | Fewer than 2 jitter BPMs have good data | Select other BPMs, or turn jitter correction off. |
| *"Profile not measured"* | That plane wasn't enabled | Check **Use … Wire** and scan again. |
| No detectors in the dropdown | The wire's configuration has no detectors | Report it to the software owner. |
| Start Scan stays greyed out | A scan is still running | Wait, or click **Abort Scan**. |
| Logbook post fails | The logbook service is unavailable | Retry later. The data is already saved. |

If a scan fails, the error window has the full details. Include them when you report a problem.

---

## Available Wires

Wires available in the GUI, by beampath and area:

### CU_HXR and CU_SXR

| Area | Wires |
|------|-------|
| IN20 | WS02 (step scan), WS03 |
| LI21 | WS11, WS12, WS13 |
| LI28 | WS27644, WS28144, WS28444, WS28744 |
| LTUH (CU_HXR only) | WS31, WS32, WS33, WS34 |
| LTUS (CU_SXR only) | WS31B, WS32B, WS33B, WS34B |

### SC_DIAG0

| Area | Wires |
|------|-------|
| HTR | WS0H04 |
| DIAG0 | WSDG01 |

### SC_BSYD, SC_HXR and SC_SXR

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

## Related Guides

- [Wire Scan GUI User Guide](wirescan-gui.md) — full reference for every control in the GUI
- [WireScanSuite User Guide](wirescan-suite.md) — running scans from Python scripts and notebooks
- [BYKIK Rate Mask](../../timing/bykik-rate-mask.md) — how the LTU timing buffers handle BYKIK
