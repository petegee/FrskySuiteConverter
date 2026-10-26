# Troubleshooting

Capture logs first — everything below keys off them:

```sh
/opt/frsky-suite/frsky-suite --no-sandbox   # terminal: main-process device logs
```

then in-app: **Suite settings → debug mode → reproduce → Log viewer** for the
renderer scopes (`Radio`, `RadioFlash`, `RadioUpdate`). Useful filters:
`hid|usb|Radio|timeout`.

| Symptom | Cause → fix |
|---|---|
| Launcher silent, terminal shows `Trace/breakpoint trap` | Electron 17 sandbox vs new kernel → the shipped desktop file already passes `--no-sandbox` |
| `invalid ELF header` (`/tmp/.org.chromium.*`) | Win32 `.node` shadowing in-asar → step 9 overwrite incomplete; re-run from `--from-step 9` |
| `Could not locate the bindings file` | In-asar `.node` missing → same as above |
| `No native build... abi=...` | In-asar prebuilds stripped or Electron major moved → check step 4 warning, never strip prebuilds |
| HID/SD seen, writes fail; `groups` lacks plugdev/dialout | Session predates usermod → **reboot** (stale sessions survive logout), replug radio, recheck `ls -l /dev/hidraw*` |
| Backup & recovery shows nothing to select (By Component **and** Full Drive empty), or backup Start stays disabled | Suite on Linux only enumerates **mounted** drives. Current builds auto-mount FrSky USB partitions (`udisksctl`; field-verified on Pop!_OS 24); if this still happens, check `lsblk -o NAME,LABEL,MOUNTPOINT` — an SD card with no FrSky USB parent and no EthOS label is not auto-mounted. Run `udisksctl mount -b /dev/sdX1`, then **leave and re-enter** the page (selections are snapshotted when it opens). The same missing mount blocks flash staging. Look for `Auto-mount … failed` in the main log if udisks2 is absent or refuses |
| DFU page "not connected" in EthOS bootloader | Expected: EthOS bootloader = `0483:5750` HID+MSC; DFU = `0483:DF11` ROM mode (different button-hold at power-on). Radio firmware flash uses HID FrSK, not DFU |
| Bootloader update loops 10 s → manual dialog (`bootloader.timeout`) | HID writes not landing (permissions) or HID+disk not back within 10 s; check the radio screen reacts |
| Flash reaches 100% then `bootloader.fail reason:deviceStatus` (`hid.flash.frsk.done status:255`) | The radio NAK'd the staged image — triage: (1) decode `RADIO/device.frsk` left on the mounted disk: u32LE bytes 8–11 = len−16, u16LE bytes 14–15 = CRC-16/CCITT-FALSE over bytes 16…, byte 4 = header version, bytes 5–7 = version, byte 12 = family (6 = RADIO), byte 13 = product id (X18RS = 23; confirm against the [FrSky product list](https://www.frsky-rc.com/wp-content/uploads/Downloads/EthosSuite/Products/frsky_products_list.json)) — wrong model or bad CRC means wrong asset/download, not transport; (2) check `stat` mtime — Suite unlinks and rewrites the staged files on *every* attempt, so a retry never tests a long-stable file; (3) on unflushed builds the radio reads a torn file (fixed by this port's drive-lock flush — field-verified on X18RS); on flushed builds a 255 is a genuine radio-side refusal — use the DFU flasher path with `bootloader.frsk`, or report to FrSky with the debug log. Note: bootloader-first ordering is Suite-enforced (`TargetUpdateRequiresBasicComponent`), and `bootloaderVersion26FlashRefused` just means the erase-FLASH confirm was cancelled — neither is a bug |
| `bootloaderVersion26FlashRefused` abort | You cancelled the "upgrading to 26.x erases FLASH" confirm dialog — re-run and confirm, after a backup |

## Permission checklist (radio plugged in, Suite running)

```sh
lsusb -d 0483:          # DF11 = DFU/ROM mode, 5750 = EthOS bootloader/firmware
ls -l /dev/hidraw*      # Suite needs read access (uaccess or plugdev)
groups                  # plugdev + dialout, effective after reboot
dfu-util -l             # DFU devices visible
ls /media/$USER/        # RADIO + FLASH mounts in firmware MSC mode
```
