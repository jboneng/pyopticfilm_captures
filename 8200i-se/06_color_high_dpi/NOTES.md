# Session notes — 06_color_high_dpi

- Date: 2026-07-29 ~17:05 local
- Windows build: Windows 11 25H2, build 26200
- Vendor app + version: **SilverFast 9**
- bcdDevice: `0x0702`; USB address **16**
- USBPcap interface used: `\\.\USBPcap1`, **inject-descriptors enabled**
- Cold power-cycle before session? no
- **DPI: 3600**, colour, **no IR pass**
- Outcome: **ok** — single complete pass
- Capture file: `06_color_3600.pcapng`, **206 MB**, 15647 frames
- Derived output geometry: **4956 × 6626 px**, 24-bit

| Op kind | Count | vs 1800 dpi |
|---------|-------|-------------|
| `bulk_in` | 14074 | 7610 |
| `reg_read` | 937 | 935 |
| `reg_write` | 862 | **862 — identical** |
| `status_probe` | 349 | 350 |
| `bulk_preamble` | 27 | **27 — identical** |
| `bulk_out` | 19 | **19 — identical** |
| `write_0x8c` | 4 | **4 — identical** |

The register-op counts are essentially unchanged from 1800 dpi despite 4× the image
data, which reconfirms the fixed-shape pipeline.

---

## Every prediction confirmed

This capture was run as a genuine test of the model fitted to 1200 and 1800 dpi.
All of it held.

| Prediction | Expected | Observed | |
|------------|----------|----------|---|
| `f = 7200 / dpi` | 2 | 2 | ok |
| `DPISET` = `dpi / 6` | `0x0258` = 600 | **`0x0258`** | ok |
| span = width × `f` | — | 9912 = 4956 × 2 | ok |
| bytes = LINCNT × width × bpp | — | 13252 × 4956 × 3 | ok |
| memory layout unchanged | byte-identical | **byte-identical** | ok |
| motor slope tables unchanged | byte-identical | **byte-identical** | ok |

### Byte-count verification

```
197,030,736 / 13252 LINCNT = 14868 bytes per line   (exact)
14868 / 3                  =  4956 output pixels
9912 span / 2              =  4956                  <- matches
13252 LINCNT / 2           =  6626 output lines
```

`13252 × 4956 × 3 = 197,030,736` exactly. The formulas from
`04_color_1800/NOTES.md` now hold at **three** resolutions.

### The memory layout is confirmed as a per-model constant

This was the riskiest open assumption, and it survives:

```
d0=0a d1=0a d2=0a e0=00 e1=68 e2=0b e3=00 e4=0b e5=01 e6=15 e7=99 e8=15 e9=9a
ea=20 eb=32 ec=20 ed=33 ee=2a ef=cb f0=2a f1=cc f2=35 f3=64 f4=35 f5=65 f6=3f
f7=fd f8=05
```

Byte-identical at 1200, 1800 **and** 3600 dpi, even though a 3600 dpi line is
4956 px versus 1728 px at 1200. So the six segment boundaries can be hardcoded.
Likewise both motor slope tables are unchanged (fast ramp `16de 06db 0540 047e …`,
slow ramp `1fb4 12ff 0fa6 0dfd …`).

### No stagger at 3600

`0x01` is `0x23` for the image scan — bit 4 (`STAGGER` in SANE) is **clear**, same
as at 1800. So no staggering correction is needed at 3600. Whether it appears at
7200 is still untested.

---

## New finding — the per-channel tables are an exposure of `14000 / f`

The three 512-byte tables at `0x10000000` / `0x10004000` / `0x10008000` were an
open question. This session resolves them. The value uploaded immediately before
the image scan is uniform across all 256 entries and equals **`14000 / f`**:

| DPI | `f` | Expected `14000 / f` | Observed |
|-----|-----|----------------------|----------|
| 1200 | 6 | 2333 | `0x091d` = 2333 |
| 1800 | 4 | 3500 | `0x0dac` = 3500 |
| 3600 | 2 | 7000 | **`0x1b58` = 7000** |
| — (native) | 1 | 14000 | `0x36b0` = 14000, the 4-byte upload at init |

So `14000` is the native line period and the per-channel table scales it by the
decimation factor — i.e. exposure per delivered line is constant in physical terms.
The 4-byte `0x36b0` upload during init is the `f = 1` case, which is a nice
consistency check.

Caveat: only the upload **immediately preceding the image scan** follows this
cleanly. Earlier uploads in the same session sometimes carry the previous pass's
value (at 1200 dpi the first upload was 3500, not 2333). Also, at 1200 and 1800 the
*first* of the 256 entries is 16 higher than the rest (`0x0dbc` vs `0x0dac`), while
at 3600 all entries are equal. Not explained.

---

## Shading table size — still not a closed form

Third data point acquired, and the rule is still not clean:

| DPI | Output width | `width × 12` | Declared size | size / 512 |
|-----|--------------|--------------|---------------|------------|
| 1200 | 1728 | 20736 | **21064** | 41.14 |
| 1800 | 2478 | 29736 | **30208** | 59.0 |
| 3600 | 4956 | 59472 | **60416** | 118.0 |

The 12-byte period (dark + white 16-bit per channel) holds at all three, and
`60416 = 2 × 30208` exactly, matching `4956 = 2 × 2478`. The 1800 and 3600 values
are exact multiples of 512; the 1200 one is not, and it breaks strict
proportionality (`21064 / 1728 = 12.1898` vs `30208 / 2478 = 12.1904`).

Several candidate rules were tested and all fail on at least one point:
`round_up_8(width × 12)`, `ceil(width × 12 / 512) × 512`,
`round_up_8(width × 12 × 65/64)`, and `12 × (width + ceil(width/64))`.

**Practical impact: low.** The size only needs to be large enough, so allocate
`width × 13` (or replicate the observed values for known DPIs) and move on. Worth
one more look if a 7200 dpi capture ever lands.

---

## Shading coefficients

Dark terms per channel are roughly DPI-independent, as expected for a fixed sensor:

| Session | DPI | Dark R / G / B |
|---------|-----|----------------|
| 04 | 1800 | `0x038b` / `0x0405` / `0x04ac` |
| 05 pass 1 | 1800 | `0x03cb` / `0x04a7` / `0x046f` |
| 06 | 3600 | `0x03a9` / `0x0404` / `0x049f` |

First upload has white terms at unity (`0x2000`); after the shading pass they
become real per-pixel values (`0x2fa4` / `0x2b0b` / `0x2be9` …). Same two-stage
pattern as every other session.

---

## Open questions

- **7200 dpi is still untested**, and it is the only place `f = 1` occurs (no
  vertical averaging at all) and where `STAGGER` might appear. Predictions if
  anyone captures it: `DPISET` = `0x04b0` = 1200, `LINCNT` = output lines exactly,
  span = width exactly.
- Whether a non-integer `f` is possible (e.g. 4800 dpi → `f = 1.5`) or whether the
  vendor only offers divisors of 7200.
- The shading-table padding rule above.
- Carried over and unchanged: the 8-vs-16-bit register (`0x33` or `0xaf`),
  `0x029`:`0x02a`, `0x02e`, `0x120`, and the 34-byte `0x000fff00`/`01` writes.
