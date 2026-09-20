---
name: astraos-bitbake
description: Run an arbitrary BitBake invocation (with any flags / multiple targets / advanced options) inside the AstraOS build environment for a target MACHINE. Use whenever the user wants something more flexible than `astraos-build-image`/`astraos-build-recipe` — e.g. "bitbake -k -v qtbase qtdeclarative for raspberrypi5", "bitbake with --runall=fetch on cm5", "build two recipes together with continue-on-error", "run BitBake with verbose tracing". Prefer the more specific skills (image, recipe, sdk) when the request fits one of them; reach for this skill when the user is doing power-user BitBake work.
---

Run the bitbake passthrough subcommand of the AstraOS build script:

```
cd /Users/akothapalli/Projects/AstraA/X/AstraOS
./scripts/build bitbake <MACHINE> <bitbake-args...>
```

Everything after `<MACHINE>` is forwarded verbatim to BitBake. The
script first sources `setup-environment <MACHINE>` (which writes
`build/conf/{bblayers,local}.conf` for that MACHINE), then runs
`bitbake <args...>`.

## Extracting parameters from the user's request

**MACHINE** — same mapping as `astraos-build-image` (raspberrypi5,
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
