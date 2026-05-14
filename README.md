# TMS99105 SBC V4 - Transparent Paged Memory System

## Introduction

The TMS99105 is a 16-bit CPU with a 64KB address space. To support larger programs
and multitasking, the SBC V4 implements a transparent paged memory system that
extends the addressable memory to 1MB without requiring application code to manage
physical addresses directly.

The design philosophy is transparency — application code uses normal memory
addresses and the hardware automatically translates them to the correct physical
location. From the application's perspective, memory simply works. The OS manages
the mapping table to allocate physical pages to logical segments.

---

## Conceptual Overview

### The Problem
The TMS99105 can only directly address 64KB. A real operating system with a shell,
BDOS, device drivers, and multiple applications needs far more than 64KB.

### The Solution — Segmented Paging
The 64KB address space is divided into 16 segments of 4KB each:

```
Segment 0   0x0000-0x0FFF   COMMON  (always physical page 0)
Segment 1   0x1000-0x1FFF   Paged
Segment 2   0x2000-0x2FFF   Paged
...
Segment 15  0xF000-0xFFFF   ROM
```

Each paged segment can be mapped to any one of 16 physical 64KB pages.
With 1MB of physical RAM divided into 16 pages of 64KB each, the full
1MB is addressable by remapping segments as needed.

### Transparency
The mapping is completely transparent to the CPU. When the CPU accesses
address 0x2000, the hardware automatically selects the correct physical
page based on the mapping table. The CPU never knows or cares which
physical page it is accessing.

---

## Physical Memory Layout

```
Two HM628512BFP-5 chips (512KB each, word-wide = 1MB total)

Physical page 0   0x00000-0x0FFFF   64KB
Physical page 1   0x10000-0x1FFFF   64KB
...
Physical page 15  0xF0000-0xFFFFF   64KB

Total: 16 pages x 64KB = 1MB
```

The physical page is selected by SA0-SA3 on the HM628512 address pins A0-A3.
These four bits extend the 16-bit CPU address to a 20-bit physical address.

---

## The Mapping Table (6116 SRAM)

A 6116 2KB SRAM acts as the mapping table. It has 16 entries, one per segment:

```
Entry 0  (A0-A3 = 0000)  Page for segment 0  (always 0 — COMMON)
Entry 1  (A0-A3 = 0001)  Page for segment 1
Entry 2  (A0-A3 = 0010)  Page for segment 2
...
Entry 15 (A0-A3 = 1111)  Page for segment 15
```

The 6116 address pins A0-A3 are connected to the CPU address lines A0-A3
(the top nibble of the 16-bit address, since TMS99105 is big-endian with A0=MSB).
As the CPU executes, the 6116 continuously presents the page number for
whatever segment is currently on the address bus.

### Why CRU for Writing?
The 6116 is written exclusively via the CRU (Communication Register Unit) —
the TMS99105's dedicated I/O bus. This is critical:

- CRU cycles do not assert /MEM — they are completely separate from memory cycles
- No risk of bus contention between the 6116 and the RAM during writes
- No risk of SA0-SA3 being corrupted during a map register write
- Map registers can safely be updated from paged memory without disturbing
  the current mapping

Writing via memory (the original approach) caused the 6116 to fight the
CPU data bus during write cycles, required complex latch timing, and meant
that MAP_SET could only safely be called from COMMON. The CRU approach
eliminates all of these problems.

---

## How SA0-SA3 Are Generated

The GAL22V10D reads D4-D7 from the 6116 output (via the 74LS373 latch)
and drives SA0-SA3 to the HM628512 physical address pins.

### The 74LS373 Latch
The 6116 /OE is driven by /MRD — the 6116 only outputs data during read
cycles. Between read cycles the 6116 outputs are Hi-Z. The 74LS373 latch
sits between the 6116 and the GAL:

```
6116 D4-D7 → 74LS373 → GAL pins D4-D7
```

The 373 LE (latch enable) is driven directly by /MRD:
- /MRD LOW  → 373 transparent → 6116 output passes through to GAL
- /MRD HIGH → 373 holds       → last valid value held for GAL

This ensures D4-D7 at the GAL are always stable and valid.

### Registered Outputs in the GAL
SA0-SA3 are implemented as registered (flip-flop) outputs in the GAL,
clocked by /MRD rising edge (end of each read cycle):

```
SA0.D = PIO_M_D4 & !PSEL & <paged address decode>
SA1.D = PIO_M_D5 & !PSEL & <paged address decode>
SA2.D = PIO_M_D6 & !PSEL & <paged address decode>
SA3.D = PIO_M_D7 & !PSEL & <paged address decode>
```

At the rising edge of /MRD:
- The 6116 has been outputting valid data for the current address
- The 373 has been transparently passing it to the GAL
- The GAL flip-flops capture the correct page value
- SA0-SA3 are held stable until the next /MRD rising edge

During write cycles (/MRD stays HIGH, /MWE asserts):
- No /MRD rising edge → SA flip-flops hold their last value
- The 6116 Hi-Z during /WE is irrelevant — SA is already latched
- The RAM sees stable SA0-SA3 throughout the write cycle

