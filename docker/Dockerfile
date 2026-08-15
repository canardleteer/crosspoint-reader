FROM python:3.13-slim-bookworm

# git: scripts/git_branch.py reads branch + SHA for default/sticky version strings.
# unzip/xz-utils: PlatformIO extracts the Espressif toolchain into PLATFORMIO_CORE_DIR.
RUN apt-get update && apt-get install -y --no-install-recommends \
        git \
        ca-certificates \
        curl \
        unzip \
        xz-utils \
    && rm -rf /var/lib/apt/lists/*

# Pins match .github/workflows/ci.yml (pioarduino 6.1.19 + platform-espressif32 builder deps).
RUN pip install --no-cache-dir uv \
    && uv pip install --system --no-cache \
        -U https://github.com/pioarduino/platformio-core/archive/refs/tags/v6.1.19.zip \
    && uv pip install --system --no-cache \
        "littlefs-python>=0.16.0" \
        "fatfs-ng>=0.1.14" \
        "pyyaml>=6.0.2" \
        "rich-click>=1.8.6" \
        "zopfli>=0.2.2" \
        "intelhex>=2.3.0" \
        "rich>=14.0.0" \
        "urllib3<2" \
        "cryptography>=45.0.3" \
        "certifi>=2025.8.3" \
        "ecdsa>=0.19.1" \
        "bitstring>=4.3.1" \
        "reedsolo>=1.5.3,<1.8" \
        "esp-idf-size>=2.0.0" \
        "esp-coredump>=1.14.0"

# First mount of the named pio-cache volume copies this mode so a non-root
# compose user can write the toolchain without a chown entrypoint.
RUN mkdir -p /cache/platformio && chmod 1777 /cache/platformio

WORKDIR /src

ENV HOME=/tmp \
    PLATFORMIO_CORE_DIR=/cache/platformio \
    PYTHONDONTWRITEBYTECODE=1

CMD ["pio"]
