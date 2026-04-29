# desktop-app-mcp-proxy-pattern

A documented design pattern + reference template for **building MCP integrations around desktop apps that already bundle their own MCP server binary** (Electron apps, typically).

When a desktop app ships an MCP binary inside its app bundle, the temptation is to reverse-engineer its WebSocket auth or extract tokens from local storage. **Don't.** The bundled binary already knows how to authenticate with the main app — just spawn it as a subprocess and proxy stdio. Then layer your own custom tools on top.

```
[MCP client] ─stdio MCP─► [Your proxy server] ─stdio MCP─► [App's bundled binary] ─WS/IPC─► [Running app]
                              │
                              └── Custom local tools (file format utilities,
                                  validation, exporters, diff, etc.)
```

## What's in this repo

- [`SKILL.md`](SKILL.md) — the full pattern, code skeleton, and pitfalls reference
- [`docs/discovery.md`](docs/discovery.md) — how to find the MCP binary in any Electron app and probe its tools
- [`docs/transports.md`](docs/transports.md) — stdio vs HTTP vs WebSocket, when each is used

## When to use this pattern

You want MCP access to a desktop app and discover it already has `app.asar.unpacked/out/mcp-server-*` somewhere in its bundle. Apps known to do this include Pencil (pencil.dev), Claude Desktop, and others.

## When NOT to use it

- The app exposes an HTTP MCP endpoint with a public token → just connect directly
- The app is fully open source and accepts contributions → submit your custom tools upstream
- You want to ship your own MCP without depending on a closed-source binary → write a standalone server

## Reference implementations

- [`pencil-mcp`](https://github.com/<your-handle>/pencil-mcp) — proxy + 7 custom tools for Pencil's `.pen` files

## License

MIT — see [`LICENSE`](LICENSE). Documentation is provided as-is; using it to integrate with a specific app is your responsibility (respect their ToS and trademarks).