### Address Decode in SA.D Equations
The SA.D equations include address decode terms that exclude:
- COMMON (0x0000-0x0FFF): A0=A1=A2=A3=0 satisfies no term → SA captures 0
- ROM (0xF000-0xFFFF): IS_ROM satisfies no term → SA captures 0

During COMMON or ROM reads, the /MRD rising edge clocks SA=0 which is
harmless — COMMON always maps to page 0, and ROM is not paged.

---

## PSEL — Mapping Enable/Disable

PSEL is a bit in the TMS99105 status register, output on a dedicated pin.
It controls whether mapping is active:

```
PSEL LOW  = mapping enabled  — SA0-SA3 driven from 6116 values
PSEL HIGH = mapping disabled — SA0-SA3 forced zero (all segments = page 0)
```

PSEL is managed by XOP2:
- XOP2 with non-zero R9 → PSEL LOW  → mapping enabled
- XOP2 with R9=0        → PSEL HIGH → mapping disabled

Critically, when an XOP, interrupt, or ROM call occurs, the CPU switches
to a new workspace which has PSEL HIGH in its status register. This
automatically disables mapping during OS/ROM operations without any
software intervention. The application's mapping state is preserved
in its workspace and restored on RTWP.

---

## Timing Walkthrough

### Power-on and Initialisation
```
1. Reset: SA flip-flops = random, 6116 = random
   (Harmless — /MEM not asserted, no RAM access yet)

2. Dispatcher runs at 0x0400 (COMMON, PSEL=HIGH):
   For each segment 0-15:
     LDCR 0x0000, 8    writes 0x0000 to 6116 entry via CRU
   All entries now = page 0

3. PSEL goes LOW (XOP2, R9 non-zero):
   SA equations now active

4. Dispatcher branches to 0x1000:
   First /MRD from 0x1000 (segment 1):
     6116 A0-A3 = 0001 → outputs entry 1 = 0x0000
     373 transparent → D4-D7 = 0 at GAL
     /MRD rising edge → SA captures 0 → page 0 ✓
   CPU fetches from physical page 0 at 0x1000 ✓
```

### Normal Execution
```
CPU fetches instruction from 0xC000 (segment 12, Shell):
  6116 A0-A3 = 1100 → outputs segment 12's page entry
  373 transparent during /MRD → D4-D7 valid
  /MRD rising edge → SA captures page value
  RAM_SEL asserts → HM628512 responds with correct physical page
```

### Remapping a Segment
```
OS remaps segment 3 to page 5:
  LDCR 0x0500, 8    (page 5 in high byte, CRU write)
  6116 entry 3 now = 0x05

Next /MRD from segment 3:
  6116 A0-A3 = 0011 → outputs 0x05
  373 → D4-D7 = 0000 0101
  /MRD rising edge → SA0=1, SA2=1, SA1=SA3=0 → page 5 ✓
```

---

## COMMON — The Unbanked Region

Segment 0 (0x0000-0x0FFF) is COMMON — it is always mapped to physical
page 0 and cannot be remapped. The address decode in the SA.D equations
means COMMON addresses never fire any SA term, so SA captures 0 = page 0.

COMMON is used for:
- System stack and workspace registers
- Interrupt vectors
- OS dispatcher and MAP_INIT routines
- Any code that must remain accessible regardless of mapping state

**Critical rule:** MAP_INIT must always run from COMMON (or ROM). The
dispatcher at 0x0400 handles this. Applications at 0x1000 can assume
the mapper is correctly initialised on entry.

---

## Memory Map

```
0x0000-0x0FFF   COMMON    RAM_SEL  Always physical page 0
0x0400          Dispatcher (MAP_INIT + jump to TPA)
0x1000-0xEFFF   Paged TPA RAM_SEL  SA from 6116 when PSEL LOW
0xF000-0xFFFF   ROM       ROM_SEL
```

### Typical OS Layout
```
Segment 0   0x0000  COMMON    OS workspace, stack, dispatcher
Segment 1   0x1000  TPA start Application code page 1
...
Segment 12  0xC000  Shell
Segment 13  0xD000  BDOS
Segment 14  0xE000  Reserved OS
Segment 15  0xF000  ROM
```

---

## Hidden CRU Storage in the 6116

The 6116 is a 2KB SRAM. Only A0-A3 are used for the 16 page registers.
Address bits A4-A10 provide 2032 bytes of completely hidden storage,
accessible only via CRU read/write (STCR/LDCR).

Properties:
- Invisible to normal memory accesses, DMA, and memory scans
- Cannot be read or written by application code using normal MOV instructions
- Only accessible to code that knows the CRU base address
- Survives across PSEL changes and context switches
- Acts as a secure private scratchpad for the OS

Potential uses:
- Process control blocks (PCBs) for multitasking
- OS mapping tables separate from user-accessible memory
- Security tokens or session keys
- Hardware configuration registers
- Persistent state across warm resets (with battery backup)

---

## Hardware Summary

