# Mesen2 Claude Code Plugin

Claude Code plugin for [Mesen2](https://github.com/brisma/Mesen2) multi-system emulator MCP integration.

Provides MCP server configuration and reference skills for Mesen2's built-in MCP server, enabling runtime debugging, memory inspection, disassembly, tracing, ROM hacking, text search, and more -- 75 MCP tools for NES, SNES, Game Boy, GBA, PC Engine, SMS/Game Gear, and WonderSwan reverse engineering.

## Installation

Install from the Claude Code plugin marketplace:

```
/plugin marketplace add brisma/claude-plugins
/plugin install mesen2@brisma-plugins
```

## Prerequisites

- [Mesen2](https://github.com/brisma/Mesen2) with MCP server support
- MCP server enabled in Mesen2:
  - GUI: **Tools > MCP Server Config**
  - Or CLI: `--mcp` flag (with optional `--mcp-port=PORT`)
- Default port: 9100

## What's Included

### MCP Server Configuration (`.mcp.json`)

Connects Claude Code to Mesen2's built-in MCP server via Streamable HTTP on `localhost:9100`.

### Skills

| Skill | Description |
|-------|-------------|
| `mesen-setup` | MCP server setup, configuration, and troubleshooting |
| `mesen-debug-execution` | CPU debugging: stepping, breakpoints, state inspection, callstack, expressions |
| `mesen-memory` | Memory read/write, search, freeze, address conversion, access counters |
| `mesen-disassembly` | Disassembly, assembly, labels, Code Data Logger, tracing, profiling |
| `mesen-emulator` | System control, config, input, recording, history/rewind, sprites/tilemaps |
| `mesen-romhacking` | Cheats, tiles, palette, Lua scripting, TBL text search, ROM patching |
| `mesen-console-reference` | Per-console reference: memory types, CPU types, tile formats for all 7 systems |
| `mesen-workflows` | Reverse engineering workflows: find text, find variables, analyze routines, patch ROMs |

## Supported Systems

NES, SNES, Game Boy, GBA, PC Engine, SMS/Game Gear, WonderSwan

## Upstream

Mesen2 with MCP support: <https://github.com/brisma/Mesen2>
