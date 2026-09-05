# pyopticfilm captures

Wireshark / USBPcap traces of **SilverFast 9** talking to a **Plustek OpticFilm 8200i SE**
(`07b3:1825`, Genesys **GL128**) during reverse-engineering of
[pyopticfilm](https://github.com/jboneng/pyopticfilm).

This repository is the **capture dataset** — raw `.pcapng` files plus per-session decode
notes and a few machine-readable fixtures. The driver, extract scripts, and tests live in
the pyopticfilm repo.

## Related repositories

| Repository | Role |
|------------|------|
| [pyopticfilm](https://github.com/jboneng/pyopticfilm) | Python USB driver, decode tooling, Scan Lab |
| This repo | Published USB captures used during 8200i SE bring-up |

## Prerequisites

Capture files are stored with **Git LFS**. After cloning:

```bash
git lfs install
git lfs pull
```

For offline inspection you also need [Wireshark](https://www.wireshark.org/) (includes
`tshark`).

## Layout

```
8200i-se/
  01_enumerate/ … 13_ppi_ladder/   # numbered capture sessions
  decoded/                         # cross-session JSON fixtures
  SESSIONS.md                      # session index (start here)

8100-v2/
  01_enumerate/ … 06_y_crop_pair/  # numbered capture sessions
  SESSIONS.md                      # session index (start here)
  README.md                        # capture methodology for this model
```

See [8200i-se/SESSIONS.md](8200i-se/SESSIONS.md) or
[8100-v2/SESSIONS.md](8100-v2/SESSIONS.md) for the full session table, capture file
names, and links to decode notes.

## Using the captures

**Wireshark display filter** (8200i SE):

```text
usb.idVendor == 0x07b3 && usb.idProduct == 0x1825
```

**Wireshark display filter** (8100 V2):

```text
usb.idVendor == 0x07b3 && usb.idProduct == 0x1824
```

Bulk image traffic only:

```text
usb.transfer_type == 0x03 && usb.idVendor == 0x07b3
```

**pyopticfilm Scan Lab** (repo checkout, not on PyPI): use **Open capture…** to replay a
`.pcapng` offline without hardware. See
[tools/scanlab/README.md](https://github.com/jboneng/pyopticfilm/blob/main/tools/scanlab/README.md).

To regenerate consolidated JSON from the captures, run the extract scripts in a
pyopticfilm checkout with `--root` pointing at this repo's `8200i-se/` folder.

## Size

The full dataset is about **2.5 GB** (26 captures). The largest single file is
`8200i-se/13_ppi_ladder/13_7200ppi_color.pcapng` (~898 MB). Plan for LFS bandwidth when
cloning.

## License

Same terms as pyopticfilm: **GPL-3.0-or-later**.
