---
description: "SMS / Game Gear reverse engineering workflows for Mesen2: find text, find variables, analyze routines, VBlank/IM1 analysis, patch ROMs, automate input. SMS-specific memory types, vectors, registers, and instruction patterns."
---

# SMS / Game Gear Reverse Engineering Workflows

Quick reference: cpuType=`Sms`, ROM=`SmsPrgRom`, RAM=`SmsWorkRam`, VRAM=`SmsVideoRam`. CPU is Z80. NOP=$00. RAM at $C000-$DFFF (8KB, mirrored to $E000-$FFFF). Sega mapper bank registers at $FFFC-$FFFF.

## Workflow A: Find Game Text (Relative Search + TBL)

Use this when you can see text on screen but do not know the ROM's character encoding. SMS/GG games typically use custom tile-based encodings where letters map to VRAM tile indices.

1. `mesen_relative_search(searchText="PLAYER", memoryType="SmsPrgRom")` -- search by byte differences between consecutive characters; works even with unknown custom encodings
2. For each match, `mesen_read_memory(address=matchAddress-4, length=30, memoryType="SmsPrgRom")` -- read surrounding bytes to verify the match looks like text (similar byte ranges, consistent patterns)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value corresponds to the first character in your search text. If 'P'=$30, then 'Q'=$31, 'R'=$32, etc.
4. `mesen_load_tbl(tblPathOrContent="30=P\n31=Q\n32=R\n33=S\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address=matchAddress, length=100, memoryType="SmsPrgRom")` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="SCORE", memoryType="SmsPrgRom")` -- find other text strings using the TBL encoding

**Tip:** Use UPPERCASE -- many SMS/GG games only store uppercase letters. Search strings of 5+ characters reduce false positives. SMS ROMs are typically 256KB-512KB, GG ROMs up to 1MB. Some games store text as tilemap entries (2 bytes per character -- tile index + attribute), which breaks relative search on single bytes. If no results, try searching with a stride or look for text in the VRAM tilemap directly.

## Workflow B: Find a Game Variable (Cheat Search)

Use this when you can see a value on screen (lives=3, HP=100) but do not know its RAM address. SMS has 8KB of work RAM at $C000-$DFFF, mirrored at $E000-$FFFF.

1. `mesen_search_memory(patternHex="03", memoryType="SmsWorkRam")` -- search for the byte value 3 in work RAM (8KB)
2. Change the value in-game (lose a life so lives becomes 2)
3. `mesen_search_memory(patternHex="02", memoryType="SmsWorkRam")` -- search for the new value
4. Cross-reference: addresses present in both result sets are candidates. With 8KB of RAM, you should narrow down quickly
5. `mesen_write_memory(address=candidate, hexData="09", memoryType="SmsWorkRam")` -- write a test value to a candidate address. If the on-screen display updates, you found it
6. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="SmsWorkRam", cpuType="Sms")` -- set a write breakpoint to watch what modifies the variable
7. `mesen_resume_execution()` -- resume and trigger the change in-game
8. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Sms")` -- check A, BC, DE, HL, PC registers to find the writing instruction
9. `mesen_disassemble(address=PC-6, lineCount=20, cpuType="Sms")` -- view surrounding code; look for LD (addr),A or LD (HL),value instructions targeting your candidate address
10. `mesen_freeze_address(startAddress=candidate, endAddress=candidate, cpuType="Sms", freeze=true)` -- freeze the value for infinite lives

