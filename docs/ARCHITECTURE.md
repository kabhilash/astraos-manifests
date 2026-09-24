# AstraOS Architecture

Custom embedded Linux distribution for the AstraX product line, built with the
Yocto Project. This document is the source of truth for the OS-level
architecture; the application (`astraX_BT`) has its own architecture doc in
its repository.

## Targets

| Role | SoC | Board | MACHINE | Status |
|---|---|---|---|---|
| Production | NXP i.MX8M Plus | Variscite VAR-SOM-MX8M-Plus on custom carrier `mcb` | `astrax-variscite-imx8mp` | **Hardware pending** — MACHINE spec authored, not yet exercised |
| Variscite dev | NXP i.MX8M Plus | Variscite VAR-SOM-MX8M-Plus on Variscite Symphony **v1.7** | `imx8mp-var-dart` (Variscite's combined machine name for both DART-MX8M-PLUS and VAR-SOM-MX8M-Plus) | Active — primary Variscite dev target until `mcb` arrives |
| RPi dev | Broadcom BCM2712 | Raspberry Pi 5 | `raspberrypi5` (upstream) | Active |
| RPi dev | Broadcom BCM2712 | Raspberry Pi Compute Module 5 on the official CM5 IO Board | `raspberrypi-cm5-io-board` (upstream) | Active |

Production is the only target with secure boot fuses blown; dev targets run
the same source tree unsigned.

**Implication of `mcb`-pending:** the full production stack (HABv4 chain,
encrypted boot, dm-verity, signed RAUC bundle, factory provisioning) cannot
be end-to-end validated until `mcb` hardware lands. Components can be
validated in pieces on Variscite Symphony v1.7 with HAB in test mode (fuses
unblown), but a clean production dry run requires real hardware. Plan
production-image milestones around `mcb` arrival, not earlier.

## Design Principles

These shape every package, recipe, and feature decision. Read before
adding anything to `IMAGE_INSTALL`, `TOOLCHAIN_TARGET_TASK`, or
`DISTRO_FEATURES`.

1. **Lean production image; lean SDK.** Only ship what's needed at
   runtime / what app developers need at build. No qtwebengine, no
   qtmultimedia, no debug CLIs, no exploratory packages "in case
   someone wants them later." Bloat in embedded products is a long-tail
   liability — eMMC pressure, longer OTA windows, more attack surface,
   more CVE noise, slower boot. Default to dev-image-only or omit
   entirely; promote to production only with a stated runtime need.
2. **Read-only by default.** Anything mutable goes through `/data`.
   Code can't write to `/etc`, `/var`, `/opt` — that's a feature.
3. **Reproducible.** Every artifact (wic, RAUC bundle, SDK installer,
   SBOM, CVE report) traces to a single git SHA. No build-machine
   nondeterminism.
4. **Production-vs-dev images split cleanly.** Production target
   (`astrax-variscite-imx8mp`) is signed, dm-verity, HMI-only. Dev images
   (RPi5, CM5, Variscite Symphony) get debug tools, SSH, writable
   rootfs. Never mix.

## Foundation

| Item | Choice |
|---|---|
| Yocto release | Scarthgap 5.0 LTS (supported until April 2028) |
| Source layout | google-repo, manifest in this repo (`manifests/*.xml`) |
| Variscite BSP | NXP L6.6.52_2.2.2 (`mx8mp-yocto-scarthgap-6.6.y_2.2.2-v1.2`) |
| DISTRO | `astraos` defined in `meta-astrax/conf/distro/astraos.conf` |
| Init system | systemd |
| Package format | `package_rpm`; runtime `dnf`/`rpm` in dev image only |

`PACKAGE_CLASSES = "package_rpm"` distro-wide. The **dev image** includes
`package-management` in `IMAGE_FEATURES`, so engineers can iterate against
a running device with `dnf install <pkg>` / `dnf upgrade` pointed at the
build machine's published dev package feed (see "Dev package feed"
below). The **production image**
deliberately omits `package-management`: zero `dnf`/`rpm` binaries in
the runtime image, all installation/update goes through RAUC bundle
swaps.

