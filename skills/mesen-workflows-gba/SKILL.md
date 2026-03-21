---
description: "GBA reverse engineering workflows for Mesen2: find text, find variables, analyze routines, VBlank/IRQ analysis, patch ROMs, automate input, BIOS calls, BG mode and sprite inspection. GBA-specific memory types, vectors, registers, and instruction patterns."
---

# GBA Reverse Engineering Workflows

Quick reference: cpuType=`Gba`, IWRAM=`GbaIntWorkRam` (32KB, fast), EWRAM=`GbaExtWorkRam` (256KB), ROM=`GbaPrgRom`, VRAM=`GbaVideoRam`, OAM=`GbaSpriteRam`, Palette=`GbaPaletteRam`. CPU is ARM7TDMI (ARM + THUMB modes). NOP=$00000000 (ARM) or $46C0 (THUMB, MOV R8,R8).

## Workflow A: Find Game Text

Use this when you can see text on screen but do not know the ROM's character encoding. GBA games often use ASCII or Shift-JIS (Japanese titles). ROM can be up to 32MB, so narrowing the search range helps.

1. `mesen_relative_search(searchText="ATTACK", memoryType="GbaPrgRom")` -- search by byte differences between consecutive characters; works even when the encoding is unknown
2. For each match, `mesen_read_memory(address=matchAddress-4, length=40, memoryType="GbaPrgRom")` -- read surrounding bytes to verify the pattern looks like text (similar byte ranges, printable character patterns)
3. From a confirmed match, calculate the byte-to-character mapping. If 'A'=$41 the game uses standard ASCII. If 'A'=$C1 or some other value, build a custom table.
4. `mesen_load_tbl(tblPathOrContent="C1=A\nC2=B\nC3=C\n...")` -- load the mapping as a TBL table (skip this step if the game uses standard ASCII)
5. `mesen_decode_text(address=matchAddress, length=300, memoryType="GbaPrgRom")` -- read decoded text from the ROM using the loaded TBL
6. `mesen_search_text(text="DEFENSE", memoryType="GbaPrgRom")` -- find other text strings using the TBL encoding
7. `mesen_search_memory(patternHex="4445 4645 4E53 45", memoryType="GbaPrgRom")` -- alternatively, if ASCII, search for raw hex of "DEFENSE"

**Tip:** Many GBA games store text as standard ASCII or with a simple offset from ASCII. Check for both. GBA ROMs are large (up to 32MB) so relative search may take a moment. Some games use Huffman or LZ77 compression for text blocks -- if relative search finds nothing, look for uncompressed strings like menu labels, item names, or error messages first. Pointer tables to text strings are common; once you find one string, read backward to find a pointer table.

## Workflow B: Find a Game Variable

Use this when you can see a value on screen (HP, gold, level) but do not know its RAM address. Search EWRAM first ($02000000, 256KB) -- most game variables live here. Also check IWRAM ($03000000, 32KB) for performance-critical variables. GBA values are commonly 32-bit or 16-bit little-endian.

1. `mesen_search_memory(patternHex="6400", memoryType="GbaExtWorkRam")` -- search for the 16-bit little-endian value 100 (e.g., HP=100) in EWRAM
2. `mesen_search_memory(patternHex="6400", memoryType="GbaIntWorkRam")` -- also search IWRAM
3. Change the value in-game (take damage so HP becomes 85)
4. `mesen_search_memory(patternHex="5500", memoryType="GbaExtWorkRam")` -- search for the new 16-bit value 85
5. Cross-reference: addresses present in both result sets are candidates
6. `mesen_write_memory(address=candidate, hexData="E703", memoryType="GbaExtWorkRam")` -- write 999 (little-endian $03E7) to test; if the game display updates, you found it
7. `mesen_breakpoint(action="set", address=candidate, type="Write", memoryType="GbaExtWorkRam", cpuType="Gba")` -- set a write breakpoint to catch what modifies the variable
8. `mesen_resume_execution()` -- resume and trigger the change in-game
9. When the breakpoint hits, `mesen_get_state(component="cpu", cpuType="Gba")` -- check R15 (PC) and CPSR to find the writing instruction and whether the CPU is in ARM or THUMB mode (CPSR bit 5 = THUMB flag)
10. `mesen_disassemble(address=PC-8, lineCount=30, cpuType="Gba")` -- view surrounding code; look for STRH (store halfword) or STR (store word) instructions targeting the variable

**Tip:** GBA games often use 32-bit values even for small numbers due to ARM alignment requirements. Try searching for 4-byte little-endian patterns if 2-byte search yields no results. EWRAM has a 16-bit bus (slower) while IWRAM has a 32-bit bus (faster), so time-critical data tends to be in IWRAM. Some games store structs with padding -- if you find one field, read nearby memory to map out the full data structure.

