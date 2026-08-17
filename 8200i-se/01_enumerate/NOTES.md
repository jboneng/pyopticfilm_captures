# Session notes — 01_enumerate

- Date: 2026-07-29 15:51 local
- Windows build: Windows 11 25H2, build 26200 (AMD Ryzen 5 5600)
- Vendor app + version: **none used** (descriptor-only session)
- Scanner serial / label: USB instance suffix `20241113`
- bcdDevice: **`0x0702`** → GL128 (matches SANE `check-usb-chip.c`)
- USBPcap interface used: **`\\.\USBPcap1`** (Wireshark 4.6.7 / Dumpcap, snaplen 65535)
- Cold power-cycle before session? n/a — device was unplugged, then replugged during capture
- DPI / mode / area: n/a
- Outcome: **ok** — complete enumeration captured
- Capture file: `01_enumerate.pcapng`, 199 kB, 3094 frames, 36.3 s

## Decoded result (checklist item A — complete)

USB device address **15**; 8 frames belong to the scanner (frames 217–224).

### Device descriptor

| Field | Value |
|-------|-------|
| `idVendor` | `0x07b3` |
| `idProduct` | `0x1825` |
| `bcdDevice` | `0x0702` |
| `bDeviceClass` | `0xff` (vendor specific) |
| `bMaxPacketSize0` | 64 |
| `bNumConfigurations` | 1 |

### Configuration descriptor

| Field | Value |
|-------|-------|
| `wTotalLength` | 39 (= 9 config + 9 interface + 3 × 7 endpoint) |
| `bNumInterfaces` | 1 |
| `bConfigurationValue` | **1** |
| `iConfiguration` | 0 |
| `bmAttributes` | `0xc0` — self-powered, no remote wakeup |
| `bMaxPower` | 5 (10 mA) |

### Interface descriptor

| Field | Value |
|-------|-------|
| `bInterfaceNumber` | **0** |
| `bAlternateSetting` | **0** |
| `bNumEndpoints` | 3 |
| class / subclass / protocol | `0xff` / `0xff` / `0xff` |

### Endpoints

| Address | Dir | Type | wMaxPacketSize | bInterval |
|---------|-----|------|----------------|-----------|
| `0x81` | IN | **Bulk** | 512 | 0 |
| `0x02` | OUT | **Bulk** | 512 | 0 |
| `0x83` | IN | Interrupt | 1 | 8 |

512-byte bulk max packet size means the device operates at USB 2.0 **high speed**.

### Enumeration sequence

| Frames | Transfer |
|--------|----------|
| 217→218 | `GET_DESCRIPTOR` device, `wLength` 18 |
| 219→220 | `GET_DESCRIPTOR` configuration, `wLength` 9 |
| 221→222 | `GET_DESCRIPTOR` configuration, `wLength` 39 (full) |
| 223→224 | `SET_CONFIGURATION` (`bRequest` 9), `wLength` 0 |

No string descriptors were requested — Windows had `iProduct` cached. The product
string from Device Manager is **`Film Scanner (A2F)`** (the bring-up doc predicted
`Film Scanner(A2M)`).

## Extra observations

- pyopticfilm locates endpoints dynamically by direction + bulk attribute, so
  `0x81` / `0x02` are picked up with **no code change needed**. The interrupt
  endpoint `0x83` is unused by pyopticfilm.
- Interface 0 / alt 0 / configuration 1 match what `device.py` already claims.
- Most of the 3094 frames belong to an unrelated busy device on the same root hub
  (address 5, ~2982 frames). Harmless — decoding filters by device address.
- Tooling fixes were required before this capture could be decoded at all; see
  the pyopticfilm repo for decode tooling history.
