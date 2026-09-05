# Session notes — 07_ppi_ladder

Modeled on the 8200i SE dataset's
[13_ppi_ladder](https://github.com/jboneng/pyopticfilm_captures/blob/main/8200i-se/13_ppi_ladder)
session, but captured as a **single continuous capture** (not one file per DPI)
covering a preview followed by a color scan at every DPI **except 7200** (already
covered by session 04, the primary 7200 dpi reference).

- Date/time:
- Windows build:
- SilverFast version:
- USBPcap interface used:
- Scanner USB address (if visible):
- Cold boot / power state before capture:
- Film / holder position: same frame as prior sessions, not moved
- Scan settings: Color, iSRD OFF, HDR OFF, plain output, full frame at each DPI
- USBPcap buffer size used:
- Action performed, in order:
  1. Preview / Prescan
  2. Scan at 150 dpi
  3. Scan at 300 dpi
  4. Scan at 600 dpi
  5. Scan at 720 dpi
  6. Scan at 900 dpi
  7. Scan at 1200 dpi
  8. Scan at 1440 dpi
  9. Scan at 1800 dpi
  10. Scan at 2400 dpi
  11. Scan at 3600 dpi
- Outcome (did every step complete normally? any step skipped/failed?):
- Capture file: `07_ppi_ladder.pcapng`, size, frame count, duration
- Any unusual event:

Record the **actual wall-clock time** (or capture timestamp) at which each DPI's
scan started, if you can note it during capture (e.g. jot down elapsed time shown
in Wireshark's status bar right before clicking each scan) — this makes it much
easier to split the single file into per-DPI segments during analysis later.