| Component | Part      | Function |
|-----------|-----------|----------|
| U44       | GAL22V10D | Address decode, SA0-SA3 registered outputs |
| IC4       | 6116      | 16-entry mapping table, hidden CRU storage |
| U26       | 74LS373   | D4-D7 latch — holds page value between /MRD pulses |
| U45       | HM628512  | 512KB RAM high byte (x2 = 1MB word-wide) |
| U31       | 74LS245   | CRU IODATA high byte bus buffer |
| U23C      | 74LS00    | MAP_REG_SEL generation |
| U27A      | 74LS04    | Signal inversion |

---

## GAL22V10D Pin Assignment (Rev 24A)

| Pin | Direction | Signal     | Description |
|-----|-----------|------------|-------------|
| 1   | Input     | CLK        | /MRD — clocks SA flip-flops |
| 2   | Input     | A0         | CPU address MSB |
| 3   | Input     | A1         | CPU address bit 1 |
| 4   | Input     | A2         | CPU address bit 2 |
| 5   | Input     | A3         | CPU address bit 3 |
| 6   | Input     | A4         | CPU address bit 4 |
| 7   | Input     | PIO_M_D4   | 6116 output bit 0 (via 373) |
| 8   | Input     | PIO_M_D5   | 6116 output bit 1 (via 373) |
| 9   | Input     | PIO_M_D6   | 6116 output bit 2 (via 373) |
| 10  | Input     | MEM        | /MEM strobe |
| 11  | Input     | PSEL       | Mapping enable (active low) |
| 13  | Input     | PIO_M_D7   | 6116 output bit 3 (via 373) |
| 14  | Output    | SA3        | Physical page bit 3 (registered) |
| 15  | Output    | SA2        | Physical page bit 2 (registered) |
| 16  | Output    | SA1        | Physical page bit 1 (registered) |
| 17  | Output    | SA0        | Physical page bit 0 (registered) |
| 18  | Output    | NC         | Spare |
| 19  | Output    | WAIT       | Wait state generator |
| 20  | Output    | NC         | Spare |
| 21  | Output    | RAM_SEL    | Active low RAM chip select |
| 22  | Output    | ROM_SEL    | Active low ROM chip select |
| 23  | Output    | NC         | Spare |

---

## Test Utilities

| Utility        | Load Address | Description |
|----------------|--------------|-------------|
| DISPATCH.A99   | 0x0400       | Mapper init + jump to TPA |
| ALIASCHECK.A99 | 0x0500       | Proves page isolation — writes unique values to all pages then reads all back |
| RAMTEST.A99    | 0x0500       | Comprehensive — cross-page alias check + 4-pattern test per page + linear |
| MEMTEST.A99    | 0x0400+0x1000| RAMTEST running from paged memory via dispatcher |
| CODETEST.A99   | 0x0500       | Writes unique subroutine to each page and executes it — proves code execution from correct page |
| SEGTEST.A99    | 0x0500       | Segment isolation test |
| SATEST.A99     | 0x1000       | SA0-SA3 logic analyser test with delays |

### Running Tests
```
; From COMMON (recommended for diagnostics):
500G

; From paged memory via dispatcher:
; Load DISPATCH at 0x0400, test at 0x1000
400G
```

---

## GAL Revision History

| Rev  | Status    | Key Change |
|------|-----------|------------|
| 08   | Obsolete  | Original with PSEL gating |
| 09   | Obsolete  | Transparent mapping, PSEL removed |
| 10   | Obsolete  | MAP_SEL gating on SA — proved wrong, blocked SA during RAM cycles |
| 10A  | Obsolete  | MAP_SEL removed from SA — first working transparent version |
| 12   | Obsolete  | !MEM removed from SA equations |
| 13-22| Obsolete  | Various registered output and PIOEN experiments |
| 23   | Obsolete  | Registered SA clocked by /MRD, OR gate removed |
| 24   | Obsolete  | CRU write path introduced, MAP_WIN freed |
| 24A  | CURRENT   | Registered SA /MRD clock, CRU write, PSEL gating |

---

## Key Design Decisions and Lessons Learned

**Why registered outputs?**
The 6116 /OE is gated by /MRD — the 6116 only drives D4-D7 during read
cycles. Between cycles D4-D7 are Hi-Z. Registered GAL outputs capture
the correct value at the end of each read cycle and hold it stable
through write cycles and idle time.

**Why CRU writes?**
Writing the 6116 via memory cycles caused bus contention (6116 fighting
the CPU data bus), required complex latch timing, and meant MAP_SET could
only be called from COMMON. CRU writes use a completely separate bus.

**Why not remove the 373?**
The 373 bridges the gap between the 6116 /OE going Hi-Z at the end of
/MRD and the GAL flip-flops clocking on the /MRD rising edge. Without it,
the GAL inputs would be floating at exactly the moment they need to be
captured.

**Why COMMON must initialise the mapper?**
At power-on SA flip-flops hold random values. Any paged memory access
before MAP_INIT will use wrong physical pages. The dispatcher at 0x0400
runs MAP_INIT from COMMON before any paged access occurs.

**The 6116 hidden storage discovery**
With A4-A10 of the 6116 unused by the page register function, 2032 bytes
of SRAM are accessible only via CRU. This provides a naturally protected
OS scratchpad area invisible to normal memory operations.
