---
description: "PC Engine / TurboGrafx-16 reverse engineering workflows for MesenCE: find text, find variables, analyze routines, VBlank/IRQ1 analysis, patch ROMs, automate input. PCE-specific memory types, vectors, registers, and instruction patterns."
---

# PC Engine / TurboGrafx-16 Reverse Engineering Workflows

Quick reference: cpuType=`Pce`, ROM=`PcePrgRom`, RAM=`PceWorkRam`, VRAM=`PceVideoRam`. CPU is HuC6280 (65C02 derivative). NOP=$EA. Zero page at $2000 (not $0000). Stack at $2100. Memory mapped via 8KB banks (MPR0-MPR7, set with TAM/TMA). CD-ROM games may have additional RAM: `PceCdromRam`, `PceCardRam`, `PceAdpcmRam`, `PceArcadeCardRam`.

## Workflow A: Find Game Text (Relative Search + TBL)

Use this when you can see text on screen but do not know the ROM's character encoding. PCE games frequently use custom tile encodings where letters map to sequential VRAM tile indices.

1. `mesen_relative_search(searchText="ATTACK", memoryType="PcePrgRom")` -- search by byte differences between consecutive characters; works even with unknown custom encodings
2. For each match, `mesen_read_memory(address=matchAddress-4, length=30, memoryType="PcePrgRom")` -- read surrounding bytes to verify the match looks like text (similar byte ranges, consistent patterns)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value corresponds to the first character in your search text. If 'A'=$20, then 'B'=$21, 'C'=$22, etc.
4. `mesen_load_tbl(tblPathOrContent="20=A\n21=B\n22=C\n23=D\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address=matchAddress, length=100, memoryType="PcePrgRom")` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="MAGIC", memoryType="PcePrgRom")` -- find other text strings using the TBL encoding

**Tip:** Use UPPERCASE -- many PCE games only store uppercase letters. Search strings of 5+ characters reduce false positives. PCE ROMs can be large (up to 2.5MB for HuCards), so searches may return more candidates than NES. If no results, the game may use compressed text or DTE (one byte = two characters). Japanese HuCard games often store kana as sequential tile indices.

## Workflow B: Find a Game Variable (Cheat Search)

Use this when you can see a value on screen (lives=3, HP=100) but do not know its RAM address. Standard PCE work RAM is 8KB. CD-ROM games may have additional RAM.

1. `mesen_search_memory(patternHex="03", memoryType="PceWorkRam")` -- search for the byte value 3 in work RAM (8KB)
2. Change the value in-game (lose a life so lives becomes 2)
3. `mesen_search_memory(patternHex="02", memoryType="PceWorkRam")` -- search for the new value
4. Cross-reference: addresses present in both result sets are candidates. With 8KB of RAM, you should narrow down quickly
5. `mesen_write_memory(address=candidate, hexData="09", memoryType="PceWorkRam")` -- write a test value to a candidate address. If the on-screen display updates, you found it
6. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="PceWorkRam", cpuType="Pce")` -- set a write breakpoint to watch what modifies the variable
7. `mesen_resume_execution()` -- resume and trigger the change in-game
8. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Pce")` -- check A, X, Y, PC registers to find the writing instruction
9. `mesen_disassemble(address=PC-8, lineCount=20, cpuType="Pce")` -- view surrounding code; look for STA (store accumulator) instructions targeting your candidate address
10. `mesen_freeze_address(startAddress=candidate, endAddress=candidate, cpuType="Pce", freeze=true)` -- freeze the value for infinite lives

