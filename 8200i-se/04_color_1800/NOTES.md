# Session notes — 04_color_1800

- Date: 2026-07-29 ~16:40 local
- Windows build: Windows 11 25H2, build 26200
- Vendor app + version: **SilverFast 9**
- bcdDevice: `0x0702`; USB address **16**
- USBPcap interface used: `\\.\USBPcap1`, **inject-descriptors enabled**
- Cold power-cycle before session? no
- **DPI: 1800**, colour (iSRD/IR off)
- **Reported output: 2474 × 1639 px, ~12 MB** → 24-bit RGB (8 bits per channel)
- Outcome: **ok** — full calibration + scan captured
- Capture file: `04_color_1800.pcapng`, 53.9 MB, 9182 frames

| Op kind | Count |
|---------|-------|
| `bulk_in` | 7610 (53,153,316 bytes) |
| `reg_read` | 935 |
| `reg_write` | 862 |
| `status_probe` | 350 |
| `bulk_preamble` | 27 |
| `bulk_out` | 19 |
| `write_0x8c` | 4 |

The op counts are almost identical to session 03 (935/862 register ops in both),
confirming the driver runs a **fixed-shape pipeline** regardless of DPI — only
register *values* change, not the sequence.

---

## Checklist item G — the DPI → geometry relationship is solved

Comparing this session against session 03 (which was a 1200 dpi preview) gives two
data points, and every relation below is **exact, with no remainder**.

Let `f = 7200 / dpi` be the decimation factor (7200 dpi is the SE's native optical
resolution).

| Quantity | Registers | Formula | 1200 dpi | 1800 dpi |
|----------|-----------|---------|----------|----------|
| `f` | — | `7200 / dpi` | 6 | 4 |
| `DPISET` | `0x2c`:`0x2d` (16-bit BE) | **`dpi / 6`** | `0x00c8` = 200 | `0x012c` = 300 |
| `LINCNT` | `0x25`:`0x26`:`0x27` (24-bit BE) | **`lines × f`** | 4836 = 806 × 6 | 6628 = 1657 × 4 |
| `STRPIXEL` | `0x82`:`0x83`:`0x84` (24-bit BE) | window start, raw units | 242 | 578 |
| `ENDPIXEL` | `0x85`:`0x86`:`0x87` (24-bit BE) | window end, raw units | 10610 | 10490 |
| span | `ENDPIXEL − STRPIXEL` | **`width × f`** | 10368 = 1728 × 6 | 9912 = 2478 × 4 |

### Byte-count verification

```
1800 dpi:  49,272,552 bytes / 6628 LINCNT = 7434 bytes per line
           7434 / 3 bytes per pixel       = 2478 output pixels
           9912 span / 4                  = 2478  <- matches exactly
           6628 LINCNT / 4                = 1657 output lines
           1657 x 2478 x 3                = 12,317,058 = the ~12 MB SilverFast estimated
```

Reported dimensions were 2474 × 1639, so the driver acquires **2478 × 1657** and
crops 4 columns and 18 rows of margin.

### The key structural insight

**The ASIC decimates horizontally but not vertically.** It delivers `LINCNT` lines
— i.e. lines at the *native* 7200 dpi step rate — each already reduced to the
output pixel width by `DPISET`. The host then averages `f` consecutive lines into
one output line. That is why the capture is 49 MB for a 12 MB image: exactly 4×.

`DPISET = dpi / 6` divides by the **6 sensor segments** from the `0xe0`–`0xf8`
memory layout, so `DPISET` is the per-segment resolution.

### Confirmation from the calibration passes

`DPISET` is `0x04b0` = **1200** during every AFE/shading pass, which is
`7200 / 6` — i.e. calibration always runs at full native resolution. The
implied widths then come out exact in both sessions:

| Pass | Read size | LINCNT | span | Bytes / output px |
|------|-----------|--------|------|-------------------|
| AFE offset/gain | 3072 | 1 | 512 | 6.000 |
| Full-width reference | 62268 | 1 | 10378 | 6.000 |
| Shading, pass 1 | 1,903,104 | 128 | 9912 | 6.000 |
| Shading, pass 2 | 1,903,104 | 128 | 9912 | 6.000 |
| **Image** | 49,272,552 | 6628 | 9912 | **3.000** |

Identical ratios in the 1200 dpi session. So calibration and shading are always
**16-bit** (6 bytes/px) and the image came back **8-bit** (3 bytes/px) because
24-bit output was requested.

---

## Only 12 registers depend on DPI

Diffing the fully-configured snapshot of session 03 (1200 dpi) against this one
(1800 dpi) — both taken from SilverFast's own 288-register readback:

