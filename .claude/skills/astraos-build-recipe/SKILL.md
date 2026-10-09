---
name: astraos-build-recipe
description: Bitbake a single AstraOS recipe (optionally a specific BitBake task on it) for a target MACHINE. Use whenever the user asks to "bitbake recipe X for Y", "build just qtbase for cm5", "rerun -c populate_sysroot on libusb1 for raspberrypi5", "build only the astrax-bt recipe", "compile spdlog by itself", or any phrasing that means "run BitBake on a single recipe rather than a whole image". For full image builds use `astraos-build-dev` or `astraos-build-prod`. For arbitrary BitBake invocations with multiple flags use `astraos-bitbake`.
---

Run the recipe subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build recipe <MACHINE> <RECIPE> [TASK]
```

The optional `[TASK]` argument is a BitBake task like `cleansstate`,
`populate_sysroot`, `compile`, `do_unpack`, etc. If the user doesn't
specify a task, omit it — BitBake will run the default task chain
(`do_build`).

## Extracting parameters from the user's request

**MACHINE** — same mapping as `astraos-build-dev`. If unspecified,
ask the user which target.

**RECIPE** — the package name as it appears in BitBake. Common ones the
user might mention:

| User says | RECIPE |
|---|---|
| "qtbase", "Qt base" | `qtbase` |
| "Qt declarative", "qml engine" | `qtdeclarative` |
| "spdlog" | `spdlog` |
| "libusb" | `libusb1` |
| "libgpiod" | `libgpiod` |
| "the AstraX app", "astrax-bt", "the benchtop app" | `astrax-bt` |
| "nexus" | `nexus` |
| "RAUC" | `rauc` |
| "the kernel", "linux", "linux kernel" | `virtual/kernel` (or the BSP-specific one) |
| "u-boot", "bootloader" | `virtual/bootloader` (or the BSP-specific one) |

**TASK** — only if the user says "rerun X" or "do task Y". Common ones:

- `cleansstate` — wipe sstate cache for this recipe
- `clean` — remove build artefacts but keep sstate
- `populate_sysroot` — make headers/libs available to dependent recipes
- `compile` — just the build step
- `unpack` / `patch` / `configure` — earlier stages
- `package` — produce the binary package
- `devshell` — drop into a shell with the recipe's build env

## Examples

User says "rebuild qtbase for raspberrypi5":
```
./scripts/build recipe raspberrypi5 qtbase
```

User says "wipe sstate for libusb on cm5":
```
./scripts/build recipe raspberrypi-cm5-io-board libusb1 cleansstate
```

User says "drop me into the qtbase devshell on the variscite":
```
./scripts/build recipe imx8mp-var-dart qtbase devshell
```

If the user wants a more complex BitBake invocation (multiple recipes,
flags like `-k -v`, environment overrides), use `astraos-bitbake`
instead.

## Local vs remote

Default is local: the build runs in the local Docker daemon, in this
checkout. `--remote` (e.g. `./scripts/build --remote recipe <MACHINE> <RECIPE> [TASK]`) dispatches to
the build machine instead; only pass it if the user asks for it. The
builder builds what is on GitHub `main` (it `git pull`s and `repo sync`s
first), so push script or layer changes before a remote build.

For local runs:

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
