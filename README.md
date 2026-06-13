# TMS99105 SBC V4 - Paged Memory System

## Overview

The TMS99105 SBC V4 implements a transparent paged memory system providing 1MB of physical RAM via two HM628512BFP-5 512KB SRAM chips (word-wide). The system divides the 64KB address space into 16 × 4KB segments, each independently mappable to one of 16 × 64KB physical pages.

## Architecture

### Schematic
![TMS99105 SBC V4 Schematic](Schematic.png)

### Truth Table
![Memory Mapper Truth Table](TruthTable.svg)

### Memory Map
```
0x0000-0x0FFF  COMMON    Always page 0 - OS, workspace, XOP vectors
0x1000-0xEFFF  PAGED TPA 14 × 4KB segments, independently mappable
0xF000-0xFFFF  ROM       Monitor/OS in EPROM
```

### Physical RAM
- 2 × HM628512BFP-5 (512KB each, word-wide = 1MB total)
- SA0-SA3 select physical page (0-15) within each chip
- SA0 = MSB (TMS99105 big-endian convention)

### Mapper Hardware
```
Component    Function
─────────────────────────────────────────────────────────
6116 SRAM    16 × 8-bit page registers (one per segment)
74LS157      MUX: CRU address (A11-A14) or CPU address (A0-A3) → 6116
GAL22V10     Address decode + SA0-SA3 registered outputs
```

### GAL22V10 (U44) - Rev 24Q
-Name     TMS9900_SBC_V4;
PartNo   00;
Date     06/05/26;
Revision 24Q;
Designer A. Cameron;
Company  None;
Assembly TMS99105 SBC;
Location U44;
Device   g22v10;

/*
  TMS9900 SBC V4 - GAL22V10 Memory Management
  TMS9900 BIG ENDIAN - A0 is MSB, A15 is LSB

  Rev 24: CRU write to 6116 mapper. 373 and MAP_WIN removed.
    6116 written via CRU (LDCR/IOWR) - no memory bus involvement.
    6116 /WE = IOWR
    6116 /CS = CRU_EN (or GND if dedicated)
    6116 /OE = MRD (outputs valid during read cycles)
    6116 D4-D7 direct to GAL pins 7,8,9,13 (no 373 latch)
    SA0-SA3 driven from PIO_D4-D7 gated by PSEL.

    PSEL LOW  = mapping enabled  - SA reflects 6116 output
    PSEL HIGH = mapping disabled - SA forced zero (COMMON behaviour)

    0xE800-0xEFFF freed up - no longer MAP_WIN
    No dummy reads needed
    No 373 latch needed
    No OR gate needed
    Code can execute from paged memory

  Address regions:
    COMMON  = !A0 & !A1 & !A2 & !A3  (0x0000-0x0FFF) always page 0
    ROM     =  A0 &  A1 &  A2 &  A3  (0xF000-0xFFFF)
    RAM     = everything else during MEM cycle
*/

/* ---- INPUT PINS ---- */
PIN  1 =  CLK;          /* 16MHz oscillator - wire TMS99105_CLKIN    */
PIN  2 =  A0;           /* CPU address bit 0 (MSB)                   */
PIN  3 =  A1;           /* CPU address bit 1                         */
PIN  4 =  A2;           /* CPU address bit 2                         */
PIN  5 =  A3;           /* CPU address bit 3                         */
PIN  6 =  A4;           /* CPU address bit 4                         */
PIN  7 =  PIO_D4;     /* PIO data bus bit 4                        */
PIN  8 =  PIO_D5;     /* PIO data bus bit 5                        */
PIN  9 =  PIO_D6;     /* PIO data bus bit 6                        */
PIN 10 = ALATCH;    /* Replaced !MEM with CPU Address Latch Strobe */
PIN 11 =  PSEL;         /* page select (active low = mapping enabled)*/
PIN 13 =  PIO_D7;     /* PIO data bus bit 7                        */

