---
name: astraos-build-prod
description: Build the **signed production** AstraOS image (`astraos-image`) for the `astrax-variscite-imx8mp` MACHINE. Use whenever the user asks to "build the production image", "build prod for mcb", "build the signed mcb image", "produce the HABv4-signed wic", "build the production rootfs for the Variscite carrier", or any phrasing that means "real production-grade build for the mcb hardware". The production image includes dm-verity, HABv4 signing chain, RAUC bundle generation. It only targets the mcb MACHINE — do not attempt for raspberrypi5/cm5/Symphony (use `astraos-build-dev` for those dev targets).
---

Run the prod-image subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build prod-image
```

No MACHINE argument needed — `prod-image` is hardwired to
`astrax-variscite-imx8mp`. The build script sources
`scripts/setup-environment astrax-variscite-imx8mp` and then `bitbake
astraos-image` (note: not `-dev`).

## Where it runs

The build runs in the local Docker daemon, in this checkout:

- a missing core layer in `sources/` (`poky` on Scarthgap, `openembedded-core`
  on Wrynose) is fetched automatically (`repo init` +
  `repo sync`); add `--force-resync` to re-run `repo init` + `repo sync --force-checkout --detach`
  on an existing `sources/`: every project goes back to its manifest
  revision on a detached HEAD, and uncommitted edits there are
  discarded. If any exist they are listed in a warning banner followed
  by a 10s Ctrl+C window. Only pass `--force-resync` when the user asks
  for it — an agent cannot press Ctrl+C
- sstate lives in `./sstate-<codename>` (e.g. `sstate-scarthgap`) of the directory the script is launched
  from (`ASTRAOS_SSTATE_DIR` to override), downloads in
  `~/yocto/downloads-<codename>` (`ASTRAOS_DL_DIR`) — launch from the same
  directory each time to keep the cache warm
- the build dir is `build-<MACHINE>/` inside the checkout
- the release is Scarthgap (`manifests/default.xml`) by default; set
  `ASTRAOS_MANIFEST=manifests/wrynose.xml` for Wrynose (use a separate
  checkout - it has its own builder image, caches and layer set)

Full walkthrough: `docs/BUILDING.md`.

## Getting the output

- Local build: the image is in
  `build-astrax-variscite-imx8mp/tmp-astrax-variscite-imx8mp/deploy/images/astrax-variscite-imx8mp/`
  in the checkout.

## When this is the wrong skill

- User wants RPi5 or CM5 build → `astraos-build-dev`
- User wants Variscite Symphony (dev) build → `astraos-build-dev` with
  `imx8mp-var-dart`
- User wants the cross-SDK installer → `astraos-build-sdk`

## Important caveats to surface for the user

- **`mcb` hardware is currently pending** — the production image's full
  bring-up (HABv4 fuse-blow, dm-verity, signed RAUC) cannot be
  end-to-end validated until real hardware lands. Pieces can be tested
  on Symphony with HAB in test mode. Mention this if the user seems to
  be expecting a flashable image to a real `mcb` board today.
- The recipe pipeline includes `azure-iot-sdk-c` and other deps with
  unpinned SRCREVs / placeholder LIC_FILES_CHKSUMs — `do_fetch` will
  fail on those until pinned. See AstraOS/docs/ARCHITECTURE.md.
