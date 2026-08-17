# 05 — Infrared / iSRD @ 1800 dpi

## What this is for

Same geometry/DPI as the color scan, with **infrared / iSRD / dust removal**
enabled. Diff against `04_color_1800` to find IR GPIO / lamp differences.

Suggested file: `05_ir_1800.pcapng`

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark + USBPcap | Capture |
| Vendor app with IR / iSRD / infrared mode | Must expose IR |
| Vendor driver | Required |

## How to capture

1. Match **DPI and crop** from session `04` as closely as possible.
2. Enable infrared / iSRD / dust mode (whatever the UI names it).
3. Start capture → run one IR scan → stop promptly.
4. Save here; in `NOTES.md` name the exact UI option and whether white lamp stayed on.

## Done when

- Capture contains a complete IR acquire (control + bulk)
- Notes make a side-by-side with `04` possible (same DPI/crop)

## Decode later

Diff GPIO (`0xa6`–`0xa9`-class or SE equivalents) and lamp power bits vs color session.
