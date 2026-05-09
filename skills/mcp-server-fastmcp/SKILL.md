---
name: mcp-server-fastmcp
description: >-
  FastMCP server setup — transport selection (stdio vs SSE), correct placement of host/port
  on the FastMCP constructor (not on run methods), SSE as a persistent macOS LaunchAgent
  service, file layout convention (server.py factory + cli entry points), and setup_mcp.py
  pattern for writing client configs. Use when building or debugging a FastMCP MCP server,
  choosing between stdio and SSE transports, running the server as a background service,
  or configuring Cursor and Claude Desktop to connect to it.
---

# MCP Server — FastMCP

## FastMCP API — constructor vs run methods

`host` and `port` are **constructor parameters**, not arguments to any `run_*` method.
Passing them to `run_sse_async()` raises `TypeError: unexpected keyword argument 'host'`.

```python
# CORRECT — host/port on the constructor
from mcp.server.fastmcp import FastMCP
server = FastMCP("my-server", host="127.0.0.1", port=8765)
asyncio.run(server.run_sse_async())        # no args needed

# WRONG — run_sse_async() does not accept host/port
asyncio.run(server.run_sse_async(host="127.0.0.1", port=8765))  # TypeError
```

### Run method signatures (as of mcp 1.x)

```python
run(self, transport: Literal['stdio', 'sse', 'streamable-http'] = 'stdio',
    mount_path: str | None = None) -> None

run_stdio_async(self) -> None
run_sse_async(self, mount_path: str | None = None) -> None
run_streamable_http_async(self) -> None
```

`run()` is the synchronous wrapper. For entry points, call `server.run()` or
`asyncio.run(server.run_sse_async())`.

### Key constructor parameters

```python
FastMCP(
    name,
    instructions = None,
    host         = "127.0.0.1",   # SSE bind address
    port         = 8000,           # SSE port (default 8000, not 8765)
    sse_path     = "/sse",         # SSE endpoint path
    message_path = "/messages/",
    log_level    = "INFO",
    debug        = False,
)
```

Default port is `8000` — always override to avoid clashing with other dev servers.
Convention in this repo: `8765`.

---

## Transport comparison

| | stdio | SSE |
|---|---|---|
| Process lifetime | spawned per session, killed on close | persistent, survives client restarts |
| Multiple clients | each client gets its own process | Cursor + Claude share one process |
| Client config | `command` + `args` in mcp.json | `url` in mcp.json |
| macOS service | not applicable | LaunchAgent plist |
| Use when | simple local dev | frequent Cursor restarts, multiple clients |

---

## File layout convention

```
src/<package>/server.py          # FastMCP object + make_server() factory
cli/run_mcp_server.py            # stdio entry point (thin wrapper)
cli/run_mcp_server_sse.py        # SSE entry point (thin wrapper)
cli/setup_mcp.py                 # writes client configs, installs LaunchAgent
```

`server.py` exports a `make_server(host, port)` factory so each transport entry point
can construct its own instance with the correct host/port. A shared `server` instance
(default host/port) is kept for the stdio case and the `forgedb-server` console script.

```python
# src/<package>/server.py
_DEFAULT_HOST = "127.0.0.1"
_DEFAULT_PORT = 8765

def make_server(host: str = _DEFAULT_HOST, port: int = _DEFAULT_PORT) -> FastMCP:
    s = FastMCP(NAME, instructions=DESCRIPTION, host=host, port=port)
    register_tools(s)
    return s

server = make_server()   # shared stdio instance

def main() -> None:      # console script entry point
    server.run()
```

```python
# cli/run_mcp_server_sse.py
from <package>.server import make_server, _DEFAULT_HOST, _DEFAULT_PORT

@app.command()
def main(host=_DEFAULT_HOST, port=_DEFAULT_PORT):
    s = make_server(host=host, port=port)
    asyncio.run(s.run_sse_async())
```

---

## Client config formats

### stdio — `~/.cursor/mcp.json`

```json
{
  "mcpServers": {
    "my-server": {
      "command": "/path/to/.venv/bin/python",
      "args": ["/path/to/cli/run_mcp_server.py"],
      "env": {}
    }
  }
}
```

### SSE — `~/.cursor/mcp.json`

```json
{
  "mcpServers": {
    "my-server": {
      "url": "http://127.0.0.1:8765/sse"
    }
  }
}
```