**Tip:** PCE values are usually 8-bit (0-255). Scores and large numbers are often stored as BCD -- the HuC6280 has a decimal mode flag (SED/CLD). Zero page on the PCE is at $2000-$20FF (not $0000 like standard 6502), so frequently-used variables will be at those addresses. The stack is at $2100-$21FF.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The HuC6280 is a 65C02 derivative with registers A (accumulator), X and Y (index), SP (stack pointer), PC (program counter), and P (processor flags: N, V, T, B, D, I, Z, C). It adds block transfer instructions (TAI, TIA, TIN, TDD, TII), bit manipulation (BBR/BBS, RMB/SMB), and special VDC I/O instructions (ST0, ST1, ST2).

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$E000", lineCount=50, cpuType="Pce")` -- view code at the target address
3. `mesen_breakpoint(action="set", address="$E000", type="Execute", memoryType="PcePrgRom", cpuType="Pce")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Pce")` -- check register values (A, X, Y, SP, PC, flags) at the breakpoint
6. `mesen_step(cpuType="Pce", stepType="Step")` -- single step one instruction; check state after each step
7. `mesen_step(cpuType="Pce", stepType="StepOver")` -- step over JSR calls to skip subroutines
8. `mesen_get_execution_trace(count=200, cpuType="Pce")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Pce")` -- see the call chain that led to this routine
10. `mesen_label(action="set", address="$E000", memoryType="PcePrgRom", label="MainLoop")` -- annotate the routine with a descriptive label

**Tip:** HuC6280 instructions are 1-3 bytes, same as 65C02. JSR (3 bytes) calls a subroutine, BSR (3 bytes) is a PC-relative call unique to the HuC6280, RTS returns. ST0/ST1/ST2 are special instructions for writing to VDC registers (ST0 sets the VDC register index, ST1 writes the low byte, ST2 writes the high byte). TAM/TMA handle bank mapping -- TAM writes A to an MPR, TMA reads an MPR into A. Block transfer instructions (TII, TDD, TIN, TIA, TAI) copy memory in hardware and are commonly used for VRAM updates.

## Workflow D: Analyze VBlank (IRQ1)

Use this to understand the main per-frame update routine. On the PCE, VBlank fires IRQ1 from the VDC (Video Display Controller). The ISR must read the VDC status register to acknowledge the interrupt, or it will fire repeatedly.

1. `mesen_read_memory(address="$FFF8", length=2, memoryType="PcePrgRom")` -- read the IRQ1 vector (2 bytes, little-endian). For example, bytes $40 $E0 means the IRQ1 handler is at $E040
2. Calculate the handler address from the two bytes (low byte first, high byte second)
3. `mesen_disassemble(address="$E040", lineCount=60, cpuType="Pce")` -- view the IRQ1 routine
4. `mesen_breakpoint(action="set", address="$E040", type="Execute", memoryType="PcePrgRom", cpuType="Pce")` -- this breakpoint will hit every frame during VBlank
5. `mesen_playback(action="resume")` -- resume and let it hit once
6. `mesen_get_execution_trace(count=500, cpuType="Pce")` -- view the full IRQ1 execution flow
7. Look for these key operations in the trace:
   - **VDC status read**: ST0 #$00 followed by a read from port $0000 (or LDA $0000), which acknowledges the interrupt and clears the IRQ
   - **VDC register writes**: ST0/ST1/ST2 sequences to update scroll, display control, or VRAM addresses
   - **SAT DMA**: VDC register $13 write to trigger sprite attribute table DMA from VRAM
   - **Palette updates**: writes to VCE ports at $0400-$0405
   - **Controller read**: reads from I/O port $1000
   - **Frame counter**: STA to a work RAM address that increments each frame
8. `mesen_label(action="set", address="$E040", memoryType="PcePrgRom", label="IRQ1_VBlank")` -- label the IRQ1 entry point

**Tip:** PCE has three interrupt levels: Timer (highest), IRQ1/VDC (VBlank, HBlank), and IRQ2 (CD-ROM, BRK). The IRQ mask register at I/O $1402 controls which are enabled. The handler checks VDC status bits to distinguish VBlank (bit 5) from scanline IRQ (bit 2). Key VDC registers (selected via ST0): $00=MAWR (VRAM write addr), $01=MARR (VRAM read addr), $02=VWR (VRAM data write), $05=CR (control: bits 0-3 enable IRQs, bits 6-7 enable BG/sprites), $06=RCR (scanline compare), $07/$08=BXR/BYR (scroll X/Y), $09=MWR (tilemap size: 32x32/64x32/128x32/32x64/64x64), $13=DVSSR (SAT DMA source). SAT entries are 4 words (8 bytes): Y position (offset -64), X position (offset -32), tile index (10-bit), flags (palette 0-15, H/V flip, priority, size: 16x16/16x32/16x64/32x16/32x32/32x64). Max 64 sprites, 16 per scanline. VCE palette at I/O $0400: 512 colors (9-bit RGB), entries 0-255 for BG, 256-511 for sprites.

