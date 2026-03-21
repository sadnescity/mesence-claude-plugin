---
description: "Mesen debugger execution tools: playback control, stepping, breakpoints, CPU/PPU state inspection, call stacks, expression evaluation. Use when debugging ROM code, setting breakpoints, stepping through instructions, or inspecting registers in any supported system (NES, SNES, Game Boy, GBA, PC Engine, SMS, WonderSwan)."
---

# Mesen Debugger Execution Tools

The debugger auto-initializes on first use -- no setup call is needed. All tools return plain text unless otherwise noted.

## Playback Control (2 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_playback(action)` | action: pause\|resume | Pause or resume emulation |
| `mesen_resume_execution()` | -- | Resume execution after a breakpoint or step. Returns `"Execution resumed."` |

**Notes:**
- `mesen_playback(action="pause")` pauses emulation globally. Use `mesen_resume_execution()` specifically to continue after hitting a breakpoint or completing a step operation.

## Stepping (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_step(cpuType, count?, stepType?)` | cpuType (required), count=1, stepType=Step | Step the CPU. Returns CPU state as a single text line (same format as `mesen_get_state(cpu)`). |

**Step types:**
- `Step` -- execute a single instruction (default)
- `StepOver` -- step over subroutine calls (executes the call and stops after it returns)
- `StepOut` -- run until the current subroutine returns
- `CpuCycleStep` -- advance by a single CPU cycle
- `PpuStep` -- advance by a single PPU cycle
- `PpuScanline` -- advance by one PPU scanline
- `PpuFrame` -- advance by one full PPU frame

**CPU types:** `Nes`, `Snes`, `Gameboy`, `Gba`, `Pce`, `Sms`, `Ws`, `Spc`, `NecDsp`, `Sa1`, `Gsu`, `Cx4`

**Notes:**
- `mesen_step` is synchronous -- returns immediately with the CPU state after the step.
- For multi-system: always specify `cpuType`. Use `mesen_list_cpu_types` to discover available CPUs for the current ROM.
- For SNES, common CPU types are `Snes` (main 65816), `Spc` (SPC700 audio), `Sa1` (SA-1 co-processor).

## Breakpoints (1 tool)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_breakpoint(action, ...)` | action: set\|remove\|remove_all\|list | Manage breakpoints and watchpoints. Returns plain text confirmation. |

**Actions:**
- `set` -- create breakpoint (requires `address`, `type`, `memoryType`, `cpuType`; optional `endAddress` for ranges, `condition` for conditional breakpoints)
- `remove` -- remove specific breakpoint (requires `address`, `memoryType`)
- `remove_all` -- remove all breakpoints
- `list` -- show all active breakpoints

**Breakpoint types (for `type` parameter):**
- `Execute` -- triggers when the CPU executes the instruction at address
- `Read` -- triggers on memory read (data read watchpoint)
- `Write` -- triggers on memory write (data write watchpoint)
- Comma-separated for multiple: `"Read,Write"`

**Condition expressions:** e.g. `"A == $42"`, `"[0x2000] > 5"`

**Notes:**
- When a breakpoint hits, execution pauses automatically. Use `mesen_get_state(component="cpu", cpuType=...)` to inspect state, then `mesen_resume_execution()` or `mesen_step()` to proceed.
- Use `endAddress` to set a breakpoint on an address range (e.g. watch all writes to a VRAM region).
- Conditional breakpoints use Mesen's expression syntax: register names (`A`, `X`, `Y`, `SP`, `PC`), hex literals with `$` prefix, memory reads with `[address]`.

## State Inspection (4 tools)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `mesen_get_state(component, cpuType)` | component: cpu\|ppu, cpuType required | Get CPU or PPU state |
| `mesen_set_program_counter(cpuType, address)` | cpuType, address (dec or 0x/$ hex) | Set the program counter. Returns `"PC=$8000"` |
| `mesen_get_callstack(cpuType)` | cpuType required | Get current call stack. Returns plain text lines: `"$818051 -> $80922B ret=$818055"` |
| `mesen_evaluate_expression(expression, cpuType)` | expression string, cpuType | Evaluate debugger expression. Returns `"66 ($42)"` |

**Expression examples:** `"A + X"`, `"$4016"`, `"[0x2000]"`, `"Y * 2 + $10"`

**Notes:**
- `mesen_get_state(component="cpu", ...)` returns a single plain text line with all CPU registers and flags. Example: `"Snes PC=$829B77 A=$0006 X=$0020 Y=$00E0 SP=$1FF3 D=$1F00 DBR=$00 K=$82 PS=$84 flags=Nv--dIzc"`
- `mesen_get_state(component="ppu", ...)` returns JSON with console-specific PPU state (scanline, cycle, frame count, etc.).
- `mesen_set_program_counter` accepts decimal, `0x`-prefixed hex, or `$`-prefixed hex addresses.
- `mesen_get_callstack` shows the chain of subroutine calls (JSR/BSR return addresses) leading to the current PC. Useful for understanding how execution reached the current point.
- `mesen_evaluate_expression` uses Mesen's built-in expression evaluator, supporting register names, arithmetic, memory reads, and hex literals.

## General Notes

- All debug tools require `cpuType`. Call `mesen_list_cpu_types` first to discover available CPUs for the currently loaded ROM.
- For SNES, common CPU types are `Snes` (main 65816), `Spc` (SPC700 audio), `Sa1` (SA-1 co-processor), `Gsu` (Super FX).
- For systems with a single CPU (NES, Game Boy, GBA, etc.), still pass the appropriate `cpuType` (e.g. `Nes`, `Gameboy`, `Gba`).
