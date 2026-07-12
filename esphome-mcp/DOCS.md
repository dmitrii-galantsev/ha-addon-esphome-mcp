# ESPHome MCP Server

This add-on runs an MCP (Model Context Protocol) server that exposes
ESPHome operations as tools for Claude Code. It delegates builds, flashes,
validation and logs to the ESPHome Device Builder dashboard (the official
ESPHome add-on) so they always run against **current** ESPHome, and keeps
native filesystem access to `/config/esphome/` for config/font transfer —
no SSH tunneling required.

## Architecture

```text
Claude Code (desktop)
     |  HTTP (MCP, port 8098, Bearer token)
     v
HA Add-on (MCP Server)
     |  HTTP/WS  -->  ESPHome Device Builder dashboard (official add-on, port 6052)
     |                    - GET /devices, GET /json-config
     |                    - WS /compile, WS /upload, WS /ws (logs)
     |  local file I/O
     v
/config/esphome/  (shared mount: push/pull YAML + fonts)
```

Because compilation happens in the ESPHome add-on's container, this add-on
ships **no** ESPHome toolchain and is not tied to any esphome version.

## Configuration

### auth_token

An authentication token to secure the MCP endpoint. If left empty, a
token is auto-generated on first start and printed in the add-on logs.

```yaml
auth_token: "my-secret-token"
```

### dashboard_url

URL of the ESPHome Device Builder dashboard the add-on delegates builds to.
Defaults to the official ESPHome add-on's internal Supervisor hostname
`http://core-esphome:6052`. If your ESPHome add-on has a different slug, set
it here (find the slug in the add-on's page URL, or via `ha addons`).

### dashboard_token

Only needed if the dashboard is protected with a password. Leave empty for
the default (open) HA add-on behind Ingress.

## Setup

1. Add this repository as a custom add-on repository in Home Assistant:
   **Settings > Add-ons > Add-on Store > ... > Repositories**
   Enter: `https://github.com/dmitrii-galantsev/ha-addon-esphome-mcp`

2. Install the **ESPHome MCP Server** add-on and start it.

3. Check the add-on logs for the auth token (if you didn't set one).

4. Set the `ESPHOME_MCP_TOKEN` environment variable on your development
   machine to the auth token value.

5. Configure `.mcp.json` in your ESPHome project:

   ```json
   {
     "mcpServers": {
       "esphome": {
         "type": "http",
         "url": "http://<your-ha-host>:8098/mcp",
         "headers": {
           "Authorization": "Bearer ${ESPHOME_MCP_TOKEN}"
         }
       }
     }
   }
   ```

6. Restart Claude Code and verify the connection with `/mcp`.

## Available Tools

| Tool | Description |
| ---- | ----------- |
| `esphome_list_devices` | List device configs with names |
| `esphome_validate` | Validate a device YAML config |
| `esphome_compile` | Compile firmware (background; returns inline or a poll handle) |
| `esphome_flash` | OTA flash a device (background; returns inline or a poll handle) |
| `esphome_build_status` | Poll the latest background compile/flash for a device |
| `esphome_logs` | Get recent device logs (snapshot) |
| `esphome_push_files` | Write YAML files to the config directory |
| `esphome_pull_files` | Read YAML files from the config directory |
| `esphome_push_fonts` | Write font files (base64-encoded) |
| `esphome_pull_fonts` | Read font files (base64-encoded) |

## Security

- All requests require a valid Bearer token in the Authorization header.
- `secrets.yaml` is explicitly rejected in push/pull operations.
- The add-on exposes port 8098 — ensure your network is trusted or use
  a reverse proxy with TLS.

## Network

The add-on listens on port **8098** (TCP). Make sure this port is
accessible from your development machine.

## Long-running builds

Compiles (and the compile step of a flash) can take several minutes,
especially the first build of a device. These run in the background: if a
build finishes within ~45s the full output is returned immediately;
otherwise the tool returns a handle and you poll `esphome_build_status`
with the device name until it reports `done` or `failed`.