## Layer Responsibility

Three Astra-owned Yocto layers, each in a separate git repository pulled in
by `repo sync`:

```
meta-astrax/                       common, hardware-agnostic
├─ conf/distro/astraos.conf        DISTRO_FEATURES, PACKAGE_CLASSES
├─ recipes-core/                   systemd units, /data bind-mounts
├─ recipes-graphics/               Qt EGLFS shared config
├─ recipes-connectivity/           BlueZ, wpa-supplicant, Avahi service files
├─ recipes-astrax/                 astraX_BT recipe + sibling daemons
├─ recipes-rauc/                   RAUC system config + signing-key consumer
├─ recipes-images/                 astraos-image.bb, astraos-image-dev.bb
└─ classes/                        shared bbclasses

meta-astrax-raspberrypi/           thin: RPi-specific overrides only
├─ recipes-bsp/                    RAUC tryboot or u-boot-rauc integration
├─ recipes-kernel/                 CONFIG fragments
└─ wic/                            astraos-rpi.wks.in

meta-astrax-variscite/             Variscite + mcb-carrier overrides
├─ conf/machine/astrax-variscite-imx8mp.conf
├─ recipes-bsp/                    U-Boot mcb defconfig fragment, RAUC bootcount
├─ recipes-kernel/linux-variscite/files/imx8mp-var-dart-mcb.dts
├─ recipes-graphics/               Qt eglfs_kms_imx config for Vivante
├─ recipes-multimedia/             imx-gpu-viv .bbappend if needed
└─ wic/                            astraos-variscite.wks.in
```

App + distro both live in `meta-astrax`. Will split out a `meta-astrax-app`
layer only if the app outgrows ~10 recipes or a separate team takes ownership.

## Display Stack

| Item | Choice |
|---|---|
| UI architecture | Transitioning from Windows-PC UI (over WebSocket) to on-device Qt HMI |
| Qt version | 6.8.3 (LTS) |
| Display server | None — Qt EGLFS direct to DRM/KMS |
| GPU on i.MX8M Plus | Vivante (`imx-gpu-viv`), Qt `eglfs_kms_imx` |
| GPU on RPi | V3D mesa (only option) |
| `DISTRO_FEATURES` | includes `opengl bluez5 bluetooth systemd`; explicitly excludes `wayland x11 vulkan-graphics` |

The Windows-PC Qt UI continues to work in parallel via the WebSocket server
on port 8765.

## Networking & Connectivity

- **`nexus`** is the unified connectivity manager — Rust daemon at
  `fi.nexus1` on the system D-Bus. It manages Ethernet, WiFi, BLE
  and GNSS via per-technology backends, delegates IP configuration
  to `systemd-networkd`, and stores connection profiles + paired-device
  state under `/var/lib/nexus` (bind-mounted from `/data/var/lib/nexus`).
  `astraX_BT`'s `platformManager` calls into `nexus` over D-Bus rather
  than driving BlueZ / wpa-supplicant directly. See sibling repo
  `../nexus/` and its `docs/nexus-architecture.md`.
- `systemd-networkd` + `systemd-resolved` (no `connman`, no `NetworkManager`)
- `wpa-supplicant` for WiFi (driven by `nexus`)
- BlueZ 5 for BLE (driven by `nexus` over D-Bus; not linked at build time)
- Avahi mDNS, advertising `_astrax._tcp.local.` with hostname `AstraX-<serial>`
- WebSocket server on port 8765, LAN-only (firewall to interface)
- WiFi/BT radio is **on the Variscite VAR-SOM-MX8M-Plus** (not the `mcb`
  carrier). Driver + firmware packages flow in via `meta-variscite-bsp-imx`.
  Standard module is Murata 1ZM (NXP 88W8987) — verify SKU before locking
  the `linux-firmware-nxp*-sdio` package.
