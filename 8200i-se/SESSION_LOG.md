# SE bring-up session log

Running log for the OpticFilm 8200i SE (GL128) capture campaign.
Master guide: [docs/gl128-bringup.md](../../docs/gl128-bringup.md).  
Campaign synthesis (protocol + unknowns): [`PROTOCOL.md`](PROTOCOL.md).

---

## Capture host (verified 2026-07-29)

| Field | Value |
|-------|-------|
| OS | Windows 11 25H2, build 26200 (AMD Ryzen 5 5600) |
| USB host controllers | AMD USB 3.10 eXtensible (`PCI\VEN_1022&DEV_149C`, `PCI\VEN_1022&DEV_43EE`) |
| Root hubs | 2 × `USB\ROOT_HUB30` — **no EHCI/USB 2.0 controller on this machine** |
| Wireshark | 4.6.7 (`C:\Program Files\Wireshark\tshark.exe`) |
| USBPcap | 1.5.4.0, service running, extcap present at `Wireshark\extcap\USBPcapCMD.exe` |
| USBPcap interfaces | `\\.\USBPcap1`, `\\.\USBPcap2` — scanner is on **`\\.\USBPcap1`** (confirmed session 01) |
| PlustekLib | 0.1.0 in `.venv` (Python 3.12.10) |

USBPcap installs as a USB **class** upper filter, so both `ROOT_HUB30` devices are
covered and `-I` / `NonStandardHWIDs` is not required here.

## Device under test (verified 2026-07-29)

| Field | Value | Note |
|-------|-------|------|
| USB ID | `07b3:1825` | Matches `MODEL_8200I_SE` |
| `bcdDevice` | **`0x0702`** (`USB\VID_07B3&PID_1825&REV_0702`) | **Confirms GL128** per SANE `check-usb-chip.c` |
| Instance ID | `USB\VID_07B3&PID_1825\20241113` | `20241113` is the unit's serial/instance suffix |
| Product / driver string | **`Film Scanner (A2F)`** | `docs/gl128-bringup.md` predicted `Film Scanner(A2M)` — actual is **A2F** |
| Driver service | `usbscan` (vendor WIA) | **Not** WinUSB — correct state for vendor captures |
| Topology | `Port_#0007.Hub_#0001`, parent `USB\ROOT_HUB30\5&19446190&0&0` (AMD `DEV_43EE`) | |
| Status | OK / started | |

## Vendor software available

| App | Version | Path | IR / iSRD? |
|-----|---------|------|------------|
| **SilverFast 9** | — | — | **Yes (iSRD)** — recognises the scanner; used from session 02 on |
| Plustek QuickScan | 6.1.0.3 | `C:\Program Files (x86)\Plustek\OpticFilm 8200i\QuickScan_x64.exe` | Unconfirmed |
| VueScan | not installed | — | — |

SilverFast 9 recognises the SE, so it is the app of choice for all sessions
including `05_ir_1800` (iSRD).

QuickScan is also registered in **Startup** — make sure it is not already holding
the device before a cold-boot capture.

---

## Tooling fixes needed for Wireshark 4.6.7

The capture/decode scripts predate this Wireshark version and could not decode
anything until these were fixed (2026-07-29):

| Problem | Fix |
|---------|-----|
| `usb.bRequest` is not a valid field in Wireshark 4.6.7 — `tshark` aborted | Renamed to `usb.setup.bRequest` in `TSHARK_FIELDS` |
| `usb.descriptor_index` is not a valid field either — enumerate filter aborted | Filter replaced (below) |
| `usb.idVendor`/`usb.idProduct` only exist on the **device-descriptor frame**, so the old filter matched 1 frame and every session would decode as empty | New `find_device_addresses()` resolves the VID/PID to a USB address, then filters `usb.device_address == N` |
| `bulk_in_candidates` was derived from observed traffic and wrongly reported control endpoint `0x80` | New `endpoint_descriptors()` parses the real endpoint descriptors; traffic-based guess now requires `transfer_type == 3` |
| Batched register writes: the vendor packs up to 32 `(addr, value)` pairs per transfer, but the decoder returned one op per transfer — **31 of every 32 init registers were silently dropped** | `decode_event_all()` now emits one op per pair; `decode_event()` kept as a first-op wrapper |
| High-address ops (`wValue` bit 8) had the `0x100` flag ignored, so register `0x101` was reported as `0x01` and collided with a real register | Flag now folded into the decoded address for both reads and writes |
| USBPcap logs a submit **and** a completion frame; control IN responses only appear on the completion frame, which has no setup fields — so **every register read value was `None`** and status histograms were empty | New `_merge_completions()` folds completion payloads (and descriptor identity) into the request, keyed off `usb.irp_info` bit 0 |
| `fe_write_path` was decided by mere presence of `0x3a`/`0x3b` vs `0x5d`/`0x5e`, which the init blast fakes — reported a useless `both_…` | Now only counts writes that **follow a `0x51` index write**; correctly reports `gl124_5d5e` for the SE |
| Frontend reconstruction was hardcoded to the GL845 `0x3a`/`0x3b` path, so the SE's frontend table came out empty | Generalised to `extract_frontend(ops, path)`; `extract_frontend_via_3a3b()` kept as a wrapper |
| 28 vendor transfers fell into `vendor_unknown` | Classified as a new `status_probe` kind (`wIndex`-selected 1-byte `REQUEST_REGISTER` read) |
| `→` / `—` in console output crashed on a cp1252 console (`UnicodeEncodeError`) | Replaced with ASCII in printed strings |
| `audit_sane_tables.py` imported `OUTPUT_PIXEL_OFFSET` / `REGISTER_DPISET` from `plusteklib.scan.geometry`, which the multi-model refactor removed | Now reads `MODEL_8200I.register_dpiset_by_dpi` / `.output_pixel_offset_by_dpi`; audit is green again |

