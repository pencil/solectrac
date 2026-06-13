# Reproducible build environment for the Solectrac ESP32-S3 CAN-monitor firmware.
#
# The build context is the repo ROOT, because the firmware embeds the dashboard
# (src/dashboard.html is a symlink into the Android tree). From the repo root:
#
#   docker build -f embedded/esp32-s3/Dockerfile -t solectrac-fw .
#
# Then extract the build artifacts onto the host so you can flash from macOS —
# Docker Desktop on macOS cannot reach the board's USB serial port, so flashing
# happens on the host:
#
#   docker run --rm -v "$PWD/out:/out" solectrac-fw
#
# Pinned to Python 3.13: PlatformIO 6.1.x segfaults during package
# post-install on Python 3.14, which is why the native Homebrew install fails.
FROM python:3.13-slim

# git + ca-certificates are needed by PlatformIO to fetch the platform,
# toolchain, and framework packages.
RUN apt-get update \
    && apt-get install -y --no-install-recommends git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Pin PlatformIO Core so builds are reproducible.
RUN pip install --no-cache-dir "platformio==6.1.19"

WORKDIR /project

# Copy the firmware project (board defs, platformio.ini, src/).
COPY embedded/esp32-s3/ ./

# src/dashboard.html is a symlink into the Android tree; Docker COPY does not
# dereference symlinks, so replace it with the real file (board_build.embed_txtfiles
# bakes it into the binary at build time).
COPY android/app/src/main/assets/dashboard.html ./src/dashboard.html

# WIFI_SSID / WIFI_PASS (the home network the board also joins for bench use)
# are embedded at compile time; the build needs them set, so default to
# placeholders. Override via --build-arg.
ARG WIFI_SSID=placeholder
ARG WIFI_PASS=placeholder
ENV WIFI_SSID=${WIFI_SSID}
ENV WIFI_PASS=${WIFI_PASS}

# Build for the LilyGo T-2CAN. This downloads the espressif32 platform and
# Xtensa toolchain on first run and caches them in the image layer.
RUN pio run -e lilygo_t2can

# Default action: copy the four images needed to flash the board to a mounted
# /out volume on the host. boot_app0.bin lives in the framework package (it
# initializes the OTA-data partition); the others are build outputs. These are
# the same images, at the same offsets, that `pio run -t upload` writes.
CMD ["sh", "-c", "set -e; mkdir -p /out; cp -v .pio/build/lilygo_t2can/firmware.bin .pio/build/lilygo_t2can/bootloader.bin .pio/build/lilygo_t2can/partitions.bin /out/; cp -v \"$(find /root/.platformio -name boot_app0.bin | head -1)\" /out/; echo 'Artifacts copied to /out — flash with esptool (see README)'"]
