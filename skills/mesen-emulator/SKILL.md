---
description: "Mesen emulator control, ROM loading, save states, input automation, recording, rewind/history, sprites, and tilemaps. Use when loading ROMs, controlling emulation, automating input, recording video/audio, navigating rewind history, or inspecting sprites and tilemap data."
---

# Mesen Emulator Control & I/O Tools

## System Control (9 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_load_rom(filepath, patchFile?)` | patchFile: IPS/BPS path | Load ROM, optional IPS/BPS patch. Returns romInfo with name, format, consoleType, hash |
| `mesen_get_rom_info()` | -- | Info about currently loaded ROM |
| `mesen_get_status()` | -- | Emulator status: running, paused, consoleType, romName, fps, frameCount, masterClock |
| `mesen_save_state(action, slotOrPath)` | action: save\|load | Save or load state by slot 1-10 or absolute file path |
| `mesen_take_screenshot(outputFile?)` | -- | Returns base64 PNG image. Optional save to disk |
| `mesen_reset(type)` | type: soft\|hard | Soft=reset button, hard=power cycle (clears RAM) |
| `mesen_shortcut(action, shortcut?, param?)` | action: list\|execute | Execute emulator shortcuts |
| `mesen_set_emulation_flag(flag, enabled)` | flag: Turbo\|Rewind\|MaximumSpeed | Enable or disable an emulation flag |
| `mesen_display_message(title, message)` | -- | Show on-screen display message |

**Actions for `mesen_shortcut`:**
- `list` -- list all available shortcuts
- `execute` -- run a shortcut by name (requires `shortcut`)

**Common shortcuts:** FastForward, Rewind, Pause, TakeScreenshot, ToggleCheats, RunSingleFrame, IncreaseSpeed, DecreaseSpeed, MaxSpeed.

## Config & Info (6 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_version()` | -- | Emulator version and build date |
| `mesen_get_screen_info()` | -- | Screen width, height, aspect ratio |
| `mesen_get_timing_info(cpuType)` | -- | FPS, frameCount, masterClock, masterClockRate, scanline |
| `mesen_get_log()` | -- | Emulator log output (debug messages, warnings, errors) |
| `mesen_list_memory_types()` | -- | List all valid memory types and sizes |
| `mesen_list_cpu_types()` | -- | Returns consoleType, mainCpu, availableCpus |

**IMPORTANT:** Call `mesen_list_memory_types()` before using any `memoryType` parameter in other tools. Call `mesen_list_cpu_types()` before using any `cpuType` parameter. Memory types and CPU types vary by loaded console.

## Input Control (6 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_key_state(action, scanCode?, pressed?)` | action: set\|reset\|get_pressed | Press/release key, release all, or list pressed keys |
| `mesen_key_info(scanCode?, keyName?)` | -- | Look up key info by scanCode or keyName |
| `mesen_mouse(mode, x, y)` | mode: relative\|absolute | Move mouse cursor |
| `mesen_input_override(action, port?, buttons?)` | action: list\|set, port: 0-7 | Override controller input on a port |
| `mesen_disable_all_keys(disabled)` | disabled: true\|false | Disable or re-enable all physical input |
| `mesen_has_control_device(controllerType)` | -- | Check if controller type is connected |

**Actions for `mesen_key_state`:**
- `set` -- press or release a key (requires `scanCode` and `pressed`: true/false)
- `reset` -- release all currently pressed keys
- `get_pressed` -- list all currently pressed keys

**Key name examples for `mesen_key_info`:** "A", "Space", "Enter", "Up", "Down", "Left", "Right".

**Mouse modes for `mesen_mouse`:**
- `relative` -- pixel deltas from current position
- `absolute` -- normalized 0.0-1.0 screen position

**Actions for `mesen_input_override`:**
- `list` -- list available buttons for the connected controller
- `set` -- override buttons on a port (requires `port` and `buttons`: comma-separated)

**Button names:** A, B, Up, Down, Left, Right, Select, Start, X, Y, L, R (varies by console).

**Controller types for `mesen_has_control_device`:** SnesController, NesController, SnesMouse, SuperScope, NesZapper, GameboyController, GbaController, PceController, SmsController, WsController, etc.

## Recording (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_video_recording(action, filepath?, codec?, compressionLevel?, recordSystemHud?, recordInputHud?)` | action: start\|stop\|status | Control AVI video recording |
| `mesen_audio_recording(action, filepath?)` | action: start\|stop\|status | Control WAV audio recording |
| `mesen_movie(action, filepath?, recordFrom?, author?, description?)` | action: record\|play\|stop\|status | Control movie (.mmo) recording and playback |

**Video codecs for `mesen_video_recording`:** None, ZMBV, CSCD, GIF. Default compressionLevel=6.

**Recording sources for `mesen_movie`:**
- `StartWithoutSaveData` -- record from power-on with no save data
- `StartWithSaveData` -- record from power-on with existing save data
- `CurrentState` -- record from current emulator state

## History/Rewind (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_history_state(action, isPaused?, volume?)` | action: check\|get\|set_options | Check rewind availability, get position/length, set options |
| `mesen_history_navigate(action, position)` | action: seek\|resume | Navigate rewind timeline |
| `mesen_history_export(action, filepath, position, endPosition?)` | action: save_state\|save_movie | Export from rewind timeline |

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
| `mesen_get_sprite_list(cpuType)` | -- | All OAM sprites: position, tile index, palette, size, mirroring, priority, visibility |
| `mesen_get_tilemap_info(cpuType, layer?)` | layer=0 | Tilemap dimensions, scroll position, tile size, format, addresses |
| `mesen_get_tilemap_tile_info(x, y, cpuType, layer?)` | layer=0 | Detailed info for tile at pixel coordinates |

**IMPORTANT:** Call `mesen_list_cpu_types()` before using the `cpuType` parameter.

**Layer parameter:** Default is 0. SNES and GBA support layers 0-3.

**Sprite list fields:** position (x, y), tile index, palette, size, horizontal/vertical mirroring, priority, visibility.

**Tile info fields:** tile index, VRAM address, palette, horizontal/vertical mirroring, priority.
