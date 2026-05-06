# TMS99105 SBC V4 — Transparant Paged Memory System

A transparent memory mapper for the TMS99105 single-board computer using a GAL22V10 and a 6116 SRAM — no 74LS612 required.

## Overview

The TMS9900 family was designed with the 74LS612 memory mapper chip as the official paging solution. This design achieves the same result — and more — using a GAL22V10 for address decode and a 6116 SRAM as the mapping table. The result is transparent paged memory: all code and data accesses are automatically mapped without any special CPU instructions, prefix opcodes, or runtime overhead.

Cross-page calls are plain `BL`/`RT`. The programmer writes normal code. The linker and loader handle everything else.

---

## Schematic

A partial schematic is shown to focus on the memory mapping component of the 99105 SBC which can be see as part of that repository.

<img src="schematic.png" alt="System Schematic" width="900">

## Hardware

### CPU and Memory

| Component | Description |
|-----------|-------------|
| CPU | TMS99105 at 16MHz, big-endian (A0=MSB) |
| Main RAM | 2× HM628512BFP-5 (512KB each, word-wide) |
| Mapper RAM | 6116 SRAM — 16-entry mapping table (high byte only; low byte unpopulated, reserved for expansion) |
| Latch | 74LS373 — isolates 6116 D4-D7 from main bus |
| Address decode | GAL22V10 (U39) |
| ROM | 27C32 at 0xF000-0xFFFF |

### Address Map

```
0x0000-0x0FFF   COMMON    always physical page 0, never remapped
0x1000-0xE7FF   PAGED     mapped via 6116 table (16 × 4KB segments)
0xE800-0xEFFF   MAP_WIN   6116 mapping table (read/write)
0xF000-0xFFFF   ROM       always present
```

### Physical Memory Layout

The two HM628512 chips provide 512KB each, word-wide. The GAL drives SA0-SA3 onto the chip address lines A0-A3, while CPU address lines A0-A14 connect to chip pins A4-A18. This gives:

```
Physical address = SA3:SA2:SA1:SA0 : A0:A1:A2:A3 : A4..A14
                   (page 0-15)       (segment 0-15) (4KB offset)
```

16 pages × 64KB = 1MB total physical space, fully addressable.

---

## How the Mapper Works

### Virtual Segments

The CPU's 64KB address space is divided into 16 × 4KB segments, selected by address bits A0-A3 (the four MSBs in big-endian notation):

```
Segment  0   0x0000-0x0FFF   COMMON — GAL forces SA=0, 6116 entry ignored
Segment  1   0x1000-0x1FFF   }
Segment  2   0x2000-0x2FFF   }
...                           }  User paged RAM (segments 1-11)
Segment 11   0xB000-0xBFFF   }
Segment 12   0xC000-0xCFFF   Shell body — protected, never remapped
Segment 13   0xD000-0xDFFF   Shell body — protected, never remapped
Segment 14   0xE000-0xE7FF   Upper shell/OS — protected
             0xE800-0xEFFF   MAP_WIN — 6116 table, hardware-protected by GAL
Segment 15   0xF000-0xFFFF   ROM — ROM_SEL fires, SA never asserted
```

### The 6116 Mapping Table

The 6116 at MAP_WIN (0xE800-0xE80F) holds 16 bytes — one per virtual segment. Each byte contains the physical page number (0-15) that segment maps to. Entry 0 is ignored for COMMON; entry 15 is not used for ROM.

To map virtual segment 3 to physical page 7:
```asm
LI      R0, 0700H       ; page 7 in high byte (MOVB writes high byte)
LI      R9, 0E806H      ; MAP_WIN + (segment 3 * 2) — word-wide entries
MOVB    R0, *R9         ; program the 6116 (high byte only)
```

### The GAL Equations

SA0-SA3 are driven on every paged memory access. No enable signal required:

```
SA0 = D4 & !MEM & !A0 & A1
    # D4 & !MEM & !A0 & A2
    # D4 & !MEM & !A0 & A3
    # D4 & !MEM &  A0 & !A1
    # D4 & !MEM &  A0 &  A1 & !A2
    # D4 & !MEM &  A0 &  A1 &  A2 & !A3 & !A4
```
(SA1, SA2, SA3 follow the same pattern with D5, D6, D7 respectively)

