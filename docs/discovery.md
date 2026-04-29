# Discovery: finding an Electron app's bundled MCP binary

Most Electron apps that ship an MCP server keep the binary in the unpacked-asar directory of the bundle. Here's how to find and probe it.

## Locate the binary

**macOS:**
```
/Applications/<App>.app/Contents/Resources/app.asar.unpacked/out/mcp-server-darwin-*
```

**Linux:**
```
/opt/<app>/resources/app.asar.unpacked/out/mcp-server-linux-*
```

**Windows:**
```
C:\Program Files\<App>\resources\app.asar.unpacked\out\mcp-server-win32.exe
```

The binary is typically 5–10 MB and lives alongside `assets/` in the unpacked asar directory.

## Quick checks

```bash
# Is the app already running an MCP process?
ps aux | grep -i <app-name>

# Listening ports?
lsof -i -P | grep -i <app-name>

# What flags does the binary support?
/path/to/mcp-server-<platform>-<arch> --help
```

## Identify the transport mode

| Behavior on its port | Likely transport |
|---|---|
| Connection refused | Not running, must spawn |
| HTTP 404 / empty body | Raw HTTP server (rare) |
| HTTP 426 Upgrade Required | WebSocket |
| HTTP 401 | HTTP MCP with auth |
| stdin/stdout (no socket) | Stdio (default) |

Common flags worth knowing about:
- `--app <name>` — which app instance to attach to (often required)
- `--http` / `--http-port <N>` — switch from stdio to HTTP
- `--conversation_id <id>` — group calls into a session
- `--enable_spawn_agents` — opt-in to agent-spawning tools when supported

## Probe what tools it exposes

Stdio:
```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' \
  | "$BINARY" --app desktop
```

HTTP (after launching with `--http --http-port 8081`):
```bash
curl -sX POST http://localhost:8081/ \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}'
```

After `initialize`, send the `initialized` notification, then `tools/list`:
```json
{"jsonrpc":"2.0","method":"initialized","params":{}}
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
```

## Common pitfalls

- **Binary architecture mismatch** — `mcp-server-darwin-arm64` won't run on Intel macOS without Rosetta. Use the matching binary or run under `arch -arm64`.
- **Port already in use** — kill stale subprocesses before spawning your own.
- **Wrong transport** — `426 Upgrade Required` means you sent HTTP to a WS endpoint; use a WebSocket client or pass `--http`.
- **Auth 401** — you connected to the app's internal IPC server expecting a session token. Spawn your own instance instead of attaching to the running one.
- **Stdio deadlock** — both stdin and stdout must be piped; blocking on one direction locks the other.
