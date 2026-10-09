# Variscite build warnings — TODO

Warnings emitted by `scripts/build dev-image imx8mp-var-dart`. None are
fatal; the build proceeds. Capture them here so we can address each
deliberately rather than letting them rot.

## 1. var-uuu-installer default-image notice

```
WARNING: .../meta-variscite-sdk-imx/recipes-variscite/var-install-packages/var-uuu-installer.bb: Using astraos-image-dev as default image.
```

We set `VAR_FLASH_IMAGE_NAME = "astraos-image-dev"` (or
`astraos-image` for prod) in `conf/local.conf.in` to silence the
"defaulting to fsl-image-gui-chromium" warning. The class re-warns at
parse-time to flag the chosen image. Harmless, but noisy.

**Possible fixes:**
- Patch the bbappend / class to use `bb.note` instead of `bb.warn`.
- Or BBMASK var-uuu-installer entirely if we're never going to ship
  Variscite's USB-boot factory-flashing flow (AstraOS uses RAUC for
  updates and a wic.zst flow for first provisioning, so UUU is mostly
  a Variscite-specific convenience).

## 2. Dangling bbappends in meta-imx/meta-imx-sdk

```
WARNING: No recipes in default available for:
  meta-imx/meta-imx-sdk/recipes-fsl/fsl-rc-local/fsl-rc-local.bbappend
  meta-imx/meta-imx-sdk/recipes-fsl/images/fsl-image-machine-test.bbappend
  meta-imx/meta-imx-sdk/recipes-fsl/images/fsl-image-multimedia.bbappend
  meta-imx/meta-imx-sdk/recipes-fsl/packagegroup/packagegroup-fsl-gstreamer1.0.bbappend
  meta-imx/meta-imx-sdk/recipes-fsl/packagegroup/packagegroup-fsl-tools-benchmark.bbappend
  meta-imx/meta-imx-sdk/recipes-fsl/packagegroup/packagegroup-fsl-tools-testapps.bbappend
  meta-imx/meta-imx-sdk/recipes-graphics/devil/devil_%.bbappend
  meta-imx/meta-imx-sdk/dynamic-layers/virtualization-layer/recipes-fsl/packagegroup/packagegroup-fsl-tools-testapps.bbappend
```

These bbappends target FSL-distro recipes (`fsl-image-gui-chromium`,
`packagegroup-fsl-gstreamer1.0`, etc.) that don't exist outside
`DISTRO=fsl-imx-*`. We downgraded the error to a warning with
`BB_DANGLINGAPPENDS_WARNONLY = "1"` in `conf/local.conf.in`.

**Possible fixes:**
- Drop `meta-imx/meta-imx-sdk` from BBLAYERS in
  `scripts/setup-environment`'s variscite branch — most of what it
  contributes are these FSL-only bbappends plus a handful of
  graphics/multimedia recipes that may or may not be needed.
- Audit each recipe in meta-imx-sdk and selectively include only the
  ones AstraOS actually needs (likely none beyond what meta-imx-bsp
  already covers).

## 3. qemu version preference mismatch

```
WARNING: preferred version 8.2.3 of qemu-native not available (for item qemu-native)
WARNING: versions of qemu-native available: 8.2.2.imx 8.2.7
WARNING: preferred version 8.2.3 of nativesdk-qemu not available (for item nativesdk-qemu)
WARNING: versions of nativesdk-qemu available: 8.2.7
... (8 lines total covering qemu-native, qemu-native-common, qemu-native-dev,
     nativesdk-qemu, nativesdk-qemu-common, nativesdk-qemu-dev)
```

Something in our stack (likely meta-astrax or one of the Variscite
layers) sets `PREFERRED_VERSION_qemu-native = "8.2.3"`, but the
available recipes are `8.2.2.imx` (NXP's patched fork in
meta-imx-bsp) and `8.2.7` (mainline scarthgap). 8.2.3 doesn't exist.

**Possible fixes:**
- Find the `PREFERRED_VERSION_qemu*` line(s) and update to a real
  version. Likely candidates:
  - `meta-astrax/conf/distro/astraos.conf` (we set PREFERRED_VERSION
    for qtbase et al. there)
  - `meta-imx-bsp` (may pin to its 8.2.2.imx fork; if so, prefer
    that to keep the i.MX patches)
- Decide whether to use NXP's `8.2.2.imx` (i.MX-patched, useful when
  running qemu-system tests against i.MX images) or mainline `8.2.7`.

## 4. RAUC system.conf and ca.cert.pem placeholders

```
WARNING: rauc-conf-1.0-r0 do_install: Please overwrite example system.conf with a project specific one!
WARNING: rauc-conf-1.0-r0 do_install: Please overwrite example ca.cert.pem with a project specific one, or set the RAUC_KEYRING_FILE variable with your file!
```

`meta-rauc`'s default recipe ships placeholder `system.conf` and
`ca.cert.pem`. For real A/B updates we need both:
- A `system.conf` describing the slot layout (compatible string,
  bootloader, system-info handler, A/B slot device paths).
- A signing keyring (`ca.cert.pem`) the device uses to verify bundles.

**Possible fixes:**
- Author a `rauc-conf.bbappend` (or recipe-specific override) in
  `meta-astrax-variscite` (and/or `meta-astrax-raspberrypi`) with
  per-MACHINE `system.conf` files. Slot layout differs by MACHINE
  (eMMC for mcb, SD for Symphony dev, NVMe for cm5).
- Generate/issue a real CA cert for AstraOS production (separate from
  dev cert) and set `RAUC_KEYRING_FILE` in the production build path.
  Dev builds can keep the example cert until we're ready to sign
  bundles for real.

This one is **load-bearing for production**: without it, RAUC won't
authenticate update bundles. Track separately from the others.
