# Session notes — 03_home_or_preview

- Date: 2026-07-29 ~16:25 local
- Windows build: Windows 11 25H2, build 26200
- Vendor app + version: **SilverFast 9**, "Prescan" pressed once
- bcdDevice: `0x0702`; USB address **16** (device already plugged in)
- USBPcap interface used: `\\.\USBPcap1`, **inject-descriptors enabled**
- Cold power-cycle before session? no — continued from session 02
- DPI / mode / area: SilverFast default prescan, full frame
- Outcome: **ok** — full calibration + preview scan + return-to-home captured
- Capture file: `03_home_or_preview.pcapng`, 28.4 MB, 8836 frames, 35.8 s
- Observed: lamp struck, carriage moved, preview image appeared in SilverFast

This session turned out to be far more than "home or preview": SilverFast ran the
**complete acquisition pipeline** — memory layout, motor slope upload, AFE
offset/gain calibration, shading calibration, the preview scan itself, and the
return to home. 6738 decoded ops from 6113 scanner frames.

| Op kind | Count |
|---------|-------|
| `bulk_in` | 4536 (27,798,588 bytes) |
| `reg_read` | 935 |
| `reg_write` | 862 |
| `status_probe` | 353 |
| `bulk_preamble` | 27 |
| `bulk_out` | 21 (51,712 bytes) |
| `write_0x8c` | 4 |

---

## Checklist item D — the status register is **`0x101`** (settled)

Read **311 times**, always via the high-address path (`wValue 0x018e`,
`wIndex 0x0122`). This is the register the driver polls; every scan start is
followed by a burst of `0x101` reads until the value settles.

`Gl128.read_status()`'s guess of **`0x41` is wrong** — `0x41` reads `0x00` and is
never written. The session-02 candidate **`0x33` is also wrong**; it is a
clock/enable register stepped to `0x1f` during init. Register **`0x101`** is the
real one. A companion register **`0x100`** is read 10 times, always alongside
`0x101`.

### Observed values

| Value | Count | Bits |
|-------|-------|------|
| `0xa5` | 164 | `1010 0101` |
| `0xd5` | 72 | `1101 0101` |
| `0xdc` | 13 | `1101 1100` |
| `0xed` | 10 | `1110 1101` |
| `0xf5` | 9 | `1111 0101` |
| `0xd8` | 8 | `1101 1000` |
| `0xbd` | 5 | `1011 1101` |
| others | 1–4 each | `0x81 0x85 0xa9 0xad 0xc4 0xc9 0xcd 0xd4 0xdd 0xe9 0xec 0xf4 0xfc 0xfd` |

**Bit 7 (`0x80`) is set in every single sample** — treat as "device alive", not a
status flag.

### What correlates with what

- **`0xa5` × 162 consecutive** while the 25 MB preview stream drains, then the
  carriage returns home. This is the "busy / motor running" state.
- **`0xd5` × 72 consecutive** during the long shading-calibration wait.
- **`0xdc` / `0xd8`** at idle between operations.
- The scan-start handshake always reads **`0xed` → `0xbd`** (sometimes
  `0xcd` → `0xed` → `0xbd`) immediately before the first bulk IN preamble, i.e.
  `0xbd` is the "data ready, come get it" edge.
- Session ends `0xad` × 3 → `0xec`.

Exact bit semantics are **not yet proven**. What *is* safe to implement is
"poll `0x101` until it changes to the ready pattern", which is what the vendor
does. Sessions 04/05 with a known DPI should pin the individual bits down.

### The `wIndex`-selected probe family (extends session 02)

| `wIndex` | Values | Count | When |
|----------|--------|-------|------|
| `0x00` | `0x00` | 1 | once, at session start |
| `0x18` | `0x02` (×26), `0x12` (×1) | 27 | after **every** bulk transfer — bulk completion |
| `0x20` | `0x55` | 263 | after almost every register write — link/ready |
| `0x21` | `0x00` (×61), `0x04` (×1) | 62 | **new** — polled during motor moves |

`wIndex 0x21` is new in this session and appears only while the carriage is
moving, so it is a second motor/status slot.

---

## The register map is **GL124-family** — now confirmed three ways

