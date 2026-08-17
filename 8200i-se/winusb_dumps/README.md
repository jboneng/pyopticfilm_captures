# WinUSB live dumps (SE)

## What this is for

After vendor `.pcapng` sessions, optionally rebind the SE to **WinUSB** and
capture live register dumps with PlustekLib for comparison against decoded
init tables.

Suggested files:

- `se_live_dump_regs.json` — from `plustekctl dump-regs`
- `se_probe.txt` — copy of verbose probe log

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| [Zadig](https://zadig.akeo.ie/) | Bind WinUSB to `07B3:1825` |
| PlustekLib (`plustekctl`) | `list` / `probe` / `dump-regs` |
| pyusb + libusb | Already required by PlustekLib |

See [docs/windows-setup.md](../../../docs/windows-setup.md).

## How to capture

1. Finish vendor Wireshark sessions first (or be ready to restore the vendor driver).
2. Unplug scanner → Zadig → WinUSB for `07B3:1825` → replug.
3. Run:

```powershell
plustekctl list
plustekctl probe -v
plustekctl dump-regs -o captures/8200i-se/winusb_dumps/se_live_dump_regs.json
```

4. **Safe ops only** — probe / dump-regs. Do **not** run warmup / home / scan on SE until GL128 is implemented.

## Restore vendor driver

Uninstall the WinUSB device entry / reinstall Plustek or SilverFast software,
then replug so vendor captures can continue later.

## Done when

- JSON dump exists and `list` shows the SE
- Notes mention WinUSB was used (so dumps aren’t confused with vendor-driver traffic)