## Workflow E: Patch ROM

Use this to modify PCE game code and save the result. The HuC6280 NOP opcode is $EA (1 byte), same as 6502/65C02.

1. `mesen_disassemble(address="$C456", lineCount=10, cpuType="Pce")` -- view current code at the patch target
2. Determine what to patch:
   - **NOP out a conditional branch** (BEQ, BNE, BCS, BCC, etc.): 2 bytes, replace with `NOP / NOP`
   - **NOP out a subroutine call** (JSR $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out a BSR** (relative subroutine call): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out an absolute store** (STA $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **Force a branch**: change BNE to BEQ or vice versa (single byte change at the opcode)
   - **Change an immediate value**: LDA #$03 -> LDA #$09 (change the operand byte)
3. `mesen_assemble(code="NOP\nNOP\nNOP", startAddress="$C456", cpuType="Pce")` -- assemble the patch over the existing code
4. `mesen_disassemble(address="$C456", lineCount=10, cpuType="Pce")` -- verify the patch was applied correctly
5. `mesen_playback(action="resume")` -- test the patch in-game
6. `mesen_take_screenshot()` -- capture the result to verify behavior
7. `mesen_save_modified_rom(filepath="C:/patched_game.pce")` -- save the full patched ROM
8. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file instead

**Tip:** HuC6280 instruction sizes follow 65C02 rules: implied/accumulator = 1 byte, immediate/zero-page/relative = 2 bytes, absolute = 3 bytes. BSR (branch to subroutine) is 3 bytes. When NOPing multi-byte instructions, fill ALL bytes with $EA. PCE uses a unique banking system: 8 MPR registers (MPR0-MPR7) map 8KB physical banks ($00-$FF) to the 64KB logical address space. MPR7 starts mapped to the last ROM bank (contains reset/IRQ vectors). To find bank-switch code, search for `TAM` instructions (writes A to selected MPRs). Use `mesen_get_address_info` to resolve CPU addresses to absolute PcePrgRom offsets. Physical banks: $00-$7F=HuCard ROM, $F7=Save RAM, $F8-$FB=Work RAM, $FF=I/O registers.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing menus, game sequences, or verifying patches. PCE controller buttons: Up, Down, Left, Right, A (I), B (II), Select, Start (Run).

1. `mesen_load_rom(filepath="C:/roms/game.pce")` -- load the PCE game
2. `mesen_playback(action="pause")` -- pause to set up the automation
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Run on controller 1
4. `mesen_step(cpuType="Pce", stepType="PpuFrame")` -- advance one frame with Run held
5. `mesen_step(cpuType="Pce", stepType="PpuFrame")` -- advance another frame (some games need multiple frames to register a press)
6. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
7. `mesen_step(cpuType="Pce", stepType="PpuFrame")` -- advance a frame with no buttons held
8. `mesen_input_override(action="set", port=0, buttons="A")` -- press button I to confirm a menu selection
9. `mesen_step(cpuType="Pce", stepType="PpuFrame")` -- advance the frame
10. `mesen_take_screenshot()` -- capture the screen to verify the result

**Tip:** Hold a direction for multiple frames by repeating `mesen_step` without changing the input override. To press two buttons simultaneously, combine them: `buttons="A,Right"`. Use `mesen_save_state` before a sequence and `mesen_load_state` to retry. PCE games read the controller via I/O port $1000. The 6-button controller (Avenue Pad 6) adds Run, Select, III, IV, V, VI buttons in a second scan. For TurboTap multitap games, use port 0-4.
