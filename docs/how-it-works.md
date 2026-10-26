# How it works

The Windows Suite installer is an NSIS self-extractor wrapping an Electron
app. The port swaps the runtime and the Windows-only pieces, keeping
everything else byte-identical.

```
FrSky-Suite-Windows-<ver>.exe
  └─ NSIS payload ($PLUGINSDIR/app-64.7z)
       └─ Windows app tree (Electron 17 win32 + resources/app.asar + thirdparty/)
            ├─ app.asar ── probe ──▶ patch ──▶ repacked app.asar (in /opt/frsky-suite)
            └─ linux natives + dfu-util + Electron linux-x64 runtime ──▶ same tree
```

## USB architecture (what must work on Linux)

| Function | Module | Device IDs | Linux handling |
|---|---|---|---|
| DFU detect | `usb` (node-usb, libusb) | STM32 `0483:DF11` only | Linux prebuilds already ship in-asar — reused |
| DFU flash | `thirdparty/dfu-util/<plat>/` | same | Only `win64/` ships upstream — supplied from Ubuntu's `dfu-util` package (links only libusb/libudev/libc) |
| HID link | `node-hid` (`manufacturer==='FrSky'` match, 3 s poll) | e.g. radio `0483:5750` | Win32-only build ships — **rebuilt** (step 7) |
| Serial | `serialport`, polled | `0483:5750` | Linux prebuilds ship — reused |
| SD card | `drivelist` + `df -k` fallback | RADIO mass storage | Win32-only build ships — **rebuilt** (step 7) |
| FFI | `koffi`, Win32-only call sites, all platform-guarded | — | `linux_x64` ships — reused |

Not ported (upstream gates or dead code): UsbDk/WinUSB driver flow (replaced
by udev rules), `elevate.exe`, PowerShell PnP helpers, Aegis updater (no
Linux binary exists upstream — a placeholder README ships and the renderer
reports `unsupportedPlatform` by design), ffmpeg transcode (upstream disables
it on Linux — `Recommends: ffmpeg`, not bundled). Dev-only modules
(`@lmdb`, `parcel`, `watcher`, `msgpackr`) are never required by the main
process and stay out of the unpacked tree.

## The three loader laws (Electron 17)

1. **`bindings` (drivelist, node-hid) resolves ONLY inside `app.asar`.**
   A win32 `.node` in-asar loads as garbage (`invalid ELF header`); an
   absent one fails with `Could not locate the bindings file`. So the
   in-asar `drivelist.node` and `HID.node` MUST be Linux ELFs, plus an added
   `HID_hidraw.node` (the Linux default driver; absent upstream).
2. **`node-gyp-build` (serialport, usb) needs its in-asar
   `prebuilds/<plat>-<arch>/` files.** Never strip them (`No native build...
   abi=...` otherwise). The Linux glibc builds already ship — keep them.
3. **`koffi` selects `build/koffi/<platform>/` explicitly** — in-asar
   `linux_x64/koffi.node` just works.

Corollary: the asar's `.node` inventory stays identical to upstream except
the two overwrites + one addition (step 12 verifies all six paths are
x86-64 ELFs).

## The three Linux patches (applied in step 9, anchors asserted exactly-once)

1. **`app/main.js` — Linux `indexPath`.** The Linux branch points at the
   dev-only `../dist/frsky-suite/browser/index.html`; the packaged file is
   `app/browser/index.html` (same as the win32 branch).
2. **`app/devices/removable-disks.js` — drive lock flush.** The handler
   implements win32 (`FSCTL_LOCK_VOLUME`, which flushes caches) and macOS
   (`diskutil`) only; on Linux it shelled a nonexistent `diskutil` and
   reported success with dirty page cache. The Linux branch now runs a
   synchronous `sync` first: Suite stages flash images to the mounted disk
   ~1 s before the HID flash command and the radio reads them off the shared
   medium — without the flush the radio can see a torn file and NAK it
   (`hid.flash.frsk.done status:255`). Field-verified: X18RS bootloader
   flash failed with `status:255` on the unflushed build and succeeded
   with this flush.
3. **`app/devices/removable-disks.js` — Linux auto-mount.** The same
   non-Windows handler is the macOS `diskutil` stub, so Suite never mounts
   anything itself; its drive scan only sees volumes the desktop already
   mounted. Before each scan the port now runs `lsblk --json` and, when a
   USB device's vendor/model contains `FrSky` (or a partition has an EthOS
   label: `RADIO`, `FLASH`, `ETHOS`, `SD`/`SDCARD`), mounts every unmounted
   partition with `udisksctl mount -b`. That covers the internal drive and
   the SD card, leaves ordinary USB storage alone, and retries a failed
   mount after a 60 s backoff. Without it, a desktop that does not
   auto-mount (seen on Pop!_OS 24) shows an empty backup page and cannot
   stage a flash image.

## Runtime notes

- The Electron major version drives the N-API ABI and the `Depends` line.
  Step 4 warns loudly on anything outside the verified line; step 12's load
  test fails closed on ABI mismatch.
- `Exec=... --no-sandbox`: Electron 17's setuid sandbox SIGTRAPs on current
  kernels despite correct 4755 `chrome-sandbox` perms; postinst still sets
  4755 so removing the flag re-enables the sandbox.
- udev rules grant `uaccess` (systemd) with `plugdev`/`dialout` fallback.
  Sessions predate group changes — **reboot** after `usermod`, then replug.
- **Linux mounting.** Suite enumerates only mounted volumes; the upstream
  non-Windows mount/unmount handler is the macOS `diskutil` stub. Patch 3
  auto-mounts FrSky USB partitions with `udisksctl`, so current builds work
  on desktops that do not auto-mount (seen on Pop!_OS 24). It depends on
  `udisks2` being present and authorized for the session; if it is not, or
  an SD card has neither a FrSky USB parent nor an EthOS label, mount it
  manually with `udisksctl mount -b /dev/sdX1` and re-enter the page.
