# FrskySuiteConverter

One command turns a FrSky Suite **Windows** installer into an installable
**Debian/Ubuntu amd64 `.deb`** with working USB (DFU flashing, HID radio
link, serial console, SD-card tools).

```sh
./frsky-port https://www.frsky-rc.com/path/to/FrSky-Suite-Windows-2.0.1.exe
# ... downloads, extracts, rebuilds, repacks, verifies ...
# Install frsky-suite_2.0.1-1_amd64.deb now? [Y/n]
```

Or point it at a file you already downloaded:

```sh
./frsky-port ~/Downloads/FrSky-Suite-Windows-2.0.1.exe --deb-rev 2 --no-install
```

## What it does (13 steps)

| # | Step | What happens |
|---|------|--------------|
| 1 | fetch | Download (or copy) the upstream `.exe`, record sha256 |
| 2 | extract-nsis | Unpack the NSIS installer (`$PLUGINSDIR/app-64.7z`) with static 7-Zip |
| 3 | extract-app | Unpack the Windows application tree |
| 4 | fingerprint | Confirm NSIS, detect the exact Electron version |
| 5 | runtime | Fetch the **exact** Electron Linux runtime (SHA-verified) + a private Node toolchain (your system node is never touched) |
| 6 | probe | Extract pristine `app.asar`; read versions, native-module inventory, patch anchors (fail-fast on upstream drift) |
| 7 | natives | Rebuild `node-hid` + `drivelist` for Linux, pinned to the app's versions, cached in `~/.cache/frsky-port` |
| 8 | dfuutil | `apt-get download dfu-util`, extract the binary (upstream ships Windows-only) |
| 9 | repack | Drop win64 dead weight, inject the 3 Linux `.node` files, apply the 2 Linux patches, repack `app.asar` |
| 10 | stage | Electron runtime, unpacked natives, dfu-util, icon, desktop file, udev rules, docs, `DEBIAN/` |
| 11 | deb | `dpkg-deb --build` → `out/frsky-suite_<ver>-<rev>_amd64.deb` |
| 12 | verify | Fail-closed checks: in-asar ELFs, N-API load test, desktop/maintainer-script lint, contents |
| 13 | install | Prompt, then `sudo dpkg -i` (`--install` / `--no-install` to skip the question) |

Resume anywhere: `./frsky-port <input> --from-step 9` reuses earlier outputs
(every step is idempotent). See `./frsky-port --help` for all flags.

## Requirements

- Debian 12+ / Ubuntu 22.04+ / Pop!_OS (or similar), **amd64**, with `sudo`.
- Network access (upstream downloads + apt).
- System packages are installed automatically: `binutils python3 gcc g++
  make libudev-dev curl xz-utils file desktop-file-utils`.

`--work`, `--cache` and `--out` directories override the defaults
(`./work`, `~/.cache/frsky-port`, `./out`).

## After installing

```sh
sudo usermod -aG plugdev,dialout $USER   # then log out/in (or reboot)
```

Then reconnect the radio. If a step misbehaves, start with
[`docs/troubleshooting.md`](docs/troubleshooting.md).

## Docs

- [`docs/how-it-works.md`](docs/how-it-works.md) — USB architecture, the
  native-module loader rules, and what the two Linux patches do and why.
- [`docs/new-release.md`](docs/new-release.md) — step-by-step checklist for
  porting the next Suite release.
- [`docs/troubleshooting.md`](docs/troubleshooting.md) — symptom → cause →
  fix playbook, including log capture.

## Scope and provenance

- This repo contains **only scripts, patches and docs**. No FrSky, Electron
  or Node binaries are checked in; everything proprietary is fetched from its
  upstream source at build time (with checksum verification where upstream
  publishes checksums).
- Unofficial community port, not affiliated with FrSky. FrSky Suite and all
  radio firmware remain property of FrSky RC.
- Upstream behaviour is preserved: Aegis updater and audio transcode stay
  unsupported on Linux, exactly as upstream gates them on non-Windows.
