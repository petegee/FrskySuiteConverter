# Porting a new Suite release — checklist

Suite releases drift (Electron majors, native-module versions, patch
anchors). Work through this list; `frsky-port` fail-fasts most of it, but
read the warnings — they exist for the cases automation can't judge.

1. **Fetch + fingerprint.** Run through step 4 and record: sha256, `file`
   output (expect NSIS), PE product version, Electron version
   (`strings` on the main exe). Compare all four against the previous port.
2. **Electron moved?** If the major version changed, NOTHING is automatic:
   re-verify the N-API ABI (step 12 load test), the `Depends` line
   (`--depends` override exists for this), and the desktop/sandbox notes.
   The script warns; you decide.
3. **Native versions.** Step 6 logs the five pinned versions. Changed
   `node-hid`/`drivelist` → step 7 rebuilds automatically (cache is keyed by
   version). Changed `serialport`/`usb`/`koffi` → confirm their Linux
   prebuilds still ship in-asar (step 6 logs the `.node` inventory count;
   investigate any drop).
4. **Patch anchors.** Steps 6 and 9 assert the `main.js` indexPath anchor
   (exactly once) and the `removable-disks.js` diskutil fallthrough. If an
   assert fires, upstream refactored: locate the new equivalent, update
   `apply_patches()` in `frsky-port`, document it in
   [`how-it-works.md`](how-it-works.md).
5. **Renderer expectations.** Spot-check the new package for `dfu-util/<plat>`
   and `aegis/<plat>` platform branches and `thirdparty/` contents
   (work/ trees are kept for inspection). No Linux aegis binary yet →
   placeholder stays.
6. **Full run + verify.** `./frsky-port <new-exe>` end to end; step 12 must
   pass clean. Then a real-radio smoke test: HID link, MSC disks, DFU detect
   (`dfu-util -l`), and one flash flow with the debug log on. If a flash
   NAKs with `status:255`, follow the staged-image triage in
   [`troubleshooting.md`](troubleshooting.md) before assuming a port bug.
7. **Ship.** Bump `--deb-rev`, keep the previous `out/*.deb` files for
   rollback, note the upstream delta in your release notes.

`--from-step N` resumes any stage for iteration (e.g. `--from-step 9` after
editing `apply_patches()`).
