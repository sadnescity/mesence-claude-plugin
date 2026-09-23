---
description: "NES reverse engineering workflows for MesenCE: find text, find variables, analyze routines, VBlank/NMI analysis, patch ROMs, automate input, ROM/mapper analysis, CHR graphics editing, PPU/nametable inspection. NES-specific memory types, vectors, registers, and instruction patterns. Use when planning or executing reverse engineering, translation or ROM hacking tasks on an NES/Famicom game in MesenCE."
---

# NES Reverse Engineering Workflows

Quick reference: cpuType=`Nes`, main memory=`NesWorkRam`, ROM=`NesPrgRom`, CHR=`NesChrRom`/`NesChrRam`. CPU is Ricoh 2A03 (6502-based). NOP=$EA.

## Workflow A: Find Game Text (Relative Search + TBL)

Use this when you can see text on screen but do not know the ROM's character encoding. Many NES games use custom encodings -- often sequential starting from a non-ASCII value (e.g., 'A'=$0A or 'A'=$D0). Relative search finds text by byte-to-byte differences, bypassing the encoding problem.

1. `mesen_relative_search(searchText="DRAGON", memoryType="NesPrgRom")` -- search by byte differences between consecutive characters; works even with unknown custom encodings
2. For each match, `mesen_read_memory(address=matchAddress-4, length=30, memoryType="NesPrgRom")` -- read surrounding bytes to verify the match looks like text (similar byte ranges, consistent patterns)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value corresponds to the first character in your search text. If 'D'=0x0A, then 'E'=0x0B, 'F'=0x0C, etc.
4. `mesen_load_tbl(tblPathOrContent="0A=D\n0B=E\n0C=F\n0D=G\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address=matchAddress, length=100, memoryType="NesPrgRom")` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="WARRIOR", memoryType="NesPrgRom")` -- find other text strings using the TBL encoding

**Tip:** Use UPPERCASE -- most NES games only store uppercase letters. Search strings of 5+ characters reduce false positives. NES character encodings are almost always sequential (A, B, C... map to consecutive byte values) but the starting offset varies per game. Spaces and punctuation often break relative search -- search for individual words instead. If no results, the game may use DTE compression (one byte = two characters).

## Workflow B: Find a Game Variable (Cheat Search)

Use this when you can see a value on screen (lives=3, HP=100) but do not know its RAM address. NES has only 2KB of work RAM ($0000-$07FF, mirrored at $0800-$1FFF), so searches produce fewer candidates than other consoles.

1. `mesen_search_memory(patternHex="03", memoryType="NesWorkRam")` -- search for the byte value 3 in work RAM (2KB total)
2. Change the value in-game (lose a life so lives becomes 2)
3. `mesen_search_memory(patternHex="02", memoryType="NesWorkRam")` -- search for the new value
4. Cross-reference: addresses present in both result sets are candidates. With only 2KB of RAM, you will typically narrow down to a handful of addresses quickly
5. `mesen_write_memory(address=candidate, hexData="09", memoryType="NesWorkRam")` -- write a test value to a candidate address. If the on-screen display updates, you found it
6. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="NesWorkRam", cpuType="Nes")` -- set a write breakpoint to watch what modifies the variable
7. `mesen_resume_execution()` -- resume and trigger the change in-game
8. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Nes")` -- check A, X, Y, PC registers to find the writing instruction
9. `mesen_disassemble(address=PC-8, lineCount=20, cpuType="Nes")` -- view surrounding code; look for STA (store accumulator) instructions targeting your candidate address
10. `mesen_freeze_address(startAddress=candidate, endAddress=candidate, cpuType="Nes", freeze=true)` -- freeze the value for infinite lives

**Tip:** NES values are usually 8-bit (0-255). Scores and large numbers are often stored as BCD (Binary-Coded Decimal) -- e.g., score 1500 stored as bytes $15 $00. Zero-page addresses ($00-$FF) are fastest to access, so games store frequently-used variables there. If NesWorkRam yields nothing, try `NesSaveRam` for cartridges with battery-backed SRAM.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The NES CPU (Ricoh 2A03) is 6502-based with registers A (accumulator), X and Y (index), SP (stack pointer), PC (program counter), and P (processor flags: N, V, -, B, D, I, Z, C).

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$C000", lineCount=50, cpuType="Nes")` -- view code at the target address
3. `mesen_breakpoint(action="set", address="$C000", type="Execute", memoryType="NesPrgRom", cpuType="Nes")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Nes")` -- check register values (A, X, Y, SP, PC, flags) at the breakpoint
6. `mesen_step(cpuType="Nes", stepType="Step")` -- single step one instruction; check state after each step
7. `mesen_step(cpuType="Nes", stepType="StepOver")` -- step over JSR calls to skip subroutines
8. `mesen_get_execution_trace(count=200, cpuType="Nes")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Nes")` -- see the call chain that led to this routine
10. `mesen_label(action="set", address="$C000", memoryType="NesPrgRom", label="MainLoop")` -- annotate the routine with a descriptive label

