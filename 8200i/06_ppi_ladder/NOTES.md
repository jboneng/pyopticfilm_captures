# Session notes — 06_ppi_ladder

This folder contains separate captures for nine color scan resolutions.
The 7200 dpi capture is in [04_color_7200](../04_color_7200/). 
1440 dpi capture was not recorded.

- Date / start timestamps: 2026-11-20, between 19:10 and 19:20 CET.
- Device: Plustek OpticFilm 8200i, USB `07b3:130d`; USB address 11
- SilverFast 9 / Windows 11 25H2
- Scan settings: DPI as per filenames; iSRD/ME deactivated
- Integrity observation: `tshark` found no frame-level captured-length
  truncation in these nine files.
- Action performed: pressed scan at indicated ppi as per filenames
- Outcome / carriage return: completed normally, no errors, no mechanical anomalies observed
- Any unusual event: none recorded

| File | Size (bytes) | Frames | Duration |
|------|-------------:|-------:|---------:|
| `06_150ppi_color.pcapng` | 8,174,928 | 7,406 | 53.312s |
| `06_300ppi_color.pcapng` | 8,168,992 | 7,316 | 27.035s |
| `06_600ppi_color.pcapng` | 8,174,932 | 7,406 | 29.859s |
| `06_720ppi_color.pcapng` | 11,203,436 | 8,152 | 34.117s |
| `06_900ppi_color.pcapng` | 16,711,220 | 8,678 | 30.342s |
| `06_1200ppi_color.pcapng` | 28,319,860 | 8,780 | 31.001s |
| `06_1800ppi_color.pcapng` | 61,048,500 | 11,502 | 39.889s |
| `06_2400ppi_color.pcapng` | 105,999,132 | 9,846 | 48.981s |
| `06_3600ppi_color.pcapng` | 233,923,168 | 16,108 | 100.833s |

