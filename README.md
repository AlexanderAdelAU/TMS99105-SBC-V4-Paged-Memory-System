# TMS99105 SBC Transparent Paged Memory

![Paged Memory Mapper Schematic](Schematic.png)

*Schematic showing the GAL22V10 (U44) page mapper, 6116 mapper RAM (IC4), 74LS157 segment address multiplexer (U26), and CRU interface (U31). Note: when PSEL_G is high, SA0-SA3 are forced low — all segments map to physical page 0.*

## Overview

This document describes the overlay manager implementation for the TMS99105 SBC. Overlays allow programs larger than a single 4KB memory segment to run by swapping code segments in and out of a fixed virtual address window. This was developed as a proof of concept and stepping stone toward running the native C compiler, which is too large for a single segment.

## Hardware — GAL22V10 Page Mapper

The SBC uses a GAL22V10 (U44) as a page mapper providing:

- 16 virtual segments × 4KB = 64KB virtual address space
- 16 physical pages × 4KB = 64KB physical RAM (from 512KB HM628512)
- CRU-based programming via `LDCR`/`STCR` at `MAP_WIN_BASE = 0x80C0`
- PSEL controlled via ST7 in the status register (set by XOP 2 handler)

### Memory Map

```
0x0000-0x00AF  Interrupt Vectors (7 user vectors at 0x00B0)
0x00B0-0x012F  Interrupt Workspace (8 x 16 bytes)
0x0130-0x022F  XOP Workspace (16 x 16 bytes)
0x0230-0x04FF  Common Workspace, Command Line, FCB, Page Variables
0x0280         Common Stack Pointer
0x02A0         OVL_PRINT_VEC (overlay print callback vector)
0x0500-0x06FF  STAGING buffer (sector I/O)
0x1000-0xCFFF  TPA — Paged Program Area (page 0 by default)
0xD000         SHELL
0xE000         BDOS
0xF000         ROM / DISC_MONITOR
```

### Key Equates

```asm
MAP_WIN_BASE:   EQU  080C0H      ; CRU base for mapper registers
BYTEWIDE:       EQU  2           ; CNT=2 = parallel byte transfer mode
PSEL_EN:        EQU  0100H       ; non-zero = enable PSEL (any non-zero value)
OVL_SEG:        EQU  2           ; virtual segment used for overlay slot
OVL_CRU:        EQU  MAP_WIN_BASE+(OVL_SEG*2)  ; = 0x80C4
OVL_PRINT_VEC:  EQU  02A0H       ; page variables area — print callback
STAGING:        EQU  0500H       ; sector staging buffer
FLATBASE:       EQU  1000H       ; flat program load address
```

## Programming the Mapper

### LDCR/STCR in Parallel Mode

On the TMS99105A, `LDCR`/`STCR` with `CNT=2` (`BYTEWIDE`) operate in **parallel byte transfer mode** — transferring a full byte, not 2 bits.

- `LDCR Rsrc,BYTEWIDE` — writes the **high byte** of Rsrc to the CRU address in R12
- `STCR Rdst,BYTEWIDE` — reads a byte from CRU into the **high byte** of Rdst

To program a mapper register:

```asm
MAP_SET:
    ; Entry: R9 = segment number, R0 = physical page number
    LI   R12,MAP_WIN_BASE       ; CRU base
    SLA  R9,1                   ; segment * 2 = CRU bit offset
    A    R9,R12                 ; R12 = CRU address for this segment
    SLA  R0,8                   ; MOVE PAGE FROM LOW BYTE TO HIGH BYTE
                                ; (LDCR BYTE TRANSFER USES HIGH BYTE)
    LDCR R0,BYTEWIDE            ; program the mapper register
    RT
```

To read back a mapper register:

```asm
    LI   R12,OVL_CRU
    STCR R1,BYTEWIDE            ; page number in HIGH BYTE of R1
    ; R1 high byte now contains current page — use directly with LDCR
    ; DO NOT SLA R1,8 again — it's already in the high byte
```

### PSEL Control

