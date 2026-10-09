---
name: astraos-shell
description: Open an interactive bash shell inside the AstraOS build environment (devcontainer) with `setup-environment` already sourced for the given MACHINE. Use whenever the user wants to "open an AstraOS shell", "give me a build shell for raspberrypi5", "drop me into the devcontainer for cm5", "I want to poke around BitBake interactively for the variscite", "let me run multiple commands in the build env without re-sourcing each time", or any phrasing that means "I need a long-lived terminal session in the AstraOS build env". For one-off invocations, prefer the more specific skills (`astraos-bitbake`, `astraos-build-recipe`) instead — they don't require leaving an interactive shell open.
---

Run the shell subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build shell <MACHINE>
```

This is **interactive**: the user lands at a bash prompt inside the
devcontainer on the local Docker daemon. The
prompt has BitBake on PATH, `setup-environment <MACHINE>` already
sourced, and `cwd = build-<MACHINE>/` in the workspace.
The user types `bitbake ...` directly, edits files, runs
`devtool modify` workflows, etc.
Exiting with `Ctrl-D` or `exit` returns control to the user's host
shell.

## Extracting MACHINE

Same as `astraos-build-dev`. If unspecified, ask which target — the
shell's `bblayers.conf` and `local.conf` are MACHINE-dependent, so
this matters.

## When to use vs. alternatives

- One-off bitbake call → `astraos-bitbake` (no need for a shell)
- One-off recipe build → `astraos-build-recipe`
- Full image → `astraos-build-dev` / `astraos-build-prod`
- Multiple commands in a row, exploration, devtool workflows,
  troubleshooting → **this skill**

## Practical note

If the user expects a long-running interactive session (e.g., devtool
modify + edit + build + deploy cycles), suggest `tmux` / `screen`, or
just keep their terminal open.

## Where it runs

The build runs in the local Docker daemon, in this checkout:

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
