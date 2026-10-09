---
name: astraos-build-container
description: Rebuild the AstraOS Yocto build devcontainer Docker image. Use whenever the user asks to "rebuild the devcontainer", "rebuild the AstraOS container", "rebuild the docker image", "refresh the build container", or whenever they've changed `.devcontainer/Dockerfile` or `devcontainer.json` and need to apply those changes. The container build runs on the local Docker daemon.
---

Run the container subcommand of the AstraOS build script:

```
# from the repo root (the astraos-manifests checkout)
./scripts/build container
```

The container image is tagged after the Yocto release of the manifest
`scripts/build` uses (`MANIFEST`): `astraos-builder:latest` for
`manifests/default.xml` (Scarthgap, the default), `astraos-builder:wrynose` for
`manifests/wrynose.xml`. After it
finishes, subsequent `./scripts/build dev-image|sdk|recipe|...` calls will
use the new image automatically.