/* ---- OUTPUT PINS ---- */
PIN 23 =  NC23;
PIN 22 =  !ROM_SEL;     /* active low - ROM 0xF000-0xFFFF            */
PIN 21 =  !RAM_SEL;     /* active low - RAM (COMMON + PAGED)         */
PIN 20 =  !PSEL_G;         /* spare - MAP_WIN freed                     */
PIN 19 =  WAIT;         /* active high - wait state                  */
PIN 18 = ALATCH2;   /* Assigned to Pin 18 to clean up logic analyzer tracking */                                    
PIN 17 =  SA0;          /* HM628512 A0 physical page bit 0           */
PIN 16 =  SA1;          /* HM628512 A1 physical page bit 1           */
PIN 15 =  SA2;          /* HM628512 A2 physical page bit 2           */
PIN 14 =  SA3;          /* HM628512 A3 physical page bit 3           */

/* ---- INTERMEDIATE NODES ---- */
IS_ROM    =  A0 &  A1 &  A2 &  A3;

/* ---- COMMON NODES ---- */
IS_COMMON    =  !A0 &  !A1 &  !A2 &  !A3;

/* ---- EQUATIONS ---- */

/* ROM: 0xF000-0xFFFF */
ROM_SEL = IS_ROM;

/* RAM: everything except ROM */
RAM_SEL = !IS_ROM;


/* WAIT: ROM only - RAM runs without wait states */
WAIT = ROM_SEL;

/* SA0-SA3: transparent mapping gated by PSEL
   PSEL LOW  = mapping enabled  - SA driven from 6116 via PIO
   PSEL HIGH = mapping disabled - SA forced zero
   No MAP_WIN decode needed - 0xE800 is now normal RAM
*/
/* Clean internal 1-cycle delayed clock synchronization */
ALATCH2.D = !ALATCH;

/* PSEL Gated with our stabilized latch window */
PSEL_G.D = !PSEL & ALATCH2;

MAP_COND = PSEL_G & !IS_COMMON;
/* IS_ROM exclusion keeps SA0-SA3 clean (zero) during ROM cycles */

SA0.D = PIO_D4 & MAP_COND;

SA1.D = PIO_D5 & MAP_COND;

SA2.D = PIO_D6 & MAP_COND;

SA3.D = PIO_D7 & MAP_COND;


/* Unused outputs */
 NC23 = 'b'0; 
/* NC20 = 'b'0; */


/*=========================================================
  MEMORY MAP
  0x0000-0x0FFF  COMMON  RAM_SEL  SA=0000 always page 0
  0x1000-0xEFFF  PAGED   RAM_SEL  SA=PIO_D4-D7 when PSEL LOW
  0xF000-0xFFFF  ROM     ROM_SEL

  0xE800-0xEFFF now normal paged RAM - MAP_WIN removed

  HARDWARE:
  6116 /WE  = IOWR
  6116 /CS  = CRU_EN (or GND)
  6116 /OE  = MRD
  6116 D4-D7 direct to GAL pins 7,8,9,13
  No 373 latch needed
  No OR gate needed
  PSEL ? GAL PIN 11

  Rev 24C: COMMON hardware enforced via (A0#A1#A2#A3) term
           Power-on safe - no MAP_INIT needed for COMMON
  Rev 24B: Address terms removed - 6116 handles naturally
  Rev 24A: Registered SA outputs clocked by /MRD (PIN 1)
  Rev 24:  CRU write path, PSEL gating, MAP_WIN freed
=========================================================*/

### 6116 Mapper SRAM
```
/CS  = /RAM_SEL & /MAP_SEL (ANDed)
/OE  = NOT(IOWR) via U38B
/WE  = IOWR
D4-D7 → GAL pins 7,8,9,13 (PIO_D4-PIO_D7)
A0-A3 ← 74LS157 Y1-Y4 outputs
```

### 74LS157 MUX (U26)
```
SEL=LOW  (MAP_SEL active)  → A inputs: A11-A14 (CRU write address)
SEL=HIGH (normal memory)   → B inputs: A0-A3   (segment address)
```

