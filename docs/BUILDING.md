# Building AstraOS

Everything goes through `scripts/build`. It runs BitBake inside the
`astraos-builder` devcontainer image on your own Docker daemon, in this
checkout.

The image tag follows the manifest `scripts/build` initialises the layers
from (`MANIFEST` near the top of the script): `astraos-builder:latest` for
`manifests/default.xml` (Scarthgap), `astraos-builder:wrynose` for Wrynose, so both
releases can be built on one Docker daemon without overwriting each other.

## Yocto releases: Scarthgap (default) and Wrynose

Two manifests live side by side: `manifests/default.xml` (Scarthgap, the
default) and `manifests/wrynose.xml`. Pick Wrynose with
`ASTRAOS_MANIFEST=manifests/wrynose.xml scripts/build ...`; use a separate
checkout per release. The release is detected from the synced core layer
(`LAYERSERIES_CORENAMES`), and `scripts/setup-environment` adapts what
differs:

| | Scarthgap | Wrynose |
|---|---|---|
| Core layers | `sources/poky` | `sources/openembedded-core` + `meta-yocto` |
| Extra layers | `meta-lts-mixins` (Rust) | `meta-freescale-distro`, `meta-perl` (i.MX) |
| `local.conf` | `TCLIBCAPPEND = ""`, `BB_DANGLINGAPPENDS_WARNONLY` | neither (parse error) |
| Builder image | `astraos-builder:latest`, Ubuntu 24.04 | `astraos-builder:wrynose`, Ubuntu 26.04 |
| Caches | `sstate-scarthgap/`, `downloads-scarthgap/` | `sstate-wrynose/`, `downloads-wrynose/` |

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

Prerequisites: Docker, and `repo` on `PATH`. A cold build is slow; run it
on a machine with plenty of cores and disk (see `docs/ARCHITECTURE.md`,
Build Machine).

- **Layers.** If `sources/openembedded-core` is missing the script runs `repo init` and
  `repo sync` in the checkout first. `--force-resync` re-runs
  `repo init` and `repo sync --force-checkout --detach` even when `sources/` exists
  (`scripts/build --force-resync dev-image ...`). `--force-checkout`
  overwrites uncommitted edits in `sources/*`; the affected projects are
  listed first. Every project ends up on a detached HEAD at its manifest
  revision; commits on local branches are kept in those branches.
  This runs on the host, so your git/SSH access to the layer remotes
  applies.
- **Caches.** `sstate-<codename>/` (e.g. `sstate-scarthgap/`) goes in the directory you launch the script from
  (`ASTRAOS_SSTATE_DIR` to override); downloads go to `~/yocto/downloads-<codename>`
  (`ASTRAOS_DL_DIR`). Launch from the same directory each time to keep the
  cache warm.
- **Workspace.** The build dir is `build-<MACHINE>/` in the checkout.

Output lands in `build-<MACHINE>/tmp-<MACHINE>/deploy/images/<MACHINE>/`:

- dev: `astraos-image-dev-<MACHINE>.rootfs.wic.zst`
- prod: the `astraos-image-astrax-variscite-imx8mp.*` files in the same
  directory

## Other useful commands

```bash
scripts/build shell <MACHINE>             # interactive shell, env sourced
scripts/build recipe <MACHINE> <recipe> [task]
scripts/build bitbake <MACHINE> <args...>
scripts/build sdk <MACHINE>
scripts/build publish <MACHINE>           # dev rpm feed, Variscite MACHINEs
scripts/build container                   # rebuild the builder image
```

`scripts/build --help` lists every subcommand and environment variable.
