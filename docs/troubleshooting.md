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
| DFU page "not connected" in EthOS bootloader | Expected: EthOS bootloader = `0483:5750` HID+MSC; DFU = `0483:DF11` ROM mode (different button-hold at power-on). Radio firmware flash uses HID FrSK, not DFU |
| Bootloader update loops 10 s → manual dialog (`bootloader.timeout`) | HID writes not landing (permissions) or HID+disk not back within 10 s; check the radio screen reacts |
| Flash reaches 100% then `bootloader.fail reason:deviceStatus` (`hid.flash.frsk.done status:255`) | The radio NAK'd the staged image. Check the staged file on the mounted disk — `device.frsk` header must show sane length/CRC, your radio's family/product id (compare with the [FrSky product list](https://www.frsky-rc.com/wp-content/uploads/Downloads/EthosSuite/Products/frsky_products_list.json)). If the image is correct and intact: pre-2.0.1-2 builds had the missing drive-lock flush (fixed); otherwise it's a genuine radio-side refusal — retry, then use the DFU flasher path or report to FrSky with the debug log |
| `bootloaderVersion26FlashRefused` abort | You cancelled the "upgrading to 26.x erases FLASH" confirm dialog — re-run and confirm, after a backup |

## Permission checklist (radio plugged in, Suite running)

```sh
lsusb -d 0483:          # DF11 = DFU/ROM mode, 5750 = EthOS bootloader/firmware
ls -l /dev/hidraw*      # Suite needs read access (uaccess or plugdev)
groups                  # plugdev + dialout, effective after reboot
dfu-util -l             # DFU devices visible
ls /media/$USER/        # RADIO + FLASH mounts in firmware MSC mode
```