**Capture requirement:** because attribution now depends on seeing the device
descriptor, every session where the scanner is *already plugged in* must be
captured with USBPcap's **"Inject already connected devices descriptors into
capture data"** enabled. Without it the extractor cannot attribute any traffic and
says so loudly instead of emitting empty tables.

---

## Session results

| # | Session | Status | File | Notes |
|---|---------|--------|------|-------|
| A | 01_enumerate | **ok** | `01_enumerate.pcapng` (199 kB) | Checklist **A complete**: addr 15, `bcdDevice 0x0702`, bulk IN `0x81`/OUT `0x02` @512, int IN `0x83`, cfg 1 / iface 0 / alt 0 |
| B | 02_cold_boot_open | **ok** | `02_cold_boot_open.pcapng` (86 kB) | Checklist **B + C + D + E**: framing matches PlustekLib; 116-register init blast; FE path is **`gl124_5d5e`** (not GL845); no soft reset; button LED lit, no lamp/motor |
| C | 03_home_or_preview | **ok** | `03_home_or_preview.pcapng` (28.4 MB) | Turned out to be the **whole pipeline**: memory layout, motor slopes, AFE calibration, shading, a 1728x4836 preview and return-to-home. Checklist **D + E + F**. Status register is **`0x101`**; lamp is `0x03` bit 4 |
| D | 04_color_1800 | **ok** | `04_color_1800.pcapng` (53.9 MB) | 1800 dpi, 2474x1639 24-bit. Checklist **G**. Gives the second geometry data point — **DPI -> register formulas now solved exactly** |
| E | 05_ir_1800 | **ok** | `05_ir_1800.pcapng` (107.8 MB) | iSRD = the colour pipeline run **twice**. Checklist **H**: IR is `0x37` bit 2 + white lamp off. `XPASEL` and GPO are *not* used |
| F | 06_color_high_dpi | **ok** | `06_color_3600.pcapng` (206 MB) | 3600 dpi, derived 4956x6626. **Third data point — all geometry predictions held.** Memory layout + slope tables still byte-identical; per-channel exposure = `14000 / f` |
| G | 07_calib | pending | | |
| H | 08_midtravel_home | **ok** | `08a`/`08b`/`08c` preview cancel pcaps | Cancel during **image** after feeds; park via `AGOHOME`. Fast-cancel attempt moved to session 12 |
| I | 09_y_crop_pair | **ok** | `09a_crop_top_1800` + `09b_crop_bottom_1800` | Feed1 always **28292**; feed2 **13128** (top) vs **20232** (bottom). Same LINCNT=3700 / span=10344 |
| J | 10_back_to_back | **ok** | `10_back_to_back_1800.pcapng` (244 MB) | 2 scans × (RGB+IR) = 4 pipelines; each parks via `AGOHOME` then repeats `28292`+`13128` from home — no warm short-feed |
| K | 11_bit_depth_1800 | **ok** | `11a` 48→24 Bit + `11b` 48 Bit HDR RAW | Depth pair confirmed; both UI modes image at `0x33=0x1F`/`0xAF=0xFF` (8-bit USB); shading stays `0x04`/`0x46` |
| L | 12_abort_during_feed | **blocked** | `12_fast_cancel_feeds_complete.pcapng` | SilverFast cannot abort mid-feed; file proves fast Cancel still finishes both feeds |

Decode checklist after sessions A+B+C+D+E: **8/9 ok** (`A_device`, `B_framing`,
`C_cold_boot`, `D_status`, `E_lamp_ir`, `F_motor_home`, `G_color_scan`,
`H_ir_deltas`). Only `I_calib` is missing, and it is keyed to the `07_calib`
session role — sessions 03, 04 and 05 each already contain two full calibration
passes, so `07_calib` is now about *confirming* rather than discovering.

## Key protocol findings so far

1. **PlustekLib's USB framing is reusable as-is** — same vendor requests, same
   `0x22 + (addr << 8)` read index, same `0x100` high-address flag, same `0x55`
   link byte, same `write_0x8c` and bulk-preamble layout (8-byte little-endian
   address + size, verified against 27 preambles).
