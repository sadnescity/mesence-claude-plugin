---
description: "WonderSwan / WonderSwan Color reverse engineering workflows for MesenCE: find text, find variables, analyze routines, VBlank analysis, patch ROMs, automate input. WS-specific memory types, vectors, registers, and instruction patterns."
---

# WonderSwan / WonderSwan Color Reverse Engineering Workflows

Quick reference: cpuType=`Ws`, ROM=`WsPrgRom`, RAM=`WsWorkRam`, VRAM is part of `WsWorkRam`. CPU is NEC V30MZ (8086-compatible). NOP=$90. 20-bit segmented addressing: physical = (segment << 4) + offset. WS mono: 16KB RAM, WSC: 64KB RAM. VRAM occupies the first 16KB (mono) or 64KB (color) of work RAM.

## Workflow A: Find Game Text (Relative Search + TBL)

Use this when you can see text on screen but do not know the ROM's character encoding. WonderSwan games typically use custom tile encodings. Many games support both ASCII-adjacent and Japanese character sets.

1. `mesen_relative_search(searchText="START", memoryType="WsPrgRom")` -- search by byte differences between consecutive characters; works even with unknown custom encodings
2. For each match, `mesen_read_memory(address=matchAddress-4, length=30, memoryType="WsPrgRom")` -- read surrounding bytes to verify the match looks like text (similar byte ranges, consistent patterns)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value corresponds to the first character in your search text. If 'S'=$53, then 'T'=$54, 'A'=$41, etc.
4. `mesen_load_tbl(tblPathOrContent="41=A\n42=B\n43=C\n53=S\n54=T\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address=matchAddress, length=100, memoryType="WsPrgRom")` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="GAME", memoryType="WsPrgRom")` -- find other text strings using the TBL encoding

**Tip:** Many WonderSwan games use near-ASCII encoding since the platform targeted a Japanese audience with some English text. Try standard ASCII values first. Search strings of 5+ characters reduce false positives. WS ROMs are typically 1-16MB. Some games store text as tilemap entries (2 bytes per character -- tile index + palette/flags), which may break relative search on single bytes. Japanese games often use Shift-JIS or custom kana tables.

## Workflow B: Find a Game Variable (Cheat Search)

Use this when you can see a value on screen (lives=3, HP=100) but do not know its RAM address. WS mono has 16KB of work RAM, WSC has 64KB. VRAM shares this space (first 16KB on mono, first 64KB on WSC), so game variables are typically stored in the upper portion of RAM.

1. `mesen_search_memory(patternHex="03", memoryType="WsWorkRam")` -- search for the byte value 3 in work RAM
2. Change the value in-game (lose a life so lives becomes 2)
3. `mesen_search_memory(patternHex="02", memoryType="WsWorkRam")` -- search for the new value
4. Cross-reference: addresses present in both result sets are candidates
5. `mesen_write_memory(address=candidate, hexData="09", memoryType="WsWorkRam")` -- write a test value to a candidate address. If the on-screen display updates, you found it
6. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="WsWorkRam", cpuType="Ws")` -- set a write breakpoint to watch what modifies the variable
7. `mesen_resume_execution()` -- resume and trigger the change in-game
8. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Ws")` -- check AX, BX, CX, DX, CS:IP registers to find the writing instruction
9. `mesen_disassemble(address=IP-8, lineCount=20, cpuType="Ws")` -- view surrounding code; look for MOV [addr],AL or MOV [BX],value instructions targeting your candidate address
10. `mesen_freeze_address(startAddress=candidate, endAddress=candidate, cpuType="Ws", freeze=true)` -- freeze the value for infinite lives