Session 02 established the frontend uses the GL124 `0x51`/`0x5d`/`0x5e` path.
This session adds two independent confirmations, and the semantics line up with
SANE's `gl124`/`gl843` register names exactly:

| Register | Observed behaviour | SANE name |
|----------|--------------------|-----------|
| `0x01` bit 0 | `0x02` → `0x03` starts a scan, `0x03` → `0x02` stops it | `REG_0x01_SCAN` |
| `0x01` bit 1 | always set (`0x02`) | `REG_0x01_SHDAREA` |
| `0x01` bit 5 | set for the final preview (`0x23`) | `REG_0x01_DVDSET` |
| `0x0d` | `0x07` written 1–3× immediately before every scan | `CLRLNCNT \| CLRMCNT` (clear counters) |
| `0x0f` | `0x01` written as the very last step before data flows | gl124 begin-scan trigger |
| `0x25`/`0x26`/`0x27` | 24-bit big-endian line count — **verified below** | `LINCNT` |
| `0x82`–`0x84` / `0x85`–`0x87` | 24-bit big-endian pixel start / end | `STRPIXEL` / `ENDPIXEL` |
| `0x03` bit 4 | toggles at lamp-on points | `LAMPPWR` |
| `0xe0`–`0xf8` | ascending buffer boundary table | `gl124_init_memory_layout` |

### `LINCNT` proof

The final preview wrote `0x25=0x00, 0x26=0x12, 0x27=0xe4` → **4836 lines**, and
the resulting bulk IN was **25,069,824 bytes**:

```
25,069,824 / 4836 = 5184 bytes per line = 1728 px x 3 bytes
```

Exact, no remainder. Calibration scans wrote `0x27=0x01` (1 line) and read 3072
bytes = 512 px × 3 ch × 16-bit — also exact.

### `STRPIXEL` / `ENDPIXEL` proof

| Phase | `0x82`–`0x84` | `0x85`–`0x87` | End − Start |
|-------|---------------|---------------|-------------|
| AFE calibration | `0x000040` = 64 | `0x000240` = 576 | **512** = the 512 px read |
| Preview | `0x0000f2` = 242 | `0x002972` = 10610 | 10368 = 1728 × 6 |

The ×6 factor on the preview matches the **6 memory segments** below, so pixel
counts in these registers are in raw sensor-clock units, not output pixels.

### Memory layout block `0xe0`–`0xf8` (rewritten before every scan, identical each time)

```
e0=00 e1=68  e2=0b e3=00  e4=0b e5=01  e6=15 e7=99  e8=15 e9=9a
ea=20 eb=32  ec=20 ed=33  ee=2a ef=cb  f0=2a f1=cc  f2=35 f3=64
f4=35 f5=65  f6=3f f7=fd  f8=05
```

Read as 16-bit big-endian pairs these are **six ascending segment boundaries**:

| Segment | End | Next start | Size |
|---------|-----|------------|------|
| 0 | `0x0b00` | `0x0b01` | 2817 |
| 1 | `0x1599` | `0x159a` | 2713 |
| 2 | `0x2032` | `0x2033` | 2713 |
| 3 | `0x2acb` | `0x2acc` | 2713 |
| 4 | `0x3564` | `0x3565` | 2713 |
| 5 | `0x3ffd` | — | 2713 |

Total `0x3ffe` ≈ 16 K words, and **`0xf8 = 0x05` = segment count − 1 = 6
segments**. This is a per-model constant table; it did not vary with DPI in this
session, but sessions 04/06 should be checked before hardcoding it.

---

## Bulk transfer framing — fully decoded

The 8-byte preamble (`bmRequestType 0x40`, `bRequest 0x04`, `wValue 0x0082`) is:

```
bytes 0..3 = target address, little-endian
bytes 4..7 = transfer size,  little-endian
```

Verified against all 27 preambles, e.g. `00 00 00 10 00 89 7e 01` →
addr `0x10000000`, size `0x017e8900` = 25,069,824. **PlustekLib's `write_ahb`
layout is correct.**

`wIndex` selects the mode:

