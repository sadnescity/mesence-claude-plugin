---
description: "Mesen emulator control, ROM loading, save states, input override, rewind/history, sprites, and tilemaps. Use when loading ROMs, controlling emulation, overriding input, navigating rewind history, or inspecting sprites and tilemap data."
---

# Mesen Emulator Control & I/O Tools

All tools return plain text unless otherwise noted.

## System Control (6 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_load_rom(filepath, patchFile?)` | patchFile: IPS/BPS path | Load ROM, optional IPS/BPS patch. Returns `"Loaded: GameName (Sfc, Snes)"` |
| `mesen_get_rom_info()` | -- | Info about currently loaded ROM |
| `mesen_get_status()` | -- | Returns `"Running Snes GameName 60.1fps frame=1234"` or `"Not running."` |
| `mesen_save_state(action, slotOrPath)` | action: save\|load | Save or load state by slot 1-10 or absolute file path. Returns `"Saved slot 3"` or `"Loaded from /path"` |
| `mesen_take_screenshot(outputFile?)` | -- | Returns base64 PNG image. Optional save to disk |
| `mesen_reset(type)` | type: soft\|hard | Soft=reset button, hard=power cycle (clears RAM). Returns `"Reset (soft)."` or `"Reset (hard)."` |

## Discovery (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_list_memory_types()` | -- | List all valid memory types and sizes. Returns tab-separated lines: `"SnesPrgRom\t2097152"` |
| `mesen_list_cpu_types()` | -- | Returns header + lines: `"Console: Snes  MainCPU: Snes\nSnes*\tSnesMemory\t131072"` |

**IMPORTANT:** Call `mesen_list_memory_types()` before using any `memoryType` parameter in other tools. Call `mesen_list_cpu_types()` before using any `cpuType` parameter. Memory types and CPU types vary by loaded console.

## Input (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_input_override(action, port?, buttons?)` | action: list\|set, port: 0-7 | Override controller input on a port |

**Actions for `mesen_input_override`:**
- `list` -- list available buttons for the connected controller
- `set` -- override buttons on a port (requires `port` and `buttons`: comma-separated)

**Button names:** A, B, Up, Down, Left, Right, Select, Start, X, Y, L, R (varies by console).

## History/Rewind (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_history_state(action, isPaused?, volume?)` | action: check\|get\|set_options | Check rewind availability, get position/length, set options. Returns plain text. |
| `mesen_history_navigate(action, position)` | action: seek\|resume | Navigate rewind timeline. Returns plain text. |
| `mesen_history_export(action, filepath, position, endPosition?)` | action: save_state\|save_movie | Export from rewind timeline. Returns plain text. |

**Actions for `mesen_history_state`:**
- `check` -- see if rewind is enabled and available
- `get` -- current position and total length in the rewind timeline
- `set_options` -- change pause state (`isPaused`) and volume (`volume`)

**Actions for `mesen_history_navigate`:**
- `seek` -- jump to a specific position in the rewind timeline
- `resume` -- resume normal emulation from current rewind position

**Actions for `mesen_history_export`:**
- `save_state` -- export a save state from a specific position
- `save_movie` -- export a movie from `position` to `endPosition`

## Sprites & Tilemaps (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_sprite_list(cpuType)` | -- | All OAM sprites. Returns TSV table with header. |
| `mesen_get_tilemap_info(cpuType, layer?)` | layer=0 | Tilemap dimensions, scroll, tile size, format, addresses. Returns plain text key=value format. |
| `mesen_get_tilemap_tile_info(x, y, cpuType, layer?)` | layer=0 | Detailed info for tile at pixel coordinates. Returns plain text key=value format. |

**IMPORTANT:** Call `mesen_list_cpu_types()` before using the `cpuType` parameter.

**Layer parameter:** Default is 0. SNES and GBA support layers 0-3.
