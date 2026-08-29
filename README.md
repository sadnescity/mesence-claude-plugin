# MesenCE Claude Code Plugin

Claude Code plugin for [MesenCE](https://github.com/sadnescity/MesenCE) multi-system emulator MCP integration.

Provides MCP server configuration and reference skills for MesenCE's built-in MCP server, enabling runtime debugging, memory inspection, disassembly, tracing, ROM hacking, text search, and more -- 57 MCP tools for NES, SNES, Game Boy, GBA, PC Engine, SMS/Game Gear, and WonderSwan reverse engineering.

## Installation

Install from the Claude Code plugin marketplace:

```
/plugin marketplace add sadnescity/claude-plugins
/plugin install mesence@sadnescity-plugins
```

## Prerequisites

- [MesenCE](https://github.com/sadnescity/MesenCE) with MCP server support
- MCP server enabled in MesenCE:
  - GUI: **Tools > MCP Server Config**
  - Or CLI: `--mcp` flag (with optional `--mcp-port=PORT`)
- Default port: 9100

## What's Included

### MCP Server Configuration (`.mcp.json`)

Connects Claude Code to MesenCE's built-in MCP server via Streamable HTTP on `localhost:9100`.

### Skills

| Skill | Description |
|-------|-------------|
| `mesen-setup` | MCP server setup, configuration, and troubleshooting |
| `mesen-debug-execution` | CPU debugging: stepping, breakpoints, state inspection, callstack, expressions |
| `mesen-memory` | Memory read/write, search, freeze, address conversion, access counters |
| `mesen-disassembly` | Disassembly, assembly, labels, Code Data Logger, tracing, profiling |
| `mesen-emulator` | System control, save states, history/rewind, sprites/tilemaps |
| `mesen-romhacking` | Cheats, tiles, palette, Lua scripting, TBL text search, ROM patching |
| `mesen-console-reference` | Per-console reference: memory types, CPU types, tile formats for all 7 systems |
| `mesen-workflows-nes` | NES reverse engineering workflows |
| `mesen-workflows-snes` | SNES reverse engineering workflows |
| `mesen-workflows-gb` | Game Boy / GBC reverse engineering workflows |
| `mesen-workflows-gba` | GBA reverse engineering workflows |
| `mesen-workflows-pce` | PC Engine / TurboGrafx-16 reverse engineering workflows |
| `mesen-workflows-sms` | SMS / Game Gear reverse engineering workflows |
| `mesen-workflows-ws` | WonderSwan reverse engineering workflows |

## Supported Systems

NES, SNES, Game Boy, GBA, PC Engine, SMS/Game Gear, WonderSwan

## Upstream

MesenCE with MCP support: <https://github.com/sadnescity/MesenCE>
