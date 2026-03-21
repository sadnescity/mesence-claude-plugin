---
description: "SNES reverse engineering workflows for Mesen2: find text, find variables, analyze routines, VBlank/NMI analysis, patch ROMs, automate input, SPC700 audio analysis, DMA/HDMA graphics analysis, BG mode and tilemap inspection. SNES-specific memory types, vectors, registers, and instruction patterns."
---

# SNES Reverse Engineering Workflows

Quick reference: cpuType=`Snes` (main 65816), `Spc` (SPC700 audio), `Sa1` (SA-1 coprocessor). Main memory=`SnesWorkRam`, ROM=`SnesPrgRom`, VRAM=`SnesVideoRam`. CPU is 65816 (24-bit addressing). NOP=$EA.

## Workflow A: Find Game Text (Relative Search + TBL)

Use this when you can see text on screen but do not know the ROM's character encoding. SNES games -- especially RPGs -- often use custom encodings, and Japanese games may use Shift-JIS or custom 16-bit tile-based encodings. Many SNES RPGs also use DTE (Dual Tile Encoding) or MTE (Multi Tile Encoding) compression. Relative search works well for alphabetic text with sequential encoding.

1. `mesen_relative_search(searchText="POTION", memoryType="SnesPrgRom")` -- search by byte differences between consecutive characters; works even with unknown custom encodings
2. For each match, `mesen_read_memory(address=matchAddress-4, length=40, memoryType="SnesPrgRom")` -- read surrounding bytes to verify the match looks like text (similar byte ranges, consistent patterns)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value corresponds to the first character. If 'P'=0x8F, then 'Q'=0x90, 'R'=0x91, etc.
4. `mesen_load_tbl(tblPathOrContent="8F=P\n90=Q\n91=R\n92=S\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address=matchAddress, length=200, memoryType="SnesPrgRom")` -- read decoded text from the ROM using the loaded TBL; SNES games often have longer text strings than NES
6. `mesen_search_text(text="SHIELD", memoryType="SnesPrgRom")` -- find other text strings using the TBL encoding

**Tip:** SNES games often have both uppercase and lowercase letters, so try mixed case if uppercase-only fails. If relative search returns no results, the game may use DTE/MTE compression (one byte = multiple characters) or dictionary-based compression common in RPGs. For Japanese text, encodings are typically 16-bit and not sequential, so relative search will not work -- you will need to find the font table in VRAM instead. Search strings of 5+ characters reduce false positives.

## Workflow B: Find a Game Variable (Cheat Search)

Use this when you can see a value on screen (HP=500, Gil=1200) but do not know its RAM address. SNES has 128KB of work RAM ($7E0000-$7FFFFF), with the first 8KB mapped to $0000-$1FFF in bank $00. More RAM means more candidates, so you may need extra search-and-narrow rounds.

1. `mesen_search_memory(patternHex="F401", memoryType="SnesWorkRam")` -- search for 500 as a 16-bit little-endian value ($01F4) in work RAM
2. Change the value in-game (take damage so HP becomes 480)
3. `mesen_search_memory(patternHex="E001", memoryType="SnesWorkRam")` -- search for 480 ($01E0) in little-endian
4. Cross-reference: addresses present in both result sets are candidates. With 128KB of RAM you may need a third round to narrow further
5. `mesen_write_memory(address=candidate, hexData="E803", memoryType="SnesWorkRam")` -- write 1000 ($03E8) as a test value. If the on-screen HP updates, you found it
6. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="SnesWorkRam", cpuType="Snes")` -- set a write breakpoint to watch what modifies the variable
7. `mesen_resume_execution()` -- resume and trigger the change in-game
8. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Snes")` -- check registers (A, X, Y, D, DBR, K, PC) to find the writing instruction
9. `mesen_disassemble(address=PC-10, lineCount=25, cpuType="Snes")` -- view surrounding code; look for STA (store accumulator) instructions. Pay attention to M/X flag state -- if M=0, STA stores 16 bits
10. `mesen_freeze_address(startAddress=candidate, endAddress=candidate+1, cpuType="Snes", freeze=true)` -- freeze both bytes of the 16-bit value for infinite HP

