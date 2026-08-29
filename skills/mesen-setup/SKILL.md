---
description: "MesenCE MCP server setup: enabling the MCP server, Streamable HTTP transport, connection troubleshooting. Use when setting up or troubleshooting MesenCE MCP connection."
---

# MesenCE MCP Server Setup

## Overview

MesenCE is a **multi-system emulator** (NES, SNES, Game Boy, GBA, PC Engine, SMS/Game Gear, WonderSwan) with a **built-in MCP server** -- no external process or sidecar is needed. The emulator itself serves the MCP protocol over Streamable HTTP (MCP Protocol Revision 2025-11-25).

## Enabling the MCP Server

### Via GUI
1. Open MesenCE
2. Go to **Tools menu > MCP Server Config**
3. Set the port and enable the server

### Via CLI
Launch MesenCE with the `--mcp` flag to auto-start the MCP server on launch:

```bash
Mesen --mcp                  # Start with MCP server on default port
Mesen --mcp-port=9200        # Start with MCP server on custom port (1-65535)
```

The default port is **9100**.

## Transport

- **Protocol:** Streamable HTTP (MCP 2025-11-25)
- **Endpoint:** `POST http://localhost:9100/mcp` (JSON-RPC 2.0)
- **SSE support:** Include `Accept: text/event-stream` header for SSE responses
- **Session management:** Server returns `MCP-Session-Id` header on initialize; include it in all subsequent requests

The server starts automatically when MesenCE launches (if enabled). It does not require a ROM to be loaded -- `mesen_get_status` works even with no ROM loaded.

## Thread Safety

All MCP tool handlers dispatch to the **emulator thread**. This is single-threaded by design -- tool calls are serialized and never race with each other or the emulation loop.

## Troubleshooting

1. **Server not responding:** Enable the MCP server via **Tools > MCP Server Config** in the GUI, or launch with the `--mcp` flag. The server must be enabled before it will accept connections.
2. **Port conflict:** Change the port in MCP Server Config or use `--mcp-port=PORT` on the command line.
3. **Connection refused:** The server only binds to localhost. Remote connections are not supported.
4. **Tools returning errors:** Most tools require a ROM to be loaded. Use `mesen_load_rom` first, or `mesen_get_status` to check current state.
5. **Timeout on tool calls:** Some operations may take longer. The emulator pauses during tool execution to ensure consistent state.
6. **Session errors (400):** Ensure you include the `MCP-Session-Id` header from the initialize response on all subsequent requests.
