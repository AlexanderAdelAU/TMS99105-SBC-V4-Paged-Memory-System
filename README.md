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

## Overlay File Format

Overlay COM files use a pagemap header so the loader knows where to place the code:

```
FFFF FFFF FFFF        opening sentinel (3 words)
start  end   page     pagemap entry: virtual start, end, physical page
FFFF FFFF FFFF        closing sentinel (3 words)
size                  block size in bytes (1 word)
[code bytes...]       overlay code
```

Example for OVLA (virtual 0x2000, physical page 2):
```
FFFF FFFF FFFF
2000 2062 0002
FFFF FFFF FFFF
0062
[62 hex bytes of code]
```

Overlays are assembled with `AORG` at the virtual address and linked with the `-P` flag:

```
r99  OVLA SCHCLC
link99 -P2 ovla.COM ovla.R99       ; page 2
link99 -P3 ovlb.COM ovlb.R99       ; page 3
```

## Loading Overlays — DOLOAD Shell Command

The shell `LOAD` command reads a COM file into paged memory without executing it:

```
%LOAD OVLA.COM      → loads OVLA into physical page 2 at virtual 0x2000
%LOAD OVLB.COM      → loads OVLB into physical page 3 at virtual 0x2000
```

DOLOAD:
1. Parses filename from command line
2. Opens file, reads to STAGING (`0x0500`)
3. Parses pagemap header — gets segment, page, block size
4. Calls `MAP_SET` to program the mapper
5. Enables PSEL, copies code from STAGING to virtual address
6. Disables PSEL, returns to shell

## Linking a Program with Overlays — EXE Format

When overlays are linked directly into an EXE (rather than loaded separately as COM files), the `link99` paged EXE format is used. The linker writes one chain block per page:

```
EXE CHAIN BLOCK pg=0 start=1000 size=...   ; main program + overlay manager
EXE CHAIN BLOCK pg=2 start=2000 size=...   ; overlay A
EXE CHAIN BLOCK pg=3 start=2000 size=...   ; overlay B
```

The shell EXE loader programs the mapper and copies each block to its virtual address automatically on launch.

The link command for a program with two overlays and a separate overlay manager module:

```
link99 prog.exe -O0x1000 -P0 main.r99 ovlmgr.r99 -P2 ovla.r99 -P3 ovlb.r99
```

| Flag | Meaning |
|------|---------|
| `-O0x1000` | Sets `cbase=0x1000` for page-0 modules — required for correct external symbol resolution when `AORG 1000H` is used |
| `-P0` | Tags page-0 modules — allows the linker to resolve `EXT`/`ENT` across them |
| `-P2` | Places overlay A at virtual `0x2000`, physical page 2 |
| `-P3` | Places overlay B at virtual `0x2000`, physical page 3 |

### Why `-O0x1000` Is Needed

In page mode the linker sets `cbase=0`. Without `-O0x1000`, external symbols in a page-0 module assembled with `AORG 1000H` resolve to offsets from zero rather than from `0x1000`. `-O0x1000` corrects this so all page-0 modules resolve their external references against the correct runtime base address.

### Why OVLMGR Has No AORG

OVLMGR is assembled without an `AORG` directive, making it purely relocatable. The linker places it immediately after the main program in the page-0 segment. An `AORG 1000H` in OVLMGR would cause the linker to treat its entry point addresses as already-absolute and refuse to add the module offset, producing wrong call targets.

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

## Example Application — BASIC99 Interpreter

The overlay mechanism is used by the BASIC99 interpreter, which splits its functionality across two overlays to fit within the paged memory model:

- **Page 2 (BASOVL)** — I/O handlers: `PRINT` string output, `INPUT` line reading and parsing
- **Page 3 (BASMATH)** — numeric processing: integer expression evaluator, decimal output

The interpreter core (BASCORE) and overlay manager (OVLMGR) both live in page 0 at `0x1000`. BASCORE selects the appropriate overlay before each operation:

```asm
    LI    R1,OVL_B             ; select BASMATH
    CALL  @OVLMGR
    MOV   R6,R1                ; R1 -> expression string
    CALL  @OVL_EVAL            ; R0 = integer result
```

This gives the interpreter effectively 12KB of code space (4KB common + 4KB × 2 overlays) while running from a single EXE launched directly from the shell.

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
0x0400-0x04FF  Variable table (A-Z, 26 words) — free loader area
0x0500-0x09FF  EXE Staging / BASIC program buffer
0x0A00         OVLMGR permanent (booted by shell)
0x0A60-0x0FEF  Free (stack grows down from 0x0FF0)
0x0FF0         Stack ceiling (STACKP)
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
STACKP:         EQU  0FF0H       ; stack ceiling — below page boundary
VARTAB:         EQU  0400H       ; variable table A-Z (26 words)
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

## Source Files

| File | Description | Page |
|------|-------------|------|
| `BASCORE.A99` | BASIC interpreter core — tokeniser, program store, RUN loop | 1 (common) |
| `OVLMGR.A99` | Overlay manager — linked with BASCORE in page 1 | 1 (common) |
| `BASOVL.A99` | I/O overlay — PRINT and INPUT statement handlers | 2 |
| `BASMATH.A99` | Maths overlay — expression evaluator, numeric output | 3 |

## Build Commands

