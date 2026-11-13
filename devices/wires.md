# SLAC Wires

Wire scanners are a type of diagnostic that measure transverse beam profiles by passing a thin wire into the beam path.

TODO: finish info on current/radiation detection.

## Scan speed

The IOC calculates each plane's on-the-fly scan speed from the range width, beam rate and scan pulses (`speed = beam_rate × (outer − inner) / scan_pulses`), and publishes it as `WIRE{X,Y,U}SPEED_CALC` (µm/s). A plane can be scanned only when

```
max(motor minimum speed VBAS, MPS speed limit MPSSPEED) < speed < motor maximum speed VMAX
```

Both limits are exclusive. In practice beam rate is fixed and operators change the range: a range that is too wide makes the wire too fast; one that is too narrow makes it too slow. See the [device layer guide](../developer-guides/wire-scanner/02-device-layer.md#speed-validation) for `range_status()`.

There are multiple types of wires in the LCLS-SC and LCLS-CU beam lines. 

## Fast wires
TODO: add info / refs

## Slow wires
TODO: add info / refs