```
0x026: 12 -> 19    LINCNT high byte  (0x0012e4=4836 -> 0x0019e4=6628)
0x02a: 13 -> ce    with 0x029=0x2c:  0x2c13 -> 0x2cce
0x02c: 00 -> 01    DPISET high  (0x00c8=200 -> 0x012c=300)
0x02d: c8 -> 2c    DPISET low
0x02e: 0f -> 05
0x037: b0 -> f0    NOT DPI-dependent - see below
0x05e: 25 -> 24    leftover AFE low byte, not meaningful
0x083: 00 -> 02    STRPIXEL  (242 -> 578)
0x084: f2 -> 42
0x086: 29 -> 28    ENDPIXEL  (10610 -> 10490)
0x087: 72 -> fa
0x120: 00 -> 0c
```

Everything else is identical. `0x27` happens to be `0xe4` at both DPIs, which is
why the LINCNT low byte does not appear.

**`0x037` is not actually DPI-dependent** — corrected by session 05, whose first
pass runs at this same 1800 dpi but reads `0xb0` rather than `0xf0`. The driver only
ever *writes* `0xc0` to `0x37`; the hardware supplies the upper nibble on read, and
bit 2 is the IR LED enable. Treat it as a mixed control/status register, not
geometry. That leaves **11** genuinely DPI-dependent registers.

Still unexplained: **`0x029`:`0x02a`** (11283 → 11470), **`0x02e`** (15 → 5) and
**`0x120`**. `0x29`:`0x2a` moves in the same direction as the window, so it may be
a feed or start-line count.

## What does **not** change with DPI — good news for implementation

- **The `0xd0`–`0xf8` memory layout block is byte-identical.** Same six segment
  boundaries, same `0xf8 = 0x05`. It is a per-model constant, not a per-DPI table.
- **Both motor slope tables are byte-identical** across DPI. The fast ramp starts
  `16de 06db 0540 047e 0406 03b1 …` and the slow ramp `1fb4 12ff 0fa6 0dfd …` in
  both sessions, at both `0x1000c000` and `0x10010000`. Also per-model constants.
- The `0x10000000`/`0x4000`/`0x8000` per-channel tables start identically
  (`0dbc` then `0dac` × 255); they only change as a *result* of calibration, not
  because of DPI.
- The whole control-flow sequence, register-op counts and probe pattern are the
  same.

So a GL128 implementation needs **one constant memory-layout table, two constant
slope tables, and a small DPI-parameterised geometry computation.**

---

## Shading table size does scale, but the padding rule is unclear

| DPI | Output width | `width × 12` | Declared size |
|-----|--------------|--------------|---------------|
| 1200 | 1728 | 20736 | **21064** |
| 1800 | 2478 | 29736 | **30208** |

The 12-byte period (dark + white 16-bit per channel) is confirmed in both. But the
padding is inconsistent: 30208 is exactly `59 × 512`, while 21064 is not a
multiple of 512 (`41 × 512 + 72`). The extra bytes are 328 and 472 respectively.
A third data point (session 06, high DPI) should settle the rule; until then,
send the size the vendor sends for a known DPI.

---

## `0x02` is the motor control register (GL843/GL124 bit names fit exactly)

Observed values `0x00`, `0x08`, `0x10`, `0x18`, `0x30` map cleanly onto SANE's
`REG_0x02`:

| Bit | SANE name | Observed use |
|-----|-----------|--------------|
| `0x20` | `AGOHOME` | set **only** for the final image scan — this is what makes the carriage return home automatically afterwards |
| `0x10` | `MTRPWR` | set whenever the carriage actually moves |
| `0x08` | `FASTFED` | set for the fast repositioning move (`0x18`, then `0x08`) |

The first shading pass runs with `0x02 = 0x00` (no motor), the second with `0x10`,
and the image with `0x30`. This directly explains the unexplained return-to-home
at the end of session 03.

---

## Open questions

- **Which register selects 8-bit vs 16-bit output.** **Settled in session 11:**
  pair is **`0x33` / `0xAF`** (`0x04`/`0x46` = 16-bit shading; `0x1F`/`0xFF` =
  8-bit image). SilverFast “48 Bit HDR RAW” does not flip the image pass to
  16-bit USB.
- `0x029`:`0x02a`, `0x02e`, `0x037`, `0x120` — DPI-dependent but unexplained.
- High registers `0x108`–`0x10d` hold non-zero values in this session
  (`0x108=0x1f 0x109=0xc1 0x10a=0xbe 0x10c=0x09 0x10d=0x7e`) but are zero in
  session 03's first snapshot. They look like counter readbacks.
- The shading-table padding rule above.

## Extra observations

- Total bulk IN was 53.15 MB across 7610 transfers, versus 49.27 MB declared for
  the image; the remainder is the calibration and shading reads.
- `0x02e` and `0x037` also differ between the two sessions' *idle* snapshots, so
  they are not purely scan-configuration.
