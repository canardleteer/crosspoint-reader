# Docker firmware build

Rebuild CrossPoint firmware with only Docker. You do not need a host PlatformIO, Python, or pioarduino install.

The image is toolchain-only (pioarduino and the Espressif builder packages, pinned to match CI). The repository is bind-mounted at run time, so editing source does not rebuild the image or re-download the ESP32 toolchain.

This path produces `.bin` files. It does not flash a device. USB passthrough is OS-specific and is not “just Docker.” Flash from the host with `pio run --target upload` or the PlatformIO IDE after you have a binary.

Nix users who already have a host toolchain can keep using `nix develop -f nix` instead. See the [README Nix section](../README.md#nixnixos).

## Prerequisites

- Docker Engine with Compose v2 (`docker compose version`)
- A recursive clone so `freeink-sdk` is present

```sh
git clone --recursive https://github.com/crosspoint-reader/crosspoint-reader
cd crosspoint-reader
```

If you already cloned without submodules:

```sh
git submodule update --init --recursive
```

`./bin/docker-build` exits immediately if `freeink-sdk/libs` is missing.

## Build

Any env in `platformio.ini` works. Pass the env name the same way you would to `pio run -e`. With no arguments, that is `default` (ESP32-C3 X3/X4) — not a Docker limitation.

```sh
./bin/docker-build              # default: X3 / X4 (ESP32-C3)
./bin/docker-build x4pro        # X4 Pro (ESP32-S3)
./bin/docker-build papermono    # M5Stack Paper Mono (ESP32-S3)
./bin/docker-build sticky       # Seeed Sticky (ESP32-S3)
./bin/docker-build default x4pro papermono sticky
```

Compose is equivalent (`-f docker/compose.yaml` from the repo root). The service default command is `pio run -e default`:

```sh
docker compose -f docker/compose.yaml run --rm firmware
docker compose -f docker/compose.yaml run --rm firmware pio run -e x4pro
```

`./bin/docker-build` exports your UID/GID so bind-mounted `.pio/` is not root-owned, then runs `docker compose -f docker/compose.yaml run --rm --build firmware pio run -e <env> …`. `--build` rebuilds the image only when `docker/Dockerfile` changed, not when firmware sources change.

If you call Compose directly on Linux, pass the host user explicitly (`UID` is read-only in bash, so use `env`):

```sh
env UID="$(id -u)" GID="$(id -g)" docker compose -f docker/compose.yaml run --rm firmware
env UID="$(id -u)" GID="$(id -g)" docker compose -f docker/compose.yaml run --rm firmware pio run -e x4pro
```

## Output

| Env | Binary |
| --- | --- |
| `default` | `.pio/build/default/firmware.bin` |
| `sticky` | `.pio/build/sticky/firmware.bin` |
| `x4pro` | `.pio/build/x4pro/firmware.bin` |
| `papermono` | `.pio/build/papermono/firmware.bin` |

Release-style envs (`gh_release`, `sticky-gh_release`, …) work the same way: `./bin/docker-build gh_release`.

## Caches

Source edits reuse three trees. None of them live in an image layer.

| Location | What it holds |
| --- | --- |
| Compose volume `pio-cache` (`PLATFORMIO_CORE_DIR=/cache/platformio`) | Espressif platform, packages, and the compiler toolchain. First run downloads this. |
| Host `.pio/` (gitignored) | libdeps and per-env object files. Incremental `pio run` after a code change. |
| Host `.cache/` (gitignored; `build_cache_dir` in `platformio.ini`) | PlatformIO build cache. |

Wipe them after a `platformio.ini` platform pin change, or if a toolchain extract looks corrupt:

```sh
docker compose -f docker/compose.yaml down -v
rm -rf .pio .cache
```

Then run `./bin/docker-build` again.

## Image only

To build or refresh the toolchain image without compiling firmware:

```sh
docker compose -f docker/compose.yaml build
```

The image is tagged `crosspoint-builder`. `docker/.dockerignore` keeps the build context (the `docker/` directory) empty; the firmware tree is not copied into the image.

## Layout

| File | Role |
| --- | --- |
| [`docker/Dockerfile`](../docker/Dockerfile) | Python 3.13 slim, git, pioarduino 6.1.19, CI builder pip pins |
| [`docker/compose.yaml`](../docker/compose.yaml) | Bind-mount repo root → `/src`, named volume `pio-cache` |
| [`docker/.dockerignore`](../docker/.dockerignore) | Empty image build context |
| [`bin/docker-build`](../bin/docker-build) | Submodule check, UID/GID, Compose wrapper |

Pins match [`.github/workflows/ci.yml`](../.github/workflows/ci.yml). Root `requirements.txt` (Pillow, CairoSVG, matplotlib) is not installed; those are for font scripts and the debug monitor, not `pio run`.
