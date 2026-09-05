# Session notes — 08_multi_exposure

Modeled on the 8200i SE dataset's
[14_multi_exposure_scans](https://github.com/jboneng/pyopticfilm_captures/blob/main/8200i-se/14_multi_exposure_scans)
session, but the 8100 V2 has no IR channel, so there is no IR variant here —
only the Multi-Exposure (ME) toggle applies. Captured as a **single
continuous capture** covering both DPIs, matching the approach used for
`07_ppi_ladder`.

- Date/time:
- Windows build:
- SilverFast version:
- USBPcap interface used:
- Scanner USB address (if visible):
- USBPcap buffer size used:
- Cold boot / power state before capture:
- Film / holder position: same frame as prior sessions, not moved
- Scan settings: Color, Multi-Exposure **ON**, iSRD OFF, HDR OFF, plain
  output, full frame
- Action performed, in order:
  1. Scan at 1200 dpi with Multi-Exposure enabled
  2. Scan at 7200 dpi with Multi-Exposure enabled
- Outcome (did both scans complete normally? any step skipped/failed?):
- Capture file: `08_multi_exposure.pcapng`, size, frame count, duration
- Any unusual event:

If you can note the approximate elapsed/Wireshark time right before starting
the second (7200 dpi) scan, that will help split the single file into the
two per-DPI segments during analysis.
