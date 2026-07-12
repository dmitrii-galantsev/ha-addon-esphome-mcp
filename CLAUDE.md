# CLAUDE.md

This file provides guidance to Claude Code when working with code in
this repository.

## Project Overview

Home Assistant custom add-on that runs an MCP (Model Context Protocol)
server for ESPHome operations. Claude Code connects to it over HTTP
instead of SSH. Builds/flashes/validation/logs are **delegated to the
ESPHome Device Builder dashboard** (the official ESPHome add-on) over its
HTTP/WS API, so they always use current ESPHome; config/font transfer uses
direct access to the shared `/config/esphome/` filesystem on the HA host.

## Repository Structure

- `repository.yaml` — HA add-on repository metadata
- `esphome-mcp/` — The add-on
  - `config.yaml` — HA add-on manifest (name, version, ports, options)
  - `build.yaml` — Multi-arch Docker build config
  - `Dockerfile` — slim `python:3.12-slim` base (no ESPHome toolchain)
  - `run.sh` — Add-on entry point (reads config, starts server)
  - `requirements.txt` — Python dependencies (mcp, uvicorn, aiohttp)
  - `server/` — Python package
    - `main.py` — FastMCP app, tool registration, uvicorn entry point
    - `tools.py` — Tool implementations (delegates builds to the dashboard;
      file/font tools use the local `/config` mount)
    - `dashboard.py` — HTTP/WS client for the Device Builder dashboard
    - `auth.py` — Bearer token middleware
  - `DOCS.md` — Add-on documentation page shown in HA UI

## Key Conventions

- **Auth**: Bearer token in `Authorization` header; auto-generated if not
  configured, persisted to `/data/auth_token`
- **Transport**: Streamable HTTP on port 8098 at `/mcp`
- **Secrets**: `secrets.yaml` is explicitly rejected in push/pull tools
- **ESPHome**: not bundled. Builds are delegated to the Device Builder
  dashboard (`dashboard_url`, default `http://core-esphome:6052`) via its
  HTTP/WS API — no local esphome binary, no version pin
- **Builds**: compile/flash consume the dashboard's WS spawn stream in a
  background thread; poll with `esphome_build_status` when a build outlives
  the sync window
- **Config mapping**: HA Supervisor maps `/config/` into the container
  (shared with the ESPHome add-on) for the file/font tools

## Building / Testing

The add-on is built by HA Supervisor when installed. For local testing:

```bash
cd esphome-mcp
docker build --build-arg BUILD_FROM=ghcr.io/home-assistant/amd64-base:latest -t esphome-mcp .
docker run -p 8098:8098 -v /path/to/config:/config \
    -e ESPHOME_MCP_AUTH_TOKEN=test \
    -e DASHBOARD_URL=http://host.docker.internal:6052 esphome-mcp
```

## Deployment

Add `https://github.com/dmitrii-galantsev/ha-addon-esphome-mcp` as a custom
add-on repository in Home Assistant, then install and start the add-on.