**Tip:** Z80 values are usually 8-bit (0-255). Scores and large numbers may be stored as BCD -- use DAA instruction to adjust after addition. RAM addresses in the CPU space are $C000-$DFFF, but SmsWorkRam offsets start from 0. The mirror at $E000-$FFFF means writing to $E000 is the same as writing to $C000. Bank registers at $FFFC-$FFFF are mapped to the last 4 bytes of RAM (SmsWorkRam offsets $1FFC-$1FFF).

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The Z80 has registers A (accumulator), BC, DE, HL (general-purpose pairs), SP (stack pointer), PC (program counter), IX/IY (index registers), and flags (S, Z, H, P/V, N, C). It also has shadow registers A', BC', DE', HL' swapped via EXX/EX AF,AF'.

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$0150", lineCount=50, cpuType="Sms")` -- view code at the target address
3. `mesen_breakpoint(action="set", address="$0150", type="Execute", memoryType="SmsPrgRom", cpuType="Sms")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Sms")` -- check register values (A, BC, DE, HL, SP, PC, flags) at the breakpoint
6. `mesen_step(cpuType="Sms", stepType="Step")` -- single step one instruction; check state after each step
7. `mesen_step(cpuType="Sms", stepType="StepOver")` -- step over CALL instructions to skip subroutines
8. `mesen_get_execution_trace(count=200, cpuType="Sms")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Sms")` -- see the call chain that led to this routine
10. `mesen_label(action="set", address="$0150", memoryType="SmsPrgRom", label="MainLoop")` -- annotate the routine with a descriptive label

**Tip:** Z80 instructions are 1-4 bytes. CALL (3 bytes) calls a subroutine, RET returns. LD loads/moves data (many addressing modes). CP compares A with a value, then JR Z / JR NZ (relative branch, 2 bytes) or JP Z / JP NZ (absolute jump, 3 bytes) branch on result. PUSH/POP save and restore register pairs. IN/OUT access I/O ports -- critical for VDP, controller, and mapper interaction. Common patterns: LD HL,addr / LD (HL),value for memory writes; LDIR for block copy; OTIR for block I/O output (used for VDP/VRAM writes).

## Workflow D: Analyze VBlank (IM 1 Interrupt)

Use this to understand the main per-frame update routine. The SMS uses Z80 interrupt mode 1 (IM 1), which jumps to $0038 on every maskable interrupt. The VDP generates this interrupt during VBlank. The ISR must read the VDP status port ($BF) to acknowledge the interrupt, or it will fire repeatedly.