**Tip:** V30MZ values can be 8-bit or 16-bit (little-endian). On mono WonderSwan, game variables are usually above $4000 in the work RAM (below that is VRAM). On WSC with 64KB RAM, the VRAM fills most of the space, so game variables may be in a smaller region. Scores and large numbers may be stored as BCD -- the V30MZ has DAA (decimal adjust) for BCD arithmetic.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The V30MZ is 8086-compatible with registers AX (AH/AL), BX (BH/BL), CX (CH/CL), DX (DH/DL), SI, DI, BP, SP, and segment registers CS, DS, ES, SS. Flags: O, D, I, T, S, Z, A, P, C. Uses segmented addressing: physical address = (segment << 4) + offset.

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$0100", lineCount=50, cpuType="Ws")` -- view code at the target address
3. `mesen_breakpoint(action="set", address="$0100", type="Execute", memoryType="WsPrgRom", cpuType="Ws")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Ws")` -- check register values (AX, BX, CX, DX, SI, DI, CS:IP, flags) at the breakpoint
6. `mesen_step(cpuType="Ws", stepType="Step")` -- single step one instruction; check state after each step
7. `mesen_step(cpuType="Ws", stepType="StepOver")` -- step over CALL instructions to skip subroutines
8. `mesen_get_execution_trace(count=200, cpuType="Ws")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Ws")` -- see the call chain that led to this routine
10. `mesen_label(action="set", address="$0100", memoryType="WsPrgRom", label="MainLoop")` -- annotate the routine with a descriptive label

**Tip:** V30MZ instructions are variable-length (1-6 bytes). CALL (3 bytes near, 5 bytes far) calls a subroutine, RET returns. MOV is the primary data transfer instruction with many addressing modes. CMP compares values, then JE/JNE/JA/JB (2-byte short or 4-byte near) branch on result. PUSH/POP save and restore registers. IN/OUT access hardware ports -- critical for VDP, sound, and system control. Common patterns: MOV SI,addr / LODSB for table reads; REP MOVSB for block copy; REP OUTSB for port I/O blocks. Segment overrides (ES:, CS:) specify which segment register to use for memory access.

## Workflow D: Analyze VBlank

Use this to understand the main per-frame update routine. The WonderSwan uses x86-style interrupt vectors. The VBlank IRQ vector number is determined by the base set in port $B0 plus offset 6 (VBlank is bit 6). The interrupt vector table (IVT) is at address 0000:0000, with each entry being 4 bytes (offset + segment).

1. First, read the IRQ vector base from port $B0. If not easily accessible, assume a common base (often $08, making VBlank vector = $08 + 6 = $0E, so IVT entry at physical address $0E * 4 = $38)
2. `mesen_read_memory(address="$0038", length=4, memoryType="WsWorkRam")` -- read the VBlank IVT entry (4 bytes: offset low, offset high, segment low, segment high). For example, bytes $00 $20 $00 $80 means the handler is at 8000:2000 (physical $82000)
3. Calculate the handler address: segment:offset from the 4-byte IVT entry
4. `mesen_disassemble(address=handlerAddress, lineCount=60, cpuType="Ws")` -- view the VBlank routine
5. `mesen_breakpoint(action="set", address=handlerAddress, type="Execute", memoryType="WsPrgRom", cpuType="Ws")` -- this breakpoint will hit every frame during VBlank
6. `mesen_playback(action="resume")` -- resume and let it hit once
7. `mesen_get_execution_trace(count=500, cpuType="Ws")` -- view the full VBlank execution flow
8. Look for these key operations in the trace:
   - **IRQ acknowledge**: OUT $B6,AL with AL=$40 (bit 6 = VBlank) to acknowledge the interrupt
   - **Display control**: OUT to ports $00-$3F for PPU register updates (scroll, BG/sprite enable)
   - **Scroll updates**: OUT to ports $10-$13 for BG0/BG1 scroll X/Y
   - **Sprite table**: writes to the sprite table address set via port $04
   - **Palette updates**: writes to VRAM offset $FE00 (WSC color palettes) or OUT to ports $1C-$3F (mono palettes)
   - **Sound updates**: OUT to ports $80-$9F for wavetable channel control
   - **Controller read**: IN from controller ports
   - **Frame counter**: MOV [addr],value to a RAM address that increments each frame
9. `mesen_label(action="set", address=handlerAddress, memoryType="WsPrgRom", label="VBlank_Handler")` -- label the VBlank entry point

**Tip:** The VBlank handler must acknowledge by writing $40 to port $B6 -- VBlank is level-triggered, so it stays asserted until acknowledged. IVT lives at $0000-$03FF in work RAM (256 vectors × 4 bytes). Port $B2 enables IRQs, $B4 reads status, $B6 acknowledges. ROM banking: port $C0 (main bank $40000-$FFFFF), $C1 (SRAM bank $10000-$1FFFF), $C2-$C3 (ROM banks at $20000/$30000).

