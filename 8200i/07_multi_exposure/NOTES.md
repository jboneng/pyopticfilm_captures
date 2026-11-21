# Session notes — 07_multi_exposure

This folder contains two separate captures at 1800 dpi.

- Device: Plustek OpticFilm 8200i, USB `07b3:130d`; USB address 11
  captures
- SilverFast 9 / Windows 11 25H2
- Integrity observation: `tshark` found no frame-level captured-length
  truncation in either file.

| File | Setting | Size (bytes) | Frames | Duration | Capture start |
|------|-----------------|-------------:|-------:|---------:|---------------|
| `07_1800ppi_color_me.pcapng` | 1800 dpi color, multi-exposure | 122,127,016 | 23,464 | 119.196s | 19:22:36 CET |
| `07_1800ppi_color_me_ir.pcapng` | 1800 dpi color, multi-exposure + IR | 183,200,496 | 35,350 | 160.042s | 19:26:04 CET |

Both files contain substantial bulk traffic. The first has 8,784 control,
14,672 bulk, and 8 interrupt frames; the second has 13,330 control,
22,008 bulk, and 12 interrupt frames.