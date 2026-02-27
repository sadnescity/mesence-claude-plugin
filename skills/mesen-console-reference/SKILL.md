---
description: "Per-console reference for Mesen2: memory types, CPU types, tile formats, cheat types for NES, SNES, Game Boy, GBA, PC Engine, SMS/Game Gear, WonderSwan"
---

## Overview

Mesen2 is a multi-system emulator. Many MCP tools require `memoryType` or `cpuType` parameters that are console-specific. Use `mesen_list_memory_types` and `mesen_list_cpu_types` at runtime to discover exact values for the current ROM. This reference lists the common types per console.

## MCP Resources

The emulator also serves MCP resources with detailed docs:

- `mesen://memory-map/{consoleType}` -- Memory map (NES, SNES, GB, GBA, PCE, SMS, WS)
- `mesen://cpu-instructions/{cpuArch}` -- CPU instruction reference (6502, 65816, Z80, ARM7)

## NES

| Property | Values |
|----------|--------|
| CPU Types | `Nes` |
| Memory Types | `NesPrgRom`, `NesWorkRam`, `NesSaveRam`, `NesChrRom`, `NesChrRam`, `NesNametableRam`, `NesInternalRam`, `NesSpriteRam`, `NesPaletteRam` |
| CPU Architecture | 6502 (Ricoh 2A03) |
| Tile Formats | `NesBpp2` |
| Cheat Types | `NesGameGenie`, `NesCustom` |
| Address Space | $0000-$FFFF (CPU), $0000-$3FFF (PPU) |

## SNES

| Property | Values |
|----------|--------|
| CPU Types | `Snes` (main 65816), `Spc` (SPC700 audio), `Sa1` (SA-1 coprocessor), `Gsu` (Super FX), `Cx4` (Cx4), `NecDsp` (NEC DSP) |
| Memory Types | `SnesPrgRom`, `SnesWorkRam`, `SnesSaveRam`, `SnesVideoRam`, `SnesSpriteRam`, `SnesCgRam`, `SnesRegister`, `SpcRam`, `SpcRom`, `DspProgramRom`, `DspDataRom`, `DspDataRam`, `Sa1InternalRam`, `GsuWorkRam` |
| CPU Architecture | 65816 (main), SPC700 (audio) |
| Tile Formats | `Bpp2`, `Bpp4`, `Bpp8`, `DirectColor` |
| Cheat Types | `SnesGameGenie`, `SnesProActionReplay` |
| Address Space | $00:0000-$FF:FFFF (24-bit) |

## Game Boy / Game Boy Color

| Property | Values |
|----------|--------|
| CPU Types | `Gameboy` |
| Memory Types | `GbPrgRom`, `GbWorkRam`, `GbCartRam`, `GbHighRam`, `GbVideoRam`, `GbSpriteRam`, `GbBootRom` |
| CPU Architecture | Z80 variant (Sharp LR35902) |
| Tile Formats | `Bpp2` |
| Cheat Types | `GbGameGenie`, `GbGameShark` |
| Address Space | $0000-$FFFF |

## Game Boy Advance

| Property | Values |
|----------|--------|
| CPU Types | `Gba` |
| Memory Types | `GbaPrgRom`, `GbaBootRom`, `GbaSaveRam`, `GbaIntWorkRam`, `GbaExtWorkRam`, `GbaVideoRam`, `GbaSpriteRam`, `GbaPaletteRam` |
| CPU Architecture | ARM7TDMI (ARM + THUMB) |
| Tile Formats | `GbaBpp4`, `GbaBpp8` |
| Cheat Types | (none built-in) |
| Address Space | 32-bit (0x00000000-0x0FFFFFFF mapped) |

## PC Engine / TurboGrafx-16

| Property | Values |
|----------|--------|
| CPU Types | `Pce` |
| Memory Types | `PcePrgRom`, `PceWorkRam`, `PceSaveRam`, `PceCdromRam`, `PceCardRam`, `PceAdpcmRam`, `PceArcadeCardRam`, `PceVideoRam`, `PceVideoRamVdc2`, `PceSpriteRam`, `PceSpriteRamVdc2`, `PcePaletteRam` |
| CPU Architecture | HuC6280 (65C02 derivative) |
| Tile Formats | `PceBpp4` |
| Cheat Types | (none built-in) |
| Address Space | $0000-$FFFF (21-bit banked) |

## SMS / Game Gear

| Property | Values |
|----------|--------|
| CPU Types | `Sms` |
| Memory Types | `SmsPrgRom`, `SmsWorkRam`, `SmsCartRam`, `SmsBootRom`, `SmsVideoRam`, `SmsPaletteRam`, `SmsPort` |
| CPU Architecture | Z80 |
| Tile Formats | `SmsBpp4` |
| Cheat Types | (none built-in) |
| Address Space | $0000-$FFFF |

## WonderSwan

| Property | Values |
|----------|--------|
| CPU Types | `Ws` |
| Memory Types | `WsPrgRom`, `WsWorkRam`, `WsCartRam`, `WsCartEeprom`, `WsBootRom`, `WsInternalEeprom`, `WsPort` |
| CPU Architecture | V30MZ (NEC, x86-like) |
| Tile Formats | `WsBpp2` |
| Cheat Types | (none built-in) |
| Address Space | 20-bit (1MB, segmented) |

## Notes

- Memory type names are exact enum values -- use them as-is in tool parameters.
- Not all memory types are available for every game. Some depend on mapper/cartridge type.
- Always call `mesen_list_memory_types` and `mesen_list_cpu_types` at runtime to get the exact set for the current ROM.
- SNES has the most CPU types due to coprocessor cartridges (SA-1, Super FX, Cx4).
