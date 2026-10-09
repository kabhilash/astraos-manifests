---
name: astraos-clean-workspace
description: Wipe `/home/akothapalli/yocto/astraos` on the AstraOS build machine and re-bootstrap from GitHub (`kabhilash/astraos-manifests`) via `repo init` + `repo sync`. Use whenever the user asks to "clean the workspace", "wipe the build server", "reset astraos", "fresh checkout", "nuke the workspace", "start from scratch on the builder", or any phrasing that implies a full workspace reset. Preserves `/home/akothapalli/yocto/{sstate,downloads}` so the next build is fast (caches still warm). Do NOT use this for routine BitBake `cleansstate` or `clean` invocations on a single recipe — those are a different operation; use `astraos-build-recipe` with task `cleansstate` instead.
---

Run the clean subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build clean
```

This is destructive on the build machine — confirm with the user
before running if there's any chance of losing in-flight work on the
server's working tree (e.g., the user has been hand-editing a recipe
under `/home/akothapalli/yocto/astraos` directly without pushing).
Once confirmed, run.

What happens behind the scenes (no need to repeat each step manually):

1. SSH to `akothapalli@10.11.12.20`
2. `rm -rf /home/akothapalli/yocto/astraos`
3. `git clone https://github.com/kabhilash/astraos-manifests.git /home/akothapalli/yocto/astraos`
4. `repo init -u https://github.com/kabhilash/astraos-manifests.git -m manifests/default.xml`
5. `repo sync -j16`

`/home/akothapalli/yocto/sstate` and `/home/akothapalli/yocto/downloads`
are preserved across the wipe. After the clean completes, the user can
immediately invoke any of the other `./scripts/build` subcommands.

The clean subcommand cannot run with `--local` — it specifically
targets the build machine.

For a fresh `sources/` on a **local** checkout this skill is not needed:
`./scripts/build --local --force-resync <subcommand> ...` re-runs
`repo sync`, and a missing `sources/poky` is fetched automatically.
