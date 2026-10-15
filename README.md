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

## After installing

```sh
sudo usermod -aG plugdev,dialout $USER   # then log out/in (or reboot)
```

Then reconnect the radio. If a step misbehaves, start with
[`docs/troubleshooting.md`](docs/troubleshooting.md).

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

## Command reference

```sh
./frsky-port <URL-or-file> [options]
```

`<URL-or-file>` (required, exactly one): an `https://` URL to a
`FrSky-Suite-Windows-*.exe` installer, or a local path to one. URLs are
downloaded with `curl` (retries enabled); local files are copied. The sha256
is always recorded to `work/upstream.exe.sha256`.

| Option | Default | Meaning |
|---|---|---|
| `--deb-rev N` | `1` | Debian revision: the `-N` in `frsky-suite_<upstream>-N_amd64.deb`. Bump when the *packaging* changes but upstream didn't, so `apt` sees an upgrade path. |
| `--expect-sha256 H` | — | Refuse to continue unless the fetched `.exe` hashes to `H` (used by CI against the digest GitHub records on the upstream asset). |
| `--version X.Y.Z` | auto | Override the upstream version. By default it is read from the app's own `package.json` after extraction (authoritative); the input filename is only cross-checked with a warning. |
| `--depends "..."` | tested line | Override the `DEBIAN/control` `Depends:` line. Needed if your distro renamed a library (e.g. Debian 12+/Ubuntu 24.04+ ship `libasound2t64` instead of `libasound2`). Quote the whole list. |
| `--work DIR` | `./work` | Scratch + extracted trees (`upstream.exe`, `extracted-nsis/`, `app/`, `probe/`, `staging/`). Safe to delete; everything in it is regenerable. |
| `--cache DIR` | `~/.cache/frsky-port` | Downloaded tools and reuse cache: static 7-Zip, Electron runtimes (one per version), private Node toolchain, native-module builds (keyed by `node-hid`+`drivelist` versions), dfu-util debs. Shared across ports — keep it. |
| `--out DIR` | `./out` | Deliverable `.deb` directory. |
| `--from-step N` | `1` | Start at step N (1–13), reusing earlier outputs. Missing prerequisites fail with a message naming the step to re-run from. |
| `--to-step N` | `13` | Stop after step N. Combine with `--from-step` to iterate one stage, e.g. `--from-step 9 --to-step 9` after editing a patch. |
| `--install` | — | Install with `sudo dpkg -i` without prompting. |
| `--no-install` | — | Build only; never install. Implied when stdin is not a terminal. |
| (neither flag) | prompt | Ask `Install <deb> now? [Y/n]` (default yes) after a successful build. |
| `-h`, `--help` | — | Print the usage summary and exit. |

Exit status is 0 on success (or on a declined install prompt), non-zero on
the first failed step, with the failing step named in the error.

Examples:

```sh
# Full port from URL, prompt before installing (the common case).
./frsky-port https://www.frsky-rc.com/path/to/FrSky-Suite-Windows-2.0.1.exe

# Re-issue the same upstream with a packaging fix, build only.
./frsky-port ~/Downloads/FrSky-Suite-Windows-2.0.1.exe --deb-rev 2 --no-install

# Iterate on the repack stage only, then verify through to a deb.
./frsky-port ~/Downloads/FrSky-Suite-Windows-2.0.1.exe --from-step 9 --to-step 12 --no-install

# Distro with renamed ALSA lib.
./frsky-port <input> --depends "libusb-1.0-0, libudev1, ..., libasound2t64, ..."
```

## Automation

`.github/workflows/port-suite.yml` converts new Suite releases without human
involvement:

- **Daily schedule** polls `FrSkyRC/ETHOS-Feedback-Community` for the newest
  *stable* `suite*` tag (prereleases and `rc`/`nightly`/`alpha`/`beta` tags
  are skipped) and converts it if this repo has no matching
  `suite-<upstream>-<deb-rev>` release yet.
- **Manual dispatch** (`Actions → port-suite → Run workflow`) can target any
  upstream tag (including RCs), set the deb revision, and create the release
  as a **draft**.
- The downloaded `.exe` is checked against the sha256 digest GitHub records
  on the upstream asset (`--expect-sha256`) before converting; the build runs
  on a stock `ubuntu-24.04` runner with the step-12 verification gate, and the
  resulting `.deb` is published as a repo release. Draft mode exists so an
  unattended build never ships to users untested — flip it on when iterating.

## Requirements

- Debian 12+ / Ubuntu 22.04+ / Pop!_OS (or similar), **amd64**, with `sudo`.
- Network access (upstream downloads + apt).
- System packages are installed automatically: `binutils python3 gcc g++
  make libudev-dev curl xz-utils file desktop-file-utils`.
  (Step 5's Node toolchain is a private tarball — your system node is never
  touched.)

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

## Disclaimer

Use at your own risk. The author makes no commitment to keeping this repo
or the converter functioning — FrSky may change the Suite installer at any
time, and fixes will land only on a best-effort basis, if at all. The author
takes no responsibility for anything resulting from its use, including but
not limited to failed or interrupted radio firmware/bootloader flashes,
which can render a radio unusable. Always back up your radio and models in
Suite before updating, verify checksums where available, and stop and ask
(file an issue) if any step reports an error or warning you do not
understand.
