# Building AstraOS

Everything goes through `scripts/build`. It runs BitBake inside the
`astraos-builder` devcontainer image, either on the dedicated build machine
(`--remote`, the default) or on your own Docker daemon (`--local`).

## Which command builds what

| Target | MACHINE | Dev image (`astraos-image-dev`) | Production image (`astraos-image`) |
|---|---|---|---|
| mcb (production carrier) | `astrax-variscite-imx8mp` | `scripts/build image astrax-variscite-imx8mp` | `scripts/build prod-image` |
| Symphony v1.7 (Variscite dev board) | `imx8mp-var-dart` | `scripts/build image imx8mp-var-dart` | not supported |

- `prod-image` takes no MACHINE: the production image (dm-verity, HABv4
  signing chain, no SSH) only exists for `astrax-variscite-imx8mp`. It sets
  `ASTRAOS_BUILD_TYPE=prod`, which turns on `cve-check` in `local.conf`.
- The dev image has a writable rootfs, root SSH and debug tools. It is the
  only image Symphony builds.
- The two Raspberry Pi MACHINEs (`raspberrypi5`, `raspberrypi-cm5-io-board`)
  also build dev images only; see `docs/ARCHITECTURE.md` (Targets, Image
  Variants).
- mcb hardware is still pending, so the production chain cannot be
  validated end to end yet (see `docs/ARCHITECTURE.md`).

## Remote builds (default)

```bash
scripts/build image imx8mp-var-dart                  # Symphony, dev
scripts/build image astrax-variscite-imx8mp          # mcb, dev
scripts/build prod-image                             # mcb, production
```

`--remote` is implied unless you are already inside the devcontainer. The
script ssh-es to `akothapalli@10.11.12.20`, then on the builder:

1. creates `sstate/` and `downloads/` and, if the workspace is missing,
   clones it and runs `repo init` + `repo sync`;
2. runs `git pull --ff-only` (retried 3x) and `repo sync` in the workspace;
3. re-runs `scripts/build --local ...` there.

Because of step 2, **the builder builds what is on GitHub `main`, not your
working tree.** Commit and push script/layer changes before a remote build.

Fetching the dev image (dev `.wic.zst` only; there is no `download` for the
production image):

```bash
scripts/build download imx8mp-var-dart [<dest-dir>]   # default: current dir
scripts/build download astrax-variscite-imx8mp
```

Other remote-only helpers: `scripts/build clean` wipes the builder
workspace and re-syncs (caches kept). Override the target with
`ASTRAOS_BUILDER_HOST`, `ASTRAOS_BUILDER_USER`, `ASTRAOS_BUILDER_PORT`,
`ASTRAOS_BUILDER_PATH`, `ASTRAOS_BUILDER_HOST_ROOT`.

## Local builds

Same commands with `--local`:

```bash
scripts/build --local image imx8mp-var-dart
scripts/build --local image astrax-variscite-imx8mp
scripts/build --local prod-image
```

Prerequisites: Docker, and `repo` on `PATH`. Local builds are slow; use them
for small iterations (see `docs/ARCHITECTURE.md`, Build Machine).

- **Layers.** If `sources/poky` is missing the script runs `repo init` and
  `repo sync` in the checkout first. `--force-resync` re-runs `repo sync`
  even when `sources/` exists (`scripts/build --local --force-resync image ...`).
  This runs on the host, so your git/SSH access to the layer remotes
  applies.
- **Caches.** `sstate/` goes in the directory you launch the script from
  (`ASTRAOS_SSTATE_DIR` to override); downloads go to `~/yocto/downloads`
  (`ASTRAOS_DL_DIR`). Launch from the same directory each time to keep the
  cache warm.
- **Workspace.** The build dir is `build-<MACHINE>/` in the checkout
  (`ASTRAOS_BUILDER_PATH` to override).

Output, for both local and remote builds, is under
`build-<MACHINE>/tmp-glibc/deploy/images/<MACHINE>/` (`tmp/` instead of
`tmp-glibc/` on non-i.MX MACHINEs). The dev image is
`astraos-image-dev-<MACHINE>.rootfs.wic.zst`.

## Other useful commands

```bash
scripts/build [--local] shell <MACHINE>              # interactive shell, env sourced
scripts/build [--local] recipe <MACHINE> <recipe> [task]
scripts/build [--local] bitbake <MACHINE> <args...>
scripts/build [--local] sdk <MACHINE>
scripts/build [--local] publish <MACHINE>            # dev rpm feed, Variscite MACHINEs
scripts/build container                              # rebuild the builder image
```

`scripts/build --help` lists every subcommand and environment variable.
