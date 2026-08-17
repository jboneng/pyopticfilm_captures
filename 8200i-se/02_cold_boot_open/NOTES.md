# Session notes — 02_cold_boot_open

- Date: 2026-07-29 ~16:05 local
- Windows build: Windows 11 25H2, build 26200
- Vendor app + version: **SilverFast 9** (recognised the scanner)
- bcdDevice: `0x0702`; USB address **16** (re-enumerated after the power cycle)
- USBPcap interface used: `\\.\USBPcap1`
- Cold power-cycle before session? **yes**
- DPI / mode / area: n/a — init only, no preview or scan
- Outcome: **ok** — complete open/init sequence, ends cleanly
- Capture file: `02_cold_boot_open.pcapng`, 86 kB, 1304 frames, 49.8 s
- Observed: front **button LED lit**; no carriage movement; no lamp strike captured

136 frames belong to the scanner (frames 735–870), decoding to 149 register
writes, 7 register reads, 2 AHB bulk writes and 28 status probes.

---

## Checklist item B — framing **confirmed compatible with pyopticfilm**

The SE speaks the same Genesys vendor-request framing that pyopticfilm's USB
protocol layer implements. Verified point by point:

| Operation | Observed | Matches `GenesysUsbProtocol` |
|-----------|----------|------------------------------|
| Register write | `bmRequestType 0x40`, `bRequest 0x04`, `wValue 0x0083`, `wIndex 0`, payload `(addr, value)` | `write_register` ✔ |
| Register write, high address | same with `wValue 0x0183`; payload carries the **low** address byte | `usb_value \|= 0x100` ✔ |
| Register read | `bmRequestType 0xc0`, `bRequest 0x04`, `wValue 0x008e`, `wIndex = 0x22 + (addr << 8)`, 2 bytes in | `read_register` ✔ |
| Register read, high address | `wValue 0x018e`; e.g. `wIndex 0x0122` → register **`0x101`** | ✔ |
| Link status byte | **`0x55`** on every read | `check_link_status` ✔ |
| `write_0x8c` | `0x40`, `bRequest 0x0c`, `wValue 0x008c`, `wIndex` = index, 1-byte payload | `write_0x8c` ✔ |
| Bulk OUT preamble | `0x40`, `bRequest 0x04`, `wValue 0x0082`, `wIndex 0x01`, 8-byte little-endian addr+size | `write_ahb` / `BULK_OUT` ✔ |
| "ASIC initialised?" probe | `0xc0`, `bRequest 0x0c`, `wValue 0x008e`, `wIndex 0x00`, 1 byte → **`0x00`** | `read_usb_speed_byte` ✔ |

`framing.matches_genesys_gl845_class` = **true**.

### Difference 1 — writes are batched

The vendor packs **many `(addr, value)` pairs into a single 64-byte transfer**;
pyopticfilm's `write_registers()` issues one transfer per register. Functionally
equivalent, but far fewer round trips. Worth mirroring for the SE init blast.

### Difference 2 — frontend uses the **GL124** path, not GL845

`hints.fe_write_path` = **`gl124_5d5e`**. The frontend is written as
`0x51` (index) → `0x5d` (high) → `0x5e` (low), *not* GL845's `0x3a`/`0x3b`.

**`GenesysUsbProtocol.write_fe_register()` must not be reused for the SE** — it
writes `0x51` + `0x3a`/`0x3b`. This needs an SE-specific frontend path, exactly as
pyopticfilm's GL128 bring-up anticipated this. Do not mutate the GL845 path.

### Difference 3 — a `wIndex`-selected status probe family

28 transfers use `0xc0` / `bRequest 0x0c` / `wValue 0x008e` with a 1-byte read
where `wIndex` selects an internal slot:

| `wIndex` | Value | Count | Interpretation |
|----------|-------|-------|----------------|
| `0x00` | `0x00` | 1 | SANE's "is the ASIC initialised" probe; `0x00` = cold, as expected |
| `0x20` | `0x55` | 25 | Issued after almost every write — link/ready check |
| `0x18` | `0x02` | 2 | Issued only after each AHB bulk write — bulk completion status |

