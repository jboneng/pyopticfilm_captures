# Decoded artifacts (SE)

## What this is for

**Not** raw Wireshark files. This folder holds script output and hand notes used
to fill `plusteklib/device/model_8200i_se.py` and `plusteklib/asic/gl128.py`.

Master checklist: [docs/gl128-bringup.md](../../../docs/gl128-bringup.md).

## Automatic extract (preferred)

After session `.pcapng` files are in the sibling folders, run from the repo root:

```powershell
uv run python scripts/extract_se_captures.py
uv run python scripts/extract_se_afe_shading.py
```

Requires **Wireshark `tshark`** on `PATH` (same install used for USBPcap capture).

Default output: **`se_extract.json`** in this folder — one consolidated JSON with:

| Section | Contents |
|---------|----------|
| `device` | VID/PID/`bcdDevice`, endpoint candidates |
| `framing` | Whether Genesys GL845-class control/bulk framing matched |
| `sessions.*` | Per-capture register writes, last-write maps, status poll, bulk summaries |
| `implementation_tables` | Best-effort `init_regs` / GPO / FE / memory / sensor slices |
| `hints` | Status address histogram, lamp/GPIO candidates, IR vs color diffs |
| `checklist` | A–I decode checklist status (`ok` / `missing`) |

Per-PPI image LINCNT + feed pairs:

```powershell
uv run python scripts/extract_se_feeds.py
```

| File | Contents |
|------|----------|
| `ppi_lincnt_feed.json` | Session 13 + 03/04/09 feed1/feed2/LINCNT; `max_image_lincnt_by_feed2` |

AFE / shading Step 1 fixtures:

| File | Contents |
|------|----------|
| `afe_shading_fixture.json` | Compact constants + session 03/04 finals (used by unit tests) |
| `afe_shading_summary.json` | Short human summary |

See also [`../CALIB.md`](../CALIB.md) and `plusteklib/scan/calib_gl128.py`.

Options:

```powershell
uv run python scripts/extract_se_captures.py --root captures/8200i-se --out captures/8200i-se/decoded/se_extract.json -v
```

**Limits:** the script cannot invent status bit meanings or prove motor slopes.
Treat `implementation_tables` as a starting point (GL845-ish address ranges) —
verify before setting `scan_ready = True`. Raw image bulk payloads are **not**
embedded (only size summaries).

## Manual notes (optional)

| File | Contents |
|------|----------|
| `NOTES_overview.md` | Human framing conclusions, bit meanings |
| `status_bits.md` | Confirmed home/lamp/motor bits |
| `scan_1800_sequence.md` | Ordered scan configure → bulk steps |

Also attach any `../winusb_dumps/*.json` — the extract script includes them under
`winusb_dumps` when present.

## Tools needed

| Tool | Why |
|------|-----|
| Wireshark (+ `tshark`) | Capture + batch decode |
| USBPcap | USB capture source |
| `scripts/extract_se_captures.py` | Consolidated JSON |
