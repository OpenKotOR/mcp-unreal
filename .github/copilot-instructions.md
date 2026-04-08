# mcp-unreal Agent Guidelines

## Project overview
Python MCP server (`mcp-unreal`) that exposes a single `execute-script` tool for executing
arbitrary Python code inside a running Unreal Engine 5 editor.

## Build and test

```bash
uv sync                       # install dependencies
uv run mcp-unreal --help      # verify CLI works
uv run python -m mcp_unreal --help
```

There are currently no automated tests; manual testing requires a running UE5 editor
with the Python Script Plugin and Remote Execution enabled.

## Key conventions

- **Single-tool design**: this server intentionally exposes only `execute-script`.
  Do not add other tools without strong justification.
- **Two backends**: `RemoteExecutionClient` (native UDP/TCP) and `AutomationBridgeClient`
  (HTTP to `ue_server.py`). Both implement `.run(code, exec_mode, unattended)` → `ExecResult`.
- **Result formatting lives in `server.py`**: `_format_result()` and helpers. Keep it
  readable Markdown — the output format was explicitly specified by the project owner.
- **No UI / no async in `ue_remote.py`**: the client is synchronous (blocking socket I/O).
  The MCP async layer (in `server.py`) runs it synchronously inside the event loop because
  scripts are short-duration. If latency becomes an issue, wrap in `asyncio.to_thread`.
- **Protocol magic**: `"ureremotexec"`, version `1`. Do not change without testing against
  multiple UE versions.

## Architecture

```
cli.py          ← argparse CLI; instantiates server + selects transport
  └─ server.py  ← MCP tool definitions + Markdown formatter
       └─ ue_remote.py ← low-level UE connection (UDP discovery, TCP framing, HTTP bridge)

plugins/ue_python_server/ue_server.py
  ← drop into UE, run once from Python console to start the HTTP bridge server
  ← required for --bridge-url mode; NOT related to McpAutomationBridge (port 8091)
```

## Backends explained

| Backend | How to activate | Requires | Output capture |
|---------|----------------|----------|----------------|
| **Native** (default) | no flags | UE Python Script Plugin + Remote Execution enabled in Project Settings | Full stdout + return value |
| **HTTP bridge** | `--bridge-url http://127.0.0.1:6800` | `plugins/ue_python_server/ue_server.py` started in UE Python console | Full stdout + return value |

McpAutomationBridge (https://github.com/ChiR24/Unreal_mcp) is a **separate** C++ plugin
for structured editor operations (spawn actors, manage blueprints, etc.) and does NOT
expose a Python REPL — do not confuse it with `ue_server.py`.

## Dependencies

- `mcp>=1.9.0` – MCP Python SDK
- `httpx` – used by `AutomationBridgeClient` only (optional at runtime)
- `uvicorn` + `starlette` – only needed for `--transport http/sse`

## UE Python API references

- Overview: https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api
- Scripting guide: https://dev.epicgames.com/documentation/en-us/unreal-engine/scripting-the-unreal-editor-using-python
- McpAutomationBridge plugin: https://github.com/ChiR24/Unreal_mcp
