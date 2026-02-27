---
description: "Mesen memory tools: read/write memory, search for byte patterns, freeze addresses, convert between CPU and absolute addresses, and track memory access counters. Use when inspecting or modifying game memory, searching for values, freezing cheat addresses, or analyzing memory access patterns."
---

# Mesen Memory Tools

## Memory Read/Write (3 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_read_memory(address, length, memoryType, outputFile?)` | address (dec/0x/$), length (max 4096), memoryType, optional outputFile path | Read memory. Returns hex dump with ASCII. Can save raw binary to file. |
| `mesen_write_memory(address, hexData, memoryType)` | address, hexData (e.g. "EAEA"), memoryType | Write hex data to memory |
| `mesen_get_memory_size(memoryType)` | memoryType | Get size of memory region in bytes |

**Tips:**
- `mesen_read_memory` returns a hex dump with ASCII sidebar, similar to a hex editor view.
- Max read size is 4096 bytes per call. For larger reads, specify `outputFile` to save raw binary to disk.
- `mesen_write_memory` takes a hex string without spaces or `0x` prefix (e.g. `"EAEA"` to write two NOP bytes on 6502).
- Use `mesen_get_memory_size` to determine the extent of a memory region before reading.

## Memory Search (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_search_memory(patternHex, memoryType, startAddress?, endAddress?, maxResults?)` | patternHex (e.g. "AD0020"), memoryType, startAddress=0, endAddress=end, maxResults=50 | Search memory for hex byte pattern. Returns matching addresses. |

**Tips:**
- `patternHex` is a hex string without spaces (e.g. `"AD0020"` to find the byte sequence `AD 00 20`).
- `endAddress` is capped at 1 MB for performance. Narrow the range with `startAddress` and `endAddress` when possible.
- Use `maxResults` to limit output when many matches are expected.

## Freeze (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_freeze_address(startAddress, endAddress, cpuType, freeze)` | startAddress, endAddress (same as start for single byte), cpuType, freeze=true/false | Freeze/unfreeze memory address range. Frozen addresses cannot be written by the game. |

**Notes:**
- This tool uses `cpuType` (not `memoryType`) because it operates on the CPU address space.
- To freeze a single byte, set `endAddress` equal to `startAddress`.
- Set `freeze=false` to unfreeze a previously frozen range.
- Useful for locking health, lives, or other game values during testing.

## Address Conversion (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_address_info(address, cpuType)` | address (dec/0x/$), cpuType | Convert between CPU (relative) and absolute (physical) addresses. Returns relativeAddress, absoluteAddress, absoluteMemoryType. |

**Notes:**
- Converts a CPU-visible (relative) address to an absolute (physical) address, revealing which memory region it maps to.
- The response includes `relativeAddress`, `absoluteAddress`, and `absoluteMemoryType`.
- Useful for understanding memory mapping -- e.g., determining whether a CPU address points to ROM, work RAM, or a hardware register.

## Access Counters (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_memory_access_counts(action, address?, length?, memoryType?)` | action: get\|reset, address (for get), length=256 (max 256), memoryType (for get) | Get or reset memory access counters. 'get' returns per-address read/write/exec counts. 'reset' clears all counters. |

**Action-specific parameters:**
- `get` -- requires `address`, `memoryType`, optional `length` (default 256, max 256). Returns per-address read, write, and exec counts.
- `reset` -- no additional parameters. Clears all access counters for all memory regions.

**Tips:**
- Access counters track how many times each memory address has been read, written, or executed since last reset.
- Useful for identifying hot code paths, frequently accessed data, or unused memory regions.
- Call `reset` before a specific game action, then `get` afterward to see exactly what was accessed.

## Important Notes

- **Always call `mesen_list_memory_types` first** to discover valid memory type names for the current ROM. Memory types are console-specific (e.g., NES uses `NesWorkRam`, SNES uses `SnesWorkRam`, Game Boy uses `GbWorkRam`).
- Addresses can be specified as decimal, `0x` hex, or `$` hex (e.g., `$C000`, `0xC000`, `49152` all refer to the same address).
- `mesen_freeze_address` and `mesen_get_address_info` use `cpuType` (not `memoryType`) -- they operate on the CPU address space.
- `mesen_read_memory`, `mesen_write_memory`, `mesen_get_memory_size`, `mesen_search_memory`, and `mesen_memory_access_counts` use `memoryType` -- they operate on physical memory regions.
