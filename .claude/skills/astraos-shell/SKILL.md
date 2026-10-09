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
devcontainer (on the local Docker daemon by default, or the build machine via SSH with `--remote`). The
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

For remote shells the user must keep the SSH session alive. If they
expect a long-running interactive session (e.g., devtool modify +
edit + build + deploy cycles), suggest `tmux` / `screen` on the build
machine, or just keep their terminal open.

## Local vs remote

Default is local: the build runs in the local Docker daemon, in this
checkout. `--remote` (e.g. `./scripts/build --remote shell <MACHINE>`) dispatches to
the build machine instead; only pass it if the user asks for it. The
builder builds what is on GitHub `main` (it `git pull`s and `repo sync`s
first), so push script or layer changes before a remote build.

For local runs:

- missing `sources/poky` is fetched automatically (`repo init` +
  `repo sync`); add `--force-resync` to re-run `repo init` + `repo sync --force-checkout --detach`
  on an existing `sources/` (discards uncommitted edits there; lists them first)
- sstate lives in `./sstate` of the directory the script is launched
  from (`ASTRAOS_SSTATE_DIR` to override), downloads in
  `~/yocto/downloads` (`ASTRAOS_DL_DIR`) — launch from the same
  directory each time to keep the cache warm
- the build dir is `build-<MACHINE>/` inside the checkout

Full walkthrough: `docs/BUILDING.md`.
