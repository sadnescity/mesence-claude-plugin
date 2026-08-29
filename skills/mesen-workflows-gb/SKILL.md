---
description: "Game Boy/GBC reverse engineering workflows for MesenCE: find text, find variables, analyze routines, VBlank analysis, patch ROMs, automate input, MBC/banking analysis, tile/sprite inspection, CGB features. GB-specific memory types, vectors, registers, and instruction patterns."
---

# Game Boy / GBC Reverse Engineering Workflows

Quick reference: cpuType=`Gameboy`, main memory=`GbWorkRam`, ROM=`GbPrgRom`, VRAM=`GbVideoRam`, HRAM=`GbHighRam`. CPU is Sharp LR35902 (Z80 variant). NOP=$00.

## Workflow A: Find Game Text

Use this when you can see text on screen but do not know the ROM's character encoding. GB games often use sequential ASCII-like encodings, but many titles (especially Japanese) use custom character tables.

1. `mesen_relative_search(searchText="POKEMON", memoryType="GbPrgRom")` -- search by byte differences between consecutive characters; works even when the encoding is unknown
2. For each match, `mesen_read_memory(address=matchAddress-4, length=30, memoryType="GbPrgRom")` -- read surrounding bytes to verify the pattern looks like text (similar byte ranges, recognizable spacing)
3. From a confirmed match, calculate the byte-to-character mapping. The first byte value at the match address corresponds to the first character in your search text. If 'P'=$8F, then 'O'=$8E, 'K'=$8A, etc.
4. `mesen_load_tbl(tblPathOrContent="8F=P\n8E=O\n8A=K\n...")` -- load the mapping as a TBL table
5. `mesen_decode_text(address=matchAddress, length=200, memoryType="GbPrgRom")` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="TRAINER", memoryType="GbPrgRom")` -- find other text strings using the TBL encoding

**Tip:** GB ROMs are banked in 16KB chunks. Bank 0 is always at $0000-$3FFF in CPU space; switchable banks are at $4000-$7FFF. When searching GbPrgRom, addresses are linear (bank * $4000 + offset). Many GB games store text sequentially in ROM. Use UPPERCASE -- many GB titles only have uppercase characters. Search strings of 5+ characters to reduce false positives. If the game uses DTE/MTE compression (common in text-heavy RPGs), relative search may still find uncompressed strings like menu labels and item names.

## Workflow B: Find a Game Variable

Use this when you can see a value on screen (HP, lives, coins) but do not know its RAM address. GB Work RAM is 8KB at $C000-$DFFF (banked to 32KB on CGB). Also check HRAM ($FF80-$FFFE, 127 bytes) -- the LDH instruction makes HRAM access faster, so games often store frequently-read variables there.

1. `mesen_search_memory(patternHex="03", memoryType="GbWorkRam")` -- search for the byte value 3 (e.g., lives=3) in Work RAM
2. `mesen_search_memory(patternHex="03", memoryType="GbHighRam")` -- also search HRAM for the same value
3. Change the value in-game (lose a life so lives becomes 2)
4. `mesen_search_memory(patternHex="02", memoryType="GbWorkRam")` -- search for the new value
5. Cross-reference: addresses present in both result sets are candidates
6. `mesen_write_memory(address=candidate, hexData="09", memoryType="GbWorkRam")` -- write a test value to a candidate address; if the game display updates, you found it
7. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="GbWorkRam", cpuType="Gameboy")` -- set a write breakpoint to catch what modifies the variable
8. `mesen_resume_execution()` -- resume and trigger the change in-game
9. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Gameboy")` -- check PC to find the writing instruction
10. `mesen_disassemble(address=PC-6, lineCount=20, cpuType="Gameboy")` -- view surrounding code; look for LD [HL],A or LD [addr],A patterns that store the value
11. `mesen_freeze_address(startAddress=candidate, endAddress=candidate, cpuType="Gameboy", freeze=true)` -- freeze the value for infinite lives