**Tip:** 6502 instructions are 1-3 bytes. JSR (3 bytes) calls a subroutine, RTS returns from it. LDA/LDX/LDY load registers, STA/STX/STY store them. CMP/CPX/CPY compare, then BEQ (branch if equal) / BNE (branch if not equal) / BCS / BCC branch based on flags. Common loop pattern: LDX #count / DEX / BNE loop. Table lookup: LDA table,X. Indirect addressing LDA ($xx),Y is used heavily for pointer-based data access.

## Workflow D: Analyze VBlank/NMI

Use this to understand the main per-frame update routine. On the NES, the NMI (Non-Maskable Interrupt) fires every frame during VBlank when enabled via PPU register $2000 bit 7. VBlank is the only safe time to update VRAM, sprites, and scroll registers.

1. `mesen_read_memory(address="$FFFA", length=2, memoryType="NesPrgRom")` -- read the NMI vector (2 bytes, little-endian). For example, bytes $0B $C4 means the NMI handler is at $C40B
2. Calculate the handler address from the two bytes (low byte first, high byte second)
3. `mesen_disassemble(address="$C40B", lineCount=60, cpuType="Nes")` -- view the NMI routine
4. `mesen_breakpoint(action="set", address="$C40B", type="Execute", memoryType="NesPrgRom", cpuType="Nes")` -- this breakpoint will hit every frame (~60 times/second)
5. `mesen_playback(action="resume")` -- resume and let it hit once
6. `mesen_get_execution_trace(count=500, cpuType="Nes")` -- view the full NMI execution flow
7. Look for these key operations in the trace:
   - **OAM DMA**: write to $4014 (copies 256 bytes of sprite data to PPU OAM)
   - **PPU scroll**: writes to $2005 (two sequential writes set X and Y scroll)
   - **PPU address/data**: writes to $2006/$2007 (nametable or pattern table updates)
   - **Controller read**: reads from $4016 (player 1) and $4017 (player 2)
   - **Sound engine call**: often a JSR to the music/SFX update routine
   - **Frame counter**: STA to a RAM address that increments each frame
8. `mesen_label(action="set", address="$C40B", memoryType="NesPrgRom", label="NMI_Handler")` -- label the NMI entry point