## Workflow C: Analyze a Routine

Use this when you want to understand what code at a specific address does. The ARM7TDMI has R0-R15 (R13=SP, R14=LR, R15=PC) plus CPSR. Most game code runs in THUMB mode (16-bit instructions) for code density. ARM mode (32-bit instructions) is used for IRQ handlers and performance-critical code placed in IWRAM.

1. `mesen_playback(action="pause")` -- pause the emulator
2. `mesen_disassemble(address="$08000000", lineCount=50, cpuType="Gba")` -- view code at the target address (GBA ROM starts at $08000000 in CPU space; the entry point is at $08000000 after the header at $08000000-$080000BF)
3. `mesen_breakpoint(action="set", address="$08000000", type="Execute", memoryType="GbaPrgRom", cpuType="Gba")` -- set an execution breakpoint
4. `mesen_playback(action="resume")` -- resume emulation and wait for the breakpoint to hit
5. `mesen_get_state(component="cpu", cpuType="Gba")` -- check register values (R0-R15, CPSR); CPSR bit 5 indicates THUMB mode
6. `mesen_step(cpuType="Gba", stepType="Step")` -- single step one instruction, check state after each step
7. `mesen_step(cpuType="Gba", stepType="StepOver")` -- step over BL (branch-and-link) calls to skip subroutine internals
8. `mesen_get_execution_trace(count=200, cpuType="Gba")` -- view recent execution flow (tracing is auto-enabled for the main CPU)
9. `mesen_get_callstack(cpuType="Gba")` -- see how the current routine was called
10. `mesen_label(action="set", address="$08000000", memoryType="GbaPrgRom", label="EntryPoint")` -- annotate the routine with a descriptive label

**Tip:** THUMB function boundaries are marked by `PUSH {r4-r7,lr}` at entry and `POP {r4-r7,pc}` at exit. `BL addr` is a subroutine call (sets LR). `BX Rn` switches between ARM and THUMB modes (bit 0 of Rn: 1=THUMB, 0=ARM). `LDR Rn,[PC,#offset]` loads constants from a literal pool after the function. `CMP/BEQ/BNE` patterns reveal conditional logic. Most game code in ROM runs as THUMB; if you see ARM instructions, the code may have been copied to IWRAM for speed.

## Workflow D: Analyze VBlank/IRQ

Use this to understand the per-frame update routine. On the GBA, the user IRQ handler pointer is at $03007FFC (top of IWRAM). The BIOS dispatches interrupts through this pointer. VBlank occurs at scanline 160 (VCOUNT=$04000006 reads 160-227 during VBlank).

1. `mesen_read_memory(address="$7FFC", length=4, memoryType="GbaIntWorkRam")` -- read the user IRQ handler pointer from the top of IWRAM (4 bytes, little-endian); this is CPU address $03007FFC
2. Calculate the handler address from the 4 bytes (little-endian 32-bit pointer, typically pointing into ROM at $08XXXXXX)
3. `mesen_disassemble(address=handlerAddress, lineCount=60, cpuType="Gba")` -- view the IRQ handler; it typically checks IE/IF to determine which interrupt fired
4. `mesen_breakpoint(action="set", address=handlerAddress, type="Execute", memoryType="GbaPrgRom", cpuType="Gba")` -- set a breakpoint on the handler; VBlank fires every frame
5. `mesen_playback(action="resume")` -- resume and let the breakpoint hit
6. `mesen_get_state(component="cpu", cpuType="Gba")` -- check register state; the handler runs in IRQ mode (CPSR mode bits = 10010)
7. `mesen_get_execution_trace(count=500, cpuType="Gba")` -- trace the full IRQ execution flow
8. Look for these key operations in the disassembly:
   - IF acknowledgment: the handler reads IE ($04000200) and IF ($04000202), ANDs them, then writes back to IF to acknowledge. It MUST also OR the acknowledged bits into the BIOS IntrCheckFlag at $03007FF8
   - DMA transfers: writes to DMA3 registers ($040000D4-$040000DE) for bulk memory copies (VRAM updates, OAM copy). DMA3 is most common for general-purpose transfers
   - OAM update: copy sprite attribute buffer to OAM ($07000000), typically via DMA3
   - BG scroll: writes to BG0HOFS/BG0VOFS ($04000010-$0400001E) set scroll positions
   - Palette update: writes to palette RAM ($05000000-$050003FF)
   - SWI 0x05 (VBlankIntrWait): many games call this in the main loop to halt until VBlank
9. `mesen_label(action="set", address=handlerAddress, memoryType="GbaPrgRom", label="IRQHandler")` -- label the handler for future reference
10. To find the main game loop, search for `SWI 0x05` in the trace -- the code just after the SWI return is the per-frame game logic

