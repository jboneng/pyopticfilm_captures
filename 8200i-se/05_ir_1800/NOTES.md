# Session notes — 05_ir_1800

- Date: 2026-07-29 ~16:50 local
- Windows build: Windows 11 25H2, build 26200
- Vendor app + version: **SilverFast 9** with **iSRD** enabled
- bcdDevice: `0x0702`; USB address **16**
- USBPcap interface used: `\\.\USBPcap1`, **inject-descriptors enabled**
- Cold power-cycle before session? no
- **DPI: 1800**, same crop as session 04
- Outcome: **ok** — two complete passes captured
- Capture file: `05_ir_1800.pcapng`, **107.8 MB**, two full-image reads
- Observed: **the scanner scans twice** — once visible, once infrared

| Op kind | Count | vs session 04 |
|---------|-------|---------------|
| `bulk_in` | 15220 | 7610 → **exactly 2×** |
| `reg_read` | 1877 | 935 → 2× + 7 |
| `reg_write` | 1710 | 862 → 2× − 14 |
| `status_probe` | 698 | 350 → 2× − 2 |
| `bulk_preamble` | 54 | 27 → **exactly 2×** |
| `bulk_out` | 38 | 19 → **exactly 2×** |
| `write_0x8c` | 8 | 4 → **exactly 2×** |

iSRD is implemented as **the whole colour pipeline run twice**, second time in
infrared. Both passes include their own AFE calibration and shading calibration.

---

## Checklist item H — IR is `0x37` bit 2, plus the white lamp off

Diffing the two passes op-by-op (9375 vs 10230 ops through a sequence matcher)
shows they are structurally identical. Only **two functional differences** exist.

### 1. White lamp off: `0x03` = `0x20` instead of `0x30`

Every scan in pass 2 runs with `0x03` bit 4 clear. This corroborates
session 03's finding that **bit 4 is the white lamp**. Bit 6 (`XPASEL` in SANE) is
**never set in either pass** — so the SE does *not* use `XPASEL` for IR, which was
my prior guess and is now ruled out.

### 2. Register `0x37` bit 2 (`0x04`) is the **IR LED enable**

The visible pass never touches `0x37` after configuration. The IR pass
**read-modify-writes it three times**, toggling exactly bit 2:

| Time | Operation | Value | bit 2 |
|------|-----------|-------|-------|
| 36.957 | `W 0x37 = 0xc0` | `1100 0000` | 0 |
| 37.330 | `W 0x37 = 0xc0` | `1100 0000` | 0 |
| 37.635 | `R 0x37 -> 0xb0` | `1011 0000` | 0 |
| 38.730 | `R 0x37 -> 0xf0` | `1111 0000` | 0 |
| 38.733 | `W 0x37 = 0xf4` | `1111 0100` | **1 — set** |
| 40.043 | `R 0x37 -> 0xb4` | `1011 0100` | 1 |
| 40.047 | `W 0x37 = 0xb0` | `1011 0000` | 0 — cleared |
| 41.280 | `R 0x37 -> 0xb0` | `1011 0000` | 0 |
| 41.285 | `W 0x37 = 0xb4` | `1011 0100` | **1 — set** |
| 45.445 | `R 0x37 -> 0xb4` | `1011 0100` | 1 — still set for the image scan |

Cross-checking every session confirms it: bit 2 of `0x37` is **never set** in
session 03 (1200 dpi preview), session 04 (1800 dpi colour), or pass 1 here. It is
set only during the IR pass, and it is still set when the final IR image is read.

The read-modify-write pattern — read, flip one bit, write back — is exactly how you
would toggle a hardware line without disturbing neighbouring bits, which makes the
interpretation solid.

### Correction to session 04

Session 04's notes listed `0x037: b0 -> f0` among the DPI-dependent registers.
**That was wrong.** Those are *read* values, and pass 1 of this session — same
1800 dpi — reads `0xb0`, not `0xf0`. So `0x37`'s upper bits are not DPI-determined.
The driver only ever *writes* `0xc0` to `0x37` during configuration; the hardware
supplies bits 4–7 on read. Treat `0x37` as a **mixed control/status register**:
bit 2 is the IR LED control, the upper nibble is hardware status.