Claude Desktop uses the same JSON structure in
`~/Library/Application Support/Claude/claude_desktop_config.json`.

**SSE is the default in `setup_mcp.py`** — it survives Cursor restarts and is shared
across Cursor and Claude Desktop. Use `--stdio` to write subprocess config instead.

---

## macOS LaunchAgent for persistent SSE service

The plist goes in `~/Library/LaunchAgents/<label>.plist`.
Use the **absolute path to the venv Python** — `sys.executable` at setup time.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.myapp.mcp-server</string>

    <key>ProgramArguments</key>
    <array>
        <string>/abs/path/to/.venv/bin/python</string>
        <string>/abs/path/to/cli/run_mcp_server_sse.py</string>
        <string>--port</string>
        <string>8765</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/you/Library/Logs/my-mcp-server.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/you/Library/Logs/my-mcp-server.log</string>

    <key>WorkingDirectory</key>
    <string>/abs/path/to/repo</string>
</dict>
</plist>
```

### launchctl lifecycle

```bash
# Install + start
launchctl load   ~/Library/LaunchAgents/com.myapp.mcp-server.plist

# Reload after code change
launchctl unload ~/Library/LaunchAgents/com.myapp.mcp-server.plist
launchctl load   ~/Library/LaunchAgents/com.myapp.mcp-server.plist

# Uninstall
launchctl unload ~/Library/LaunchAgents/com.myapp.mcp-server.plist
rm               ~/Library/LaunchAgents/com.myapp.mcp-server.plist

# Check running
launchctl list com.myapp.mcp-server

# Watch logs
tail -f ~/Library/Logs/my-mcp-server.log
# Expected healthy line: "Uvicorn running on http://127.0.0.1:8765"
```

`KeepAlive: true` causes launchd to restart the process if it crashes.
After code changes, always unload + load (not just load) — launchd caches
the plist and will not pick up changes otherwise.

---

## setup_mcp.py pattern

```python
# Default: SSE service
python cli/setup_mcp.py              # install LaunchAgent + write URL config

# Opt-out: stdio subprocess
python cli/setup_mcp.py --stdio      # write command/args config

# Teardown
python cli/setup_mcp.py --uninstall  # unload + remove plist, revert to stdio config
```

`setup_mcp.py` generates the plist with `sys.executable` (the current Python) so it
always points to the right venv. Re-run it after switching venvs or moving the repo.

---

## Troubleshooting

**Cursor shows the MCP server as "Error" after startup (ECONNREFUSED race)**

Cursor attempts to connect to the SSE server exactly once at startup. If the LaunchAgent
server is still starting (e.g. recovering from a crash with `KeepAlive: true`), Cursor gets
`ECONNREFUSED`, marks the server as `error`, and never retries automatically.

Fix — toggle the server in **Cursor Settings → Tools & MCP**: turn the toggle off, then on.
The server reconnects immediately. No Cursor restart required.

`mcp.retryExhaustedServers` is a registered internal command but is **not accessible from
the Command Palette** — it has no display title and does not appear in search results.

To reduce the race window, set `ThrottleInterval: 2` in the plist so crash-restarts complete
in 2 s instead of the default 10 s:

```xml
<key>ThrottleInterval</key>
<integer>2</integer>
```

**Tools not showing in Cursor after restart (other causes)**
- Check the URL in `~/.cursor/mcp.json` matches what the server is listening on
- Verify the service is running: `launchctl list com.myapp.mcp-server`
- Check logs: `tail -20 ~/Library/Logs/my-mcp-server.log`
- Cursor requires a full quit (Cmd-Q), not just window close

**`TypeError: run_sse_async() got an unexpected keyword argument 'host'`**
- Move `host=` and `port=` to the `FastMCP()` constructor call, not to `run_sse_async()`

**`launchctl load` fails with "Input/output error"**
- Try `launchctl bootout user/$(id -u) ~/Library/LaunchAgents/com.myapp.mcp-server.plist`
  then `launchctl bootstrap user/$(id -u) ~/Library/LaunchAgents/com.myapp.mcp-server.plist`
- Or re-run `python cli/setup_mcp.py` which handles unload-before-load idempotently

**Port already in use**
- Find the process: `lsof -i :8765`
- Use a different port: `python cli/setup_mcp.py --port 8766`
