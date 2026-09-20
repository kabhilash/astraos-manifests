---
name: astraos-build-prod
description: Build the **signed production** AstraOS image (`astraos-image`) for the `astrax-variscite-imx8mp` MACHINE. Use whenever the user asks to "build the production image", "build prod for mcb", "build the signed mcb image", "produce the HABv4-signed wic", "build the production rootfs for the Variscite carrier", or any phrasing that means "real production-grade build for the mcb hardware". The production image includes dm-verity, HABv4 signing chain, RAUC bundle generation. It only targets the mcb MACHINE — do not attempt for raspberrypi5/cm5/Symphony (use `astraos-build-image` for those dev targets).
---

Run the prod-image subcommand of the AstraOS build script:

```
cd /Users/akothapalli/Projects/AstraA/X/AstraOS
./scripts/build prod-image
```

No MACHINE argument needed — `prod-image` is hardwired to
`astrax-variscite-imx8mp`. The build script sources
`scripts/setup-environment astrax-variscite-imx8mp` and then `bitbake
astraos-image` (note: not `-dev`).

## When this is the wrong skill

- User wants RPi5 or CM5 build → `astraos-build-image`
- User wants Variscite Symphony (dev) build → `astraos-build-image` with
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