**Tip:** GB values are usually 8-bit. For 16-bit values (like score), search with 2-byte little-endian patterns. BCD encoding is common for displayed scores -- e.g., score 1500 stored as bytes $15 $00. On CGB, Work RAM banks 1-7 ($D000-$DFFF) are switchable, so a variable's CPU address depends on the active bank -- searching the full GbWorkRam memory type avoids this complication.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The LR35902 has registers A (accumulator), B, C, D, E, H, L, F (flags), SP, and PC. Register pairs BC, DE, HL, AF are used for 16-bit operations. HL is the primary pointer register.

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$0150", lineCount=50, cpuType="Gameboy")` -- view code at the target address (GB ROM entry point is $0100, game code typically starts at $0150)
3. `mesen_breakpoint(action="set", address="$0150", type="Execute", memoryType="GbPrgRom", cpuType="Gameboy")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Gameboy")` -- check register values (A, BC, DE, HL, SP, PC, flags)
6. `mesen_step(cpuType="Gameboy", stepType="Step")` -- single step one instruction, check state after each step
7. `mesen_step(cpuType="Gameboy", stepType="StepOver")` -- step over CALL instructions to skip subroutine internals
8. `mesen_get_execution_trace(count=200, cpuType="Gameboy")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Gameboy")` -- see the call chain that reached this routine
10. `mesen_label(action="set", address="$0150", memoryType="GbPrgRom", label="GameInit")` -- annotate the routine with a descriptive label

**Tip:** Common LR35902 patterns: `LD A,[HL+]` / `LD A,[HL-]` for sequential memory reads (auto-increment/decrement HL). `CP n` / `JR Z,offset` / `JR NZ,offset` for comparisons and branches. `CALL addr` / `RET` for subroutines. `RST $XX` for fast calls to fixed addresses ($00, $08, $10, $18, $20, $28, $30, $38). `PUSH/POP` pairs around CALL instructions show which registers are preserved. `DI` / `EI` around critical sections disable/enable interrupts.

## Workflow D: Analyze VBlank

Use this to understand the per-frame update routine. On the GB, VBlank fires when the LCD finishes drawing scanline 143 and enters the vertical blank period (scanlines 144-153). The CPU jumps to the fixed address $0040 (VBlank interrupt vector).

1. `mesen_disassemble(address="$0040", lineCount=5, cpuType="Gameboy")` -- view the VBlank vector; typically contains a JP instruction to the real handler
2. Note the JP target address (e.g., JP $0250 means the real VBlank handler is at $0250)
3. `mesen_disassemble(address="$0250", lineCount=60, cpuType="Gameboy")` -- view the full VBlank handler
4. `mesen_breakpoint(action="set", address="$0250", type="Execute", memoryType="GbPrgRom", cpuType="Gameboy")` -- set a breakpoint on the real handler; this fires every frame
5. `mesen_playback(action="resume")` -- resume and let the breakpoint hit
6. `mesen_get_state(component="cpu", cpuType="Gameboy")` -- check register state at VBlank entry
7. `mesen_get_execution_trace(count=500, cpuType="Gameboy")` -- trace the full VBlank execution flow
8. Look for these key operations in the disassembly:
   - OAM DMA: a write to $FF46 initiates sprite DMA (source = written_value * $100). A tight loop in HRAM waits for it to complete (40 cycles)
   - Scroll updates: writes to SCY ($FF42) and SCX ($FF43) set background scroll position
   - LCDC writes: $FF40 controls LCD enable, tile data area, BG map area, sprite size
   - BGP/OBP writes: $FF47 (BG palette), $FF48-$FF49 (sprite palettes); on CGB, palette writes go through $FF68-$FF6B
   - VRAM tile/map updates: writes to $8000-$9FFF (tile data and background maps)
9. `mesen_label(action="set", address="$0250", memoryType="GbPrgRom", label="VBlankHandler")` -- label the handler for future reference

**Tip:** VRAM ($8000-$9FFF) and OAM ($FE00-$FE9F) are only safely accessible during VBlank (LCD mode 1) and HBlank (LCD mode 0). Writing during rendering causes graphical glitches. The LY register ($FF44) contains the current scanline (0-153); LY >= 144 means VBlank is active. On CGB, double-speed mode (toggle via KEY1 register $FF4D + STOP instruction) gives twice as much VBlank CPU time. CGB HBlank DMA ($FF51-$FF55) can transfer $10 bytes per HBlank automatically -- commonly used for animated tiles.