- `syncManager` (in `astraX_BT`) uploads XRF test results to **AstraX cloud**.
  Authenticates via the per-device X.509 cert provisioned at factory time.
  Currently uses Qt6::Network HTTPS (no Azure IoT SDK link); endpoint,
  transport (HTTPS / MQTT / gRPC), retention policy, and retry/backoff are
  open threads — see below.

## Storage & Updates

### Partition Layout

```
boot       FAT32 / per-machine bootloader artifacts
rootfs.A   squashfs (zstd), read-only, ~400-700 MB
rootfs.B   squashfs (zstd), read-only, same size as A
data       ext4, remainder of eMMC
           /data/astrax/identity/    serial + per-device cert
           /data/astrax/db/          SQLite results store
           /data/var/lib/bluetooth   BLE pairings (bind-mounted to /var/lib/bluetooth)
           /data/var/lib/astrax      app state
           /data/log/journal         persistent journald (Storage=persistent, SystemMaxUse=200M)
           /data/rauc/               RAUC bundle staging
```

Read-only-rootfs `IMAGE_FEATURE` handles the standard bind-mounts; custom
mounts for BlueZ, app state, and journald are owned by `meta-astrax`.

### OTA — RAUC, A/B Slots

- `meta-rauc` layer + per-target bundle recipes
- Bundle = full rootfs slot (squashfs), signed with PKI tied to HABv4 chain
- Bootloader (U-Boot on i.MX, tryboot or u-boot-rauc on RPi) tracks
  `BOOT_ORDER` + `bootcount` for atomic slot flip and rollback on failed
  boot
- Update source: signed bundles served from public S3/CDN (no server-side
  auth needed; signature is the trust anchor). Per-device cert used for
  mTLS to the OTA endpoint as defense-in-depth
- Telemetry / fleet console: deferred to v2

### Dev package feed

Dev images (Variscite only for now) update in place with `dnf` from an
rpm feed on the build machine. `astrax-updater` (sibling repo) is the
on-device client. The feed never serves the production image.

`scripts/build publish <MACHINE>` (`imx8mp-var-dart` or
`astrax-variscite-imx8mp`):

1. builds `astraos-image-dev` and runs `package-index` (a no-op build
   when the image is current)
2. writes `astrax-manifest.json` via `scripts/feed-manifest`: the image's
   package set with the exact NEVRA, rpm path and rpm header SHA-256 of
   each package, plus the git SHA of every layer
3. snapshots `deploy/rpm/` into
   `${ASTRAOS_FEED_ROOT}/<MACHINE>/<build-id>/` (rpms hardlinked,
   repodata copied) and atomically points `<MACHINE>/main` at it
4. keeps the newest `ASTRAOS_FEED_KEEP` (5) snapshots per MACHINE
5. makes sure the `astraos-feed` nginx container is serving
   `${ASTRAOS_FEED_ROOT}` on `ASTRAOS_FEED_PORT` (8090)

```
http://10.11.12.20:8090/<MACHINE>/main/                  current snapshot
http://10.11.12.20:8090/<MACHINE>/<build-id>/            retained snapshots
http://10.11.12.20:8090/<MACHINE>/main/astrax-manifest.json
```

Why snapshots rather than serving `deploy/rpm/` directly: any bitbake
run after the metadata changes prunes the stale outputs of changed
recipes from `deploy/rpm/`, and the next build rewrites it in place. A
board reading the live directory can see a feed that no longer matches
any image. Snapshots stay put.