PSEL is controlled via XOP 2. The XOP 2 handler sets/clears ST7 (the map enable bit):

```asm
    ; Enable PSEL
    LI   R9,PSEL_EN             ; any non-zero value
    PSEL R9                     ; XOP 2 — sets ST7, enables mapping

    ; Disable PSEL
    CLR  R9
    PSEL R9                     ; XOP 2 — clears ST7, disables mapping
```

**Important:** XOP entry clears ST7-ST11. The XOP 2 handler sets ST7 via `ORI` on the saved status before `RTWP`, so PSEL state is correctly restored on return.

**Important:** With PSEL enabled (ST7=1), the processor is in **user mode**. Privileged instructions (`LWPI`, `LIMI`, `LST`) must only be executed with PSEL disabled or before PSEL is first enabled.

## Output File Format — Shell V5.x EXE

When overlays are linked with `-P#` flags, `link99` produces a single **EXE file** in Shell V5.x chain-block format. There are no separate overlay files — everything is in one EXE launched directly from the shell.

### Format Marker

The EXE file begins with `FFFF FFFF FFFF` (3 words). The shell uses this as a format marker to distinguish EXE files from flat COM binaries — if the first word is `0xFFFF` it is an EXE, otherwise it is a flat COM loaded at a fixed address. The EXE loader itself ignores the sentinel and goes straight to the chain blocks.

### Chain Blocks

After the sentinel the file consists of chain blocks, one per page:

```
[next_offset:word][page:word][start:word][size:word][data...]
```

- `next_offset` — byte distance from this header to the next (0 = last block)
- `page` — physical page number (0 = common memory, no mapper programming needed)
- `start` — virtual load address
- `size` — byte count of data following the header

Large modules are automatically split into multiple blocks, each limited to `0x1F8` bytes (one 512-byte sector minus the 8-byte header) for Shell V5.8 loader compatibility.

The loader reads two sectors into staging at `0x0500`, walks the chain, programs the mapper for each page>0 block, copies data to the virtual address, then follows `next_offset` to the next block. When `next_offset=0` it launches at the first block's start address.

### Link Command

```
link99 basic.exe -O0x1000 -P0 bascore.r99 ovlmgr.r99 -P2 basovl.r99 -P3 basmath.r99
```

| Flag | Meaning |
|------|---------|
| `-O0x1000` | Sets `cbase=0x1000` — required for correct external symbol resolution when `-P0` modules use `AORG 1000H` |
| `-P0` | Tags modules as page-0 — allows the linker to resolve `EXT`/`ENT` across them |
| `-P2` | Places overlay A at virtual `0x2000`, physical page 2 |
| `-P3` | Places overlay B at virtual `0x2000`, physical page 3 |

## Overlay Manager — OVLMGR

OVLMGR is a relocatable module linked with the main program in page 0. It manages swapping overlays into the fixed virtual slot at `0x2000`.

### OVL_PAGES Table

```asm
OVL_PAGES:
    WORD  0         ; unused (index 0)
    WORD  2         ; overlay A = physical page 2
    WORD  3         ; overlay B = physical page 3

CURRENT_OVL:
    WORD  0         ; currently mapped overlay ID (0 = none)

SAVE_R3:
    WORD  0         ; R3 scratch save — no stack use during mapper programming
```

### OVLMGR_INIT

Call once at program startup:

```asm
    CALL  @OVLMGR_INIT     ; clears CURRENT_OVL, enables PSEL
```

### OVLMGR

Call before each overlay function:

```asm
    LI    R1,OVL_A         ; overlay ID (1=A, 2=B)
    CALL  @OVLMGR          ; swap if needed, re-enables PSEL on return
```

OVLMGR compares R1 to CURRENT_OVL — if already mapped, returns immediately. Otherwise it programs segment 2 with the new page. R1 is restored from CURRENT_OVL on return; R3 is saved to a scratch word to avoid any stack activity during the mapper programming window:

