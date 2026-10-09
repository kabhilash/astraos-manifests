---
name: astraos-build-dev
description: Build the AstraOS development image (`astraos-image-dev`) for a target MACHINE via BitBake. Use whenever the user asks to "build the dev image for X", "build AstraOS for raspberrypi5", "bitbake astraos-image-dev", "build the image for cm5", "make a dev wic for the Variscite Symphony", or any similar phrasing that means "produce a runnable AstraOS development rootfs/wic for one of the supported MACHINEs". Use this for **dev** images only; for production (signed, HABv4-fused, mcb-only) use `astraos-build-prod` instead.
---

Run the image subcommand of the AstraOS build script with the user's
chosen MACHINE:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build dev-image <MACHINE>
```

## Extracting MACHINE from the user's request

Map natural-language target names to the canonical MACHINE string:

| User says | MACHINE |
|---|---|
| "raspberry pi 5", "rpi 5", "pi 5", "raspberrypi5" | `raspberrypi5` |
| "compute module 5", "cm5", "rpi cm5", "cm5 io board", "raspberrypi-cm5" | `raspberrypi-cm5-io-board` |
| "variscite", "var-som", "symphony", "Symphony v1.7", "imx8mp-var-dart" | `imx8mp-var-dart` |
| "mcb", "main carrier board", "the production carrier", "astrax-variscite-imx8mp" | `astrax-variscite-imx8mp` |

If the user mentions "production", "signed", or "HABv4" alongside `mcb`,
they probably want the production image (`astraos-build-prod`), not
the dev image — clarify if ambiguous.

If the MACHINE is ambiguous (e.g. just "build an image" with no target),
ask which one before running. The four valid options are listed above.

## Notes

- This builds `astraos-image-dev` (writable rootfs, debug tools, ssh,
  no signing, no dm-verity).
- `bitbake` runs inside the devcontainer on the local Docker daemon
  (default); the script handles all of that — just invoke it.
- The image lands in `build-<MACHINE>/tmp-<MACHINE>/deploy/images/<MACHINE>/`
  (`astraos-image-dev-<MACHINE>.rootfs.wic.zst`) in the checkout. For a
  `--remote` build it is in `/home/akothapalli/yocto/astraos/` on the
  build machine instead, and `scripts/build download-dev <MACHINE> [<dest>]`
  scp's the dev wic to the user's box. (`download-dev` is the dev-image
  counterpart; the prod image has `download-prod`.)
- First-time builds for a given MACHINE take ~1–2 hours (cold sstate);
  subsequent builds are far faster thanks to the per-MACHINE sstate
  cache (`<sstate dir>/<MACHINE>/`; on the build machine
  `/home/akothapalli/yocto/sstate`).
- Both Variscite MACHINEs build the dev image: `imx8mp-var-dart`
  (Symphony) and `astrax-variscite-imx8mp` (mcb). Only mcb has a
  production image.

## Local vs remote

Default is local: the build runs in the local Docker daemon, in this
checkout. `--remote` (e.g. `./scripts/build --remote dev-image <MACHINE>`) dispatches to
the build machine instead; only pass it if the user asks for it. The
builder builds what is on GitHub `main` (it `git pull`s and `repo sync`s
first), so push script or layer changes before a remote build.

For local runs:

- missing `sources/openembedded-core` is fetched automatically (`repo init` +
  `repo sync`); add `--force-resync` to re-run `repo init` + `repo sync --force-checkout --detach`
  on an existing `sources/`: every project goes back to its manifest
  revision on a detached HEAD, and uncommitted edits there are
  discarded. If any exist they are listed in a warning banner followed
  by a 10s Ctrl+C window. Only pass `--force-resync` when the user asks
  for it — an agent cannot press Ctrl+C
- sstate lives in `./sstate` of the directory the script is launched
  from (`ASTRAOS_SSTATE_DIR` to override), downloads in
  `~/yocto/downloads` (`ASTRAOS_DL_DIR`) — launch from the same
  directory each time to keep the cache warm
- the build dir is `build-<MACHINE>/` inside the checkout

Full walkthrough: `docs/BUILDING.md`.
