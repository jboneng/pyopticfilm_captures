# 03 — Home / preview

## What this is for

Capture **motor, home seek, and status polling** without a full high-res scan.
Usually a preview, autofocus, or “home” action in the vendor UI.

Feeds: status register address/bits, home/park/stop-motor sequences, slope /
feed setup in `gl128.py`.

Suggested file: `03_home_or_preview.pcapng`

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark + USBPcap | Capture |
| SilverFast / Plustek / VueScan | Trigger preview or home |
| Vendor driver | Required |

## How to capture

1. Scanner already open and ready in the vendor app (after a cold boot if needed).
2. Start capture.
3. Trigger **one** preview or home/autofocus action only.
4. Stop as soon as motion finishes / preview appears.
5. Save `03_home_or_preview.pcapng`; note in `NOTES.md` exactly which UI control you used.

If the UI has no explicit home, a short preview is acceptable.

## Safety

If the carriage stalls or grinds: **stop capture, power-cycle**. Do not repeat
failed moves blindly.

## Done when

- Control traffic shows status polling (tight read loops)
- Some motor-related setup (AHB/bulk OUT slope tables and/or feed registers)
- File stays shorter than a full 1800 dpi scan
