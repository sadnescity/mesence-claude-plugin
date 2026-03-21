---
description: "Mesen ROM hacking tools: cheats, palette editing, tile pixel editing, ROM header/output, Lua scripting, and text search via TBL tables and relative search. Use when applying cheat codes, editing graphics or colors, running Lua scripts, searching for text in ROMs, or saving modified ROMs."
---

# Mesen ROM Hacking Tools

All tools return plain text unless otherwise noted.

## Cheats (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_set_cheats(codes[])` | codes: string array | Apply cheat codes (replaces all active cheats). Returns `"3 cheats applied."` |
| `mesen_clear_cheats()` | -- | Clear all active cheat codes. Returns `"Cheats cleared."` |

**Code format:** Each entry in `codes[]` is `"Type:Code"`. Valid types:
- `NesGameGenie` -- e.g. `"NesGameGenie:SXIOPO"`
- `NesCustom` -- NES raw address/value cheats
- `SnesGameGenie` -- SNES Game Genie format
- `SnesProActionReplay` -- e.g. `"SnesProActionReplay:7E0DBF01"`
- `GbGameGenie` -- Game Boy Game Genie format
- `GbGameShark` -- Game Boy GameShark format

## Palette (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_palette(cpuType)` | cpuType | Get all palette colors. Returns header + space-separated RGB hex values. |
| `mesen_set_palette_color(cpuType, colorIndex, colorHex)` | colorHex: 6-digit RGB hex | Set a single palette color at runtime. Returns `"Color 5 set to #FF0000"` |

**Notes:**
- `colorHex` is 6-digit RGB hex (e.g. `"FF0000"` for red). Optional `#` or `$` prefix accepted.
- `colorIndex` is the palette entry to modify.

## Tiles (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_tile_pixel(tileAddress, format, x, y, memoryType)` | x/y within tile | Read a pixel from a tile in memory. Returns `"Color index: 2"` |
| `mesen_set_tile_pixel(tileAddress, format, x, y, color, memoryType)` | color: palette index | Write a pixel to a tile in memory. Returns `"Pixel set to color 2"` |

**Parameters:**
- `tileAddress` -- Address of the tile data (decimal, `0x` hex, or `$` hex prefix)
- `x`, `y` -- Pixel coordinates within the tile (0-7 for 8x8 tiles, 0-15 for 16x16 tiles)
- `format` -- Tile pixel format (see below)
- `memoryType` -- Which memory region contains the tile data

**Tile pixel formats:** `Bpp2`, `Bpp4`, `Bpp8`, `NesBpp2`, `SmsBpp4`, `GbaBpp4`, `GbaBpp8`, `PceBpp4`, `WsBpp2`, `DirectColor`

## ROM Header & Output (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_rom_header()` | -- | Get ROM header. Returns JSON (console-specific structured data). |
| `mesen_save_modified_rom(filepath, saveAsIps?, stripOption?)` | saveAsIps=false | Save modified ROM. Returns `"ROM saved to /path"` or `"IPS patch saved to /path"` |

**Parameters for `mesen_save_modified_rom`:**
- `filepath` -- Output file path
- `saveAsIps` -- `false` saves full ROM (default); `true` creates an IPS patch instead
- `stripOption` -- CDL-based stripping: `StripNone` (default), `StripUnused`, `StripUsed`

## Lua Scripting (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_run_lua_script(code, waitMs?, persistent?)` | waitMs=200, persistent=false | Run Lua code. Non-persistent: returns log output directly. Persistent: returns `"Script #7 running (persistent).\n{log}"` |
| `mesen_remove_lua_script(scriptId)` | scriptId from run_lua_script | Remove a persistent script by ID |

**Parameters for `mesen_run_lua_script`:**
- `code` -- Lua source code to execute
- `waitMs` -- Time in ms to wait for output (default 200, max 5000)
- `persistent` -- `false` (default): script runs once and auto-cleans up; `true`: script stays running

**Notes:**
- Non-persistent scripts run once and auto-cleanup.
- Persistent scripts (`persistent=true`) stay running -- use for `emu.addCallback`-based scripts. Call `mesen_remove_lua_script` to stop them.

## Text Search -- TBL (5 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_relative_search(searchText, memoryType, startAddress?, endAddress?, maxResults?)` | -- | Find text by matching byte differences. Returns text with matches per line. |
| `mesen_load_tbl(tblPathOrContent)` | -- | Load a TBL character mapping table. Returns text confirmation. |
| `mesen_search_text(text, memoryType, startAddress?, endAddress?, maxResults?)` | -- | Search memory for text using loaded TBL. Returns text with matches. |
| `mesen_decode_text(address, length, memoryType, endMarker?)` | -- | Decode memory region as text using loaded TBL. Returns `"24 bytes: Hello, World!"` |
| `mesen_get_tbl_info()` | -- | Show all mappings in the currently loaded TBL. Returns text. |

### Relative Search

`mesen_relative_search` is the standard ROM hacking technique for finding text when the character encoding is unknown. It matches byte *differences* between consecutive characters rather than absolute values. Use UPPERCASE text for best results. Returns candidate addresses and the inferred base value (the offset from ASCII to the game's encoding).

### TBL File Format

TBL (table) files map hex byte values to characters. Load via file path or raw content string.

Each line: `HH=C` where `HH` is one or more hex bytes and `C` is the character. Supports multi-byte entries and DTE (dual-tile encoding).

Example:
```
00=A
01=B
02=C
80=th
81=he
FF=<END>
```

### Workflow

1. `mesen_relative_search` -- find text candidates without any encoding knowledge
2. Build a TBL file from the inferred base value
3. `mesen_load_tbl` -- load the TBL
4. `mesen_search_text` / `mesen_decode_text` -- search and decode using the TBL

**Notes:**
- `mesen_relative_search` works without a TBL (encoding-agnostic)
- `mesen_search_text` and `mesen_decode_text` REQUIRE a loaded TBL
- `endMarker` in `mesen_decode_text` is a hex byte value for end-of-string detection (e.g. `"FF"`, `"00"`)