```asm
OVLMGR:
    C     R1,@CURRENT_OVL
    JEQ   OVLMGR_RET           ; already loaded
    MOV   R1,@CURRENT_OVL      ; save overlay ID
    MOV   R3,@SAVE_R3          ; save R3
    MOV   R1,R3
    SLA   R3,1                 ; word offset into OVL_PAGES
    MOV   @OVL_PAGES(R3),R0    ; R0 = physical page number
    SLA   R0,8                 ; page to high byte for LDCR
    CLR   R9
    PSEL  R9                   ; disable PSEL before programming mapper
    LI    R12,OVL_CRU
    LDCR  R0,BYTEWIDE          ; program segment 2
    LI    R9,PSEL_EN
    PSEL  R9                   ; re-enable PSEL
    MOV   @SAVE_R3,R3          ; restore R3
    MOV   @CURRENT_OVL,R1      ; restore R1
OVLMGR_RET:
    RET
```

**Note:** You cannot use R0 as an index register on TMS9900 — `MOV @TABLE(R0),R0` does not index. Use R3 or any other non-zero register.

## Overlay Code Structure

Each overlay is assembled at `AORG 2000H` with a fixed entry point table:

```asm
    AORG  2000H

OVLA_FUNC1:  B  @OVLA_F1      ; 0x2000 — function 1 entry
             NOP
OVLA_FUNC2:  B  @OVLA_F2      ; 0x2004 — function 2 entry
             NOP
```

## Calling Overlays from the Main Program

### Print Callback Vector

Overlays cannot directly call routines in the main program since their addresses are not known at overlay assembly time. A callback vector in COMMON solves this:

```asm
OVL_PRINT_VEC:  EQU  02A0H    ; page variables area
```

The main program installs the callback after `OVLMGR_INIT`:

```asm
    CALL  @OVLMGR_INIT
    LI    R0,PRINT
    MOV   R0,@OVL_PRINT_VEC   ; store PRINT address in common vector
```

The overlay calls it via register indirect — **load the address first, then call through the register**:

```asm
    MOV   @OVL_PRINT_VEC,R0   ; load PRINT address from vector
    CALL  *R0                  ; call PRINT via register indirect
```

This is equivalent to a C function pointer: `(*print_func)(str)`.

An alternative is a **trampoline** — store a `B` opcode and address at the vector location:

```
02A0: 0460   ; B opcode
02A2: xxxx   ; PRINT address
```

Then `CALL @0x02A0` jumps through the trampoline with a single indirection.

### CALL and RET

Use `CALL` (XOP 6) and `RET` (XOP 7) throughout — they handle workspace and return address management correctly across paged boundaries:

```asm
    DXOP  CALL,6
    DXOP  RET,7

    CALL  @OVLMGR             ; call overlay manager
    CALL  @OVL_FUNC1          ; call overlay function at 0x2000
```

Overlays return with `RET`:

```asm
OVLA_F1:
    LI    R1,TXT_OVLA1
    MOV   @OVL_PRINT_VEC,R0
    CALL  *R0
    RET                        ; return to caller
```

## Example Application

The overlay mechanism scales naturally to larger programs. The BASIC99 interpreter for example splits across two overlays — I/O handling in page 2 and numeric expression evaluation in page 3 — giving it effectively 12KB of code space while launching as a single EXE from the shell. The interpreter core selects the appropriate overlay before each operation:

```asm
    LI    R1,OVL_B             ; select maths overlay
    CALL  @OVLMGR
    MOV   R6,R1                ; R1 -> expression string
    CALL  @OVL_EVAL            ; R0 = integer result
```

## Complete Test Run

```
%LOAD OVLA.COM
Loaded
%LOAD OVLB.COM
Loaded
%OVLTEST
Overlay POC Test Starting
Test 1: A.Func1 (first load)
Overlay A - Function 1
Test 2: B.Func1 (swap A->B)
Overlay B - Function 1
Test 3: A.Func2 (swap B->A)
Overlay A - Function 2
Test 4: A.Func1 (no swap)
Overlay A - Function 1
Overlay POC Test COMPLETE
%
```
