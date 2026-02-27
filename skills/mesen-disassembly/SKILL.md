---
description: "Mesen disassembly, assembly, tracing, profiling, and code analysis tools: disassemble code, search disassembly, assemble instructions, manage labels, Code Data Logger coverage and function detection, execution tracing, and per-function profiling. Use when analyzing ROM code, patching instructions, tracing execution, or profiling performance in any supported system (NES, SNES, Game Boy, GBA, PC Engine, SMS, WonderSwan)."
---

# Mesen Disassembly, Assembly & Code Analysis Tools

## Disassembly (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_disassemble(address, lineCount, cpuType, outputFile?)` | address (dec/0x/$), lineCount (max 500), cpuType, optional outputFile | Disassemble code. Returns lines with address, byteCode, text, comment. |
| `mesen_find_occurrences(searchString, cpuType, matchCase?, matchWholeWord?)` | searchString (e.g. "JSR", "LDA #$"), cpuType, matchCase=false, matchWholeWord=false | Search disassembly for text pattern. Returns up to 500 matches. |

**Notes:**
- Address can be specified in decimal, hex with `0x` prefix, or 6502-style `$` prefix.
- Output includes address, raw byte code, disassembled text, and any auto-generated comments.
- Use `outputFile` to write large disassembly dumps to disk instead of returning them inline.
- `mesen_find_occurrences` searches the full disassembly view, useful for finding all calls to a subroutine or all uses of a specific instruction pattern.

## Assembly (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_assemble(code, startAddress, cpuType)` | code (newline-separated), startAddress, cpuType | Assemble code at address. Returns success, byteCount, hexBytes, errors. |

**Notes:**
- Pass multiple instructions separated by newlines.
- Returns the assembled bytes as hex, the total byte count, and any assembler errors.
- The assembled bytes are written directly into memory at `startAddress`.

## CPU Types

The following `cpuType` values are valid for disassembly, assembly, and tracing tools:

`Nes`, `Snes`, `Gameboy`, `Gba`, `Pce`, `Sms`, `Ws`, `Spc`, `Sa1`, `Gsu`, `Cx4`

- `Spc` is the SPC-700 audio coprocessor (SNES).
- `Sa1`, `Gsu`, `Cx4` are SNES cartridge coprocessors (SA-1, Super FX, CX4).

## Labels (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_label(action, address?, memoryType?, label?, comment?)` | action: set\|clear_all | Set a label on an address or clear all labels. 'set' requires address, memoryType, label. Optional comment. |

**Notes:**
- Labels appear in the disassembly view and trace output, replacing raw addresses with meaningful names.
- Use `set` to create or update a label at a specific address. Requires `address`, `memoryType`, and `label`.
- Use `clear_all` to remove every label at once.
- The optional `comment` field adds a note displayed alongside the label in the disassembly.

## Code Data Logger (4 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_cdl_statistics(memoryType)` | memoryType | CDL coverage: percentage of code, data, and unknown bytes |
| `mesen_get_cdl_functions(memoryType)` | memoryType | List function entry points detected by CDL |
| `mesen_mark_bytes_as(startAddress, endAddress, memoryType, flags)` | startAddress, endAddress, memoryType, flags: None\|Code\|Data\|JumpTarget\|SubEntryPoint | Mark byte range in CDL |
| `mesen_cdl_file(action, memoryType, filepath)` | action: save\|load, memoryType, filepath | Save or load CDL data to/from file |

**Notes:**
- CDL (Code Data Logger) tracks which bytes have been executed as code vs read as data during emulation.
- Use `mesen_get_cdl_statistics` to see coverage -- how much of the ROM has been identified as code, data, or remains unknown.
- Use `mesen_get_cdl_functions` to find function entry points that the CDL has detected through `JSR`/`CALL` targets and similar instructions.
- `mesen_mark_bytes_as` manually annotates a byte range with CDL flags. Use `None` to clear flags, `Code` or `Data` to classify bytes, `JumpTarget` for branch destinations, and `SubEntryPoint` for function entry points.
- CDL data can be saved/loaded with `mesen_cdl_file` for persistent analysis across sessions.

## Tracing (4 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_set_trace_options(cpuType, enabled?, format?, condition?, useLabels?, indentCode?)` | cpuType, enabled=true, optional format string, condition expression, useLabels=true, indentCode=false | Configure trace logging. Must be called before reading trace. |
| `mesen_get_execution_trace(count, cpuType?, outputFile?)` | count (max 30000), optional cpuType filter, optional outputFile | Get last N executed instructions with register state |
| `mesen_clear_execution_trace()` | -- | Clear the trace buffer |
| `mesen_trace_file(action, filepath?)` | action: start\|stop, filepath (required for start) | Log execution trace to file continuously |

**Notes:**
- Call `mesen_set_trace_options` to enable tracing before using `mesen_get_execution_trace`. Tracing is off by default.
- Tracing is per-CPU. Use the `cpuType` parameter on `mesen_get_execution_trace` to filter to a specific processor.
- The `format` parameter controls what appears in each trace line. Example: `"[Disassembly][Align,24] A:[A,2h] X:[X,2h] Y:[Y,2h]"`
- The `condition` parameter is a Mesen expression that filters which instructions are logged (e.g., `"A == $FF"` to only trace when the accumulator is 0xFF).
- Set `indentCode=true` to indent trace lines based on subroutine call depth.
- For large analysis, use `mesen_trace_file` to log to disk continuously instead of buffering in memory. Call with `action="start"` and a `filepath` to begin, `action="stop"` to end.
- `mesen_clear_execution_trace` empties the in-memory trace buffer without changing trace options.

## Profiler (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_profiler(action, cpuType, outputFile?)` | action: get\|reset, cpuType, optional outputFile | Per-function profiling: call count, cycles, min/max. 'get' returns statistics, 'reset' clears data. |

**Notes:**
- The profiler collects per-function statistics: total call count, total cycles spent, and min/max cycles per call.
- Use `get` to retrieve current profiling data. Use `reset` to clear all accumulated statistics and start fresh.
- Use `outputFile` to write profiler results to disk for large datasets.
- Profiling data is based on CDL-detected function boundaries. Run the game for a while to build up CDL data before profiling for best results.