### CRU Mapper Interface
- Base address: **0x80C0H**
- 16 registers at 80C0H, 80C2H ... 80DEH
- Written via LDCR (2 bits, page value in high byte)
- Read via STCR

### PSEL (TMS99105 PIN 31)
- LOW = paging enabled (SA driven from 6116)
- HIGH = paging disabled (SA = 0000, flat memory)
- Forced HIGH by: XOP, RTWP, interrupts, all I/O instructions
- Controlled via XOP 2

## Programming Model

### Initialisation
```asm
; Clear all page registers (all segments → page 0)
BL      @MAP_CLR

; Enable paging - leave enabled throughout application
LI      R9, 0100H
PSEL    R9
```

### Page Register Setup
```asm
; MAP_SET - Entry: R9=segment, R0=page
MAP_SET:
    LI      MAP_PORT, 80C0H
    SLA     R9, 1           ; segment * 2
    A       R9, MAP_PORT    ; point to register
    SLA     R0, 8           ; page to high byte
    LDCR    R0, 2           ; write to 6116
    RT
```

### Accessing Paged Memory
```asm
; Set page register
LI      R9, 2       ; segment 2
LI      R0, 5       ; page 5
BL      @MAP_SET

; Enable paging (once at startup - leave enabled)
LI      R9, 0100H
PSEL    R9

; Access paged memory transparently
LI      R2, 2000H   ; segment 2 base address
MOV     *R2, R1     ; read from segment 2, page 5
MOV     R1, *R2     ; write to segment 2, page 5
```



## Test Utility - MAPDEBUG

Load at 0x0500. Parameters at fixed addresses:
```
0502: COMMAND
0504: SEG_PAGE   HIGH=PAGE(0-15)  LOW=SEG(0-15)
0506: VALUE      write value (CMD 5)
```

### Commands
```
1=PSEL_EN   Enable mapping
2=PSEL_DIS  Disable mapping
3=SMREG     Set map register + verify (0504=SEG:PAGE)
4=RMREG     Read map register (0504=SEG:PAGE)
5=WPGSEG    Write value to SEG:PAGE (0504=SEG:PAGE 0506=VAL)
6=RPGSEG    Read from SEG:PAGE (0504=SEG:PAGE)
7=MAP_CLR   Initialise - set all segments to page 0
8=MAP_LST   List all map registers
9=RDLOOP    Continuous read loop (reset to exit)
A=PGLOOP    Cycle all pages on segment (reset to exit)
B=HELP      Help
C=SETALL    Set seg N = page N for segs 0-14, seg15=page0
D=SCYC      PSEL boundary cycle for scope analysis (reset to exit)
E=INIT      Init segs 0-14 = seg N, seg15=page0
F=CODE      Cross-page code execution test
```

### Test Sequence
```
; Prove page isolation:
E 0502 0007 / 500G          ; clear all registers
E 2000 1234                 ; write 1234 to page 0 seg 2
E 0502 0005
E 0504 0102                 ; seg 2, page 1
E 0506 A5A5 / 500G          ; write A5A5 to page 1
E 0502 0006 / 500G          ; read back - should be A5A5
2000                        ; page 0 should still be 1234

; Code execution test:
E 0502 000F / 500G          ; RESULT:BEEF PASS
```

## Hardware Summary

| Signal | Source | Destination | Function |
|--------|--------|-------------|----------|
| MAP_SEL | U30B pin 10 | U26 SEL, 6116 /CS | CRU mapper select |
| RAM_SEL | GAL PIN 21 | 6116 /CS (ANDed) | RAM cycle select |
| IOWR | U14 | 6116 /WE, /OE via U38B | CRU write strobe |
| PIO_D4-D7 | 6116 Q4-Q7 | GAL PIN 7,8,9,13 | Page value to GAL |
| SA0-SA3 | GAL PIN 17,16,15,14 | HM628512 A0-A3 | Physical page select |
| PSEL | CPU PIN 31 | GAL PIN 11 | Mapping enable |
| 16MHz CLK | Oscillator | GAL PIN 1 | SA register clock |