2. **The SE is a GL124-family register map, not GL845.** Confirmed three ways:
   the frontend path (`0x51`/`0x5d`/`0x5e`), the `0xe0`–`0xf8` memory-layout block,
   and the scan-control semantics of `0x01`/`0x0d`/`0x0f`.
   `GenesysUsbProtocol.write_fe_register()` (GL845 `0x3a`/`0x3b`) must **not** be
   reused — add an SE-specific path and leave GL845 byte-identical to SANE.
3. **The AFE is 16-bit**: `0x5d` = high byte, `0x5e` = low byte. Indices `0x02`–`0x04`
   are per-channel offsets, `0x05`–`0x07` per-channel gains, both searched at
   runtime during calibration rather than loaded from a constant table.
4. **The status register is `0x101`** (high-address read, `wIndex 0x0122`), polled
   311 times in session 03. `Gl128.read_status()`'s `0x41` is wrong — `0x41` is
   never touched. `0x33` is also wrong; it is a clock/enable register.
   `0xbd` is the "data ready" value, `0xa5` means busy/moving.
5. **Lamp is `0x03` bit 4** (`0x30` held during all acquisition).
6. **Infrared is `0x37` bit 2, set by read-modify-write, with `0x03` bit 4 cleared.**
   Nothing else changes. `XPASEL` (`0x03` bit 6) is never set in any session, and
   the GPO block is identical between visible and IR — so the SE does *not* select
   IR via GPO the way SANE's GL845 8200i does. iSRD is simply the whole colour
   pipeline (calibration included) run a second time in infrared, at identical
   geometry, so the two images are pixel-aligned.
7. **Geometry registers are verified exactly**: `LINCNT` = `0x25`:`0x26`:`0x27`
   (24-bit big-endian), `STRPIXEL` = `0x82`–`0x84`, `ENDPIXEL` = `0x85`–`0x87`,
   `DPISET` = `0x2c`:`0x2d`. With `f = 7200 / dpi` (7200 dpi is native):

   ```
   DPISET                 = dpi / 6          (6 = sensor segment count)
   LINCNT                 = lines x f
   ENDPIXEL - STRPIXEL    = width x f
   bytes delivered        = LINCNT x width x bytes_per_pixel
   ```

   Confirmed exactly at **1200, 1800 and 3600 dpi**. The ASIC decimates
   *horizontally* only; it returns `LINCNT` native-rate lines and the host averages
   `f` of them into each output line.
8. **Only 11 registers depend on DPI**, and neither the `0xd0`–`0xf8` memory layout
   nor either motor slope table changes at any tested resolution — they are
   per-model constants and can be hardcoded. See `04_color_1800/NOTES.md` and
   `06_color_high_dpi/NOTES.md`.
9. **The per-channel tables at `0x10000000`/`0x4000`/`0x8000` are a line exposure
   of `14000 / f`**, uniform across all 256 entries: 2333 at 1200 dpi, 3500 at
   1800, 7000 at 3600, and the `0x36b0` = 14000 init upload is the `f = 1` case.
10. **`0x02` is the motor register** with SANE's `REG_0x02` bit names: `0x20`
    `AGOHOME` (set only for the final scan, which is what returns the carriage
    home), `0x10` `MTRPWR`, `0x08` `FASTFED`.
11. **Motor positioning is two constant feeds from home**, not GL845-style
    `starty`. Every colour/IR session: `FEEDL=28292` (`0x02=0x18`→`0x08`), then
    `FEEDL=13704` (`0x02=0x18`), then image with `FEEDL=1` and `0x02=0x30`.
    Preview used `13128` for the second feed. Fast feeds upload the fast slope
    table, start with `0x0f=0x01` only (no `0x0d`), and complete when vendor
    probe `wIndex=0x21` returns `0x04`. Details: [`MOTOR.md`](MOTOR.md).
12. **Scan start is a fixed 6-step recipe**: memory layout, geometry, AFE,
    `0x0d = 0x07`, `0x01 |= SCAN`, `0x0f = 0x01`, then poll `0x101`.
13. **Bulk reads need `wIndex 0x00`, and the image stream uses `wIndex 0x08`.**
    PlustekLib only ever sends `0x01` (RAM write).
14. **No soft reset** on cold boot; registers `0x0e`–`0x10` are never written.
15. **GPO block differs**: SE writes `0xa6`–`0xa9`; GL845's `0x6b`–`0x6f` is unused.
    `0xaf: 0x00 -> 0xff` before a scan is a strong output-enable candidate.
16. **`STAGGER` (`0x01` bit 4) is never set** at 1200, 1800 or 3600 dpi. Only 7200
    remains untested.
17. **Bit depth pair is `0x33` / `0xAF`.** Shading: `0x04` / `0x46` (16-bit).
    Colour image: `0x1F` / `0xFF` (8-bit). Session 11: SilverFast **48 Bit HDR
    RAW** still uses the 8-bit USB image path (host expands to 16-bit in-file).
