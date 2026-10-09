---
name: astraos-bitbake
description: Run an arbitrary BitBake invocation (with any flags / multiple targets / advanced options) inside the AstraOS build environment for a target MACHINE. Use whenever the user wants something more flexible than `astraos-build-dev`/`astraos-build-recipe` — e.g. "bitbake -k -v qtbase qtdeclarative for raspberrypi5", "bitbake with --runall=fetch on cm5", "build two recipes together with continue-on-error", "run BitBake with verbose tracing". Prefer the more specific skills (image, recipe, sdk) when the request fits one of them; reach for this skill when the user is doing power-user BitBake work.
---

Run the bitbake passthrough subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build bitbake <MACHINE> <bitbake-args...>
```

Everything after `<MACHINE>` is forwarded verbatim to BitBake. The
script first sources `setup-environment <MACHINE>` (which writes
`build-<MACHINE>/conf/{bblayers,local}.conf` for that MACHINE), then runs
`bitbake <args...>`.

## Extracting parameters from the user's request

**MACHINE** — same mapping as `astraos-build-dev` (raspberrypi5,
raspberrypi-cm5-io-board, imx8mp-var-dart, astrax-variscite-imx8mp). If
unspecified, ask.

**bitbake-args** — the flags + recipes the user mentioned, in order.
Common BitBake flags:

| Flag | Meaning |
|---|---|
| `-k` | continue on errors (don't stop at first failure) |
| `-v` | verbose |
| `-D` | debug output (more `-D`s = more verbose) |
| `-c <task>` | run a specific task (alternative: use `astraos-build-recipe`) |
| `--runall=<task>` | force-run the named task on all matched recipes |
| `-f` | force-rerun even if already complete |
| `-S printdiff` | show signature differences (debug sstate misses) |
| `-e` | dump full build environment for inspection |
| `--show-versions` | show available versions for all recipes |

## Examples

User says "bitbake qtbase qtdeclarative -k -v for raspberrypi5":
```
./scripts/build bitbake raspberrypi5 qtbase qtdeclarative -k -v
```

User says "force fetch everything for cm5":
```
./scripts/build bitbake raspberrypi-cm5-io-board --runall=fetch astraos-image-dev
```

User says "show me the full env for the spdlog recipe on the
variscite":
```
./scripts/build bitbake imx8mp-var-dart -e spdlog
```

If the user's request fits the simpler `image` / `prod-image` / `sdk` /
`recipe` skills, prefer those — they're easier to read in chat history
than a flag-heavy `bitbake` line.

## Local vs remote

Default is local: the build runs in the local Docker daemon, in this
checkout. `--remote` (e.g. `./scripts/build --remote bitbake <MACHINE> <args...>`) dispatches to
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
