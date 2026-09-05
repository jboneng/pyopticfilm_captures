# Session notes — 05_midtravel_home

- Capture file: `05_midtravel_home.pcapng`, 6,167,976 bytes (~5.9 MiB), 4,790
  frames, 10.516s
- Integrity: **Clean.** All frames are target-device (`07b3:1824`) traffic,
  no unrelated USB noise.
- Content:
  - 3,310 control transfers (`bRequest`=4/12 dominate — register read/write;
    357+1293=1650 of those, rest unlabeled continuations, plus 4
    GET_DESCRIPTOR, 1 SET_CONFIGURATION)
  - 1,480 bulk transfers, sizes matching the calibration/preview pattern
    seen in `03_prescan` (11,776 / 12,288 / 24,064 / 24,576 / 25,088 /
    61,952-byte chunks, plus 512-byte per-channel uploads and 3,072-byte
    AFE-sized transfers) — consistent with a preview cycle that was
    interrupted partway through
  - Tail of capture (last ~1s): a run of alternating 8-byte/2-byte control
    transfers, spaced ~15ms apart — matches the status-polling signature of
    the carriage settling/parking after a cancel, as documented in the SE
    dataset's `08_midtravel_home` session
- Action performed: pressed Preview/Prescan, then pressed Cancel partway
  through the operation (before it finished normally)
- Outcome: capture is structurally consistent with a clean cancel + park —
  confirm from your own observation whether the carriage physically
  recovered/homed correctly and without any grinding/stalling
- Date/time: 2026-09-05, ~23:06 local

Modeled on the 8200i SE dataset's
[08_midtravel_home](https://github.com/jboneng/pyopticfilm_captures/blob/main/8200i-se/08_midtravel_home/NOTES.md)
session. Register-level decode of the cancel sequence belongs in later
analysis, not here.