pyopticfilm only implements the `wIndex 0x00` case (`read_usb_speed_byte`).
Semantics of `0x20` / `0x18` are **not yet proven** — `0x55` is the same value as
the link-OK byte, so `0x20` may simply be another link check.

---

## Checklist item C — cold boot / init register map

### Init blast — 116 registers in 4 batched transfers

```
0x01=0x22 0x02=0x78 0x03=0x20 0x04=0x02 0x05=0x48 0x06=0x18 0x07=0x00 0x08=0x00
0x09=0x00 0x0a=0x40 0x0b=0x6c 0x0c=0x00 0x0d=0x00 0x11=0x00 0x12=0x04 0x13=0x08
0x14=0x01 0x15=0x80 0x16=0x27 0x17=0x0c 0x18=0x10 0x19=0x02 0x1a=0x00 0x1b=0x00
0x1c=0x00 0x1d=0x00 0x1e=0x10 0x1f=0x00 0x20=0x0c 0x21=0x00 0x22=0x1a 0x23=0x00
0x24=0x1a 0x25=0x00 0x26=0x00 0x27=0x00 0x2b=0x20 0x2c=0x12 0x2d=0xc0 0x30=0x6f
0x31=0x00 0x32=0x22 0x33=0x04 0x34=0x80 0x35=0x2f 0x36=0x1c 0x37=0xc0 0x38=0x44
0x39=0x00 0x3a=0x00 0x3b=0xff 0x3c=0xff 0x3d=0x00 0x3e=0x00 0x3f=0x01 0x4f=0x03
0x52=0x07 0x53=0x09 0x54=0x0b 0x55=0x01 0x56=0x03 0x57=0x05 0x5a=0x12 0x5b=0x00
0x5c=0x40 0x5e=0x1f 0x5f=0x05 0x60=0x00 0x61=0x00 0x63=0x20 0x67=0x7f 0x68=0x7f
0x69=0x01 0x70=0x01 0x71=0x02 0x72=0x03 0x73=0x04 0x74=0x00 0x75=0x00 0x76=0x00
0x77=0x00 0x78=0x00 0x79=0x0f 0x7a=0xff 0x7b=0xff 0x7c=0xff 0x7d=0x00 0x7e=0x2a
0x7f=0xf8 0x80=0x00 0x81=0x22 0x82=0x00 0x83=0x01 0x84=0x18 0x85=0x00 0x86=0x00
0x87=0x00 0x93=0x00 0x94=0x00 0x95=0x00 0x9d=0x08 0xa0=0x12 0xa4=0x00 0xa5=0x20
0xa6=0x00 0xa7=0x00 0xa8=0x00 0xa9=0x00 0xaa=0x00 0xab=0x30 0xb8=0x00 0xb9=0x38
0xba=0x00 0xbd=0x00 0xbe=0x00 0xbf=0x00
```

Addresses are strictly ascending with gaps — a straight table blast, no
read-modify-write. Transfer boundaries: `0x01`–`0x23`, `0x24`–`0x5b`,
`0x5c`–`0x86`, `0x87`–`0xbf`.

**No soft reset was observed.** `hints.soft_reset_from_cold_boot` is empty; there
is no `0x0e = 01` then `00`. Registers `0x0e`, `0x0f`, `0x10` are never written at
all. The blast goes straight in after the `wIndex 0x00` probe returns `0x00`.

### Ordered sequence after the blast

