# 8200i SE capture sessions

| Field | Value |
|-------|-------|
| Model | Plustek OpticFilm 8200i SE |
| USB | `07b3:1825` |
| ASIC | Genesys GL128 (`bcdDevice` `0x0702`) |
| Capture app | SilverFast 9 (vendor Windows driver) |
| Host OS | Windows 11 25H2 |

USB traffic was recorded with Wireshark + USBPcap during bring-up of
[pyopticfilm](https://github.com/jboneng/pyopticfilm).

## Session index

| # | Folder | Capture file(s) | Status | Notes |
|---|--------|-----------------|--------|-------|
| 01 | [01_enumerate/](01_enumerate/) | `01_enumerate.pcapng` | Done | [NOTES](01_enumerate/NOTES.md) |
| 02 | [02_cold_boot_open/](02_cold_boot_open/) | `02_cold_boot_open.pcapng` | Done | [NOTES](02_cold_boot_open/NOTES.md) |
| 03 | [03_home_or_preview/](03_home_or_preview/) | `03_home_or_preview.pcapng` | Done | [NOTES](03_home_or_preview/NOTES.md) · [timeline.txt](03_home_or_preview/timeline.txt) |
| 04 | [04_color_1800/](04_color_1800/) | `04_color_1800.pcapng` | Done | [NOTES](04_color_1800/NOTES.md) |
| 05 | [05_ir_1800/](05_ir_1800/) | `05_ir_1800.pcapng` | Done | [NOTES](05_ir_1800/NOTES.md) |
| 06 | [06_color_high_dpi/](06_color_high_dpi/) | `06_color_3600.pcapng` | Done | [NOTES](06_color_high_dpi/NOTES.md) |
| 07 | — | — | Skipped | Calibration captured inline in sessions 03–05 (no separate UI step) |
| 08 | [08_midtravel_home/](08_midtravel_home/) | `08a_preview_cancel_x3.pcapng`, `08b_preview_cancel.pcapng`, `08c_preview_cancel.pcapng` | Done | [NOTES](08_midtravel_home/NOTES.md) |
| 09 | [09_y_crop_pair/](09_y_crop_pair/) | `09a_crop_top_1800.pcapng`, `09b_crop_bottom_1800.pcapng` | Done | [NOTES](09_y_crop_pair/NOTES.md) |
| 10 | [10_back_to_back/](10_back_to_back/) | `10_back_to_back_1800.pcapng` | Done | [NOTES](10_back_to_back/NOTES.md) |
| 11 | [11_bit_depth_1800/](11_bit_depth_1800/) | `11a_color_1800_24bit.pcapng`, `11b_color_1800_48bit.pcapng` | Done | [NOTES](11_bit_depth_1800/NOTES.md) |
| 12 | [12_abort_during_feed/](12_abort_during_feed/) | `12_fast_cancel_feeds_complete.pcapng` | Blocked | [NOTES](12_abort_during_feed/NOTES.md) — SilverFast 9 cannot abort mid-feed |
| 13 | [13_ppi_ladder/](13_ppi_ladder/) | `13_{150,300,600,720,900,1200,1440,1800,2400,3600,7200}ppi_color.pcapng` (11 files) | Done | [NOTES](13_ppi_ladder/NOTES.md) · [decoded_ppi_ladder.json](13_ppi_ladder/decoded_ppi_ladder.json) |

## Decoded fixtures

Machine-readable extracts kept alongside the captures. Regenerate from a
[pyopticfilm](https://github.com/jboneng/pyopticfilm) checkout (`scripts/extract_se_*.py`,
`--root` pointing at this `8200i-se/` directory).

| File | Contents |
|------|----------|
| [decoded/afe_shading_fixture.json](decoded/afe_shading_fixture.json) | AFE/shading constants + session 03/04/05 finals (unit-test fixture) |
| [decoded/afe_shading_summary.json](decoded/afe_shading_summary.json) | Human-readable AFE/shading summary |
| [decoded/ppi_lincnt_feed.json](decoded/ppi_lincnt_feed.json) | Feed1/feed2/LINCNT rows (sessions 03, 04, 09, 13) |
| [decoded/dvdset_choreography.json](decoded/dvdset_choreography.json) | Shading upload timing across sessions |
| [decoded/gl128_tables.json](decoded/gl128_tables.json) | Per-session init regs, RAM uploads, scan tables |
| [decoded/session_05_ir_afe_shading.json](decoded/session_05_ir_afe_shading.json) | IR-pass AFE gains/offsets (session 05) |
| [decoded/dvdset_session04_window.txt](decoded/dvdset_session04_window.txt) | Annotated shading/image register window (session 04) |
| [13_ppi_ladder/decoded_ppi_ladder.json](13_ppi_ladder/decoded_ppi_ladder.json) | Per-PPI register snapshot (150–7200) |

Per-session decode write-ups are in each folder's `NOTES.md`.