COMMON (A0-A3 all zero) never satisfies any term — SA stays zero, always page 0. The address space partitioning replaces PSEL entirely.

### The 74LS373 Latch

The 6116 data outputs D4-D7 are latched by a 74LS373 (U26):
- MAP_SEL LOW → latch transparent (6116 data flows through)
- MAP_SEL HIGH → latch holds the last value

This ensures SA0-SA3 remain stable throughout the memory cycle even as the data bus changes.

---

## Memory Map (Common Area)

```
0x0000-0x022F   OS vectors, XOP table
0x0230-0x024F   Shell workspace (R0-R15)
0x0250-0x02FF   CPU workspace (LWPI target)
0x0300-0x03FF   Sector buffer / FCB
0x0400-0x0FFF   Loader, OS code (stack grows down from 0x0FFF)
0x1000          TPA — Transient Program Area base
```

### 6116 Layout

```
0xE800-0xE80F   16 map registers (segment 0-15 → physical page)
0xE810-0xEFFF   OS control structures (page map table, free)
```

---

## Software

### Toolchain

| Tool | Description |
|------|-------------|
| `a99` | TMS9900 assembler |
| `link99` v3.7 | Relocatable linker with transparent mapping support |
| `SHELLV50.A99` | Command processor with integrated loader |
| WinCUPL II | GAL equation compiler |

### link99 Usage

```
link99 [-B] [-S] [-G#] [-D#] [-P#] [-M] outname [module/library ...]

  -P#   Page assignment 0-15 for following modules
        Libraries auto-reset to page 0 unless explicitly tagged.
  -G#   Absolute load address (produces .LGO output)
  -D#   Data segment base address
  -B    Big program: force code to disk, maximise symbol table
  -S    Generate Small-C call wrapper to main()
  -M    Monitor: verbose progress output
```

### Linking a Multi-Page Program

```
link99 -G1000 myapp -P0 main.R99 io.R99 clib99.LIB -P1 bigmod.R99
```

Modules tagged `-P0` share physical page 0. Modules tagged `-P1` go to physical page 1. The linker assigns virtual segments automatically and detects collisions at link time — two modules on different pages sharing a virtual segment is a hard error.

### Output File Format

```
[pagemap entry 0]   start:word  end:word  page:word    6 bytes
[pagemap entry N]   ...
[FFFF FFFF FFFF]    sentinel                            6 bytes
[program code]
[program data]
[sector padding]
```

User paged virtual space is segments 1-11 (0x1000-0xBFFF) — 44KB of virtual address space, each segment independently mappable to any of 16 physical pages, giving up to 704KB of physical RAM accessible to user programs.

Non-paged programs have only the sentinel (6 bytes) before the code. The loader handles both cases identically.

---

## Programming Examples

### Initialising the Map Registers

The ROM clears all map registers on startup. To set them manually:

```asm
;--- Clear all 16 entries to page 0 ---
        CLR     R0              ; page 0 value
        LI      R9, 0E800H      ; MAP_WIN base
        LI      R1, 16
CLRLOOP:
        MOVB    R0, *R9         ; MOVB writes high byte
        INC     R9
        DEC     R1
        JNE     CLRLOOP

;--- Map segment 3 to page 1 ---
        LI      R0, 0100H       ; page 1 in high byte
        LI      R9, 0E803H      ; entry for segment 3
        MOVB    R0, *R9
```

### Writing Data to a Remote Page

With transparent mapping, a normal `MOV` suffices once the map registers are set:

```asm
        MOV     R0, @3000H      ; writes to segment 3 → physical page 1
```

No `LDS`, `LDD`, or `XOP` required.

### Executing Code in a Remote Page

Copy the subroutine into the target segment, then call it normally:

```asm
;--- Copy SUBR from page 0 into segment 3 (page 1) ---
        LI      R2, SUBR        ; source in page 0
        LI      R3, 03000H      ; destination in segment 3
        LI      R1, SUBR_LEN
COPYLOOP:
        MOV     *R2+, *R3+
        DEC     R1
        JNE     COPYLOOP

;--- Call it ---
        BL      @03000H         ; plain branch-and-link
        ...                     ; returns here via RT

SUBR:
        A       R1, R0          ; position-independent subroutine
        RT
SUBR_END:
SUBR_LEN EQU    (SUBR_END - SUBR) / 2
```

