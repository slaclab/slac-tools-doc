# BYKIK Rate Mask

Timing masks restrict which beam shots a CU timing buffer (EDEF) records. slac-timing uses them so that buffers in the LTU only record shots that get past the BYKIK/BYKIKS kickers in the BSY.

When BYKIK is kicking, only a fraction of the CU beam reaches the LTU. At a 120 Hz beam rate with BYKIK at 30 Hz, only 1 shot in 4 gets through. A buffer without a rate mask records all 4, and the 3 beam-less shots carry only detector baseline. On a wire scan they are interleaved with the real signal and distort the profile fit.

## Masks applied

`create_buffer()` chooses the masks from the beampath and the area:

| beampath | area | BYKIK rate PV | inclusion masks |
|---|---|---|---|
| CU_HXR | LTUH | `IOC:BSY0:MP01:BYKIK_RATE` | `pockcel_perm`, `RATE_MPS_HXR_<rate>` |
| CU_HXR | other | — | `pockcel_perm` |
| CU_SXR | LTUS | `IOC:BSY0:MP01:BYKIKS_RATE` | `pockcel_perm`, `RATE_MPS_SXR_<rate>` |
| CU_SXR | other | — | `pockcel_perm` |

The rate bit is applied only in LTUH and LTUS, matching the MATLAB wirescan GUI (`whatEDefParams`). SC buffers (`BSABuffer`) don't use these masks.

The BYKIK rate PV is read as a string and mapped as follows:

| BYKIK rate | rate mask | `rate_limit_hz` |
|---|---|---|
| `1 Hz` / `10 Hz` / `30 Hz` / `60 Hz` | `RATE_MPS_<path>_{01,10,30,60}HZ` | 1 / 10 / 30 / 60 |
| `120 Hz` | none | `None` |
| `0 Hz` | — | raises `NoBeamError` |
| anything else (`Invalid`, `Unknown`, ...) | none, with a logged warning | `None` |

If the rate PV can't be read, reservation raises `ReservationError`.

## Reserving a buffer

Pass the device's `area` to `create_buffer()`. Without it, no rate mask is applied.

```python
from slac_timing import create_buffer, NoBeamError

try:
    buf = create_buffer(
        beampath="CU_HXR",
        n_measurements=400,
        user="operator",
        area="LTUH",
    )
except NoBeamError:
    ...  # BYKIK is at 0 Hz: no beam past the BSY, don't start the scan

with buf:
    buf.start()
    buf.wait(timeout=30)
    data = buf.get("PMT:LTUH:756:QDCRAW")
```

`buf.rate_limit_hz` is the rate of shots the masks let through, or `None` if the buffer sees the full beam rate. Use it wherever you compute how long acquisition takes or how many points to expect:

```
recording rate = min(beam_rate, rate_limit_hz)   # beam_rate if rate_limit_hz is None
```

For example, a 400-point buffer at 120 Hz beam with BYKIK at 30 Hz takes about 13 s to fill, not 3.3 s.

## Mask behavior

- **Masks replace the defaults.** At reservation, every INCLUSION and EXCLUSION word is cleared, then exactly the requested masks are set. This drops the IOC's reservation defaults (`pockcel_perm`, and the TS2/TS3/TS5/TS6 exclusions) and any bits left by a previous user. `create_buffer()` adds `pockcel_perm` back explicitly.
- **Masks are set once, at reservation.** The BYKIK rate is read when the buffer is reserved and isn't refreshed. Reserve a new buffer for each scan so a rate change is picked up. Reattaching to an existing EDEF (`number=` given) doesn't touch its masks.
- **Unknown mask names raise.** All names are checked against `PNBN:SYS0:{1..140}:NAME` before anything is written. An unknown name raises `ReservationError` with the missing name.

To set masks directly, construct an `EventDefinition` yourself:

```python
from slac_timing import EventDefinition

edef = EventDefinition(
    name="My EDEF",
    user="operator",
    n_measurements=100,
    beamcode=1,
    inclusion_masks=["pockcel_perm", "RATE_MPS_HXR_30HZ"],
    exclusion_masks=["TS2", "TS3", "TS5", "TS6"],  # restore the timeslot defaults
    rate_limit_hz=30.0,
)
```

`rate_limit_hz` is informational only; it isn't derived from the masks, so set it to match.

## Mask PVs

Mask bit positions come from the `PNBN:SYS0:<i>:NAME` / `:BITP` table. BITP numbering starts at 32 (TS1 = 32 … TS6 = 37). Each bit is written to the 32-bit words `EDEF:SYS0:<n>:INCLUSION{1..5}` / `EXCLUSION{1..5}`, where `word = BITP // 32` and `bit = BITP % 32`.

| Name | BITP | | Name | BITP |
|---|---|---|---|---|
| RATE_MPS_HXR_01HZ | 76 | | RATE_MPS_SXR_01HZ | 86 |
| RATE_MPS_HXR_10HZ | 69 | | RATE_MPS_SXR_10HZ | 84 |
| RATE_MPS_HXR_30HZ | 68 | | RATE_MPS_SXR_30HZ | 82 |
| RATE_MPS_HXR_60HZ | 67 | | RATE_MPS_SXR_60HZ | 81 |
| pockcel_perm | 83 | | | |

For example, an LTUH buffer at 30 Hz has `INCLUSION2 = 0x00080010` (BITP 68 and 83) and every other word 0. To check a live buffer, read back these words, or MATLAB's per-bit view `EDEF:SYS0:<n>:INCM<k>`.

## Wire scans

The wire scan collection passes the wire's `area` through to `create_buffer()` and uses the recording rate to size the buffer, set the step-mode wire speed and set the acquisition timeout. See the [measurement layer guide](../developer-guides/wire-scanner/03-measurement-layer.md#timing-buffer-management-collectionbufferpy) for details.
