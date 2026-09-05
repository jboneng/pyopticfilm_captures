# Session notes — 08_multi_exposure

Modeled on the 8200i SE dataset's
[14_multi_exposure_scans](https://github.com/jboneng/pyopticfilm_captures/blob/main/8200i-se/14_multi_exposure_scans)
session, but the 8100 V2 has no IR channel, so there is no IR variant here —
only the Multi-Exposure (ME) toggle applies. Captured as a single continuous
capture covering both DPIs.

- Capture file: `08_multi_exposure.pcapng`, 1,896,314,064 bytes (~1.77 GiB),
  53,722 frames, 517.163s
- USBPcap interface used: same confirmed interface as prior sessions
- USBPcap buffer size used: 128 MiB
- Integrity: **Clean.** All frames are target-device (`07b3:1824`) traffic,
  no unrelated USB noise. Total bulk payload: 1,893,001,968 bytes (~1.76 GiB).
- Content: 2 clearly separated scan segments (idle gaps of ~5s and ~10.6s
  around setup/DPI-change in the UI):

  | Segment | Time range (s) | Duration (s) | Expected step |
  |---------|-----------------|---------------|----------------|
  | (setup) | 0.00 – ~0.0 | — | initial control frame(s) |
  | 1 | 5.03 – 76.91 | 71.87 | 1200 dpi, Multi-Exposure ON |
  | 2 | 87.49 – 517.16 | 429.68 | 7200 dpi, Multi-Exposure ON |

  Segment 2 duration (~430s) is notably longer than session 04's plain 7200
  dpi scan (~176s, no ME) — consistent with Multi-Exposure taking multiple
  passes per line, but not yet confirmed from register decode (e.g. number
  of exposure passes, whether `LPERIOD`/`EXPOSURE` differ from session 04).
  Confirm during analysis before treating the DPI/ME mapping as certain.

- Film / holder position: same frame as prior sessions, not moved
- Scan settings: Color, Multi-Exposure ON, iSRD OFF, HDR OFF, plain output,
  full frame
- Action performed, in order: scan at 1200 dpi with ME enabled, then scan at
  7200 dpi with ME enabled, in the same capture
- Outcome: both scans present and cleanly separated; no dropped-packet
  warning reported (confirm from your own Wireshark session if you checked)
- Any unusual event: none observed in the trace
- Windows build: TODO — fill in
- Date/time: 2026-09-05, ~23:26 local