```
r99 bascore SCHCLC
r99 OVLMGR  SCHCLC
r99 BASOVL  SCHCLC
r99 basMATH SCHCLC

link99 basic.exe -O0x1000 -P0 bascore.r99 ovlmgr.r99 -P2 basovl.r99 -P3 basmath.r99
```

### Link Command Flags

| Flag | Meaning |
|------|---------|
| `-O0x1000` | Sets `cbase=0x1000` for page-0 modules so external symbol resolution uses the correct runtime addresses |
| `-P0` | Tags BASCORE and OVLMGR as page-0 modules — allows the linker to resolve `EXT`/`ENT` across them |
| `-P2` | Places BASOVL at virtual `0x2000`, physical page 2 |
| `-P3` | Places BASMATH at virtual `0x2000`, physical page 3 |

### Why `-O0x1000` Is Needed

In page mode the linker sets `cbase=0`. Without `-O0x1000`, external symbols exported by OVLMGR (which has `AORG 1000H`) resolve to offsets from zero rather than from `0x1000`, placing them inside BASCORE code rather than after it. `-O0x1000` corrects this so OVLMGR lands at `0x1498` (immediately after BASCORE's `0x498` bytes) and its symbols resolve correctly.

### Why OVLMGR Has No AORG

OVLMGR is assembled without an `AORG` directive, making it purely relocatable. The linker places it immediately after BASCORE in the page-0 segment. An `AORG 1000H` in OVLMGR would cause the linker to treat its entry point addresses as already-absolute and refuse to add the module offset, producing wrong addresses.

## Overlay File Format

EXE files use the link99 paged EXE format. The linker writes one chain block per page:

```
EXE CHAIN BLOCK pg=0 start=1000 size=...   ; BASCORE + OVLMGR
EXE CHAIN BLOCK pg=2 start=2000 size=...   ; BASOVL
EXE CHAIN BLOCK pg=3 start=2000 size=...   ; BASMATH
```

The shell EXE loader programs the mapper and copies each block to its virtual address.

## Overlay Manager — OVLMGR

OVLMGR is a separate relocatable module linked into page 0 alongside BASCORE. It manages swapping overlays into the fixed virtual slot at `0x2000`.

### OVL_PAGES Table

```asm
OVL_PAGES:
    WORD  0         ; unused (index 0)
    WORD  2         ; overlay A (BASOVL)  = physical page 2
    WORD  3         ; overlay B (BASMATH) = physical page 3

CURRENT_OVL:
    WORD  0         ; currently mapped overlay ID (0 = none)

SAVE_R3:
    WORD  0         ; R3 scratch save — avoids any XOP dependency
```

### OVLMGR_INIT

Call once at program startup:

```asm
    CALL  @OVLMGR_INIT     ; clears CURRENT_OVL, enables PSEL
```

### OVLMGR

Call before each overlay function:

```asm
    LI    R1,OVL_A         ; overlay ID (1=BASOVL, 2=BASMATH)
    CALL  @OVLMGR          ; swap if needed, re-enables PSEL on return
```

OVLMGR compares R1 to CURRENT_OVL — if already mapped, returns immediately. Otherwise it programs segment 2 with the new page. R1 is restored from CURRENT_OVL on return; R3 is saved to a scratch word (no stack use during the swap):

```asm
OVLMGR:
    C     R1,@CURRENT_OVL
    JEQ   OVLMGR_RET           ; already loaded
    MOV   R1,@CURRENT_OVL      ; save overlay ID
    MOV   R3,@SAVE_R3          ; save R3 (no stack use during swap)
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

OVL_DOPRINT:  B  @BAS_PRINT    ; 0x2000 — PRINT handler (BASOVL)
OVL_DOINPUT:  B  @BAS_INPUT    ; 0x2004 — INPUT handler (BASOVL)
```

```asm
    AORG  2000H

OVL_EVAL:     B  @EVAL         ; 0x2000 — expression evaluator (BASMATH)
OVL_NUMOUT:   B  @NUMOUT       ; 0x2004 — numeric output (BASMATH)
```

## Calling Overlays from BASCORE

```asm
    LI    R1,OVL_A             ; select BASOVL
    CALL  @OVLMGR              ; swap if needed
    MOV   R6,R1                ; R1 -> expression/string in PROGBUF
    CALL  @OVL_DOPRINT         ; call at fixed address 0x2000

    LI    R1,OVL_B             ; select BASMATH
    CALL  @OVLMGR
    MOV   R6,R1
    CALL  @OVL_EVAL            ; R0 = result, R1 advanced past expression
```

## Memory Layout Notes

- **VARTAB at `0x0400`** — variable table lives in the free loader area below PROGBUF. The loader occupies `0x0300`-`0x04FF` during loading only; after the EXE starts, this is free RAM. The XOP workspace (`0x0130`-`0x022F`) and page variables (`0x0230`-`0x02FF`) do not reach `0x0400`.
- **PROGBUF at `0x0500`** — grows upward. Upper limit `PROGBUF_END EQU 0A00H` keeps it clear of the stack.
- **Stack at `0x0FF0`** — ceiling is set below the `0x1000` page boundary to avoid any ambiguity about which side of the boundary stack frames live on. Grows downward; normal call depth leaves ample headroom above PROGBUF.
- **BASCORE + OVLMGR in page 0** — both assembled with `AORG 1000H` (BASCORE) and no origin (OVLMGR). Linked with `-P0 -O0x1000`. BASCORE loads at `0x1000`; OVLMGR follows immediately at `0x1498`.

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
