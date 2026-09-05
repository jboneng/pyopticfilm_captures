# 8100 V2 capture sessions

| Field | Value |
|-------|-------|
| Model | Plustek OpticFilm 8100 V2 |
| USB | `07b3:1824` |
| ASIC | Genesys GL128 (`bcdDevice` `0x0702`) |
| Capture app | SilverFast 9 (vendor Windows driver) |
| Host OS | Windows 11 |

USB traffic was recorded with Wireshark + USBPcap during 8100 V2 driver bring-up in
[pyopticfilm](https://github.com/jboneng/pyopticfilm), for comparison against the
[8200i-se](../8200i-se/) dataset. The 8100 V2 has no infrared channel / iSRD support,
so there is no IR-equivalent session here.

## Session index

| # | Folder | Capture file(s) | Status | Notes |
|---|--------|------------------|--------|-------|
| 01 | [01_enumerate/](01_enumerate/) | `01_enumerate.pcapng` | Done | [NOTES](01_enumerate/NOTES.md) |
| 02 | [02_cold_boot_open/](02_cold_boot_open/) | `02_cold_boot_open.pcapng` | Done | [NOTES](02_cold_boot_open/NOTES.md) |
| 03 | [03_prescan/](03_prescan/) | `03_prescan.pcapng` | Done | [NOTES](03_prescan/NOTES.md) |
| 04 | [04_color_7200/](04_color_7200/) | `04_color_7200.pcapng` | Done | [NOTES](04_color_7200/NOTES.md) — primary 7200 dpi reference |
| 05 | [05_midtravel_home/](05_midtravel_home/) | `05_midtravel_home.pcapng` | Done | [NOTES](05_midtravel_home/NOTES.md) — preview + Cancel mid-travel, modeled on SE session 08 |
| 06 | [06_y_crop_pair/](06_y_crop_pair/) | `06a_crop_top_7200.pcapng`, `06b_crop_bottom_7200.pcapng` | Pending | [NOTES](06_y_crop_pair/NOTES.md) |
| 07 | [07_ppi_ladder/](07_ppi_ladder/) | `07_ppi_ladder.pcapng` | Done | [NOTES](07_ppi_ladder/NOTES.md) — single continuous capture: preview + scan at every DPI except 7200 |
| 08 | [08_multi_exposure/](08_multi_exposure/) | `08_multi_exposure.pcapng` | Pending | [NOTES](08_multi_exposure/NOTES.md) — single continuous capture: ME scan at 1200 dpi then 7200 dpi |

See [README.md](README.md) in this folder for capture settings, physical safety, and
film/SilverFast consistency rules.