1. `mesen_disassemble(address="$0038", lineCount=60, cpuType="Sms")` -- view the ISR at $0038 (the IM 1 fixed vector). Most games jump from here to the real handler
2. If the code at $0038 is a JP instruction, follow the jump target to the real VBlank handler
3. `mesen_breakpoint(action="set", address="$0038", type="Execute", memoryType="SmsPrgRom", cpuType="Sms")` -- this breakpoint will hit every frame during VBlank
4. `mesen_playback(action="resume")` -- resume and let it hit once
5. `mesen_get_state(component="cpu", cpuType="Sms")` -- check register state at the interrupt entry
6. `mesen_get_execution_trace(count=500, cpuType="Sms")` -- view the full VBlank execution flow
7. Look for these key operations in the trace:
   - **VDP status read**: IN A,($BF) -- this acknowledges the interrupt and returns status flags (bit 7 = VBlank flag, bit 6 = sprite overflow, bit 5 = sprite collision)
   - **VDP register writes**: two sequential OUT to port $BF (data byte, then $80|register#) to update VDP registers (scroll, display mode, etc.)
   - **VRAM writes**: set VRAM address via two OUTs to port $BF (address low, then $40|address high for write mode), then write data to port $BE
   - **Sprite table update**: OUTI/OTIR sequences writing sprite data to VRAM
   - **CRAM update**: set CRAM address via port $BF (address, then $C0), then write color bytes to port $BE
   - **Controller read**: IN A,($DC) for player 1, IN A,($DD) for player 2
   - **Frame counter**: LD (addr),A to a RAM address that increments each frame
8. `mesen_label(action="set", address="$0038", memoryType="SmsPrgRom", label="IRQ_VBlank")` -- label the interrupt entry point

**Tip:** Reading port $BF is mandatory -- it clears the interrupt flag. Most games check bit 7 of the status to confirm VBlank (vs line interrupt). Key VDP registers (set via two OUTs to $BF: data, then $80|reg#): $00=Mode Control 1 (bit 4=line IRQ enable, bit 5=left column blank), $01=Mode Control 2 (bit 5=VBlank IRQ enable, bit 6=display enable, bit 1=sprite size 8x8/8x16), $02=Name Table addr, $05=SAT addr, $06=Sprite tile base ($0000 or $2000), $07=Border color, $08=H-Scroll, $09=V-Scroll, $0A=Line counter. Mode 4 nametable entries are 2 bytes: bits 0-8=tile index (9-bit), bit 9=H flip, bit 10=V flip, bit 11=palette select, bit 12=priority. Tiles are 4bpp (32 bytes/tile). SAT in VRAM: Y coords at offset $00-$3F (64 entries, $D0=end), X/tile pairs at $80-$FF. Max 64 sprites, 8 per scanline. NMI at $0066 handles Pause button (SMS only, not GG).

## Workflow E: Patch ROM

Use this to modify SMS/GG game code and save the result. The Z80 NOP opcode is $00 (1 byte).

1. `mesen_disassemble(address="$1A00", lineCount=10, cpuType="Sms")` -- view current code at the patch target
2. Determine what to patch:
   - **NOP out a relative branch** (JR Z, JR NZ, JR C, JR NC): 2 bytes, replace with `NOP / NOP`
   - **NOP out a CALL** (CALL $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out a conditional CALL** (CALL Z, CALL NZ, etc.): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out an absolute jump** (JP $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **Force a branch**: change JR NZ to JR Z (single byte change: $20 to $28, or $C2 to $CA for JP)
   - **Change an immediate value**: LD A,$03 -> LD A,$09 (change the operand byte)
3. `mesen_assemble(code="NOP\nNOP\nNOP", startAddress="$1A00", cpuType="Sms")` -- assemble the patch over the existing code
4. `mesen_disassemble(address="$1A00", lineCount=10, cpuType="Sms")` -- verify the patch was applied correctly
5. `mesen_playback(action="resume")` -- test the patch in-game
6. `mesen_take_screenshot()` -- capture the result to verify behavior
7. `mesen_save_modified_rom(filepath="C:/patched_game.sms")` -- save the full patched ROM
8. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file instead

**Tip:** Z80 instruction sizes: 1-byte (NOP, LD r,r, INC/DEC, PUSH/POP, RET), 2-byte (JR, LD r,n, IN/OUT with immediate port, CB-prefixed bit ops), 3-byte (JP, CALL, LD rr,nn, LD (nn),A). When NOPing multi-byte instructions, fill ALL bytes with $00. Be aware of bank mapping -- most games use the **Sega mapper** (registers at $FFFC-$FFFF), but some use the **Codemasters mapper** (registers at $0000, $4000, $8000 -- writes to the start of each bank slot). To identify: check for writes to $FFFF (Sega) vs $8000 (Codemasters) in the disassembly. Use `mesen_get_address_info` to resolve banked addresses to absolute ROM offsets before patching.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing menus, game sequences, or verifying patches. SMS buttons: Up, Down, Left, Right, A (Button 1), B (Button 2). Game Gear adds Start.

1. `mesen_load_rom(filepath="C:/roms/game.sms")` -- load the SMS or Game Gear game
2. `mesen_playback(action="pause")` -- pause to set up the automation
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Start/Pause (use Start for GG; SMS pause is hardware NMI)
4. `mesen_step(cpuType="Sms", stepType="PpuFrame")` -- advance one frame with button held
5. `mesen_step(cpuType="Sms", stepType="PpuFrame")` -- advance another frame (some games need multiple frames to register a press)
6. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
7. `mesen_step(cpuType="Sms", stepType="PpuFrame")` -- advance a frame with no buttons held
8. `mesen_input_override(action="set", port=0, buttons="A")` -- press Button 1 to confirm a menu selection
9. `mesen_step(cpuType="Sms", stepType="PpuFrame")` -- advance the frame
10. `mesen_take_screenshot()` -- capture the screen to verify the result

**Tip:** Hold a direction for multiple frames by repeating `mesen_step` without changing the input override. To press two buttons simultaneously, combine them: `buttons="A,Right"`. Use `mesen_save_state` before a sequence and `mesen_load_state` to retry. SMS reads controller input via I/O port $DC (player 1) and $DD (player 2). On Game Gear, the Start button is read from port $00 bit 7. For player 2, use port=1.
