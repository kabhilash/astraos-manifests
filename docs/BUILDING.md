# Building AstraOS

Everything goes through `scripts/build`. It runs BitBake inside the
`astraos-builder` devcontainer image, on your own Docker daemon by default
(`--local`) or on the dedicated build machine (`--remote`).

## Which command builds what

| Target | MACHINE | Dev image (`astraos-image-dev`) | Production image (`astraos-image`) |
|---|---|---|---|
| mcb (production carrier) | `astrax-variscite-imx8mp` | `scripts/build dev-image astrax-variscite-imx8mp` | `scripts/build prod-image` |
| Symphony v1.7 (Variscite dev board) | `imx8mp-var-dart` | `scripts/build dev-image imx8mp-var-dart` | not supported |

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

## Local builds (default)

```bash
scripts/build dev-image imx8mp-var-dart              # Symphony, dev
scripts/build dev-image astrax-variscite-imx8mp      # mcb, dev
scripts/build prod-image                             # mcb, production
```

(`--local` is accepted but implied, unless noted otherwise.)

Prerequisites: Docker, and `repo` on `PATH`. Local builds are slow; the
build machine has far more cores (see `docs/ARCHITECTURE.md`, Build
Machine).

- **Layers.** If `sources/poky` is missing the script runs `repo init` and
  `repo sync` in the checkout first. `--force-resync` re-runs
  `repo init` and `repo sync --force-checkout --detach` even when `sources/` exists
  (`scripts/build --force-resync dev-image ...`). `--force-checkout`
  overwrites uncommitted edits in `sources/*`; the affected projects are
  listed first. Every project ends up on a detached HEAD at its manifest
  revision; commits on local branches are kept in those branches.
  This runs on the host, so your git/SSH access to the layer remotes
  applies.
- **Caches.** `sstate/` goes in the directory you launch the script from
  (`ASTRAOS_SSTATE_DIR` to override); downloads go to `~/yocto/downloads`
  (`ASTRAOS_DL_DIR`). Launch from the same directory each time to keep the
  cache warm.
- **Workspace.** The build dir is `build-<MACHINE>/` in the checkout
  (`ASTRAOS_BUILDER_PATH` to override).

Output lands in `build-<MACHINE>/tmp-<MACHINE>/deploy/images/<MACHINE>/`:

- dev: `astraos-image-dev-<MACHINE>.rootfs.wic.zst`
- prod: the `astraos-image-astrax-variscite-imx8mp.*` files in the same
  directory

## Remote builds

Add `--remote`:

```bash
scripts/build --remote dev-image imx8mp-var-dart     # Symphony, dev
scripts/build --remote dev-image astrax-variscite-imx8mp # mcb, dev
scripts/build --remote prod-image                    # mcb, production
```

The script ssh-es to `akothapalli@10.11.12.20`, then on the builder:

1. creates `sstate/` and `downloads/` and, if the workspace is missing,
   clones it and runs `repo init` + `repo sync`;
2. runs `git pull --ff-only` (retried 3x) and `repo sync` in the workspace;
3. re-runs `scripts/build --local ...` there.

Because of step 2, **the builder builds what is on GitHub `main`, not your
working tree.** Commit and push script/layer changes before a remote build.
The builder keeps its sstate at `/home/akothapalli/yocto/sstate`.

Fetching the result, matching the build you ran:

```bash
scripts/build download-dev imx8mp-var-dart [<dest-dir>]   # dev wic.zst, default dest: current dir
scripts/build download-dev astrax-variscite-imx8mp [<dest-dir>]
scripts/build download-prod [<dest-dir>]                   # prod wic, mcb
```

Other remote-only helpers: `scripts/build clean` (always targets the
builder; rejects `--local`) wipes the builder workspace and re-syncs,
keeping caches. Override the target with `ASTRAOS_BUILDER_HOST`,
`ASTRAOS_BUILDER_USER`, `ASTRAOS_BUILDER_PORT`, `ASTRAOS_BUILDER_PATH`,
`ASTRAOS_BUILDER_HOST_ROOT`.

## Other useful commands

```bash
scripts/build [--remote] shell <MACHINE>             # interactive shell, env sourced
scripts/build [--remote] recipe <MACHINE> <recipe> [task]
scripts/build [--remote] bitbake <MACHINE> <args...>
scripts/build [--remote] sdk <MACHINE>
scripts/build [--remote] publish <MACHINE>           # dev rpm feed, Variscite MACHINEs
scripts/build [--remote] container                   # rebuild the builder image
```

`scripts/build --help` lists every subcommand and environment variable.