---

## Everything else is identical

Confirmed byte-for-byte across both passes:

- **The 34-byte writes to `0x000fff00` / `0x000fff01`** — including the lone
  `0x33` at offset 32. Identical in both passes, so still unexplained but
  definitely not IR-related.
- **Both motor slope tables** at `0x1000c000` / `0x10010000` — fast ramp
  (`16de 06db 0540 …`, tail `… 00d1 00d0 00d0`) and slow ramp
  (`1fb4 12ff 0fa6 …`, tail `… 0412 0411`).
- **The per-channel 512-byte tables** at `0x10000000` / `0x10004000` /
  `0x10008000` — `0dbc` then `0dac` × 255 in both passes. So these are **not** LED
  or exposure selection, which rules out my earlier hypothesis about them.
- **All four `write_0x8c` calls** — `(0x10, 0x0c)` and `(0x13, 0x0c)`, twice per pass.
- **The GPO block** `0xa5`–`0xaf` — identical at every scan start. IR does **not**
  go through GPO, unlike SANE's GL845 8200i.
- **Geometry** — `LINCNT` 6628, span 9912, `DPISET` 0x012c, 3 bytes/output pixel.
  The IR image is delivered in the same 3-channel shape as colour.

---

## Checklist item I groundwork — the shading table proves the illumination changed

The shading uploads to `0x10014000` (30208 bytes, same size as session 04) differ
in a way that independently confirms IR:

**Pass 1 (visible)** — per-channel dark terms all distinct and non-zero:

```
dark/white pairs:  03cb 2000 | 04a7 2000 | 046f 2000     (first upload, unity white)
after shading:     03cb 3048 | 04a7 2c48 | 046f 2b7d
```

**Pass 2 (IR)** — dark terms **all zero**, white terms nearly equal:

```
dark/white pairs:  0000 2000 | 0000 2000 | 0000 2000     (first upload)
after shading:     0000 32e3 | 0000 3265 | 0000 324e
```

Zero dark offsets and three near-identical white gains are exactly what you expect
when all three CCD sub-exposures see the *same* infrared illumination instead of
separate R/G/B. The visible pass's spread (`0x3048` / `0x2c48` / `0x2b7d`) reflects
genuinely different per-channel response.

The IR pass also **skips some AFE gain iterations** — pass 1 has two extra
three-channel gain-adjust rounds (FE indices 5–7) that pass 2 omits, presumably
because a single illuminant converges faster.

### AFE values reached

| Pass | offsets (FE 2/3/4) | final gains (FE 5/6/7) |
|------|--------------------|------------------------|
| Visible | `0x26` / `0x1d` / `0x24` | `0x14` / `0x22` / `0x16` |
| IR | (converged separately) | `0x30` / `0x29` / `0x31` region |

IR needs noticeably **higher gains**, consistent with a dimmer IR return.

---

## Implementation implication

An IR scan is not a special code path — it is the ordinary scan path with two
changes:

```
0x03  bit 4 = 0      (white lamp off)
0x37  bit 2 = 1      (IR LED on, via read-modify-write)
```

then run calibration and acquisition exactly as for colour. To produce an
iSRD-style result, run the visible pass first, then the IR pass, at identical
geometry — the two images are pixel-aligned because every geometry register is the
same.

---

## Open questions

- The upper nibble of `0x37` (`0xb0` vs `0xf0` on read) — hardware status of some
  kind, meaning unknown. Not DPI-dependent.
- Whether bit 2 of `0x37` can be set *without* clearing the lamp bit, i.e. whether
  a simultaneous RGB+IR pass is possible. The vendor never tries it.
- Why bit 2 is set, cleared, then set again during the IR pass (t=38.7 / 40.0 /
  41.3) rather than set once — possibly the dark-reference pass deliberately runs
  with the IR LED off.
- The still-unexplained items carry over from session 04: `0x029`:`0x02a`, `0x02e`,
  `0x120`, the shading-table padding rule, and the 8-vs-16-bit register.