**Tip:** SNES values are often 16-bit (little-endian). Always search for both 8-bit and 16-bit representations. The direct page register (D) shifts zero-page addressing -- if D=$1A00, then `STA $42` actually writes to $1A42. Variables in bank $7E beyond $1FFF are accessed via long addressing (STA $7E2000) or by setting DBR to $7E. If SnesWorkRam yields nothing, try `SnesSaveRam` for games with battery backup.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The SNES main CPU (65816) has registers: A (accumulator, 8 or 16-bit), X/Y (index, 8 or 16-bit), SP (stack pointer), D (direct page), DBR (data bank register), K (program bank), PC (program counter), and PS (processor status with flags: N, V, M, X, D, I, Z, C). The M flag controls A width and the X flag controls X/Y width.

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$00:8000", lineCount=50, cpuType="Snes")` -- view code at the target address (use $BB:AAAA 24-bit format)
3. `mesen_breakpoint(action="set", address="$008000", type="Execute", memoryType="SnesPrgRom", cpuType="Snes")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Snes")` -- check register values; pay special attention to M and X flags to know if A/X/Y are 8-bit or 16-bit
6. `mesen_step(cpuType="Snes", stepType="Step")` -- single step one instruction; check state after each step
7. `mesen_step(cpuType="Snes", stepType="StepOver")` -- step over JSR and JSL calls to skip subroutines
8. `mesen_get_execution_trace(count=200, cpuType="Snes")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Snes")` -- see the call chain; JSR stays in-bank, JSL crosses banks
10. `mesen_label(action="set", address="$008000", memoryType="SnesPrgRom", label="GameLoop")` -- annotate the routine with a descriptive label

**Tip:** 65816 instructions are 1-4 bytes. Instruction size for immediate addressing depends on M/X flags -- LDA #$xx is 2 bytes when M=1 (8-bit A) but 3 bytes when M=0 (16-bit A). REP #$20 clears M flag (A becomes 16-bit), SEP #$20 sets it (A becomes 8-bit). REP #$10 / SEP #$10 do the same for X/Y. JSR (3 bytes) calls within the current bank, JSL (4 bytes) calls across banks. PHB/PLB save/restore the data bank. Common pattern: PHK / PLB sets DBR to the current code bank for data access.

## Workflow D: Analyze VBlank/NMI

Use this to understand the main per-frame update routine. On the SNES, the NMI fires every frame during VBlank when enabled via register $4200 bit 7. VBlank is the primary time to perform DMA transfers to VRAM, OAM, and CGRAM.

1. `mesen_read_memory(address="$FFEA", length=2, memoryType="SnesPrgRom")` -- read the NMI vector (2 bytes, little-endian) from the native mode vector table. For example, bytes $00 $81 means the NMI handler is at $00:8100
2. Calculate the handler address: the vector provides the 16-bit PC within bank $00
3. `mesen_disassemble(address="$00:8100", lineCount=80, cpuType="Snes")` -- view the NMI routine; SNES NMI handlers are often longer than NES due to DMA setup
4. `mesen_breakpoint(action="set", address="$008100", type="Execute", memoryType="SnesPrgRom", cpuType="Snes")` -- this breakpoint will hit every frame (~60 times/second for NTSC)
5. `mesen_playback(action="resume")` -- resume and let it hit once
6. `mesen_get_execution_trace(count=800, cpuType="Snes")` -- view the full NMI execution flow; SNES NMI is typically more complex than NES
7. Look for these key operations in the trace:
   - **DMA to VRAM**: writes to $4300-$437F (DMA channel registers) followed by $420B (DMA enable). Registers $4301-$4302 set the source address, $2118 is the VRAM data port
   - **OAM update**: DMA transfer targeting $2104 (OAM data write)
   - **CGRAM update**: DMA transfer targeting $2122 (palette data write)
   - **Scroll registers**: writes to $210D-$2114 (BG1-BG4 horizontal/vertical scroll)
   - **HDMA setup**: writes to $420C (HDMA enable) and $43x0-$43xA (HDMA channel config)
   - **SPC communication**: reads/writes to $2140-$2143 (APU I/O ports) for music and sound effect commands
   - **Frame counter**: STA to a RAM address that increments each frame
8. `mesen_label(action="set", address="$008100", memoryType="SnesPrgRom", label="NMI_Handler")` -- label the NMI entry point

**Tip:** SNES NMI is more complex than NES because of DMA. Instead of manually copying bytes, the SNES sets up DMA channel registers ($4300-$437F: source address, destination register, transfer mode, byte count) and triggers the transfer with a single write to $420B. A typical NMI sets up 2-4 DMA channels to update VRAM tiles, tilemaps, OAM, and CGRAM in rapid succession. HDMA (configured via $420C) runs automatically during each scanline and is used for gradient effects, window shaping, and mode 7 parameter changes. The NMI handler usually starts by reading $4210 to acknowledge the interrupt.

