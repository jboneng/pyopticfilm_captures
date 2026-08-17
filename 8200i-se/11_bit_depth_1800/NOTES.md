# Session notes — 11 bit depth @ 1800 dpi

- Date: 2026-08-03
- Vendor app: SilverFast 9
- Files:
  - `11a_color_1800_24bit.pcapng` (~19 MB) — UI **48 → 24 Bit**
  - `11b_color_1800_48bit.pcapng` (~19 MB) — UI **48 Bit HDR RAW**
- Crop / DPI: same small colour crop, **1800 ppi**, iSRD / HDRi **off**
- Outcome: **ok** — registers settled; SilverFast “48-bit” does **not** change
  the USB image bit depth

## Result

Both captures use the **same** on-wire image format:

| Phase | `0x33` | `0xAF` | Host size formula | Wire samples |
|-------|--------|--------|-------------------|--------------|
| Shading / calib | `0x04` | `0x46` | 16-bit path | 16-bit |
| Image START | **`0x1F`** | **`0xFF`** | ``LINCNT×w×3`` | **16-bit LE chunky** (``LINCNT/2`` rows) |

Image-pass register write window (last `0x01=0x23` ± margins) is **byte-identical**
between 11a and 11b. Image bulk preamble size is **15 705 144** bytes
(= ``3196 × 1638 × 3`` = ``1598 × 1638 × 6``).

The only unique write-value diffs are AFE `0x5E` low bytes (run-to-run calib
drift), not depth.

Image geometry (both): `DPISET=300` (= 1800/6), `LINCNT=3196`,
`STRPIXEL=1418`–`ENDPIXEL=7970`, `FEEDL=1`, `0x02=0x30` (`AGOHOME`).

## Interpretation

- **`0x33` / `0xAF` are the depth *register* pair** (confirms session 04 candidates):
  - shading: `0x33=0x04`, `0xAF=0x46`
  - image: `0x33=0x1F`, `0xAF=0xFF`
- SilverFast still sizes the image bulk as ``LINCNT × width × 3`` with those
  DEPTH8 regs, but the **wire samples are 16-bit little-endian chunky RGB**
  (oracle on `11a`: recognizable film frame only when decoded as
  ``(LINCNT/2) × width × 6``). Treating the same bytes as 8-bit is rainbow noise.
- SilverFast **48 Bit HDR RAW** uses the same USB image path as **48 → 24 Bit**.

## Implications for pyopticfilm

- Program image depth regs as capture does (`0x33=0x1F`, `0xAF=0xFF`).
- Decode / allocate with **`usb_image_depth=16`**, chunky layout, and
  ``optical_line_count = LINCNT / 2`` (bulk size stays ``LINCNT × width × 3``).
- Keep **DEPTH16** regs for shading / calib only.

## Open

- Whether **any** host path programs a 16-bit colour image pass on GL128 SE
  (other software, or a SilverFast option not tried). Not blocking 8-bit
  bring-up.
