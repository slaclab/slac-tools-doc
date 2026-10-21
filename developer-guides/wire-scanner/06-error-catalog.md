# Wire Scan Error Catalog

Complete catalog of errors, exceptions, and failure conditions that can occur during a wire scan.

Any unhandled exception from phases 2–8 propagates to the scan thread's top-level `except` in `slacwire/ws_gui.py`, which emits `scan_failed` and shows a `QMessageBox.Critical` dialog (or prints `ERROR:` in remote mode).

## Phase 1: GUI Startup / Wire Selection

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| Wire not found in beampath/area | `--wire` CLI arg doesn't match YAML hierarchy | `ws_gui.py:198` | Warning (non-fatal) |
| Area/beampath/detector not found | CLI arg mismatch for area, beampath, or detector | `ws_gui.py:206-233` | Warning (non-fatal) |
| Failed to create wire device | `create_wire()` raises or returns None | `measurement.py:201` | Fatal (no device) |
| Pydantic ValidationError | Wire model fields invalid | `reader.py:97-104` | Fatal (returns None) |
| Wire name empty | Empty string to `_resolve_wire_and_area` | `_base.py:132` | ValueError |
| Wire not in WIRE_AREA_LOOKUP | Wire name missing from lookup dict | `_base.py:136` | KeyError |
| Config DB unreachable | SQLite DB inaccessible | `wire_config.py:72` | Silent (returns defaults) |

## Phase 2: Scan Mode Validation

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| Invalid scan_mode | Mode is not `'otf'` or `'step'` | `_run.py:120`, `_collect.py:27`, `_motion.py:33`, `base.py:374` | ValueError |

## Phase 3: Timing Buffer Reservation

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| Unknown beampath | Beampath doesn't start with `CU_` or `SC_` | `factory.py:37` | ValueError |
| Username empty | `getpass.getuser()` returns falsy | `buffer.py:23` | BufferError |
| Invalid beam rate | `beam_rate` is None or non-positive | `buffer.py:72` | ValueError |
| No BSA buffers available | `nfree` PV reports 0 | `bsa_buffer.py:46` | ReservationError |
| BSA reservation timeout | 5s loop exhausted, slot not found | `bsa_buffer.py:61` | ReservationError |
| BSA buffer won't start | `is_complete` stays True >1s after start | `bsa_buffer.py:94` | ReservationError |
| eDef PV unreachable | Any CU reservation PV put returns None | `event_definition.py:47-168` | ReservationError |
| No eDefs available | Available count <1 | `event_definition.py:78` | ReservationError |
| eDef reservation timeout | 5s loop exhausted | `event_definition.py:85` | ReservationError |
| eDef config PV timeout | avgcnt/meascnt/beamcode put returns None | `event_definition.py:97` | ReservationError |
| eDef mask PV timeout | Inclusion/exclusion mask put fails | `event_definition.py:143-168` | ReservationError |
| Buffer not reserved | Calling start/stop/release on unreserved buffer | `buffer.py:77` | ReservationError |
| Log directory missing | `/u1/lcls/physics/data/wire_scan/logs` doesn't exist | `base.py:323` | FileNotFoundError |

## Phase 4: Detector / Device Setup

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| No detectors available | No detectors in metadata, no default | `detector_util.py:92` | RuntimeError |
| Unknown device type | Detector prefix not LBLM/PMT/BPM/IM | `base.py:147` | Warning (skipped) |
| Device creation returned None | `create_*` failed for a detector | `base.py:152` | Warning (skipped) |

## Phase 5: Wire Motion / Initialization

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| OTF init failure | Wire not homed+on after 3 retries (10s each) | `otf.py:36` | RuntimeError |
| Step init failure | Wire not enabled after 3 retries | `step.py:37` | RuntimeError |
| Step position timeout | Wire didn't reach target within 10s / 250 um | `step.py:76` | RuntimeError |
| scan_status PV undefined | Accessing `scan_status` on device without it | `wire.py:256` | AttributeError |
| Speed below MPS limit | Calculated speed too slow for machine protection | `wire.py:540` | ValueError |
| Speed above hardware max | Calculated speed exceeds max | `wire.py:545` | ValueError |
| Invalid scan_pulses | scan_pulses <= 0 | `wire.py:508` | ValueError |
| Invalid range | inner >= outer | `wire.py:510` | ValueError |

