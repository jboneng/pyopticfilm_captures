# Session notes — 03_prescan

- Capture file: `03_prescan.pcapng`, 40,120,536 bytes (~38.3 MiB),
  11,038 frames
- Date / start timestamp: 2026-11-20 18:59:12
- Duration: 34.744s
- Device: Plustek OpticFilm 8200i, USB `07b3:130d`; USB address 11
- Capture Software / OS: SilverFast 9 / Windows 11 25H2
- Scan settings: iSDR/ME deactivated
- Content: 4,204 control, 6,830 bulk, and 4 interrupt frames; bulk traffic
  includes 6,792 frames on endpoint `0x81` and 38 on `0x02`
- Integrity observation: no frame-level captured-length truncation found.
  Frames use USB device address 11; the VID/PID is present in the device
  descriptor traffic.
- Action performed: prescan activated
- Outcome / carriage return: completed normally, no errors, no mechanical anomalies observed
- Any unusual event: none observed