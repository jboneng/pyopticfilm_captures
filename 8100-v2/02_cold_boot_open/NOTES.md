# Session notes — 02_cold_boot_open

- Capture file: `02_cold_boot_open.pcapng`, 30,528 bytes, 398 frames, 21.860s
- USBPcap interface used: same single interface confirmed in 01_enumerate
  (fill in exact `\\.\USBPcapN` number)
- Integrity: **Clean.** No unrelated USB devices in the capture.
- Content: three distinct init bursts, separated by idle gaps —
  - Burst 1: frames ~1–6, t=0.000s (initial enumeration-adjacent activity)
  - Burst 2: frames ~7–14, starting t=7.097s
  - Burst 3: frames ~15–142, starting t=18.760s (largest — bulk of register
    init traffic)
  - Burst 4/tail: frames ~143–398, starting t=21.651s through t=21.860s
    (SilverFast device-open register batch)
  - 386 control transfers total: `bRequest`=4 and `bRequest`=12 (vendor
    GL128 register read/write) dominate (90 + 90 = 180), plus 205 unlabeled
    control-transfer continuations, 11 GET_DESCRIPTOR (bRequest=6), 2
    SET_CONFIGURATION (bRequest=9)
  - 12 bulk transfers of exactly 512 bytes each — consistent with motor
    slope table uploads (matches the pattern from the 8200i SE dataset and
    the earlier V2 pilot capture)
  - Device address: 1 throughout
- Action performed: power-cycle scanner, open SilverFast 9, wait for
  frame/scanner-ready view, no prescan run
- Outcome: completed normally, no errors, no mechanical anomalies
- Any unusual event: none
- Windows build: TODO — fill in
- Date/time: 2026-09-05, ~22:44 local
