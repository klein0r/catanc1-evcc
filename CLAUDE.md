# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A packaging repo, not application source. It wraps the upstream `docker.io/evcc/evcc` Docker image into a PLCnext app (`.app` squashfs) for the Phoenix Contact **Catan C1** controller (arm64). There is no code to compile, and there are no tests or linters. It is a port of the sibling repo `catanc1-zigbee2mqtt`, which has the same structure.

## Build

```bash
sudo apt install -y squashfs-tools docker-ce docker-ce-cli
bash build.sh <app version>   # e.g. 1.2.0; a leading "v" is stripped
```

- The app version is passed as the only argument and is independent of the evcc version in [EVCC_VERSION](EVCC_VERSION).
- Build on Linux: `build.sh` uses GNU `sed -i` and `md5sum`, so it fails on macOS as written.
- Needs Docker >= 25 so that `docker save` emits an OCI archive; the script checks this.
- Output: `evcc_<VERSION>_arm64.app` and a matching `.md5` in the repo root. Both are gitignored, as is the `app/` staging directory.

## Releases

[release.yml](.github/workflows/release.yml) runs when a GitHub release is published: it calls `build.sh` with the release tag as app version and attaches the `.app`, `.md5`, `LICENSE` and `LICENSE-evcc` to the release, zipped, because GitHub rejects `.app` release assets (HTTP 422). `zip` only warns about missing files, so a missing license file does not fail the step. It then appends a "Source code and licenses" section to the release notes (source pointers for the evcc version and this repo, plus a note on bundled third-party licenses); it skips this if the section already exists, so re-runs are safe. It installs its own Docker via `docker/setup-docker-action` with the containerd image store, because `build.sh` reads `.Identity` from `docker inspect` (Docker >= 29.2) and the runner's preinstalled Docker is older.

## Licenses

- [LICENSE](LICENSE): MIT, for the packaging in this repo.
- [LICENSE-evcc](LICENSE-evcc): evcc's own MIT license, shipped with the release zip because the image contains only the evcc binary, not its license text.

## Updating evcc

Most commits only bump the upstream image tag in [EVCC_VERSION](EVCC_VERSION) (just the tag, e.g. `0.318.1`; evcc tags have no `v` prefix). `build.sh` reads it into `TAG`, which goes verbatim into `IMAGE_TAG`; the app version comes from the release tag, not from this file.

## How the pieces fit together

`build.sh` pulls the arm64 image, saves it as `app/image.tar.gz`, copies the two templates into `app/`, fills in build-time placeholders, and then runs `mksquashfs` with uid/gid 1001/1002.

There are two kinds of placeholders. Don't mix them up:

- **Build-time, `{CURLY}`**: `{VERSION}` (app version), `{IMAGE_TAG}` (evcc image tag), `{IMAGE_ID}` and `{IMAGE_IDENTITY}`, filled in by `sed` in `build.sh`. New placeholders also need a dummy value in [validate-app-info.yml](.github/workflows/validate-app-info.yml).
- **Runtime, `${SHELL}`**: `${APP_ID}`, `${APP_PERSISTENT_DIR}`, `${IMAGE_NAME}`, `${IMAGE_TAG}` and the `${FIRST_*_PORT}` / `${SECOND_*_PORT}` variables in [evcc.container](evcc.container). PLCnext fills these in on the device. Some come from the firmware (`APP_ID`, `APP_PERSISTENT_DIR`); the rest come from `ocicontainer.environmentVariables` in [app_info.json](app_info.json).

[app_info.json](app_info.json) is the PLCnext app manifest. It declares:

- the version `evcc {IMAGE_TAG} ({VERSION})`, in the PLCnext format `<Name> (<major>.<minor>.<patch>.<build> <state>)`. On updates PLCnext compares only the part in parentheses, so it must be the app version; every evcc bump therefore needs a new app release. `build.sh` only accepts 1 to 4 numeric parts for the app version;
- the persistent `data` directory, mounted into the container as `/app/data`;
- the image and quadlet file paths inside the app (`/image.tar.gz` and `/evcc.container`);
- the port mappings: container `7070` (evcc web UI) to host `57070`, and container `8887` (OCPP) to host `58887`.

The `PortLink` URL `"57070/"` must stay in sync with `FIRST_OUTGRESS_PORT`. evcc shows OCPP URLs with its internal port `8887`, but chargers must use the host port `58887` (`SECOND_OUTGRESS_PORT`); the README says so. Keep `identifier` stable, because it identifies the app on the device for updates.

[evcc.container](evcc.container) is a Podman Quadlet unit; see the [podman-systemd.unit docs](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html). Notes:

- evcc keeps all settings (configured via the web UI), sessions and vehicle tokens in its SQLite database. Its default path `~/.evcc/evcc.db` depends on `$HOME`, so `Exec=evcc --database /app/data/evcc.db` points it to the persistent volume. The image's entrypoint passes `evcc ...` through. Without an `evcc.yaml`, evcc runs in "database-only mode"; it only looks for one in `$HOME`, `/app` and `/etc`.
- It runs as `User=1001:1002` (`app_user`:`plcnext`, as the [CatanIO app docs](https://github.com/PLCnext/CatanIO/tree/main/apps) require for store apps), with `GroupAdd=keep-groups` and `UserNS=keep-id`.
- The two RS-485 ports are passed in with `AddDevice=` (`/dev/ttymxc3`, `/dev/ttymxc2`), USB serial devices by bind-mounting `/dev/usb-devices` with `rslave` propagation. Users point evcc at a path under `/dev/usb-devices/symlinks/...`.
- It uses bridge networking, not `Network=host`. So multicast and discovery (SMA Energy Meter / Sunny Home Manager on UDP 9522, EEBus with mDNS) do not reach evcc.

## Constraints

- USB passthrough requires firmware 2026.0.3 or later (`minfirmware_version` in the manifest).
- CH340/CH341-based USB dongles are not supported.
