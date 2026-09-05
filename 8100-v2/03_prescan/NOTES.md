# Session notes — 03_prescan

- Capture file: `03_prescan.pcapng`, 40,068,240 bytes (~38.2 MiB), 10,228
  frames, 28.344s
- Integrity: **Clean.** All frames are target-device (`07b3:1824`) traffic,
  no unrelated USB noise.
- Content:
  - 3,394 control transfers (`bRequest`=4 and 12 dominate — vendor GL128
    register read/write, consistent with calibration + status polling),
    plus 4 GET_DESCRIPTOR and 1 SET_CONFIGURATION
  - 6,834 bulk transfers, ~39.4 MB total payload. Size distribution includes
    512-byte transfers (likely per-channel exposure/table uploads, matching
    the pattern in `02_cold_boot_open` and the SE dataset), several
    3,072-byte transfers (AFE offset/gain-sized, matching SE session
    conventions), and large chunks at 11,776 / 12,288 / 24,064 / 24,576 /
    25,088 / 61,952 bytes (calibration strips + preview image data)
  - Tail of capture (last ~1s, frames ~10,219–10,228) is a run of small
    control transfers alternating 8-byte/2-byte payloads — consistent with
    status polling while the carriage settles/returns home
- Action performed: SilverFast already open (warm session, no power-cycle),
  started capture first, then pressed Prescan, allowed preview to complete
  and carriage to return home before stopping
- Outcome: completed normally, no errors, no mechanical anomalies observed
- Any unusual event: none
- Date/time: 2026-09-05, ~22:47 local