| `wIndex` | Meaning | Count |
|----------|---------|-------|
| `0x01` | RAM write (bulk OUT follows) | 20 |
| `0x00` | RAM read (bulk IN follows) | 6 |
| `0x08` | **streaming image read** — used *only* for the 25 MB preview | 1 |

`wIndex 0x08` is new and important: the small calibration reads use `0x00`, but
the real image acquisition uses `0x08`. PlustekLib only ever sends `0x01`.

**Caveat:** for the three 4-byte writes the preamble declared `size = 4` but a
full 512-byte bulk packet was sent (only the first 4 bytes non-zero). So the
device pads to `wMaxPacketSize`; the declared size is what matters.

### ASIC RAM map (banks are `0x4000` apart)

| Address | Size | Contents |
|---------|------|----------|
| `0x000fff00` / `0x000fff01` | 34 | The two mystery init blocks from session 02, unchanged |
| `0x10000000` | 512 | per-channel table, **R** |
| `0x10004000` | 512 | per-channel table, **G** |
| `0x10008000` | 512 | per-channel table, **B** |
| `0x1000c000` | 512 | **motor slope table 1** |
| `0x10010000` | 512 | **motor slope table 2** (always identical to table 1) |
| `0x10014000` | 21064 | **shading coefficients** |
| `0x10000000` | up to 25 MB | image data on read (`wIndex 0x08`) |

---

## Checklist item F — motor slope tables

Two 512-byte tables = **256 × 16-bit little-endian**, uploaded to `0x1000c000`
and `0x10010000` (byte-identical to each other). Values **decay monotonically** —
classic step-period acceleration ramps.

**Fast ramp** (uploaded at t=4.90 and again t=10.12, 404 of 512 bytes non-zero):

```
16de 06db 0540 047e 0406 03b1 0371 033e 0314 02f2 02d4 02ba
02a3 028e 027c 026b 025c 024e 0241 0235 022a 0220 0216 020d
0205 01fd 01f5 01ee 01e7 01e0 01da 01d4 01ce 01c9 01c4 01bf
01ba 01b5 01b1 01ac 01a8 01a4 01a0 019d 0199 0195 0192 018f ...
```

**Slow ramp** (uploaded at t=9.00, 511 of 512 bytes non-zero):

```
1fb4 12ff 0fa6 0dfd 0cea 0c21 0b8b 0b0c 0aa6 0a4f 0a01 09bf
0984 094f 091e 08f1 08c8 08a2 087f 0860 0841 0825 0809 07f0
07d9 07c1 07ad 0798 0784 0771 0760 074f 073e 072f 071f 0710
0702 06f5 06e8 06db 06cf 06c3 06b7 06ac 06a1 0696 068c 0682 ...
```

Full tables are in the capture; extract with the decoder when implementing.
The fast ramp is used for the calibration/positioning moves, the slow one for the
shading pass.

### Scan-start sequence (identical every time, 8 occurrences)

```
1.  write memory layout 0xd0-0xd2, 0xe0-0xf8
2.  write geometry / mode registers (batched)
3.  write AFE registers via 0x51 / 0x5d / 0x5e
4.  0x0d = 0x07          (clear line + motor counters, 1-3x)
5.  0x01 = 0x03          (set SCAN bit)
6.  0x0f = 0x01          (trigger)
7.  poll 0x101 until 0xbd   (via 0xed / 0xcd)
8.  full 288-register readback (only on some passes)
9.  bulk preamble + bulk IN
10. 0x01 = 0x02          (clear SCAN)
11. read 0x100 and 0x101
```

### Return to home (t=26.18 → 28.89)

```
0x03 = 0x30      lamp on
0x03 = 0x20      lamp off
0x01 = 0x22      clear SCAN
0x03 = 0x10 -> 0x00 -> 0x20 -> 0x30 -> 0x20 -> 0x30
poll 0x101: 0x81, 0x81, 0x85 x4, 0xa5 x162 (~2.5 s of movement), 0xad x3, 0xec
```

The lamp ends **on** (`0x03 = 0x30`), which matches the lamp still being lit when
the session ended.

---

## Checklist item E — lamp is `0x03` bit 4

All 25 writes to `0x03`, in order:

