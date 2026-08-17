# Captures — OpticFilm 8200i SE (GL128)

| Field | Value |
|-------|--------|
| Model | Plustek OpticFilm 8200i SE |
| USB | `07b3:1825` |
| ASIC | Genesys GL128 (`bcdDevice` often `0x702`) |
| Goal | Vendor USB traces → fill `model_8200i_se.py` + `gl128.py` |

PlustekLib SE path exists but **motor moves are gated** until capture-faithful
recipes are re-tested. Collect `08`–`12` with the vendor driver only.
**Do not** run GL845 or SE plusteklib home/scan while capturing.

Master guide: [docs/gl128-bringup.md](../../docs/gl128-bringup.md).

**Protocol bible (all findings):** [`PROTOCOL.md`](PROTOCOL.md) — USB framing,
pipeline, motor, IR, geometry, bit depth, and open unknowns.

**SilverFast scan flowcharts:** [`FLOW.md`](FLOW.md) — colour one-pass, iSRD
two-pass, and colour-vs-IR deltas (capture-derived Mermaid).

## Folder map

| Subfolder | Capture | Priority |
|-----------|---------|----------|
| [01_enumerate/](01_enumerate/) | Plug-in / descriptors | Done |
| [02_cold_boot_open/](02_cold_boot_open/) | Soft reset + init blast | Done |
| [03_home_or_preview/](03_home_or_preview/) | Motor / home / status poll | Done |
| [04_color_1800/](04_color_1800/) | Mid-DPI color scan + bulk | Done |
| [05_ir_1800/](05_ir_1800/) | IR / iSRD deltas | Done |
| [06_color_high_dpi/](06_color_high_dpi/) | 3600 color | Done |
| [07_calib/](07_calib/) | Vendor calibrate / shading | Optional |
| [08_midtravel_home/](08_midtravel_home/) | Preview Cancel → park via `AGOHOME` | **Done** |
| [09_y_crop_pair/](09_y_crop_pair/) | Same DPI, top vs bottom Y crop | **Done** |
| [10_back_to_back/](10_back_to_back/) | Two scans without closing app | **Done** |
| [11_bit_depth_1800/](11_bit_depth_1800/) | 24-bit vs 48-bit @ 1800 | **Done** — image always 8-bit USB |
| [12_abort_during_feed/](12_abort_during_feed/) | Mid-feed Cancel — blocked in SilverFast 9 | **Blocked** (evidence pcap present) |
| [13_ppi_ladder/](13_ppi_ladder/) | Colour scans at each SilverFast PPI | **Done** — full list wired |
| [PROTOCOL.md](PROTOCOL.md) | Full protocol & behaviour synthesis | **Reference** |
| [FLOW.md](FLOW.md) | SilverFast colour / iSRD Mermaid flowcharts | **Reference** |
| [CALIB.md](CALIB.md) | AFE search + ASIC shading (Steps 1–3) | **Reference** |
| [MOTOR.md](MOTOR.md) | Decoded feed/home recipes | Reference |
| [decoded/](decoded/) | Hand-extracted JSON / notes | After decode |
| [winusb_dumps/](winusb_dumps/) | `plustekctl dump-regs` after Zadig | Optional |

Motor follow-ups (`08`–`12`) use the **vendor driver + SilverFast** only — do not
run plusteklib home/scan while collecting them.

## Session notes template

Copy into each session folder as `NOTES.md` when you capture:

```markdown
# Session notes

- Date:
- Windows build:
- Vendor app + version (SilverFast / VueScan / …):
- Scanner serial / label (if any):
- bcdDevice:
- USBPcap interface used:
- Cold power-cycle before session? (yes/no)
- DPI / mode / area:
- Outcome (ok / failed / truncated):
- Extra observations:
```

Also keep a root `SESSION_LOG.md` here if you prefer one running log for the whole bring-up day.
