# AGENTS

## Scope and entrypoints
- This repo builds the **live installer ISO** only; the installer app itself comes from the external `seapath-installer` `.deb` fetched in `build.sh`.
- Main build entrypoint is `./build.sh`; it wraps `make build` and then post-processes the ISO.
- `auto/config` is the live-build source of truth for distro/arch/image settings (`bookworm`, `amd64`, `iso-hybrid`).

## Build commands (verified)
- Preferred path (matches CI/tooling): `cqfd init` once, then `cqfd` for local builds.
- CI flavor commands (see `.github/workflows/ci-seapath-installer.yml`):
  `cqfd -b ci_empty` (empty variant) then `cqfd -b ci` (full variant).
- Non-container build: `./build.sh` (requires `sudo` and host deps equivalent to `.cqfd/docker/Dockerfile`).
- Build an empty installer (no SEAPATH images in `DATA/images/`):
  `./build.sh --empty` or `cqfd -b ci_empty`.

## `build.sh` behavior that affects edits
- Version pins are hardcoded at top-level vars: `SEAPATH_IMAGES_VERSION` and `SEAPATH_INSTALLER_VERSION`.
- Default behavior fetches `seapath-installer_${SEAPATH_INSTALLER_VERSION}_all.deb` from GitHub into `config/packages/`.
- `--no-installer-fetch` skips download and expects that exact `.deb` to already exist in `config/packages/`.
- On successful `make build`, script appends a FAT `DATA` partition (`extra_partition.img`, 10 GiB) using `xorriso`, then writes `seapath-live-installer-${SEAPATH_INSTALLER_VERSION}.iso`.
- `--empty` skips SEAPATH artifacts fetch (`fetch_seapath_artifacts`): `DATA/images` stays empty and the output is renamed to `seapath-live-installer-${SEAPATH_INSTALLER_VERSION}-empty.iso`.
- After a successful build the script removes `extra_partition.img` and `live-image-amd64.hybrid.iso` so chained builds (e.g. `ci_empty` then `ci`) start from a clean state.

## Repo structure that matters
- `config/package-lists/*.list.chroot|binary`: package selection for the image.
- `config/includes.chroot/`: files copied into the live filesystem (notably `usr/bin/start-seapath-installer`, `usr/lib/systemd/system/seapath.mount`, `usr/lib/systemd/system/seapath-ssh-keys.service`).
- `config/hooks/normal/*.hook.chroot|binary`: build-time hooks; includes SEAPATH-specific service enabling (`0900-enable-seapath-mount`, `0900-allow-ssh`).
- `Makefile.extra` pulls extra third-party artifacts into `cache/downloads/` and `config/packages.chroot/` during `download_extra`.

## Validation and test reality
- There is no lint/typecheck pipeline in this repo.
- Fast check: `make test_imagesize` (expects built ISO under `iso/*.iso`).
- Full `make tests` also runs libvirt install boots (`virt-install` BIOS+UEFI) and assumes ISO is available in `/var/lib/libvirt/images/`.

## Known doc/config mismatch
- `README.md` references replacing `config/packages/seapath-installer_1.0_all.deb`; executable source (`build.sh`) requires the filename/version tied to `SEAPATH_INSTALLER_VERSION`.
