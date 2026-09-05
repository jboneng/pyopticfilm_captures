# Session notes — 07_ppi_ladder

Modeled on the 8200i SE dataset's
[13_ppi_ladder](https://github.com/jboneng/pyopticfilm_captures/blob/main/8200i-se/13_ppi_ladder)
session, but captured as a **single continuous capture** (not one file per DPI)
covering a preview followed by a color scan at every DPI **except 7200** (already
covered by session 04, the primary 7200 dpi reference).

- Capture file: `07_ppi_ladder.pcapng`, 562,829,880 bytes (~537 MiB), 98,544
  frames, 321.718s
- USBPcap interface used: same confirmed interface as prior sessions
- Integrity: **Clean.** All frames are target-device (`07b3:1824`) traffic,
  no unrelated USB noise.
- Content: 11 clearly separated segments (idle gaps of 3.2–7.7s between
  each), matching the intended preview + 10-DPI sequence, with segment
  duration increasing monotonically as expected with rising DPI:

  | Segment | Time range (s) | Duration (s) | Expected step |
  |---------|-----------------|---------------|----------------|
  | 1 | 0.00 – 26.08 | 26.08 | Preview |
  | 2 | 29.73 – 44.99 | 15.26 | 150 dpi |
  | 3 | 51.06 – 66.32 | 15.26 | 300 dpi |
  | 4 | 72.55 – 87.81 | 15.26 | 600 dpi |
  | 5 | 95.50 – 112.11 | 16.62 | 720 dpi |
  | 6 | 117.19 – 135.81 | 18.62 | 900 dpi |
  | 7 | 141.91 – 163.93 | 22.02 | 1200 dpi |
  | 8 | 167.15 – 191.88 | 24.73 | 1440 dpi |
  | 9 | 197.58 – 226.37 | 28.79 | 1800 dpi |
  | 10 | 229.57 – 265.13 | 35.55 | 2400 dpi |
  | 11 | 271.20 – 321.72 | 50.52 | 3600 dpi |

  Segment boundaries were derived purely from timing gaps (>1.5s idle between
  bursts of traffic), not from register decode — the DPI labels above are the
  intended sequence, not yet confirmed from `DPISET` register values. Confirm
  by decoding `DPISET` per segment during analysis before treating the DPI
  mapping as certain.

- USBPcap buffer size used: 128 MiB
- Film / holder position: same frame as prior sessions, not moved
- Scan settings: Color, iSRD OFF, HDR OFF, plain output, full frame at each DPI
- Action performed, in order: Preview, then scans at 150 / 300 / 600 / 720 /
  900 / 1200 / 1440 / 1800 / 2400 / 3600 dpi
- Outcome: all 11 steps present and cleanly separated; no dropped-packet
  warning reported (confirm from your own Wireshark session if you checked)
- Any unusual event: none observed in the trace
- Windows build: TODO — fill in
- Date/time: 2026-09-05, ~23:09–23:15 local