```
t=3.10  0x20        t=4.74  0x00        t=11.40 0x30
t=3.18  0x10        t=4.75  0x20        t=26.18 0x30
t=3.18  0x00        t=4.76  0x20        t=26.19 0x20
t=3.36  0x20        t=4.77  0x30        t=26.21 0x10
t=3.37  0x30        t=4.77  0x20        t=26.21 0x00
t=3.38  0x20        t=4.78  0x30        t=26.23 0x20
t=3.38  0x30        t=6.17  0x20        t=26.24 0x30
t=3.48  0x20        t=7.43  0x30        t=26.24 0x20
                                        t=26.25 0x30
```

`0x30` is held during **all** image acquisition and `0x20` between passes, so
**bit 4 (`0x10`) = lamp power** and bit 5 (`0x20`) is set almost always
(`AVEENB` in SANE). The repeated `0x20`/`0x30` toggle pairs are unexplained —
possibly a lamp-stabilise strobe. Bit 6 (`0x40`, `XPASEL` in SANE) is **never
set**, so the IR LED is *not* selected here — expect it in session 05.

---

## Calibration — AFE offset/gain and shading

### The AFE is **16-bit**: `0x5d` = high byte, `0x5e` = low byte

This corrects session 02, which recorded 8-bit values from `0x5e` alone. Gains
above 255 prove it, e.g. `0x51=0x05, 0x5d=0x01, 0x5e=0x6a` → **`0x016a`**.

### FE index map (now unambiguous)

| Index | Role | Values seen |
|-------|------|-------------|
| `0x00` | config | `0x00f8` (init only) |
| `0x01` | config | `0x0080` (init only) |
| `0x02` | **offset R** | `0x0000` → `0x0024` |
| `0x03` | **offset G** | `0x0000` → `0x001a` |
| `0x04` | **offset B** | `0x0000` → `0x0021` |
| `0x05` | **gain R** | `0x0080` → `0x00ff` → `0x016a` → `0x000c` → `0x0012` |
| `0x06` | **gain G** | `0x0080` → `0x00ff` → `0x0142` → `0x0016` → `0x001e` |
| `0x07` | **gain B** | `0x0080` → `0x00ff` → `0x015b` → `0x0010` → `0x0017` |

Indices 2–4 are offsets, 5–7 are gains — the ordering matches the GL845 8200i
table (`0x28 0x20 0x28` offsets, `0x2f 0x2d 0x23` gains) but the SE searches for
them at runtime instead of using constants.

### Calibration loop (the `0x0000`/`0x00ff` bracket search)

Each iteration: set FE, `0x0d=0x07`, `0x01=0x03`, `0x0f=0x01`, poll, read
**3072 bytes** (= 1 line × 512 px × 3 ch × 16-bit), `0x01=0x02`, adjust, repeat.
The values bracket `0x0000` → `0x00ff` → converged, i.e. a binary search on a
1-line strip. One larger read of **62,268 bytes** appears mid-sequence.

### Checklist item I groundwork — shading table at `0x10014000`

21,064 declared bytes (20,992 captured; the final 72-byte short packet was not
attributed). Structure is a repeating **12-byte period = 3 channels × 2 × 16-bit**:

**First upload, t=7.41** — white terms all `0x2000` (unity), dark terms constant:

```
038b 2000 | 0405 2000 | 04ac 2000     (repeats identically)
```

**Second upload, t=8.55** — after the shading pass, white terms are real:

```
038b 3335 | 0405 2dc8 | 04ac 2ce4
038b 331f | 0405 2d6d | 04ac 2c95
038b 32f3 | 0405 2d89 | 04ac 2d12
038b 3288 | 0405 2d67 | 04ac 2ce3   ...
```

The dark terms `0x038b` / `0x0405` / `0x04ac` stay **constant per channel** while
the white terms vary per pixel. `(21064 − 4) / 12 = 1755` entries exactly, i.e.
1755 pixels — 1728 output pixels plus 27 of margin. So this is
**(dark offset, white gain) × R,G,B per pixel**, with a 4-byte pad.

### Per-channel 512-byte tables at `0x10000000` / `0x10004000` / `0x10008000`

256 × 16-bit little-endian, first entry differs from the rest:

| When | Contents | Note |
|------|----------|------|
| t=4.99 | `36b0 36b0` then zero, declared size **4** | only 2 words meaningful |
| t=7.62 | `0dbc` then `0dac` × 255 | before shading pass |
| t=11.34 | `0dbc` then `0d1d` × 255 | before preview — **changed by calibration** |

All three channels always get identical contents. The value dropping
`0x0dac` → `0x0d1d` across calibration suggests a per-channel exposure / LED
on-time, but this is **not proven**.

---

## Full register-file snapshots

SilverFast reads **all 288 registers** (`0x001`–`0x120`) twice, which gives exact
ground-truth snapshots. Both are recorded here; the diffs are the useful part.

### Diff: session-02 cold-boot final → sweep 1 (idle, t=3.58)

25 registers changed — this is the rest of "open the device":

```
0x003: 00->20   0x006: 18->f0   0x013: 0f->08   0x01c: 00->20
0x01d: 00->80   0x01e: 10->20   0x031: 00->7e   0x032: 22->f6
0x033: 1f->07   0x037: c0->b0   0x03b: ff->01   0x052: 07->0b
0x053: 09->0d   0x054: 0b->0f   0x056: 03->05   0x057: 05->07
0x05a: 12->31   0x05b: 00->79   0x070: 01->0a   0x071: 02->0b
0x072: 03->0c   0x073: 04->0d   0x07a: ff->03   0x07e: 2a->36
0x07f: f8->b0
```

`0x006: 0x18 → 0xf0` sets `SCANMOD = 7` and clears `GAIN4`.

### Diff: sweep 1 (idle) → sweep 2 (configured for the 4836-line preview, t=11.43)

56 registers changed — **this is the scan configuration**:

```
0x001: 22->23   0x002: 78->30   0x003: 20->30   0x004: 02->42
0x005: 48->40   0x026: 00->12   0x027: 00->e4   0x029: 00->2c
0x02a: 00->13   0x02b: 20->02   0x02c: 12->00   0x02d: c0->c8
0x02e: 09->0f   0x032: f6->f2   0x033: 07->1f   0x051: 07->04
0x05e: 00->25   0x081: 22->40   0x083: 01->00   0x084: 18->f2
0x086: 00->29   0x087: 00->72   0x0a5: 20->02   0x0ab: 30->02
0x0ad: 00->01   0x0af: 00->ff   0x0d0: 00->0a   0x0d1: 00->0a
0x0d2: 00->0a   0x0e1..0x0f8: memory layout (see above)
0x100: b0->30   0x101: d8->c4   0x114: 00->80   0x115: 00->80
```

`0x051`/`0x05e` are just leftovers from the last AFE write. `0x0af: 00->ff` is a
strong GPO output-enable candidate; `0x0a5`/`0x0ab`/`0x0ad` also move.

Sweep 2 additionally shows `0x0d0`–`0x0d2 = 0x0a` and the whole `0x0e1`–`0x0f8`
block populated, neither of which is touched at init.

---

## Open questions

- **Exact bit meanings of `0x101`.** We know which value means "ready" and which
  means "busy", but not per-bit. Needs a session with a single, known move.
- **`wIndex 0x21`** — polled only during motor moves, returns `0x00` / once `0x04`.
- **`wIndex 0x08`** on the streaming image read — what do bits 1–3 select?
- **The `0x20`/`0x30` toggle pairs on `0x03`** around lamp transitions.
- **`0x000fff00` / `0x000fff01`, 34 bytes** — still unexplained, identical to
  session 02, with the lone `0x33` at offset 32.
- **`0x114` / `0x115` = `0x80`** — written once per session, purpose unknown.
- **62,268-byte calibration read** — doesn't fit the 3072-byte pattern.
- Whether the `0xe0`–`0xf8` memory layout and the slope tables vary with DPI.
  Session 06 (high DPI) would answer this.

## Extra observations

- A compressed chronological listing is in `timeline.txt` (1301 lines) for
  reference; regenerate it from the pcapng with the decoder if needed.
- No soft reset here either, consistent with session 02.
- The preview geometry — **1728 px × 4836 lines × 3 bytes** — is a clean,
  fully-verified data point to test any future implementation against.