**Other interrupt vectors:** VBlank ($0040) is the most common, but games also use:
- **STAT** ($0048): LCD status interrupt -- fires on specific LCD modes/scanlines. Used for raster effects (per-scanline palette changes, split-screen scrolling). Configured via STAT register ($FF41).
- **Timer** ($0050): Fires at a configurable rate via TAC ($FF07). Used for music engines and time-based events.
- **Serial** ($0058): Link cable communication.
- **Joypad** ($0060): Input change detection (rarely used in practice).

Check IE register ($FFFF) to see which interrupts the game enables, IF ($FF0F) for pending flags.

## Workflow E: Patch ROM

Use this to modify game code and save the result. The LR35902 NOP is $00 (1 byte). Most instructions are 1-2 bytes; some (LD rr,nn / CALL / JP) are 3 bytes.

1. `mesen_disassemble(address=patchAddress, lineCount=10, cpuType="Gameboy")` -- view the current code at the target address
2. `mesen_read_memory(address=patchAddress, length=10, memoryType="GbPrgRom")` -- check the raw bytes before patching
3. `mesen_assemble(code="NOP\nNOP", startAddress=patchAddress, cpuType="Gameboy")` -- assemble new instructions over the existing code (e.g., NOP out a 2-byte JR conditional branch)
4. `mesen_disassemble(address=patchAddress, lineCount=10, cpuType="Gameboy")` -- verify the patch was applied correctly
5. `mesen_read_memory(address=patchAddress, length=10, memoryType="GbPrgRom")` -- confirm raw bytes match expected values
6. `mesen_save_modified_rom(filepath="C:/patched_game.gb")` -- save the full patched ROM
7. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file for distribution

**Tip:** Common GB patches: NOP out a `JR Z,offset` or `JR NZ,offset` (2 bytes each) to skip conditional branches. NOP out a `CALL addr` (3 bytes = 3 NOPs) to disable a subroutine call. To force a branch, replace `JR NZ,offset` with `JR offset` ($18 instead of $20). GB ROMs use 16KB banks -- bank 0 is always mapped at $0000-$3FFF, switchable banks at $4000-$7FFF. The GbPrgRom memory type uses linear addressing (bank * $4000 + offset within bank), so address $4000 in bank 2 = GbPrgRom address $8000.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing. GB buttons: A, B, Up, Down, Left, Right, Select, Start. Input is read through the joypad register at $FF00.

1. `mesen_load_rom(filepath="C:/roms/game.gb")` -- load the game
2. `mesen_step(cpuType="Gameboy", stepType="PpuFrame")` -- advance past the boot screen (repeat several times or use a save state)
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Start to begin the game
4. `mesen_step(cpuType="Gameboy", stepType="PpuFrame")` -- advance 1 frame with Start held
5. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
6. `mesen_step(cpuType="Gameboy", stepType="PpuFrame")` -- advance a few frames to let the game process the input
7. `mesen_input_override(action="set", port=0, buttons="A")` -- press A to confirm a menu selection
8. `mesen_step(cpuType="Gameboy", stepType="PpuFrame")` -- advance 1 frame
9. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
10. `mesen_take_screenshot()` -- capture the screen to verify the resulting state

**Tip:** GB games typically read joypad input once per frame during VBlank. Hold a button for at least 1 full frame for it to register. For directional movement, hold the direction across multiple frames. Save states are useful checkpoints -- save before a sequence and reload to retry. Some games require buttons to be released for at least 1 frame between presses to register them as separate inputs (especially menu navigation).

## Workflow G: MBC & Banking Analysis

Use this to understand the ROM layout and bank switching before patching banked code. GB ROMs use Memory Bank Controllers (MBCs) for ROM and RAM banking. Knowing the MBC type is critical for patching code in switchable banks.

1. `mesen_get_rom_header()` -- read the cartridge header. The mapper field tells you the MBC type, and PrgRomSize/ChrRomSize give the total ROM/RAM sizes
2. Identify the MBC type:
   - **No MBC (mapper 0)**: 32KB ROM, no banking. Simplest to patch.
   - **MBC1 (mapper 1)**: Up to 2MB ROM / 32KB RAM. Bank select via writes to $2000-$3FFF (5-bit bank number, banks 1-31). Bank $00 reads as $01.
   - **MBC2 (mapper 2)**: Up to 256KB ROM + 512x4-bit internal RAM. Bank select via $2100-$21FF.
   - **MBC3 (mapper 3)**: Up to 2MB ROM / 32KB RAM + RTC. Bank select via $2000-$3FFF (7-bit, banks 1-127). Most common in later DMG games (Pokemon Gen 2).
   - **MBC5 (mapper 5)**: Up to 8MB ROM / 128KB RAM. Bank select via $2000-$2FFF (low 8 bits) + $3000-$3FFF (bit 8). Can select bank 0. Most common in CGB games.
