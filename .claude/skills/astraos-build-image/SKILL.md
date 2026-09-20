---
name: astraos-build-image
description: Build the AstraOS development image (`astraos-image-dev`) for a target MACHINE via BitBake. Use whenever the user asks to "build the dev image for X", "build AstraOS for raspberrypi5", "bitbake astraos-image-dev", "build the image for cm5", "make a dev wic for the Variscite Symphony", or any similar phrasing that means "produce a runnable AstraOS development rootfs/wic for one of the supported MACHINEs". Use this for **dev** images only; for production (signed, HABv4-fused, mcb-only) use `astraos-build-prod` instead.
---

Run the image subcommand of the AstraOS build script with the user's
chosen MACHINE:

```
cd /Users/akothapalli/Projects/AstraA/X/AstraOS
./scripts/build image <MACHINE>
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
- `bitbake` runs on the AstraOS build machine via SSH (default), inside
  the devcontainer. The script handles all of that — just invoke it.
- The image lands at
  `/home/akothapalli/yocto/astraos/build-<MACHINE>/tmp/deploy/images/<MACHINE>/`
  on the build machine when done. The user may want to scp the `.wic`
  artefact to their dev box afterward (the `scripts/build download`
  subcommand wraps this), or use `flash`-style tooling.
- First-time builds for a given MACHINE take ~1–2 hours (cold sstate);
  subsequent builds are far faster thanks to the per-MACHINE
  `/home/akothapalli/yocto/sstate/<MACHINE>/` cache.