| Step | Operation | Note |
|------|-----------|------|
| 1 | probe `wIndex 0x00` → `0x00` | ASIC not yet initialised |
| 2 | init blast (4 transfers, 116 registers) | above |
| 3 | `write_0x8c(0x10, 0x0c)` | |
| 4 | `write_0x8c(0x13, 0x0c)` | |
| 5 | `0x0b = 0x44` | overrides init `0x6c` |
| 6 | `0x13 = 0x0f` | overrides init `0x08` |
| 7 | `0x0b = 0x4c` | overrides again — final value `0x4c` |
| 8 | read `0x101` → `0xdc` (twice) | high-address register |
| 9 | `0x03 = 0x10`, then `0x03 = 0x00` | init set `0x20`; final `0x00` |
| 10 | frontend: 8 × (`0x51`=idx, `0x5d`=high, `0x5e`=low) | table below |
| 11 | AHB bulk write addr `0x000fff00`, size 34 | 34 zero bytes |
| 12 | probe `wIndex 0x18` → `0x02` | |
| 13 | AHB bulk write addr `0x000fff01`, size 34 | zeros except **byte[32] = 0x33** |
| 14 | probe `wIndex 0x18` → `0x02` | |
| 15 | read `0x33` → `0x07`, write `0x33 = 0x07` | read-modify-write ×4 |
| 16 | read `0x33` → `0x07`, write `0x33 = 0x07` | |
| 17 | read `0x33` → `0x07`, write `0x33 = 0x17` | sets bit 4 |
| 18 | read `0x33` → `0x17`, write `0x33 = 0x1f` | sets bits 3+ |
| 19 | read `0x101` → `0xd8` | bit 2 cleared vs `0xdc` in step 8 |

Sequence ends here — SilverFast then sat idle.

### Frontend registers (via `0x51` / `0x5d` / `0x5e`)

Session 03 established that the AFE value is **16-bit**: `0x5d` is the high byte
and `0x5e` the low byte. The values below are re-stated accordingly.

| FE index | Value |
|----------|-------|
| `0x00` | `0x00f8` |
| `0x01` | `0x0080` |
| `0x02`–`0x07` | `0x0000` |

FE `0x00`/`0x01` match the GL845 8200i values (`0xf8`, `0x80`); indices 2–7 are
**zeroed at init** where GL845 loads gains/offsets (`0x28 0x20 0x28 0x2f 0x2d 0x23`).
Session 03 confirms indices `0x02`–`0x04` are **offsets** and `0x05`–`0x07` are
**gains**, searched for at runtime during calibration rather than taken from a
constant table.

### GPO candidates

Only `0xa6`–`0xa9` appear, all `0x00`. GL845's `0x6b`–`0x6f` GPO block is **never
written** on the SE. Nearby writes in the same region: `0xa0=0x12`, `0xa4=0x00`,
`0xa5=0x20`, `0xaa=0x00`, `0xab=0x30`.

---

## Open questions for later sessions

Resolved by session 03 — see `../03_home_or_preview/NOTES.md`:

- **`0x101` is the status register**, polled 311 times there. Both the `0x33`
  guess below and `Gl128.read_status()`'s `0x41` are wrong.
- **`0x33`** is not a status register; it is a clock/enable stepped to `0x1f`,
  and session 03 shows it moving back to `0x07` at idle.
- **FE indices 2–7** are offsets (`0x02`–`0x04`) and gains (`0x05`–`0x07`),
  searched at runtime; values are 16-bit via `0x5d`/`0x5e`.
- **Lamp** is `0x03` bit 4.

Still open:

- **AHB writes to `0x000fff00` / `0x000fff01`, 34 bytes each** — purpose unknown.
  Not the `0x10000000` image address. The lone non-zero byte (`0x33` at offset 32)
  is suspicious; possibly a small config or LED/button table. Session 03 repeats
  these two writes byte-for-byte, so they are part of "open device".
- **No lamp-on and no motor traffic** — expected, since we stopped before preview.
  The button LED lighting is not explained by any obvious GPO write.

## Extra observations

- Decoding this capture required five more tooling fixes (batched writes, high
  addresses, submit/completion merging, FE path detection, probe classification).
- Registers written more than once, final values: `0x03` → `0x00`,
  `0x0b` → `0x4c`, `0x13` → `0x0f`, `0x33` → `0x1f`.
