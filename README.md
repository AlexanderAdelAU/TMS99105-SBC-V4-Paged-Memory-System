# TMS99105 SBC V4 — Transparent Memory Mapper

## Architecture

The system uses a **6116 SRAM** as a page register file, programmed via CRU, with a **GAL22V10** generating the physical bank-select signals. The design is called "transparent" because page 0 maps every segment to its natural physical address — no initialisation is needed for common memory and the system boots safely without MAP_INIT.

```
CPU Address Bus (A0-A15)
        │
        ├──► GAL22V10 ──► ROM_SEL  (0xF000-0xFFFF)
        │              ──► RAM_SEL  (everything else)
        │              ──► SA0-SA3  (physical bank select → HM628512)
        │
        └──► 6116 SRAM (page register file)
                │
                └── D4-D7 ──► GAL pins 7,8,9,13
                    A0-A3 ◄── CRU address decode
                    /WE   ◄── IOWR
                    /CS   ◄── CRU_EN
                    /OE   ◄── MRD
```

## Memory Map

| Range | Region | Behaviour |
|---|---|---|
| 0x0000–0x0FFF | COMMON | Always page 0 — forced transparent by GAL |
| 0x1000–0xBFFF | PAGED | SA0-SA3 driven from 6116 when PSEL active |
| 0xC000–0xFFFF | PROTECTED | Shell / BDOS / ROM — segments ≥ 12 bypass mapper |

## GAL22V10 Equations (Rev 24T)

```
IS_ROM     =  A0 &  A1 &  A2 &  A3;          /* 0xF000-0xFFFF         */
IS_COMMON  = !A0 & !A1 & !A2 & !A3;          /* 0x0000-0x0FFF         */

ROM_SEL    = IS_ROM;
RAM_SEL    = !IS_ROM;

PSEL_G.D   = !PSEL;                           /* registered — clocked  */

/* SA lines active only outside COMMON and ROM, when PSEL enabled      */
SA0.D = PIO_D4 & PSEL_G & (A0 # A1 # A2 # A3);
SA1.D = PIO_D5 & PSEL_G & (A0 # A1 # A2 # A3);
SA2.D = PIO_D6 & PSEL_G & (A0 # A1 # A2 # A3);
SA3.D = PIO_D7 & PSEL_G & (A0 # A1 # A2 # A3);
```

**PSEL LOW = mapping enabled.** SA0-SA3 follow 6116 outputs gated by PSEL_G.  
**PSEL HIGH = mapping disabled.** SA0-SA3 forced to zero — all segments transparent.

The `!IS_ROM` exclusion was removed (Rev 24T) to save product terms. ROM cycles never assert RAM_SEL so the SA state during ROM access is irrelevant.

## 6116 Register File

One 2-bit register per segment (16 segments × 2 bits = 32 CRU bits).

| Signal | Connection |
|---|---|
| /WE | IOWR — written via LDCR |
| /CS | CRU_EN |
| /OE | MRD — outputs valid during read |
| D4-D7 | Directly to GAL pins 7, 8, 9, 13 |
| A0-A3 | Decoded from CRU address |

CRU base address: **0x80C0**  
Segment N register: CRU address `0x80C0 + N×2`

## Programming the Registers

### MAP_INIT — clear all segments to page 0

```asm
MAP_INIT:
    LI   R12, 080C0H     ; CRU base
    CLR  R0              ; page 0
    LI   R1, 16          ; 16 segments
MAPI_L:
    LDCR R0, 2           ; write 2-bit page value
    INCT R12             ; next segment
    DEC  R1
    JNE  MAPI_L
    RT
```

### MAP_SET — set one segment to a page

```asm
; Entry: R9 = segment (0-15), R0 = page (0-3)
MAP_SET:
    LI   R12, 080C0H
    SLA  R9, 1           ; segment × 2 = CRU bit offset
    A    R9, R12
    SLA  R0, 8           ; page to bits 15-14 for LDCR (MSB first)
    LDCR R0, 2
    RT
```

### PSEL — enable / disable the mapper

PSEL is **XOP 2**. R9 holds the control value.

```asm
LI   R9, 0100H   ; enable  (PSEL LOW via GAL)
PSEL R9

CLR  R9          ; disable (PSEL HIGH — all transparent)
PSEL R9
```

XOP calls execute with PSEL temporarily transparent (the XOP handler is in ROM, segment 15, which is unprotected). The workspace is always in segment 0 (COMMON), so it remains visible regardless of mapper state.

## Loader Sequence (LOADERCODE)

When the shell loads a paged `.COM` file (link99 v3.9 format):

1. **MAP_INIT** — clear all registers to page 0
2. **Read first sector** to TPA (0x0500)
3. **Detect sentinel** — `FFFF FFFF FFFF` at file offset 6 identifies a paged file
4. **Walk page map** — for each `[start, end, page]` entry call MAP_SET for every segment in range
5. **Enable PSEL** — `LI R9, 0100H / PSEL R9`
6. **Copy first sector** from staging buffer to paged virtual address
7. **Additional sectors** — SETDMA to paged destination, RDSEQ writes directly into physical bank via mapper (BDOS passes DMA address unchanged)
8. **Launch** — `CLR R15 / ORI R15, 0087H / LST R15 / B *R8`

The launch address (R8) is saved from the page map start field before the copy loop advances R2.

## link99 Paged File Format

```
[start:word][end:word][page:word]   ← page map entry (one per segment range)
[FFFF][FFFF][FFFF]                  ← sentinel at byte offset 6
[size:word]                         ← code block byte count
[code data ...]                     ← relocatable code + data
```

Programs link with `-P<n>` to assign a page number (0-3).  
Segments 0 (COMMON) and 12-15 (shell/ROM) are never remapped by the loader.