**Tip:** The GBA IRQ system requires careful setup: DISPSTAT bit 3 ($04000004) enables VBlank IRQ signaling, IE bit 0 ($04000200) enables VBlank in the interrupt controller, and IME ($04000208) is the master enable. The handler MUST acknowledge interrupts by writing to both IF ($04000202) and the BIOS flag at $03007FF8, or VBlankIntrWait will hang. Unlike the GB, the GBA has no restriction on VRAM access timing -- VRAM is always accessible, but writes during active display may cause tearing. DMA transfers during VBlank are the preferred method for flicker-free updates.

## Workflow E: Patch ROM

Use this to modify game code and save the result. Most GBA game code is THUMB (16-bit instructions). THUMB NOP = $46C0 (MOV R8,R8, 2 bytes). ARM NOP = $E1A00000 (MOV R0,R0, 4 bytes). Note: game code is at $08000000+ in CPU address space but $00000000+ in the GbaPrgRom memory type.

1. `mesen_disassemble(address=patchAddress, lineCount=15, cpuType="Gba")` -- view the current code at the target address (use GbaPrgRom addresses, not CPU addresses)
2. `mesen_read_memory(address=patchAddress, length=16, memoryType="GbaPrgRom")` -- check the raw bytes before patching
3. `mesen_assemble(code="MOV R8,R8\nMOV R8,R8", startAddress=patchAddress, cpuType="Gba")` -- assemble THUMB NOPs over the existing code (e.g., NOP out a 4-byte conditional BL call)
4. `mesen_disassemble(address=patchAddress, lineCount=15, cpuType="Gba")` -- verify the patch was applied correctly
5. `mesen_read_memory(address=patchAddress, length=16, memoryType="GbaPrgRom")` -- confirm raw bytes match expected values ($C046 repeated for THUMB NOPs, little-endian)
6. `mesen_save_modified_rom(filepath="C:/patched_game.gba")` -- save the full patched ROM
7. Or `mesen_save_modified_rom(filepath="C:/patch.ips", saveAsIps=true)` -- create an IPS patch file for distribution

**Tip:** THUMB conditional branches (BEQ, BNE, etc.) are 2 bytes -- one THUMB NOP replaces them. THUMB BL (branch-and-link, function call) is 4 bytes (two 16-bit words) -- use two THUMB NOPs. ARM instructions are always 4 bytes -- one ARM NOP replaces any ARM instruction. Remember: GbaPrgRom address = CPU address - $08000000. For example, code at CPU address $0800A000 is at GbaPrgRom address $A000. To force a branch, change the condition code: replace BEQ ($D0xx) with B ($E0xx, unconditional). To disable a function entirely, place `BX LR` at its start to return immediately.

## Workflow F: Automate Input

Use this to script repeatable input sequences for testing. GBA buttons: A, B, L, R, Up, Down, Left, Right, Select, Start. Input is read from KEYINPUT at $04000130 (active low -- bits are 0 when pressed).

1. `mesen_load_rom(filepath="C:/roms/game.gba")` -- load the game
2. `mesen_step(cpuType="Gba", stepType="PpuFrame")` -- advance past the BIOS intro (repeat several times or use a save state)
3. `mesen_input_override(action="set", port=0, buttons="Start")` -- press Start to begin the game
4. `mesen_step(cpuType="Gba", stepType="PpuFrame")` -- advance 1 frame with Start held
5. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
6. `mesen_step(cpuType="Gba", stepType="PpuFrame")` -- advance a few frames for the game to process
7. `mesen_input_override(action="set", port=0, buttons="A")` -- press A to confirm a selection
8. `mesen_step(cpuType="Gba", stepType="PpuFrame")` -- advance 1 frame
9. `mesen_input_override(action="set", port=0, buttons="")` -- release all buttons
10. `mesen_take_screenshot()` -- capture the screen to verify the resulting state

**Tip:** GBA games read KEYINPUT once per frame, typically during VBlank processing. Hold a button for at least 1 full frame for it to register. For simultaneous button presses (e.g., L+R for special actions), combine them: `buttons="L,R"`. The GBA has L and R shoulder buttons that many games use for menu navigation or special actions. Save states are useful checkpoints -- save before a sequence and reload to retry. Some games implement key repeat with a delay (hold for N frames, then auto-repeat), so multi-frame holds may trigger repeated actions.

## Workflow G: BIOS Calls & Decompression

Use this to identify and understand BIOS function calls (SWI instructions). GBA games frequently use BIOS routines for decompression, math, and memory operations. Finding these calls reveals data format and helps locate compressed assets.

1. `mesen_find_occurrences(searchString="SWI", cpuType="Gba")` -- find all BIOS calls in the disassembly
2. Identify the SWI number to determine the function:

| SWI # | Function | What it does |
|-------|----------|-------------|
| $05 | VBlankIntrWait | Halt CPU until VBlank (main loop sync) |
| $06 | Div | Signed division: R0/R1 → R0=quotient, R1=remainder |
| $08 | Sqrt | Square root of R0 |
| $0B | CpuSet | Memory copy/fill (word or halfword, general purpose) |
| $0C | CpuFastSet | Fast memory copy/fill (32-bit, 8-word blocks) |
| $11 | LZ77UnCompWram | LZ77 decompress to WRAM (byte writes) |
| $12 | LZ77UnCompVram | LZ77 decompress to VRAM (halfword writes) |
| $13 | HuffUnComp | Huffman decompress |
| $14 | RLUnCompWram | Run-length decompress to WRAM |
| $15 | RLUnCompVram | Run-length decompress to VRAM |

3. For a decompression call (SWI $11/$12), set a breakpoint just before the SWI and check R0 (source pointer) and R1 (destination pointer)
4. `mesen_read_memory` at the source address to see the compressed data header. LZ77 starts with a 4-byte header: byte 0 = $10 (LZ77 marker), bytes 1-3 = decompressed size (24-bit LE)
5. After the SWI returns, `mesen_read_memory` at the destination to see the decompressed output (tile data, tilemap, or other assets)
6. For CpuSet/CpuFastSet calls: R0=source, R1=destination, R2=length|flags (bit 24: 0=copy, 1=fill; bit 26: 0=16-bit, 1=32-bit)

**Tip:** LZ77-compressed data is extremely common in GBA ROMs -- graphics, tilemaps, and sometimes even code are stored compressed. The $10 header byte is a reliable marker. Huffman ($20 header) and RLE ($30 header) are also used but less common. When you find a decompression call, the source address points into ROM and the destination is usually VRAM or EWRAM. Tracing these calls maps out where the game stores its graphical assets. Some games use custom decompression instead of BIOS calls -- look for tight loops with shift/mask operations if SWI search yields few results.

## Workflow H: BG Mode & Sprite Inspection

Use this to understand how the game renders its graphics. The GBA supports 6 display modes with different BG layer configurations.

1. `mesen_get_state(component="ppu", cpuType="Gba")` -- check current BG mode and forced blank state
2. `mesen_get_tilemap_info(cpuType="Gba", layer=0)` -- inspect BG0: dimensions, tile size, BPP, scroll position, addresses
3. `mesen_get_tilemap_info(cpuType="Gba", layer=1)` -- inspect BG1 (if available in current mode)
4. `mesen_get_tilemap_tile_info(x=120, y=80, cpuType="Gba", layer=0)` -- inspect a specific tile at screen coordinates
5. `mesen_get_sprite_list(cpuType="Gba")` -- list all 128 OAM sprites with positions, tile indices, palettes, sizes, affine transforms, priority
6. `mesen_get_palette(cpuType="Gba")` -- view all palette colors. GBA uses 15-bit RGB (5 bits per channel): 256 BG colors ($05000000) + 256 OBJ colors ($05000200)
7. `mesen_read_memory(address="$0000", length=64, memoryType="GbaVideoRam")` -- read raw VRAM tile data
8. To find what updates a BG layer, `mesen_breakpoint(action="set", address="$04000008", type="Write", memoryType="GbaIntWorkRam", cpuType="Gba")` -- watch BG0CNT register writes to catch mode/tileset changes

**BG Mode quick reference:**

| Mode | BG0 | BG1 | BG2 | BG3 | Notes |
|------|-----|-----|-----|-----|-------|
| 0 | Tile | Tile | Tile | Tile | 4 regular tile layers, most common |
| 1 | Tile | Tile | Affine | -- | 2 regular + 1 rotation/scaling layer |
| 2 | -- | -- | Affine | Affine | 2 rotation/scaling layers |
| 3 | -- | -- | Bitmap | -- | 240x160 16-bit direct color framebuffer |
| 4 | -- | -- | Bitmap | -- | 240x160 8-bit indexed, double-buffered |
| 5 | -- | -- | Bitmap | -- | 160x128 16-bit direct color, double-buffered |

**Tip:** Mode 0 is by far the most common in GBA games. Tile layers use either 4bpp (16 colors from a 16-color sub-palette, `GbaBpp4`) or 8bpp (256 colors, `GbaBpp8`). DISPCNT ($04000000) bits 8-11 enable BG0-BG3, bit 12 enables sprites. OAM at $07000000 holds 128 sprites, each 8 bytes (2 attributes of 16 bits + 16-bit tile/palette info + 16-bit affine parameter index). Affine sprites support rotation and scaling via the affine parameter matrix (PA/PB/PC/PD interleaved in OAM). GBA VRAM is 96KB: $06000000-$06017FFF. In tile modes, VRAM is split into charblocks (16KB each for tile data) and screenblocks (2KB each for tilemaps), with shared address space.
