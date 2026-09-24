---
name: astraos-build-sdk
description: Build the AstraOS cross-SDK installer (`astraos-sdk` recipe) for a target MACHINE. Use whenever the user asks to "build the SDK for X", "populate the SDK", "build the cross-toolchain installer", "make a Yocto SDK for raspberrypi5", "produce the AstraX app developer SDK", or any phrasing that implies "give me a self-extracting cross-compile toolchain that an app developer can use to build Qt6/C++/Rust code targeting AstraOS". The SDK includes Qt6 sysroot (via populate_sdk_qt6), spdlog/libusb1/libgpiod headers, and the latest Rust on the host side (nativesdk-rust + nativesdk-cargo).
---

Run the sdk subcommand of the AstraOS build script with the user's
chosen MACHINE:

```
cd /Users/akothapalli/Projects/AstraA/X/AstraOS
./scripts/build sdk <MACHINE>
```

## Extracting MACHINE from the user's request

Same mapping as `astraos-build-image`:

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
`/home/akothapalli/yocto/astraos/build-<MACHINE>/tmp/deploy/sdk/` named
something like
`astraos-glibc-x86_64-astraos-sdk-cortexa53-<MACHINE>-toolchain-0.1.0.sh`
on the build machine. Running it on a developer Mac/Linux box installs
the cross-sysroot under `/opt/astraos-sdk/`. The user then sources
`environment-setup-...` to put the cross-compile toolchain on PATH and
build app code (e.g. `astraX_BT`) against the AstraOS target sysroot.

Cold SDK builds take a while (the Qt6 stack is heavy); subsequent
builds for the same MACHINE hit `/home/akothapalli/yocto/sstate/<MACHINE>/`
and finish fast.
