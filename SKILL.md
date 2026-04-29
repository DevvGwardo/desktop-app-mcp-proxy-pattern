---
name: desktop-app-mcp-proxy-pattern
description: Hybrid MCP server pattern for desktop apps that bundle their own MCP binary — spawn as subprocess, bridge stdio messages, add custom tools
---

## When to use

You're building an MCP integration for a **desktop application** (typically Electron) that already **bundles an MCP server binary** inside its app package. Common scenario: the app exposes MCP tools but they're only accessible when the app is running, not as a standalone server you can connect to.

Examples:
- Pencil by Evolus (`mcp-server-darwin-arm64` inside the app bundle)
- Other Electron apps with `app.asar.unpacked/out/mcp-server-*` binaries

## The pattern

**Do NOT** try to reverse-engineer the binary's WebSocket protocol or extract auth tokens from LevelDB/SQLite. The binary already knows how to authenticate with the main app — just let it do its thing.

**Instead**, build a **hybrid proxy server**:

1. Your Python MCP server **spawns the app's binary as a subprocess** (stdio mode)
2. Your server connects to its **client via stdio MCP**  
3. For **native tools**: forward the MCP request to the subprocess, pipe response back
4. For **custom tools**: handle locally, adding developer superpowers

```
[Hermes/Claude] ←stdio MCP→ [Your Python Server] ←stdio MCP→ [App's Native Binary] ←WebSocket→ [Running App]
```

## Why this works

- The app's binary already handles authentication, WebSocket connection, state sync
- Stdio is simpler than WebSocket client — no token extraction, no reconnection logic
- You get **all native tools for free** by proxying
- You can **layer custom tools** on top without modifying the original binary

## Step-by-step

### 1. Locate the binary
Typically at:
```
/Applications/<App>.app/Contents/Resources/app.asar.unpacked/out/mcp-server-<platform>-<arch>
```

Or via env var override (useful for non-macOS or custom builds):
```python
binary = os.environ.get("APP_MCP_BINARY", DEFAULT_PATH)
```

### 2. Subprocess manager

```python
class AppProcess:
    def __init__(self, binary: str):
        self.binary = binary
        self.proc = None
        self._next_id = 1
        self._pending = {}  # rid → Future

    async def start(self):
        self.proc = await asyncio.create_subprocess_exec(
            self.binary, "--app", "desktop",  # flag varies by app
            stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
        )
        asyncio.create_task(self._stderr_logger())
        asyncio.create_task(self._reader())
        await asyncio.sleep(1.5)  # let app connect via WebSocket

    async def _reader(self):
        buf = b""
        while True:
            chunk = await self.proc.stdout.read(4096)
            if not chunk: break
            buf += chunk
            while b"\n" in buf:
                line, buf = buf.split(b"\n", 1)
                try:
                    msg = json.loads(line.decode())
                except: continue
                rid = msg.get("id")
                if rid is not None and rid in self._pending:
                    fut = self._pending.pop(rid)
                    if not fut.done(): fut.set_result(msg)

    def _next_req_id(self) -> int:
        nid = self._next_id
        self._next_id += 1
        return nid

    async def request(self, method: str, params: dict = None) -> dict:
        req_id = self._next_req_id()
        msg = {"jsonrpc": "2.0", "id": req_id, "method": method, "params": params or {}}
        fut = asyncio.get_event_loop().create_future()
        self._pending[req_id] = fut
        self.proc.stdin.write((json.dumps(msg) + "\n").encode())
        await self.proc.stdin.drain()
        return await asyncio.wait_for(fut, timeout=20)
```

### 3. MCP server skeleton

```python
from mcp.server import Server, NotificationOptions
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("app-mcp")
app: Optional[AppProcess] = None

@server.list_tools()
async def list_tools() -> List[Tool]:
    resp = await app.request("tools/list", {})
    native = [Tool(**t) for t in resp.get("result", {}).get("tools", [])]
    return native + CUSTOM_TOOLS

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> List[TextContent]:
    if name in CUSTOM_TOOLS:
        return await CUSTOM_TOOLS[name](arguments)
    resp = await app.request("tools/call", {"name": name, "arguments": arguments})
    content = resp.get("result", {}).get("content", [])
    return [TextContent(type="text", text="\n".join(c["text"] for c in content if c["type"] == "text"))]
```

### 4. Custom tools (examples)

**Parse app-specific file format** — many design apps use JSON or ZIP:
```python
def parse_app_file(path: str) -> dict:
    if zipfile.is_zipfile(path):
        with zipfile.ZipFile(path) as zf:
            return {"doc": json.loads(zf.read("doc.json")), "assets": zf.namelist()}
    return {"doc": json.load(open(path)), "assets": []}
```

**Extract design tokens** (colors, fonts, spacing) from hierarchical scene graph:
```python
def extract_tokens(doc: dict) -> dict:
    tokens = {"colors": {}, "fonts": {}, "spacing": []}
    def walk(node):
        for attr in ["fill", "stroke", "color"]:
            if attr in node: 
                # normalize value, count usage
                ...
        for child in node.get("children", []):
            walk(child)
    for page in doc.get("children", []):
        walk(page)
    return tokens
```

**Tree traversal with depth limiting** (common gotcha: don't use `max_depth <= 0` when recursing — pass depth as param):
```python
def flatten_tree(doc, max_depth=5):
    flat = []
    def walk(node, path="", depth=0):
        flat.append({...})
        if depth >= max_depth or not node.get("children"):
            return
        for child in node["children"]:
            walk(child, f"{path}/{node['id']}", depth + 1)
    for child in doc.get("children", []):
        walk(child)
    return flat
```

**Diff two documents** (structural comparison by node IDs and types):
```python
def diff_docs(doc_a, doc_b):
    a_nodes = {n["id"]: n for n in flatten_tree(doc_a)}
    b_nodes = {n["id"]: n for n in flatten_tree(doc_b)}
    return {
        "only_in_a": [v for k,v in a_nodes.items() if k not in b_nodes],
        "only_in_b": [v for k,v in b_nodes.items() if k not in a_nodes],
        "common_ids": list(set(a_nodes) & set(b_nodes)),
    }
```

## Pitfalls observed

- **Binary shutdown noise**: The subprocess often logs "Failed to start stdio server: context canceled" on exit — harmless, the connection already worked. Filter it from stderr or ignore.
- **Native tool parameter quirks**: Some native tools require specific params (e.g. `get_editor_state` requires `include_schema`). Call `tools/list` first to inspect schemas, or test interactively.
- **Depth recursion bugs**: Common mistake: checking `max_depth <= 0` without tracking current depth. Always pass `depth` param and compare `depth >= max_depth`.
- **Request ID collisions**: Ensure `_next_req_id` uses an instance attribute, not a method name that shadows the field (`self._next_id` vs `self._next_id_val` to avoid naming conflict with the method itself).
- **JSON-RPC line buffering**: Binary may send logs on stderr mixed with JSON on stdout. Only parse stdout lines that successfully decode as JSON.
- **Headers in schema**: Don't blindly trust `native_raw` tool dicts — wrap with `Tool(name=..., description=..., inputSchema=..., annotations=...)` to satisfy MCP SDK's stricter validation.

## Related skills

- `software-development/mcp-server-trim`: After proxying all native tools, your server's list just grows. Use `trim` if you only need a subset.
- `software-development/requesting-code-review`: Run a reviewer pass on the proxy logic before shipping — concurrency bugs in request/response matching are subtle.
