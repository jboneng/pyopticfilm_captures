# Session notes — 04_color_7200

- Capture file: `7200dpi_color.pcapng`, 483,807,032 bytes (~461.5 MiB),
  32,630 frames 
- Date / start timestamp: 2026-11-19 20:45:47 CET
- Duration: 264.254s
- Device: Plustek OpticFilm 8200i, USB `07b3:130d`; USB address 16
- SilverFast 9 / Windows 11 25H2
- Scan settings: 7200 dpi, color, positive
  iSRD/HDR, crop, and output settings were not recorded
- Film/holder position: not recorded
- Content: 4,206 control, 28,420 bulk, and 4 interrupt frames; bulk traffic
  includes 28,382 frames on endpoint `0x81` and 38 on `0x02`
- Integrity: **truncated.** In a `tshark` frame-length check, 7,063 frames
  have a reported length greater than their captured length. The difference
  totals about 398 MB across those frames. 
- Action performed: color scan at 7200 dpi;
- Outcome / carriage return: completed normally, no errors, no mechanical anomalies observed
- Any unusual event: none observed, substantial frame truncation as noted above