3. `mesen_get_address_info(address="$4000", cpuType="Gameboy")` -- check which absolute ROM offset is currently mapped to the switchable bank ($4000-$7FFF)
4. `mesen_find_occurrences(searchString="LD [$2000]", cpuType="Gameboy")` -- find bank switch code (writes to $2000-$3FFF select the ROM bank). Also try `"LD [$2100]"` for MBC2
5. `mesen_breakpoint(action="set", address="$2000", type="Write", memoryType="GbPrgRom", cpuType="Gameboy")` -- watch for bank switches to understand when and which banks get swapped in
6. When patching code in a switchable bank, always use `mesen_get_address_info` to convert the CPU address to an absolute GbPrgRom offset. CPU address $4000 in bank 5 = GbPrgRom address $14000 (bank * $4000)

**Tip:** Bank 0 ($0000-$3FFF) is always accessible and contains the interrupt vectors, header, and usually the core engine code. Switchable banks ($4000-$7FFF) contain level data, graphics, music, and specialized routines. When the game calls code in a different bank, it typically uses a "bankswitch trampoline" -- a routine in bank 0 that switches the bank, calls the target, then restores the original bank. Finding this trampoline is key to understanding the game's code organization.

## Workflow H: Tile & Sprite Inspection

Use this to inspect and modify graphics. GB tiles are 8x8 pixels, 2bpp (4 shades/colors). Tile data is 16 bytes per tile (2 bytes per row, interleaved bit planes).

1. `mesen_get_sprite_list(cpuType="Gameboy")` -- list all 40 OAM sprites with positions, tile indices, palettes, and flip flags
2. For a sprite of interest, note the `TileAddr` in VRAM
3. `mesen_get_tile_pixel(tileAddress=addr, format="Bpp2", x=0, y=0, memoryType="GbVideoRam")` -- read a pixel from the tile (color index 0-3)
4. `mesen_set_tile_pixel(tileAddress=addr, format="Bpp2", x=0, y=0, color=2, memoryType="GbVideoRam")` -- modify a pixel
5. `mesen_get_tilemap_info(cpuType="Gameboy")` -- get background map info: dimensions, tile size, scroll position, tilemap/tileset VRAM addresses
6. `mesen_get_tilemap_tile_info(x=80, y=72, cpuType="Gameboy")` -- inspect a specific background tile by screen pixel coordinate
7. `mesen_get_palette(cpuType="Gameboy")` -- view palette entries. DMG: 4 shades via BGP ($FF47), OBP0 ($FF48), OBP1 ($FF49). CGB: 8 BG palettes + 8 sprite palettes, 4 colors each, 15-bit RGB
8. `mesen_read_memory(address="$0000", length=16, memoryType="GbVideoRam")` -- read raw tile bytes. Each tile is 16 bytes: pairs of bytes (low plane, high plane) per row, 8 rows

**Tip:** The GB has two tile data addressing modes controlled by LCDC bit 4: mode 0 uses $8800 base with signed tile indices ($9000 = tile 0), mode 1 uses $8000 base with unsigned indices. Most games use $8000 (unsigned). The background map at $9800 (or $9C00, LCDC bit 3) is 32x32 tile indices. OAM ($FE00-$FE9F) holds 40 sprites, 4 bytes each: Y position, X position, tile index, attributes (DMG: bit 4=palette, bit 5=X flip, bit 6=Y flip, bit 7=BG priority; CGB adds bits 0-2=palette, bit 3=VRAM bank). Max 10 sprites per scanline. On CGB, VRAM bank 1 ($FF4F) stores BG tile attributes: bits 0-2=palette, bit 3=tile VRAM bank, bit 5=H flip, bit 6=V flip, bit 7=BG-over-OBJ priority.
