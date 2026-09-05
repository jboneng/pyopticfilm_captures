# Session notes — 01_enumerate

## Attempt 1 — INVALID, discarded

- Wrong USBPcap interface selected (captured a different root hub/controller —
  only 1 stray frame of `07b3:1824` amid unrelated devices). Superseded by
  Attempt 2 below. Details of the misconfiguration are in git history of this
  file if needed later.

## Attempt 2 — VALID

- Capture file: `01_enumerate.pcapng`, 1,320 bytes, 14 frames, 26.196s
- USBPcap interface used: identified via multi-interface probe (the one
  isolated to the scanner's hub — confirm exact `\\.\USBPcapN` number here)
- Integrity: **Clean.** Every frame matches `usb.idVendor == 0x07b3 &&
  usb.idProduct == 0x1824` — no unrelated USB devices captured.
- Content: two full enumeration rounds —
  - Round 1: frames 1–6 (t=0.000s) — GET_DESCRIPTOR (bRequest=6) → device
    descriptor (bcdDevice=0x0702, idProduct=0x1824) → GET_DESCRIPTOR →
    (unlabeled) → SET_CONFIGURATION (bRequest=9) → ack
  - Round 2: frames 7–14 (t=26.19s) — same sequence repeated (Windows
    re-enumeration after driver attach, same pattern as the 8200i SE dataset)
  - Device address: 1 throughout
- Action performed: unplug scanner USB cable, wait ~5s, replug, allow
  enumeration to complete
- Outcome: completed normally, no errors, no mechanical action involved
- Any unusual event: none
- Windows build: TODO — fill in (Settings → System → About)
- Date/time: 2026-09-05, ~22:41 local