## Workflow E: Patch ROM

Use this to modify SNES game code and save the result. The 65816 NOP opcode is $EA (1 byte), same as 6502.

1. `mesen_disassemble(address="$00:A350", lineCount=15, cpuType="Snes")` -- view current code at the patch target
2. Determine what to patch:
   - **NOP out a conditional branch** (BEQ, BNE, BCS, BCC, BRA): 2 bytes, replace with `NOP / NOP`
   - **NOP out a short subroutine call** (JSR $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out a long subroutine call** (JSL $xxxxxx): 4 bytes, replace with `NOP / NOP / NOP / NOP`
   - **Force a branch**: change BNE ($D0) to BEQ ($F0) or vice versa (single byte opcode change)
   - **Change an immediate value**: LDA #$03 -> LDA #$09 (change the operand byte; note the operand is 1 byte when M=1, 2 bytes when M=0)
3. `mesen_assemble(code="NOP\nNOP\nNOP\nNOP", startAddress="$00:A350", cpuType="Snes")` -- assemble the patch (4 NOPs for a JSL)
4. `mesen_disassemble(address="$00:A350", lineCount=15, cpuType="Snes")` -- verify the patch was applied correctly and subsequent instructions are not misaligned
5. `mesen_playback(action="resume")` -- test the patch in-game
6. `mesen_take_screenshot()` -- capture the result to verify behavior
7. `mesen_save_modified_rom(filepath="C:/patched_game.sfc")` -- save the full patched ROM
8. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file instead

**Tip:** 65816 instruction sizes vary: implied = 1 byte, immediate = 2 or 3 bytes (depending on M/X flags), direct page = 2 bytes, absolute = 3 bytes, long = 4 bytes. When NOPing, fill ALL bytes with $EA. Be careful with bank boundaries -- code at the end of one bank does not flow into the next bank on all mapper configurations. For patching SPC700 audio code, use cpuType="Spc" (SPC NOP is also $00, not $EA). Common SNES patches: disable damage by NOPing the STA that writes HP, skip cutscenes by NOPing JSL calls to the cutscene engine, or modify item quantities by changing immediate operands.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing menus, game sequences, or verifying patches. SNES controller buttons: A, B, X, Y, L, R, Up, Down, Left, Right, Select, Start.

1. `mesen_load_rom(filepath="C:/roms/game.sfc")` -- load the SNES game
2. `mesen_playback(action="pause")` -- pause to set up the automation
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Start on controller 1
4. `mesen_step(cpuType="Snes", stepType="PpuFrame")` -- advance one frame with Start held
5. `mesen_step(cpuType="Snes", stepType="PpuFrame")` -- advance another frame (some menus need multiple frames to register input)
6. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
7. `mesen_step(cpuType="Snes", stepType="PpuFrame")` -- advance a frame with no buttons held (prevents repeat detection)
8. `mesen_input_override(action="set", port=0, buttons="A")` -- press A to confirm a menu selection
9. `mesen_step(cpuType="Snes", stepType="PpuFrame")` -- advance the frame
10. `mesen_take_screenshot()` -- capture the screen to verify the result

**Tip:** The SNES controller has more buttons than the NES (X, Y, L, R in addition to A, B). To press multiple buttons simultaneously, combine them: `buttons="Y,Right"` (many SNES games use Y for run + direction for movement). Use `mesen_save_state` before an input sequence and `mesen_load_state` to retry. For player 2, use port=1. SNES games read controllers via the auto-joypad feature ($4200 bit 0) which reads during VBlank, or manually via $4016/$4017 -- either way, one frame of held input is sufficient for most games.

## Workflow G: SPC700 Audio Analysis

Use this to understand the sound engine and SPC700 coprocessor communication. The SPC700 runs independently at 1.024 MHz with 64KB RAM. The main CPU communicates via four 8-bit ports ($2140-$2143 on the SNES side, $F4-$F7 on the SPC side).

1. `mesen_set_trace_options(cpuType="Spc", enabled=true)` -- enable tracing for the SPC700
2. `mesen_get_state(component="cpu", cpuType="Spc")` -- inspect SPC registers (A, X, Y, SP, PC, PSW)
3. `mesen_disassemble(address="$0200", lineCount=100, cpuType="Spc")` -- view SPC code. The SPC driver typically lives at $0200-$3FFF after being uploaded via the IPL boot protocol
4. `mesen_breakpoint(action="set", address="$00F4", type="Write", memoryType="SpcRam", cpuType="Spc")` -- watch SPC writes to communication port $F4 (responses to the main CPU)
5. `mesen_resume_execution()` -- resume and trigger a sound effect in-game
6. `mesen_get_execution_trace(count=500, cpuType="Spc")` -- view the SPC execution around the port write
7. On the main CPU side, `mesen_find_occurrences(searchString="STA $2140", cpuType="Snes")` -- find where the main CPU sends commands to the SPC
8. `mesen_breakpoint(action="set", address="$2140", type="Write", memoryType="SnesRegister", cpuType="Snes")` -- watch main CPU writes to SPC port 0 to capture the command protocol
9. `mesen_read_memory(address="$0000", length=256, memoryType="SpcRam")` -- read SPC zero page to find driver variables, sample pointers, and state

**Tip:** The SPC700 uses a different instruction set from the 65816 -- it's closer to 6502 but with unique features (MOV for loads/stores, CBNE/DBNZ for loops, SET1/CLR1 for bit manipulation). The DSP is accessed via register pair $F2/$F3 (address/data). Key DSP registers: $4C (KON, key on voices), $5C (KOF, key off), $6C (FLG, flags/noise clock), $7D (EDL, echo delay). Sound samples use BRR compression (4-bit ADPCM, 9 bytes = 16 samples). The sample directory at DIR*$100 contains start/loop address pairs for each sample.

## Workflow H: DMA & HDMA Graphics Analysis

Use this to understand how the game transfers graphics data and creates per-scanline effects. SNES DMA is critical for VRAM/OAM/CGRAM updates. HDMA runs automatically during HBlank for effects like gradients, parallax scrolling, and mode 7 parameter changes.

### Analyzing DMA Transfers (VBlank)

1. Break in the NMI handler (see Workflow D) and step through to find DMA setup code
2. Look for writes to the DMA channel registers. Key registers per channel (n=0-7):
   - `$43n0` (DMAPn): transfer direction and pattern. Patterns: 0=1 byte, 1=2 bytes alternating low/high (VRAM), 2=2 bytes same register (OAM/CGRAM)
   - `$43n1` (BBADn): B-bus destination register ($18=VRAM data, $04=OAM data, $22=CGRAM data)
   - `$43n2-$43n4` (A1Tn): 24-bit source address in ROM/RAM
   - `$43n5-$43n6` (DASn): byte count (0 = 65536 bytes)
3. The transfer is triggered by writing to `$420B` -- each bit enables one DMA channel
4. `mesen_find_occurrences(searchString="STA $420B", cpuType="Snes")` -- find all DMA trigger points
5. `mesen_breakpoint(action="set", address="$420B", type="Write", memoryType="SnesRegister", cpuType="Snes")` -- break on DMA triggers to inspect channel configuration

### Analyzing HDMA Effects

6. `mesen_find_occurrences(searchString="STA $420C", cpuType="Snes")` -- find where HDMA is enabled. Each bit of $420C enables one HDMA channel
7. Break at the HDMA enable point and read the channel registers to understand the effect:
   - `$43n0` bit 6: indirect mode (0=direct table, 1=table contains pointers to data)
   - `$43n1`: which PPU register is being modified per-scanline
   - `$43n2-$43n4`: address of the HDMA table in ROM/RAM
8. `mesen_read_memory` at the HDMA table address on `SnesPrgRom` to decode the table:
   - Each entry starts with a line count byte: $01-$7F = write once then skip N-1 lines, $81-$FF = write every scanline for N-$80 lines, $00 = end of table
   - Followed by data bytes (direct mode) or a 16-bit pointer (indirect mode)
   - Data size depends on the transfer pattern in DMAPn bits 0-2

### Common HDMA Uses

| Target Register | Effect |
|-----------------|--------|
| $2100 (INIDISP) | Brightness fading per scanline (spotlight/iris effects) |
| $2105 (BGMODE) | Mode switching mid-frame |
| $210D-$2114 (BGnHOFS/VOFS) | Parallax scrolling, wavy water effects (pattern 3: 4 bytes per write) |
| $211A-$2120 (Mode 7) | Mode 7 rotation/scaling parameters per scanline |
| $2126-$2129 (Window) | Window position changes for shaped transparency |
| $2130-$2132 (Color math) | Color gradient effects, underwater tinting |

**Tip:** HDMA is one of the SNES's most powerful and complex features. A single HDMA channel can change a PPU register every scanline, and up to 8 channels can run simultaneously. Transfer pattern 3 (4 bytes: low+low, high+high) is commonly used for scroll registers since BGnHOFS/VOFS require two sequential writes. Indirect HDMA tables allow the same table structure to point to different data per frame (useful for animated effects). Never enable DMA and HDMA simultaneously on the same channel -- DMA should only run during VBlank with HDMA channels disabled or using different channel numbers. Debug HDMA by reading the table at the source address and manually decoding the line count / data entries.

## Workflow I: BG Mode & Graphics Inspection

Use this to understand how the game renders its backgrounds and to inspect graphics data. The SNES supports 8 BG modes with different layer counts and color depths.

1. `mesen_get_state(component="ppu", cpuType="Snes")` -- check the current BG mode, screen brightness, and VRAM address
2. `mesen_get_tilemap_info(cpuType="Snes", layer=0)` -- inspect BG1: dimensions, tile size, BPP, scroll position, tilemap/tileset VRAM addresses
3. `mesen_get_tilemap_info(cpuType="Snes", layer=1)` -- inspect BG2 (available in modes 0-4)
4. `mesen_get_tilemap_tile_info(x=128, y=112, cpuType="Snes", layer=0)` -- inspect a specific tile by pixel coordinate on BG1
5. `mesen_get_sprite_list(cpuType="Snes")` -- list all 128 OAM sprites with positions, tile indices, palettes, sizes, priority, flip flags
6. `mesen_get_palette(cpuType="Snes")` -- view all 256 CGRAM colors (8 BG palettes + 8 sprite palettes, 16 colors each). Colors are 15-bit RGB (5 bits per channel)
7. `mesen_read_memory(address="$0000", length=64, memoryType="SnesVideoRam")` -- read raw VRAM tile data. SNES tile sizes vary: 2bpp=16 bytes/tile, 4bpp=32 bytes/tile, 8bpp=64 bytes/tile
8. To find what code updates a specific BG layer, `mesen_breakpoint(action="set", address="$2118", type="Write", memoryType="SnesRegister", cpuType="Snes")` -- watch VRAM data writes, then trace back to find the DMA source and the original tile/map data in ROM

**BG Mode quick reference:**

| Mode | BG1 | BG2 | BG3 | BG4 | Notes |
|------|-----|-----|-----|-----|-------|
| 0 | 2bpp | 2bpp | 2bpp | 2bpp | 4 layers, 4 colors each. Used in UI-heavy games |
| 1 | 4bpp | 4bpp | 2bpp | -- | Most common. 16 colors per tile on BG1/BG2 |
| 2 | 4bpp | 4bpp | OPT | -- | Offset-per-tile on BG3 (column scrolling effects) |
| 3 | 8bpp | 4bpp | -- | -- | 256-color BG1, used for photorealistic backgrounds |
| 4 | 8bpp | 2bpp | OPT | -- | 256-color + offset-per-tile |
| 5 | 4bpp | 2bpp | -- | -- | Hi-res (512px wide), 16x8 tiles |
| 6 | 4bpp | -- | OPT | -- | Hi-res + offset-per-tile |
| 7 | 8bpp | -- | -- | -- | Rotation/scaling (Mode 7). 128x128 tilemap, 256 tiles |

**Tip:** Mode 1 is by far the most common -- nearly all SNES action games, RPGs, and platformers use it. BG1/BG2 are the main playfield layers (16 colors per tile from 8 palettes), BG3 is typically used for status bars or HUD overlays (4 colors). In Mode 1, BG3 can be given highest priority by setting bit 3 of $2105 (the "Mode 1 BG3 priority" flag), which is how games show BG3 text in front of everything else. Mode 7 is used for rotation/scaling effects (F-Zero, Mario Kart, Final Fantasy airship). SNES tiles use planar format -- 2bpp tiles interleave two bit planes per row (16 bytes/tile), 4bpp adds two more planes (32 bytes/tile). The tilemap is 32x32 or 64x64 entries, each 2 bytes: 10-bit tile number + palette + priority + H/V flip.
