# Changelog

All notable changes to this project will be documented in this file.

## Attributors

- **Bert Berrevoets** — Project author
- **Claude Code** — AI-assisted development

## [1.4.0] - 2026-07-12

### Changed

- Run on `host_network` and default `dashboard_url` to `http://127.0.0.1:<port>`.
  The HA ESPHome add-on serves its dashboard ingress-only on
  `127.0.0.1:<ingress_port>` (no fixed 6052), and its peer guard trusts only
  loopback/Supervisor — so a bridge-network add-on gets 403. host_network lets
  us reach it over loopback as a trusted peer, mirroring HA core's ESPHome
  integration. Set `dashboard_url` to `http://127.0.0.1:<ingress_port>` (find
  it via `ha addons info <esphome-slug> | grep ingress_port`).

## [1.3.2] - 2026-07-12

### Fixed

- Run delegated dashboard calls on a worker thread. FastMCP invokes the sync
  tools on its own event-loop thread, so the direct `asyncio.run()` in
  `list_devices`/`validate`/`logs` raised "cannot be called from a running
  event loop". Offload via a one-shot thread pool. Verified end-to-end
  against a live dashboard (list, validate, and WS compile streaming).

## [1.3.1] - 2026-07-12

### Fixed

- Build on the trusted HA Alpine base (`ghcr.io/home-assistant/{arch}-base`)
  and install `python3`/`py3-pip`/`bash` via apk. Supervisor rejects
  untrusted base images like `python:3.12-slim` (Docker Hub library) and
  silently falls back to the HA base, which lacks pip — so 1.3.0 failed to
  build with `pip3: not found`.

## [1.3.0] - 2026-07-12

### Changed

- **Builds are now delegated to the ESPHome Device Builder dashboard** (the
  official ESPHome add-on) over its HTTP/WS API instead of running a bundled
  `esphome` binary. This removes the pinned esphome version that lived in the
  Docker base image — configs needing newer ESPHome features (e.g. the
  `WAVESHARE-ESP32-C6-LCD-1.47` display model) no longer fail with an
  "Unknown value" error because the add-on always builds with current ESPHome.
- `compile`/`flash` drive the dashboard's `/compile` and `/upload` WebSocket
  spawn protocol; `validate` uses `GET /json-config`; `list_devices` uses
  `GET /devices`; `logs` streams the `/ws` `devices/logs` command. File and
  font tools still read/write the shared `/config/esphome` mount directly.
- Dropped the heavy `ghcr.io/esphome/esphome` base image for a slim
  `python:3.12-slim` — no ESP toolchain or PlatformIO is shipped anymore.

### Added

- Options `dashboard_url` (default `http://core-esphome:6052`) and
  `dashboard_token` (only needed if the dashboard has a password set).

## [1.2.0] - 2026-05-20 (glibc fork)

### Changed

- Rebased the image on the official `ghcr.io/esphome/esphome` (Debian/glibc)
  image. The previous Alpine/musl base could not run ESPHome's glibc ESP
  cross-toolchains (`xtensa-lx106-elf-g++`), so every compile failed with
  `not found` (exit 127). Compiles/flashes now work.
- Replaced bashio/`with-contenv` startup with a plain `/data/options.json`
  read; cleared the base image's inherited `ENTRYPOINT` and `HEALTHCHECK`
  (the dashboard healthcheck caused a ~60s restart loop).
- Default port moved to **8098** so the fork can run beside the original.

### Added

- Background builds: `esphome_compile` / `esphome_flash` run in a thread and
  return a pollable handle for long builds, with new `esphome_build_status`
  to check progress — avoids MCP request timeouts on multi-minute compiles.
- `esphome_flash` forces OTA (`--device <name>.local`) so it no longer hangs
  on the interactive serial/OTA chooser when USB adapters are present.

## [1.0.0] - 2026-03-17

### Added

Author: *Bert Berrevoets, Claude Code*

- Initial release as Home Assistant add-on
- FastMCP server with streamable HTTP transport on port 8099
- Bearer token authentication (auto-generated or user-configured)
- Nine MCP tools: list_devices, validate, compile, flash, logs,
  push_files, pull_files, push_fonts, pull_fonts
- Direct filesystem access to `/config/esphome/` — no SSH required
- Alpine-based Docker image with ESPHome and PlatformIO pre-installed
- Multi-architecture support (aarch64, amd64)
- Add-on documentation (DOCS.md)
- secrets.yaml protection in push/pull operations
