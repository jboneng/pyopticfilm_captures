# 07 — Vendor calibration / shading

## What this is for

Optional capture of whatever the vendor UI does for **calibrate**, first-run
shading, or “optimize” — dark/white strip behaviour for later SE calib port.

Suggested file: `07_calib.pcapng` (or split `07_calib_dark.pcapng` /
`07_calib_white.pcapng` if the UI has separate steps).

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark + USBPcap | Capture |
| Vendor app with a calibrate / shading action | Not all UIs expose this |
| Vendor driver | Required |

## How to capture

1. Only if the UI has an explicit calibrate / shading / first-run wizard step.
2. Start capture → run **only** that action → stop.
3. If calibrate is baked into every scan with no separate UI, skip this folder and note that in `NOTES.md`.

## Done when

- Either a dedicated calib capture exists, or notes say “no separate calib UI”
- Any lamp-off then lamp-on strip behaviour is called out

## Priority

Optional. Host-side calib can follow after boot + first color scan work.
