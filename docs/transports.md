# Transports: stdio vs HTTP vs WebSocket

Bundled MCP binaries usually support more than one transport. Pick the right one for your use case.

## Stdio (default for most app-bundled binaries)

JSON-RPC over stdin/stdout. The simplest case — no ports, no auth, no reconnection logic. Your wrapper spawns the binary, writes requests to its stdin, reads responses from its stdout.

**Use when:** You're embedding the proxy in another process (Hermes, Claude Desktop, IDE extensions). This is the recommended default.

**Gotchas:**
- The binary may write logs to stderr mixed with JSON on stdout — only parse stdout lines that decode as JSON.
- Buffered writes can deadlock — always `await drain()` after writing on Python async streams.
- Some binaries print a noisy "context canceled" message on shutdown. Filter or ignore.

## HTTP

Plain `POST /` with a JSON-RPC body. Used when you want multiple clients connecting to one running binary, or when you can't pipe stdio (some platforms).

**Use when:** You want to share one running binary across processes, debug interactively with `curl`, or run the binary on a different host than the client.

**Gotchas:**
- The binary needs explicit flags (often `--http --http-port N`).
- Default ports vary; pick a known one to avoid clashing with the app's internal port.
- If the binary expects an internal IPC token (issued by the main app), HTTP requests from outside will get `401`. Spawn your own instance with no auth, or extract the token if the app exposes one.

## WebSocket

Used by some apps for streaming or bidirectional updates. RFC 6455 framing with `Sec-WebSocket-Key` handshake.

**Use when:** The app explicitly requires it (you'll see `HTTP 426 Upgrade Required` on the port). Otherwise prefer stdio or HTTP.

**Gotchas:**
- Frame masking is required for client → server messages; servers don't mask their replies.
- Reconnection logic is on you. Stdio reconnects "for free" by re-spawning.

## Decision flow

```
Need MCP tools from a running desktop app?
│
├─ Are you embedding in another process? ─── yes ──► stdio
│
├─ Need multiple clients on one binary? ─── yes ──► HTTP
│
├─ App listens with HTTP 426? ─── yes ──► WebSocket
│
└─ Default ──► stdio
```