**Tip:** The NMI is the backbone of the NES game loop. Most games split work between NMI (time-critical PPU updates that must happen during VBlank's ~2273 CPU cycles) and the main loop (game logic like physics, AI, collision). The NMI handler typically starts with PHA/TXA/PHA/TYA/PHA to save registers and ends with PLA/TAY/PLA/TAX/PLA/RTI to restore them. If the NMI seems too short, the game may use a "ready flag" pattern -- the main loop sets a flag when it finishes computing a frame, and NMI only does the PPU update if that flag is set.

## Workflow E: Patch ROM

Use this to modify NES game code and save the result. The 6502 NOP opcode is $EA (1 byte).

1. `mesen_disassemble(address="$C123", lineCount=10, cpuType="Nes")` -- view current code at the patch target
2. Determine what to patch:
   - **NOP out a conditional branch** (BEQ, BNE, BCS, BCC, etc.): 2 bytes, replace with `NOP / NOP`
   - **NOP out a subroutine call** (JSR $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **NOP out an absolute store** (STA $xxxx): 3 bytes, replace with `NOP / NOP / NOP`
   - **Force a branch**: change BNE to BEQ or vice versa (single byte change at the opcode)
   - **Change an immediate value**: LDA #$03 -> LDA #$09 (change the operand byte)
3. `mesen_assemble(code="NOP\nNOP\nNOP", startAddress="$C123", cpuType="Nes")` -- assemble the patch over the existing code
4. `mesen_disassemble(address="$C123", lineCount=10, cpuType="Nes")` -- verify the patch was applied correctly
5. `mesen_playback(action="resume")` -- test the patch in-game
6. `mesen_take_screenshot()` -- capture the result to verify behavior
7. `mesen_save_modified_rom(filepath="C:/patched_game.nes")` -- save the full patched ROM
8. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file instead

**Tip:** NES 6502 instruction sizes: implied/accumulator = 1 byte, immediate/zero-page/relative = 2 bytes, absolute/indirect = 3 bytes. When NOPing multi-byte instructions, you must fill ALL bytes with $EA to avoid misaligning the instruction stream. Common patches: make a game skip a "lives check" by NOPing the DEC + BEQ sequence, or give infinite health by NOPing the STA that writes the damage value. IPS patches are small and shareable.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing menus, game sequences, or verifying patches. NES controller buttons: A, B, Up, Down, Left, Right, Select, Start.

1. `mesen_load_rom(filepath="C:/roms/game.nes")` -- load the NES game
2. `mesen_playback(action="pause")` -- pause to set up the automation
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Start on controller 1
4. `mesen_step(cpuType="Nes", stepType="PpuFrame")` -- advance one frame with Start held
5. `mesen_step(cpuType="Nes", stepType="PpuFrame")` -- advance another frame (some games need multiple frames to register a press)
6. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
7. `mesen_step(cpuType="Nes", stepType="PpuFrame")` -- advance a frame with no buttons held
8. `mesen_input_override(action="set", port=0, buttons="A")` -- press A to confirm a menu selection
9. `mesen_step(cpuType="Nes", stepType="PpuFrame")` -- advance the frame
10. `mesen_take_screenshot()` -- capture the screen to verify the result

**Tip:** Hold a direction for multiple frames by repeating `mesen_step` without changing the input override. To press two buttons simultaneously, combine them: `buttons="A,Right"`. Use `mesen_save_state(action="save")` before an input sequence and `mesen_save_state(action="load")` to retry. For player 2, use port=1. NES games typically read the controller once per frame during NMI via $4016, so each frame of input counts.

## Workflow G: ROM & Mapper Analysis

Use this to understand the ROM layout, mapper, and bank switching before doing any serious patching. Bank switching awareness is critical on the NES -- the same CPU address ($8000-$FFFF) can map to different PRG ROM banks depending on the mapper state.

1. `mesen_get_rom_header()` -- read the iNES/NES 2.0 header to get mapper number, PRG/CHR sizes, mirroring type, battery flag
2. Identify the mapper from the header. Common mappers:
   - **Mapper 0 (NROM)**: No bank switching. 16KB or 32KB PRG, 8KB CHR. Simplest to patch.
   - **Mapper 1 (MMC1/SxROM)**: 256KB PRG max, switched in 16KB or 32KB chunks via serial register writes. Common in early RPGs (Zelda, Metroid, Final Fantasy).
   - **Mapper 2 (UxROM)**: 16KB switchable bank at $8000 + 16KB fixed bank at $C000. Simple bankswitch via STA to $8000-$FFFF.
   - **Mapper 3 (CNROM)**: Fixed PRG, switchable 8KB CHR banks. Write to $8000-$FFFF to select CHR bank.
   - **Mapper 4 (MMC3/TxROM)**: 8KB PRG banks, 1KB/2KB CHR banks, scanline counter for IRQ. Most popular mapper (Mega Man 3-6, Super Mario Bros 3, many later games).
   - **Mapper 7 (AxROM)**: 32KB PRG bank switching, single-screen mirroring. Used by Battletoads, Rare games.
3. `mesen_get_address_info(address="$8000", cpuType="Nes")` -- check which absolute PRG ROM offset maps to CPU $8000 in the current bank configuration
4. `mesen_get_address_info(address="$C000", cpuType="Nes")` -- check the fixed bank (on mappers like UxROM/MMC3, the last bank is often fixed at $C000-$FFFF)
5. To find the bankswitch routine, `mesen_find_occurrences(searchString="STA $8000", cpuType="Nes")` or `mesen_find_occurrences(searchString="STA $E000", cpuType="Nes")` -- mapper register writes vary by mapper
6. `mesen_breakpoint(action="set", address="$8000", type="Write", memoryType="NesWorkRam", cpuType="Nes")` -- watch for bankswitch writes (mapper registers are in the ROM address space)
7. When patching banked code, always use `mesen_get_address_info` to find the absolute PRG ROM offset first, then verify with `mesen_read_memory` on `NesPrgRom` at that offset

**Tip:** ROM expansion is one of the hardest NES hacking tasks and is highly mapper-specific. Each mapper has different bank sizes, register interfaces, and maximum ROM sizes. The workflow above helps you understand the current layout and find free space within existing banks -- always try that first before expanding. Full ROM expansion (adding banks, writing bankswitch stubs, relocating code, updating pointer tables) requires deep mapper knowledge. Refer to the nesdev wiki for mapper-specific details: https://www.nesdev.org/wiki/Mapper

## Workflow H: CHR Graphics Inspection & Editing

Use this to inspect and modify tile graphics. NES tiles are 8x8 pixels, 2bpp (4 colors per tile from a 4-color palette). CHR data can be in ROM (`NesChrRom`) or RAM (`NesChrRam`).

1. `mesen_get_rom_header()` -- check if the game uses CHR ROM or CHR RAM (ChrRomSize=0 means CHR RAM, tiles are copied from PRG ROM to CHR RAM at runtime)
2. `mesen_list_memory_types()` -- confirm available CHR memory types (`NesChrRom` and/or `NesChrRam`)
3. `mesen_get_sprite_list(cpuType="Nes")` -- see all 64 OAM sprites with their tile indices, positions, palettes, and flip flags
4. For a sprite of interest, note its `TileAddr` -- this is the address in CHR memory where the tile graphics live
5. `mesen_get_tile_pixel(tileAddress=spriteAddr, format="NesBpp2", x=0, y=0, memoryType="NesChrRom")` -- read individual pixels from the tile (returns a color index 0-3)
6. `mesen_set_tile_pixel(tileAddress=spriteAddr, format="NesBpp2", x=0, y=0, color=2, memoryType="NesChrRom")` -- modify a pixel in the tile
7. `mesen_get_palette(cpuType="Nes")` -- view all palette colors to understand which RGB colors map to which indices
8. `mesen_read_memory(address="$0000", length=16, memoryType="NesChrRom")` -- read raw tile bytes (each tile is 16 bytes: 8 bytes for bit plane 0 + 8 bytes for bit plane 1)
9. To find a specific tile in CHR, `mesen_search_memory(patternHex="...", memoryType="NesChrRom")` -- search for known tile byte patterns

**Tip:** NES tile format (NesBpp2): each 8x8 tile is 16 bytes -- two 8-byte bit planes interleaved. Bit plane 0 (bytes 0-7) stores the low bit of each pixel, bit plane 1 (bytes 8-15) stores the high bit. The PPU has two pattern tables: $0000-$0FFF (256 tiles) and $1000-$1FFF (256 tiles). BG tiles typically use one table, sprites the other (controlled by $2000 bits 3-4). For CHR RAM games, tiles are loaded from PRG ROM -- to find where, set a write breakpoint on NesChrRam and trace back to find the copy routine and its source address in PRG ROM.

## Workflow I: PPU & Nametable Inspection

Use this to understand what's displayed on screen and how the game builds its visual layout. The NES PPU has two nametables (32x30 tiles each = 960 bytes + 64 attribute bytes).

1. `mesen_get_tilemap_info(cpuType="Nes", layer=0)` -- get nametable dimensions, scroll position, tile size, tilemap/tileset addresses
2. `mesen_get_tilemap_tile_info(x=128, y=120, cpuType="Nes")` -- inspect a specific tile on screen by pixel coordinate. Returns tile index, palette, flip flags, VRAM address
3. `mesen_read_memory(address="$0000", length=960, memoryType="NesNametableRam")` -- read the first nametable (32x30 = 960 tile indices). Each byte is a tile index into the pattern table
4. `mesen_read_memory(address="$03C0", length=64, memoryType="NesNametableRam")` -- read the attribute table for the first nametable (64 bytes control palette assignment for 2x2 tile groups)
5. `mesen_read_memory(address="$0000", length=32, memoryType="NesPaletteRam")` -- read all 32 palette entries (4 BG palettes + 4 sprite palettes, 4 colors each). First color ($3F00) is the universal background color
6. `mesen_get_palette(cpuType="Nes")` -- view actual RGB colors for all palette entries
7. To find what code writes a specific nametable tile, `mesen_breakpoint(action="set", address="$2007", type="Write", memoryType="NesWorkRam", cpuType="Nes")` -- catch PPU writes, then trace back to find the tile index source

**Tip:** NES nametable mirroring (from ROM header): Horizontal mirroring = vertical scrolling games (nametables A/A/B/B), Vertical mirroring = horizontal scrolling games (A/B/A/B). The attribute table is notoriously tricky -- each byte controls a 4x4 tile area (32x32 pixels), with 2 bits per 2x2 quadrant selecting palettes 0-3. Palette RAM is 32 bytes but only 25 unique entries ($3F00 is shared, and $3F04/$3F08/$3F0C mirror $3F00 for BG). For scrolling games, understanding the nametable layout and scroll register values ($2005) is key to finding the map data format in PRG ROM.