### Proving Physical Isolation

Poison page 0 at the target address before copying to page 1. If the wrong page is accessed the CPU will execute garbage and crash:

```asm
;--- Write garbage to page 0 at 0x3000 (mapping disabled for segment 3) ---
        LI      R0, 0E803H
        LI      R1, 0000H       ; segment 3 → page 0
        MOVB    R1, *R0

        LI      R0, 0DEADH
        MOV     R0, @03000H     ; poisons page 0

;--- Now remap segment 3 to page 1 and copy good code there ---
        LI      R1, 0100H       ; segment 3 → page 1
        MOVB    R1, *R0
        ... (copy and call as above)

;--- Result 0003 proves page 1 was used ---
```

---

## Why Not the 74LS612?

The 74LS612 was TI's official companion chip for the TMS9900 family. This design replaces it for several reasons:

**Flexibility** — The GAL equations define exactly which address regions are paged. Changing the memory map is a recompile, not a board respin.

**Transparency** — The 74LS612 requires PSEL to gate each access. This design maps all paged accesses automatically — no `LDS`, `LDD`, or XOP calls scattered through application code.

**Expandability** — Adding SA4 for 32 pages requires one more GAL output pin and one equation change. The 74LS612 topology does not extend this cleanly.

**Visibility** — Every mapping decision is documented in WinCUPL equations. The entire mapper is readable and auditable as source code.

**Accessibility** — GAL22V10 chips and USB programmers are cheap and widely available. The 74LS612 is obsolete.

---

## GAL Pin Assignment (Revision 09)

```
PIN  1 = CLK    spare
PIN  2 = A0     CPU address bit 0 (MSB, big-endian)
PIN  3 = A1     CPU address bit 1
PIN  4 = A2     CPU address bit 2
PIN  5 = A3     CPU address bit 3
PIN  6 = A4     CPU address bit 4
PIN  7 = D4     6116 output bit 0 (via 373 latch)
PIN  8 = D5     6116 output bit 1 (via 373 latch)
PIN  9 = D6     6116 output bit 2 (via 373 latch)
PIN 10 = MEM    /MEM strobe (active low)
PIN 11 = NC     spare (formerly PSEL)
PIN 13 = D7     6116 output bit 3 (via 373 latch)

PIN 22 = !ROM_SEL   0xF000-0xFFFF
PIN 21 = !RAM_SEL   0x0000-0xE7FF (COMMON + PAGED)
PIN 20 = !MAP_SEL   0xE800-0xEFFF (6116 map table)
PIN 19 = WAIT       active high wait state
PIN 17 = SA0        HM628512 A0 — physical page bit 0
PIN 16 = SA1        HM628512 A1 — physical page bit 1
PIN 15 = SA2        HM628512 A2 — physical page bit 2
PIN 14 = SA3        HM628512 A3 — physical page bit 3
```

---

## Repository Contents

```
gal/
    tms9900_sbc_v4.pld      GAL22V10 equations (WinCUPL, revision 09)
    tms9900_sbc_v4.jed      Compiled fuse map

linker/
    link99.c                Relocatable linker/loader v3.7
    rel99.h                 REL file format definitions

shell/
    SHELLV50.A99            Command processor with integrated loader v5.0

tests/
    pseltest.a99            Map register read/write test
    segtest.a99             Remote page code execution test

docs/
    memory_map.md           Detailed address map and segment table
```

---

## Revision History

| Version | Change |
|---------|--------|
| GAL rev 09 | Removed PSEL condition from SA0-SA3 — transparent mapping |
| GAL rev 08 | Initial paged mapping with PSEL gate |
| link99 v3.7 | Trampolines removed, segment collision detection, TPA at 0x1000 |
| link99 v3.6 | Trampoline stubs, page map table, -P# flag |
| Shell v5.0 | New transparent loader, LOADERCODE2 removed, stack to 0x0FFF |

---

## Acknowledgements

- Original Small-MAC linker: J. E. Hendrix (1985)
- TMS9900 port and extensions: A. Cameron (1984-2026)
- Original Small/Shell: J. E. Hendrix (1981)
- TMS9900 Shell port: A. Cameron
