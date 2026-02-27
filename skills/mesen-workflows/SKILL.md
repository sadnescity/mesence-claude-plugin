---
description: "Mesen2 reverse engineering workflows: step-by-step patterns for finding text, variables, analyzing routines, VBlank, patching ROMs, automating input"
---

# Mesen2 Reverse Engineering Workflows

## Workflow A: Find Game Text (Relative Search + TBL)

Use this when you can see text on screen but do not know the ROM's character encoding.

1. `mesen_relative_search(searchText="DRAGON", memoryType="NesPrgRom")` -- search by byte differences between consecutive characters, works even when the encoding is unknown
2. For each match, `mesen_read_memory` around the address (20-30 bytes) to verify surrounding bytes look like text (similar byte ranges, recognizable patterns)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value corresponds to the first character in your search text. If 'D'=0x83, then 'E'=0x84, 'F'=0x85, etc.
4. `mesen_load_tbl(tblPathOrContent="83=D\n84=E\n85=F\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address, length=100, memoryType)` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="WARRIOR", memoryType)` -- find other text strings using the TBL encoding

**Tip:** Use UPPERCASE (many games only use uppercase). Search strings of 5+ characters reduce false positives. If no results, the game may use DTE/MTE compression (one byte = multiple characters). Spaces and punctuation may break relative search -- search for individual words instead.

## Workflow B: Find a Game Variable (Cheat Search)

Use this when you can see a value on screen (lives=3, HP=100) but do not know its RAM address.

1. `mesen_search_memory(patternHex="03", memoryType="NesWorkRam")` -- search for the byte value 3 in work RAM
2. Change the value in-game (lose a life so lives becomes 2)
3. `mesen_search_memory(patternHex="02", memoryType="NesWorkRam")` -- search for the new value
4. Cross-reference: addresses present in both result sets are candidates
5. `mesen_write_memory(address=candidate, hexData="09", memoryType)` -- write a test value to a candidate address. If the game display updates, you found it
6. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType, cpuType)` -- set a write breakpoint to watch what modifies the variable
7. `mesen_resume_execution()` -- resume and trigger the change in-game
8. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType)` -- check the PC register to find the writing instruction
9. `mesen_disassemble(address=PC-8, lineCount=20, cpuType)` -- view surrounding code for context
10. `mesen_freeze_address(startAddress=candidate, endAddress=candidate, cpuType, freeze=true)` -- freeze the value for infinite lives

**Tip:** The value might be stored as BCD (Binary-Coded Decimal) -- e.g., score 1500 stored as bytes $15 $00. Try searching WorkRam first, then SaveRam. For 16-bit values, search for 2-byte patterns in little-endian order.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does.

1. `mesen_init_debugger()` -- ensure the debugger is initialized
2. `mesen_playback(action="pause")` -- pause the emulator
3. `mesen_disassemble(address="$C000", lineCount=50, cpuType)` -- view the code at the target address
4. `mesen_breakpoint(action="set", address="$C000", type="Execute", memoryType, cpuType)` -- set an execution breakpoint
5. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
6. `mesen_get_state(component="cpu", cpuType)` -- check register values at the breakpoint
7. `mesen_step(cpuType, stepType="Step")` -- single step one instruction, check state after each
8. `mesen_step(cpuType, stepType="StepOver")` -- step over subroutine calls to skip into JSR/JSL targets
9. `mesen_set_trace_options(cpuType, enabled=true)` -- enable tracing, resume briefly, then `mesen_get_execution_trace(count=200, cpuType)` to see the execution flow
10. `mesen_get_callstack(cpuType)` -- see how the routine was called
11. `mesen_label(action="set", address="$C000", memoryType, label="MainLoop")` -- annotate the routine with a descriptive label

**Tip:** JSR/JSL are subroutine calls, JMP is a jump -- these reveal control flow. LDA/STA patterns reveal data access (load/store). CMP/BEQ/BNE are comparisons and branches -- key for understanding game logic. Common pattern: loop = LDX #count / DEX / BNE loop; table lookup = LDA table,X.

## Workflow D: Analyze VBlank/NMI

Use this to understand the main per-frame update routine. VBlank runs once per frame (~60 times/second) during the vertical blanking period -- the only safe time to update VRAM, sprites, and scroll registers.

1. Find the interrupt vector:
   - NES: read 2 bytes little-endian from NesPrgRom at $FFFA-$FFFB (NMI vector)
   - SNES: NMI vector at $00:FFEA-$00:FFEB in the vector table
   - GB/GBC: VBlank handler at $0040
   - GBA: VBlank interrupt handler in the IRQ table at 0x03007FFC
2. `mesen_read_memory` to read the vector bytes and calculate the handler address
3. `mesen_disassemble(address=vectorAddress, lineCount=50, cpuType)` -- view the VBlank routine
4. `mesen_breakpoint(action="set", address=vectorAddress, type="Execute", memoryType, cpuType)` -- this breakpoint will hit every frame
5. `mesen_set_trace_options(cpuType, enabled=true)` -- enable tracing, resume for a few frames, then `mesen_get_execution_trace` to see the full VBlank execution flow
6. Look for: OAM DMA (sprite transfer, NES: write to $4014), PPU register writes (scroll, control), VRAM updates (tile data, nametable), sound engine call, input reading ($4016/$4017 on NES), frame counter increment

**Tip:** The VBlank/NMI is the backbone of the game loop. Most games split work between VBlank (time-critical PPU updates) and the main loop (game logic). Tracing VBlank reveals how the game structures its per-frame rendering pipeline.

## Workflow E: Patch ROM

Use this to modify game code permanently and save the result.

1. `mesen_disassemble(address=patchAddress, lineCount=10, cpuType)` -- view the current code at the target address
2. `mesen_assemble(code="NOP\nNOP\nNOP", startAddress=patchAddress, cpuType)` -- assemble new instructions over the existing code
3. `mesen_disassemble(address=patchAddress, lineCount=10, cpuType)` -- verify the patch was applied correctly
4. `mesen_save_modified_rom(filepath="/path/to/patched.nes")` -- save the full patched ROM
5. Or `mesen_save_modified_rom(filepath="/path/to/patch.ips", saveAsIps=true)` -- create an IPS patch file instead

**Tip:** NOPing out instructions: use $EA for 6502 (NES/SNES), $00 for Z80/ARM (GB/GBA). Use `mesen_read_memory` to check bytes before and after patching. IPS patches are small and shareable; full ROM saves include all modifications.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing menus, dialogs, or game sequences.

1. `mesen_load_rom(filepath="/path/to/game.sfc")` -- load the game
2. `mesen_disable_all_keys(disabled=true)` -- prevent physical input from interfering with the script
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press the Start button
4. Advance a few frames: `mesen_step(cpuType, stepType="PpuFrame")` or `mesen_shortcut(action="execute", shortcut="RunSingleFrame")`
5. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
6. Advance frames, then set the next button combination as needed
7. `mesen_take_screenshot()` -- verify the resulting screen state
8. `mesen_disable_all_keys(disabled=false)` -- re-enable physical input when done

**Tip:** Use save states before a sequence and load them to retry. `mesen_movie(action="record")` can record input for exact replay later.
