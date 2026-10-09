---
name: astraos-build-sdk
description: Build the AstraOS cross-SDK installer (`astraos-sdk` recipe) for a target MACHINE. Use whenever the user asks to "build the SDK for X", "populate the SDK", "build the cross-toolchain installer", "make a Yocto SDK for raspberrypi5", "produce the AstraX app developer SDK", or any phrasing that implies "give me a self-extracting cross-compile toolchain that an app developer can use to build Qt6/C++/Rust code targeting AstraOS". The SDK includes Qt6 sysroot (via populate_sdk_qt6), spdlog/libusb1/libgpiod headers, and the latest Rust on the host side (nativesdk-rust + nativesdk-cargo).
---

Run the sdk subcommand of the AstraOS build script with the user's
chosen MACHINE:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build sdk <MACHINE>
```

## Extracting MACHINE from the user's request

Same mapping as `astraos-build-dev`:

| User says | MACHINE |
|---|---|
| "raspberrypi5", "rpi 5", "pi 5" | `raspberrypi5` |
| "cm5", "compute module 5", "cm5 io board" | `raspberrypi-cm5-io-board` |
| "variscite", "var-som", "symphony" | `imx8mp-var-dart` |
| "mcb", "main carrier board", "production carrier" | `astrax-variscite-imx8mp` |

The SDK MACHINE only matters for the **target sysroot** (which arch the
SDK is built to compile for). If the user just says "build the SDK"
without specifying, ask which target. Most app dev work uses the
production-target SDK (`astrax-variscite-imx8mp`) for the canonical sysroot,
or `imx8mp-var-dart` while `mcb` hardware is pending.

## What the user gets

A self-extracting `.sh` installer in
`build-<MACHINE>/tmp-<MACHINE>/deploy/sdk/` in the checkout (or
`/home/akothapalli/yocto/astraos/` on the build machine for `--remote`
builds), named
something like
`astraos-glibc-x86_64-astraos-sdk-cortexa53-<MACHINE>-toolchain-0.1.0.sh`. Running it on a developer Mac/Linux box installs
the cross-sysroot under `/opt/astraos-sdk/`. The user then sources
`environment-setup-...` to put the cross-compile toolchain on PATH and
build app code (e.g. `astraX_BT`) against the AstraOS target sysroot.

Cold SDK builds take a while (the Qt6 stack is heavy); subsequent
builds for the same MACHINE hit the per-MACHINE sstate cache
(`/home/akothapalli/yocto/sstate/<MACHINE>/` on the build machine) and
finish fast.

## Local vs remote

Default is local: the build runs in the local Docker daemon, in this
checkout. `--remote` (e.g. `./scripts/build --remote sdk <MACHINE>`) dispatches to
the build machine instead; only pass it if the user asks for it. The
builder builds what is on GitHub `main` (it `git pull`s and `repo sync`s
first), so push script or layer changes before a remote build.

For local runs:

- missing `sources/poky` is fetched automatically (`repo init` +
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
