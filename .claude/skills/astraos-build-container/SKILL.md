---
name: astraos-build-container
description: Rebuild the AstraOS Yocto build devcontainer Docker image. Use whenever the user asks to "rebuild the devcontainer", "rebuild the AstraOS container", "rebuild the docker image", "refresh the build container", or whenever they've changed `.devcontainer/Dockerfile` or `devcontainer.json` and need to apply those changes. By default the container build runs on the local Docker daemon; pass `--remote` if the user wants it to run on the AstraOS build machine via SSH instead.
---

Run the container subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build container
```

If the user explicitly says "build it on the build machine" or similar,
prepend `--remote`:

```
./scripts/build --remote container
```

With `--remote` the script handles the SSH dispatch to the build machine
(`akothapalli@10.11.12.20`, workspace `/home/akothapalli/yocto/astraos`)
on its own. Just invoke the command — the
script knows what to do.

The container image is tagged after the Yocto release of the manifest
`scripts/build` uses (`MANIFEST`): `astraos-builder:latest` for
`manifests/default.xml` (Scarthgap, the default), `astraos-builder:wrynose` for
`manifests/wrynose.xml`. After it
finishes, subsequent `./scripts/build dev-image|sdk|recipe|...` calls will
use the new image automatically.