Why the manifest pins exact NEVRA and header digests: there is no PR
server, so a rebuilt package can keep its version-release, and stale
rpms left in `deploy/rpm/` make "latest" ambiguous (`+git<sha>` versions
don't sort by age). Boards install the exact NEVRA the manifest names,
and compare header digests to spot rebuilds.

Plain HTTP, no auth, `gpgcheck=0`: the feed is reachable only on the LAN
and tailnet, and carries dev packages only.

## Security

| Layer | Mechanism |
|---|---|
| Boot ROM → SPL → U-Boot → FIT image | HABv4 (i.MX High Assurance Boot v4) chain of trust |
| Boot confidentiality | **Encrypted boot via HABv4 + DEK blob** — boot artifacts are encrypted, not just signed. Per-device DEK wrapped to CAAM master key |
| Rootfs integrity | dm-verity, hash tree built into wic at build time |
| OTA bundles | RAUC bundles signed with key chained to HABv4 PKI |
| Fuses | SRK fuses blown at factory provisioning, **production only** — dev units stay open |
| App secret storage | **CAAM-backed key blobs** — accessed from normal world via kernel `caam` driver; no OP-TEE in v1 |
| Secure-world execution | None — **no OP-TEE in v1**. Revisit if attestation or sealed compute is needed |

**Encrypted boot flow:** SPL contains the wrapped DEK (encrypted to CAAM
master key, unique per chip). At boot, ROM verifies signed SPL via HABv4;
SPL asks CAAM to unwrap the DEK; SPL uses DEK to decrypt U-Boot + FIT image
before transferring control. Anyone dumping eMMC sees ciphertext only.

**CAAM usage pattern for app secrets:** application generates a key, calls
CAAM (via `/dev/crypto` or `caam_keygen`) to wrap it as a blob, stores blob
on `/data/astrax/secrets/`. To use, unwrap via CAAM. The master key
(per-chip, fused) never leaves the CAAM hardware. Per-device cert private
key, sync-data encryption keys, and any future PII keys all use this
pattern.

PKI custody, HSM model, and signing-key rotation: deferred design topic.
Production signing (HABv4 SRK + DEK wrapping + RAUC bundle key) happens in
a dedicated locked-down CI runner with HSM-held keys accessed via PKCS#11.

## Image Variants

Two image recipes in `meta-astrax/recipes-images/` sharing a base:

- `astraos-image.bb` (production)
  - `IMAGE_FEATURES = "read-only-rootfs license-pkg"`
  - dm-verity enabled
  - HMI service + workers + Avahi + network stack only
  - No SSH, no shell access by default
  - HABv4-signed boot chain
  - Targets `astrax-variscite-imx8mp` only
- `astraos-image-dev.bb` (development)
  - `IMAGE_FEATURES += "debug-tweaks tools-debug ssh-server-dropbear tools-profile"`
  - Writable rootfs
  - `gdbserver`, `strace`, `tcpdump`, `perf`, `htop`
  - Root SSH with known dev key
  - journald to console
  - Targets `raspberrypi5`, `raspberrypi-cm5-io-board`, `imx8mp-var-dart`

## Application

`astraX_BT` (XRF Benchtop) lives in a sibling repo. Yocto integration:

- `meta-astrax/recipes-astrax/astrax-bt/astrax-bt_git.bb` — fetches pinned
  SRCREV, builds via `inherit cmake systemd qt6-cmake`
- `DEPENDS = "qtbase qtdeclarative qtquickcontrols2 qtwebsockets spdlog libusb1 libgpiod sqlite3"` —
  derived from the project's actual `find_package` / `pkg_check_modules` /
  `target_link_libraries` calls, kept lean per design principle
- `RDEPENDS` adds `bluez5`, `dbus`, `nexus`, `avahi-daemon` — used at
  runtime via D-Bus, not linked at build time
- One systemd unit per daemon (xrfapiManager, testManager, etc.) with
  explicit ordering and `Restart=on-failure`
- Per the architecture doc, components communicate via Qt signal-slot
  in-process; structure may be a single binary or split. To be confirmed
- App-developer iteration: `devtool modify astrax-bt` → edit → `devtool
  build astrax-bt` → `devtool deploy-target astrax-bt root@<dev-ip>`

The exported Yocto SDK includes:
- Qt 6.8.3 cross sysroot
- `nativesdk-rust` + `nativesdk-cargo` (latest Rust via `meta-lts-mixins/scarthgap/rust`)
- protobuf, sqlite3, bluez5 dev headers

## Compliance & Release Engineering

In `astraos.conf`:

```
INHERIT += "create-spdx archiver cve-check"
SPDX_INCLUDE_SOURCES = "1"
ARCHIVER_MODE[src] = "patched"
LICENSE_CREATE_PACKAGE = "1"
COPY_LIC_MANIFEST = "1"
COPY_LIC_DIRS = "1"
CVE_CHECK_FORMAT_JSON = "1"
CVE_CHECK_REPORT_PATCHED = "1"

BB_HASHSERVE = "auto"
BB_SIGNATURE_HANDLER = "OEEquivHash"
```

CI publishes per-build artifacts:

- `astraos-image-<machine>-<sha>.wic`
- `astraos-bundle-<machine>-<sha>.raucb`
- `astraos-image-<machine>-<sha>.spdx.json` (SBOM)
- `astraos-image-<machine>-<sha>-license-manifest.txt`
- `astraos-image-<machine>-<sha>-cve.json` + `.html`

CVE-check gates releases: block on Critical/High, allow-with-acknowledgment
on Medium/Low via `CVE_CHECK_IGNORE_FILES`.

## CI/CD

### Build Machine

A single dedicated build server handles all Yocto builds. Local laptops do
not run BitBake (the devcontainer can build, but practical builds go to
the server because it has the cores, RAM, and persistent caches).

| Property | Value |
|---|---|
| Hostname / LAN IP | `10.11.12.20` |
| Tailscale IP | `100.82.113.92` |
| SSH | port 22, user `akothapalli`, key-based (passwordless) |
| Host OS | Ubuntu 26.04 |
| CPU | 48 cores / 96 threads |
| RAM | 1 TB |
| Workspace path | `/home/akothapalli/yocto/astraos/` (NVMe-backed; sstate/downloads sit alongside under `/home/akothapalli/yocto/`) |
| Builds run inside | the AstraOS devcontainer (Ubuntu 24.04, Yocto-supported) |

`scripts/build` defaults to remote execution against this host (`--local`
opts out for tiny iteration on the dev machine). BB_NUMBER_THREADS and
PARALLEL_MAKE auto-detect to 96 inside the container, fully utilising the
machine.

### Other CI infrastructure

- Self-hosted runner pool: the build machine above is the runner;
  ≥2 hosts in time for resilience
- Persistent caches on local NVMe (mirrored to S3 for restore):
  - `SSTATE_DIR` shared across branches
  - `DL_DIR` upstream tarball cache
- Containerized builds (`crops/poky` or equivalent), bind-mounting caches
- Build matrix per branch / PR:
  - `astraos-image-dev` × `raspberrypi5`
  - `astraos-image-dev` × `raspberrypi-cm5-io-board`
  - `astraos-image-dev` × `imx8mp-var-dart`
  - `astraos-image` × `astrax-variscite-imx8mp` (signed, on `main`/release branches)
- Production signing: dedicated locked-down signing runner, HABv4 SRK +
  RAUC bundle keys held in HSM, accessed via PKCS#11. Keys never leave HSM,
  never visible to general CI runners

## Factory Provisioning

Two-stage flow, HAB fuse-blow last:

```
Factory PC
    │ uuu (NXP universal update util) over USB OTG
    ▼
Stage 1: astraos-image-factory  (signed by factory key, NOT prod key)
    1. read OTP UID from i.MX8M Plus fuses
    2. derive serial: AX-<YYWW>-<6-digit-counter>
    3. mTLS to Astra factory-CA → per-device X.509 cert
    4. CAAM blob-wrap the cert private key → /data/astrax/identity/
       (master key is per-chip, never leaves CAAM)
    5. generate per-device DEK via CAAM, encrypt boot artifacts (SPL,
       U-Boot, FIT image) with DEK; store DEK blob alongside artifacts
    6. write serial + cert + identity manifest → /data/astrax/identity/
    7. flash encrypted+signed boot artifacts to boot partition
    8. flash production astraos-image to rootfs.A
    9. flash same to rootfs.B
   10. write RAUC slot.status (slot.A=good, slot.B=good)
   11. set U-Boot env: boot rootfs.A
   12. *** blow HABv4 SRK fuses (IRREVERSIBLE) ***
   13. reboot
    │
    ▼
Stage 2: production astraos-image
    ROM verifies signed SPL → U-Boot → FIT → dm-verity rootfs
    mounts /data, reads identity, advertises AstraX-<serial> via mDNS
    runs astrax-bt service
```

Dev devices (RPi5, CM5) skip all of this: `dd astraos-image-dev-*.wic` to SD
card. CM5 first-eMMC flash via `rpiboot`. Serial = built-in CPU serial.

Dependencies (open threads):
- Astra factory-CA infra (e.g., `step-ca` on hardened VM) — set up before
  first production run
- Serial / counter registry — embedded in factory CA's issued cert serial
- Factory PC tooling — Python/Bash wrapper around `uuu` + factory-CA call
  + fuse-blow command, in `AstraOS/scripts/factory/`

## Open Threads

Decisions deliberately deferred. Each will need its own grilling session
when the time comes.

| Thread | Why deferred | When to resolve |
|---|---|---|
| PKI custody (HSM model, key rotation, DEK wrapping) | Operational, not architectural | Before first production signing |
| OP-TEE / TrustZone | Locked OUT of v1 | Revisit only if attestation or sealed compute needed |
| Variscite SOM WiFi/BT module SKU | Need to confirm 88W8987 vs 88W8997 vs other | Before locking `linux-firmware-nxp*-sdio` package |
| AstraX cloud sync transport | HTTPS vs MQTT vs gRPC, endpoint, retention/retry | When AstraX cloud API is defined |
| Camera / GPS in `platformManager` | Scope unclear in v1 | Confirm before kernel CONFIG fragments |
| Telemetry / observability agent | Deferred to v2 | When fleet management needed |
| Recovery / failsafe boot policy | RAUC bootcount tuning | During RAUC integration |
| RPi RAUC bootloader path | tryboot vs u-boot-rauc | During RAUC integration |
| Kernel CONFIG fragments | Derive from `astraX_BT` hardware needs | During first kernel build |
| `astraX_BT` process structure | Single binary or split daemons? | Confirm during recipe authoring |

## Source-of-Truth Pointers

- Variscite BSP release notes:
  https://dev.variscite.com/var-som-mx8m-plus/mx8mp-yocto-scarthgap-6.6.y_2.2.2-v1.2/release-notes/
- AstraOS (this repo): `https://github.com/kabhilash/astraos-manifests.git`
- `meta-astrax`: `https://github.com/kabhilash/meta-astrax.git`
- `meta-astrax-raspberrypi`: `https://github.com/kabhilash/meta-astrax-raspberrypi.git`
- `meta-astrax-variscite`: `https://github.com/kabhilash/meta-astrax-variscite.git`
- `nexus` (Rust connectivity manager): `https://github.com/kabhilash/nexus.git`
- `astraX_BT` (XRF Benchtop application): sibling repo at `../astraX_BT/` —
  GitHub: `git@github.com:AstraAnalytical/astraX_BT.git` (different org;
  the build machine's SSH key has read access)
  - Architecture doc: `astraX_BT/Docs/ARCHITECTURE.md`
  - Service discovery: `astraX_BT/Docs/SERVICE_DISCOVERY.md`
- meta-qt6 (need SRCREV that ships Qt 6.8.3 on Scarthgap):
  https://code.qt.io/cgit/yocto/meta-qt6.git/log/?h=scarthgap
- meta-lts-mixins (rust branch for latest Rust on Scarthgap):
  https://git.yoctoproject.org/meta-lts-mixins/log/?h=scarthgap/rust
- meta-rauc, meta-rauc-community: TBD pinning during RAUC integration
