# 8100 V2 capture sessions

| Field | Value |
|-------|-------|
| Model | Plustek OpticFilm 8200i |
| USB | `07b3:130d` |
| ASIC | Genesys GL845 |
| Capture app | SilverFast 9 (vendor Windows driver) |
| Host OS | Windows 11 25H2 |

## Session index

| # | Folder | Capture file(s) | Status | Notes |
|---|--------|------------------|--------|-------|
| 01 | [01_enumerate/](01_enumerate/) | `01_enumerate.pcapng` | Pending | [NOTES](01_enumerate/NOTES.md) |
| 02 | [02_cold_boot_open/](02_cold_boot_open/) | `02_cold_boot_open.pcapng` | Pending | [NOTES](02_cold_boot_open/NOTES.md) |
| 03 | [03_prescan/](03_prescan/) | `03_prescan.pcapng` | Done | [NOTES](03_prescan/NOTES.md) |
| 04 | [04_color_7200/](04_color_7200/) | `04_color_7200.pcapng` | Done | [NOTES](04_color_7200/NOTES.md) — primary 7200 ppi reference |
| 05 | [05_midtravel_home/](05_midtravel_home/) | `05_midtravel_home.pcapng` | Done | [NOTES](05_midtravel_home/NOTES.md) — preview + Cancel mid-travel (50% progress)|
| 06 | [06_ppi_ladder/](06_ppi_ladder/) | `06_ppi_ladder.pcapng` | Done | [NOTES](06_ppi_ladder/NOTES.md) — scan at every DPI except 1440 ppi, 7200 ppi - no prescans|
| 07 | [07_multi_exposure/](07_multi_exposure/) | `07_multi_exposure.pcapng` | Done | [NOTES](07_multi_exposure/NOTES.md) — ME scan at 1800 ppi - IR deactivated, ME scan at 1800 ppi - IR activated |
| 08 | [08_ir_3600/](08_ir_3600/) | `08_ir_3600.pcapng` | Done | [NOTES](08_ir_3600/NOTES.md) — scan at 3600 ppi with IR activated  |