**Display hardware:** 224x144, two BG layers (Screen 1/Screen 2) + 128 sprites (32/scanline). Tilemap entries are 2 bytes: bits 0-8=tile index (9-bit), bits 9-12=palette, bit 13=tile bank (WSC), bit 14=H flip, bit 15=V flip. Sprite entries are 4 bytes: byte 0=tile index low, byte 1=attributes (bit 0=tile bit 8, bits 1-3=palette 8-15, bit 5=priority, bit 6=H flip, bit 7=V flip), byte 2=Y, byte 3=X. Sprite table location set via port $04, tilemap addresses via port $07 (low nibble=Screen 1, high nibble=Screen 2). Color palettes at VRAM $FE00 (WSC, 16 palettes × 4 colors × 2 bytes, 12-bit RGB). Mono palettes via ports $1C-$3F (8 shades). Sound: 4 wavetable channels, samples stored in RAM at (port $8F << 6), each channel uses 16 bytes of 32 × 4-bit samples.

## Workflow E: Patch ROM

Use this to modify WonderSwan game code and save the result. The V30MZ NOP opcode is $90 (1 byte).

1. `mesen_disassemble(address="$0200", lineCount=10, cpuType="Ws")` -- view current code at the patch target
2. Determine what to patch:
   - **NOP out a short conditional jump** (JE, JNE, JB, etc.): 2 bytes, replace with `NOP / NOP`
   - **NOP out a near CALL** (CALL $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out a near jump** (JMP $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **Force a branch**: change JNE to JE (single byte change: $75 to $74)
   - **Change an immediate value**: MOV AL,$03 -> MOV AL,$09 (change the operand byte)
3. `mesen_assemble(code="NOP\nNOP\nNOP", startAddress="$0200", cpuType="Ws")` -- assemble the patch over the existing code
4. `mesen_disassemble(address="$0200", lineCount=10, cpuType="Ws")` -- verify the patch was applied correctly
5. `mesen_playback(action="resume")` -- test the patch in-game
6. `mesen_take_screenshot()` -- capture the result to verify behavior
7. `mesen_save_modified_rom(filepath="C:/patched_game.ws")` -- save the full patched ROM
8. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file instead

**Tip:** V30MZ instruction sizes vary: 1-byte (NOP, PUSH/POP reg, RET, IRET), 2-byte (short Jcc, MOV reg,imm8, IN/OUT with immediate port), 3-byte (near CALL, near JMP, MOV reg,imm16), and longer for memory-referencing instructions with ModR/M bytes and displacements. When NOPing multi-byte instructions, fill ALL bytes with $90. Be aware of segment addressing -- the physical ROM offset depends on the CS segment value. Execution starts at FFFF:0000 (physical $FFFF0), which is near the end of the ROM.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing menus, game sequences, or verifying patches. WonderSwan buttons: Up, Down, Left, Right, A, B, Start.

1. `mesen_load_rom(filepath="C:/roms/game.ws")` -- load the WonderSwan game
2. `mesen_playback(action="pause")` -- pause to set up the automation
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Start on the WonderSwan
4. `mesen_step(cpuType="Ws", stepType="PpuFrame")` -- advance one frame with Start held
5. `mesen_step(cpuType="Ws", stepType="PpuFrame")` -- advance another frame (some games need multiple frames to register a press)
6. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
7. `mesen_step(cpuType="Ws", stepType="PpuFrame")` -- advance a frame with no buttons held
8. `mesen_input_override(action="set", port=0, buttons="A")` -- press A to confirm a menu selection
9. `mesen_step(cpuType="Ws", stepType="PpuFrame")` -- advance the frame
10. `mesen_take_screenshot()` -- capture the screen to verify the result

**Tip:** Hold a direction for multiple frames by repeating `mesen_step` without changing the input override. To press two buttons simultaneously, combine them: `buttons="A,Right"`. Use `mesen_save_state(action="save")` before a sequence and `mesen_save_state(action="load")` to retry. The WonderSwan has two sets of directional buttons (X1-X4 and Y1-Y4) since the console can be held vertically or horizontally. In horizontal mode, Y buttons are the D-pad and X buttons are face buttons. Some games use both sets for different purposes.
