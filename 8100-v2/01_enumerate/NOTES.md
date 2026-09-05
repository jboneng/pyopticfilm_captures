# Session notes — 01_enumerate

- Capture file: `01_enumerate.pcapng`, 1,320 bytes, 14 frames, 26.196s
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
- Date/time: 2026-09-05, ~22:41 local