## Phase 6: Data Acquisition

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| OTF acquisition timeout | Buffer didn't fill in expected time | `otf.py:29` | TimeoutError |
| Step acquisition timeout | Buffer didn't fill in expected time | `step.py:56` | TimeoutError |
| Invalid buffer point count | `n_measurements` is None or <= 0 | `base.py:302` | RuntimeError |
| Buffer wait timeout | Generic 60s buffer wait exhausted | `buffer.py:109` | TimeoutError |
| Buffer data size mismatch | PV data length != n_measurements after retries | `buffer.py:162` | BufferSizeError |
| Batch size mismatch | `get_many()` PV lengths inconsistent after retries | `buffer.py:204` | BufferSizeError |
| Beamless poll timeout | RBV polling thread didn't finish in 60s | `beamless_step.py:68` | TimeoutError |

## Phase 7: Analysis / Fitting

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| Detector not in metadata | Requested detector missing from collected list | `analysis.py:90` | ValueError |
| Positions all identical | Wire didn't move (min == max position) | `analysis.py:391` | RuntimeError |
| Insufficient scan range | Wire travel didn't reach profile inner bound | `analysis.py:408` | RuntimeError |
| Toroid not in raw_data | Specified charge toroid missing from data | `charge_normalization.py:78` | ValueError (caught, non-fatal) |
| No toroids defined | No toroids in metadata and none specified | `charge_normalization.py:86` | ValueError (caught, non-fatal) |
| No toroid data found | Toroids in metadata but no data collected | `charge_normalization.py:94` | ValueError (caught, non-fatal) |
| Too few BPMs for jitter | Fewer than 2 BPMs with valid data | `jitter_correction.py:47` | ValueError (fatal) |
| No valid BPM data | All BPM data is NaN | `jitter_correction.py:147` | ValueError (fatal) |
| No signal above threshold | Detector signal below triangle threshold | `analysis.py:260` | Warning (falls back to argmax) |
| All-NaN detector data | Detector returns all NaN for a profile | `analysis.py:297` | Warning (skips detector) |
| All pulses insufficient charge | Zero valid pulses after charge masking | `analysis.py:492` | Warning (skips profile) |

## Phase 8: Results / Data Persistence

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| No results for wire | `latest_run` called before any scan | `_results.py:23` | KeyError |
| H5 save failure | Any exception during HDF5 write | `ws_gui.py:651` | Logged (returns None) |
| H5 load failure | Exception during file load | `ws_gui.py:665` | Silent (absorbed) |
| Registry corrupt/inaccessible | JSON decode error or OS error | `registry.py:114` | Warning (non-fatal) |
| Registry write failure | Cannot write to registry file | `registry.py:125` | Warning (non-fatal) |

## Phase 9: Logbook / Remote / Multi-Wire

| Error | Trigger | Source | Severity |
|-------|---------|--------|----------|
| Logbook post failure | `physicselog.submit_entry` raises | `ws_gui.py:706` | Logged (non-fatal) |
| Remote: BUSY | SCAN command while already scanning | `ws_gui.py:557` | Protocol error |
| Remote: missing arg | SCAN/SELECT without wire name | `ws_gui.py:560` | Protocol error |
| Remote: unknown command | Unrecognized command string | `ws_gui.py:589` | Protocol error |
| Remote: wire/detector not found | Wire or detector not in hierarchy | `ws_gui.py:594-604` | Protocol error |
| run_all per-wire failure | Any single wire fails during batch scan | `_run.py:57` | Logged (continues) |
| Buffer release failure | Exception during `buf.release()` in finally block | `base.py:95` | Logged (swallowed) |
