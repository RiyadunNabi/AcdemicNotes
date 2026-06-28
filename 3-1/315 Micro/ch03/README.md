Got it Riyad. Full breakdown of Chapter 3 from the actual PDF.

---

# Chapter 3 — Organization of the IBM Personal Computers

---

## Overview

Chapter 1 covered general microcomputer organization. Chapter 3 zooms into the IBM PC specifically. It covers:
- The Intel 8086 processor family (section 3.1)
- The 8086 architecture — registers and segmented memory (section 3.2)
- The overall IBM PC structure — memory map, I/O ports, DOS, BIOS (section 3.3)

---

## 3.1 The Intel 8086 Family of Microprocessors

IBM PC models and what processor they use:

| IBM Model | Processor |
|---|---|
| PC, PC XT | 8088 |
| PC AT, PS/1 | 80286 |
| Some laptops | 80186 |
| PS/2 | 8086, 80286, 80386, or 80486 |

### The 8086 and 8088

- Intel introduced the **8086 in 1978** — first 16-bit microprocessor (can operate on 16 bits of data at once)
- **8088 introduced in 1979** — internally identical to the 8086
- Key difference: 8086 has a **16-bit data bus**, 8088 has an **8-bit data bus**
- IBM chose the 8088 for the original PC because it was **cheaper** to build around
- Both share the **same instruction set** — this set is the foundation for all later processors in the family

### The 80186 and 80188

- Enhanced versions of the 8086/8088
- Incorporate the functions of the 8086/8088 **plus some support chips** on one chip
- Can execute an **extended instruction set**
- But offered no significant advantage — quickly overshadowed by the 80286

### The 80286

Introduced **1982**, also 16-bit but faster (12.5 MHz vs 10 MHz). Three major advances:

1. **Two modes of operation:**
   - **Real address mode** — behaves exactly like the 8086. Old 8086 programs run without modification.
   - **Protected (virtual address) mode** — supports:
     - **Multitasking** — run several programs simultaneously
     - **Memory protection** — one program can't mess with another program's memory

2. **More addressable memory** — in protected mode, can address **16 MB** (vs 1 MB for 8086)

3. **Virtual memory** — can treat disk storage as if it were RAM. Programs up to **1 GB** can run even if they don't fit in physical memory

### The 80386 and 80386SX

- Intel's **first 32-bit processor**, introduced **1985**
- 32-bit data path + up to 33 MHz clock + executes instructions in fewer cycles = much faster than 80286
- Modes: real mode (acts like 8086), protected mode (emulates 80286), and a new **virtual 8086 mode** (runs multiple 8086 programs under memory protection)
- In protected mode: addresses **4 GB** physical memory, **64 TB** virtual memory
- **386SX** = same internal structure as 386, but only a **16-bit data bus** (cheaper)

### The 80486 and 80486SX

- Introduced **1989**, 32-bit, fastest in the family
- Incorporates the 386 **plus**:
  - **80387 numeric processor** — handles floating-point math
  - **8 KB cache memory** — fast buffer between the slow RAM and the CPU
- Result: **3× faster** than a 386 at the same clock speed
- **486SX** = same as 486 but **without the floating-point processor**

---

## 3.2 Organization of the 8086/8088 Microprocessors

From here the book focuses on the 8086/8088 (simplest structure). "8086" in the rest of the chapter means both 8086 and 8088.

---

## 3.2.1 Registers

Registers are fast storage circuits inside the CPU. Three types:
- **Data registers** — hold data for operations
- **Address registers** — hold addresses of instructions or data
- **Status register** — tracks current processor state

The 8086 has **14 registers total**, all 16-bit. See Figure 3.1 in the book. Categories:
- Data registers: AX, BX, CX, DX
- Segment registers: CS, DS, SS, ES
- Pointer and index registers: SP, BP, SI, DI, IP
- FLAGS register

---

## 3.2.2 Data Registers: AX, BX, CX, DX

Four general-purpose registers for data manipulation. Key point: **registers are faster than memory** — same instruction takes fewer clock cycles when data is in a register vs in RAM. This is why processors have many registers.

### High/Low byte access

Each 16-bit data register can be split into two 8-bit halves:

| 16-bit | High byte (bits 8–15) | Low byte (bits 0–7) |
|---|---|---|
| AX | AH | AL |
| BX | BH | BL |
| CX | CH | CL |
| DX | DH | DL |

This gives you more options when working with byte-sized data.

### Special functions of each:

**AX (Accumulator Register)**
Preferred register for arithmetic, logic, and data transfer — using AX generates the **shortest machine code**. In multiplication and division, one of the numbers must be in AX or AL. I/O operations also require AL and AX.

**BX (Base Register)**
Also serves as an **address register**. Used in table look-up with the `XLAT` (translate) instruction.

**CX (Count Register)**
Used as a **loop counter** in program loops. Also used with `REP` (repeat) instruction for string operations. `CL` (low byte of CX) is used as a bit-shift/rotate count.

**DX (Data Register)**
Used in **multiplication and division** operations. Also used in **I/O operations**.

---

## 3.2.3 Segment Registers: CS, DS, SS, ES

This is the most important section of Chapter 3. It explains segmented memory — the core concept of 8086 addressing.

### The Problem

The 8086 uses **20-bit physical addresses** (to address 1 MB). But all registers are only **16 bits**. A 16-bit register can only hold values up to FFFFh = 65,535. You can't fit a 20-bit address into a 16-bit register.

### The Solution: Memory Segments

Memory is divided into **segments**.

**What is a memory segment?**
A block of **2¹⁶ = 65,536 bytes = 64 KB** of consecutive memory. Each segment has a **segment number** (16-bit, so highest is FFFFh).

**What is an offset?**
Within a segment, a location is identified by its **offset** — how many bytes from the start of the segment. Also 16-bit, so offsets go from 0000h to FFFFh.

### Segment:Offset (Logical Address)

A memory location is written as `segment:offset` — called a **logical address**.

Example: `A4FB:4872h` means offset 4872h within segment A4FBh.

### Converting to Physical Address

```
Physical Address = Segment × 10h + Offset
```

(Multiplying by 10h = shifting left 4 bits in hex, i.e. appending a 0 to the right)

Example from the book:
```
Segment = A4FBh  →  A4FB0h  (shift left, append 0)
Offset  =          +  4872h
Physical address = A9822h
```

### Location of Segments

Segments start every **16 bytes (= 1 paragraph)** in memory. The starting address of any segment always ends in hex digit `0` — this is called a **paragraph boundary**.

Segments **overlap** heavily. This means the same physical address can be represented by many different segment:offset combinations — the logical address is **not unique**.

**Example 3.1 from the book:**
Physical address `1256Ah` can be expressed as:
- `1256:000A` (in segment 1256h, offset = Ah)
- `1240:016A` (in segment 1240h, offset = 16Ah)

**Example 3.2 from the book:**
Physical address `80FD2h`, offset `BFD2h` → what segment?
```
Segment × 10h = 80FD2h − BFD2h = 75000h
Segment = 7500h
```

### Program Segments

A running program has three parts in memory:
- **Code segment** — the instructions
- **Data segment** — the variables
- **Stack segment** — used for procedure calls

The four segment registers track these:

| Register | Holds |
|---|---|
| CS | Segment number of code segment |
| DS | Segment number of data segment |
| SS | Segment number of stack segment |
| ES | Extra segment (for a second data area) |

At any moment, only the **4 segments pointed to by CS, DS, SS, ES** are accessible. But a program can change segment register values to access other parts of memory.

A program segment doesn't have to fill all 64 KB — it can be smaller, and due to overlapping, multiple segments can be packed close together.

---

## 3.2.4 Pointer and Index Registers: SP, BP, SI, DI

These hold **offset addresses** (pointers into memory). Unlike segment registers, they can be used in arithmetic operations.

**SP (Stack Pointer)**
Works with SS to access the **stack segment**. Points to the top of the stack. Covered in detail in Chapter 8.

**BP (Base Pointer)**
Primarily accesses **data on the stack**. Unlike SP, BP can also access data in other segments.

**SI (Source Index)**
Points to memory locations in the **data segment** (addressed by DS). Incrementing SI lets you walk through consecutive memory locations easily.

**DI (Destination Index)**
Same functions as SI. Used with **string operations** to access memory in the **ES segment**.

---

## 3.2.5 Instruction Pointer: IP

To access *instructions* (not data), the 8086 uses **CS and IP together**:
- CS = segment number of the current instruction
- IP = offset of the *next* instruction

IP is **automatically updated** after each instruction executes — always pointing ahead to the next one.

Critical rule from the book: **IP cannot be directly manipulated** by an instruction. You can't write `MOV IP, AX` — IP is off-limits as an operand.

---

## 3.2.6 FLAGS Register

Reflects the **current status of the processor**. Individual bits called **flags** are set or cleared based on operations.

Two types of flags:

**Status flags** — reflect the result of the last operation:
- Example: **ZF (Zero Flag)** — set to 1 when a result is zero. A later instruction can check ZF and branch accordingly.
- Covered in Chapter 5.

**Control flags** — enable or disable processor behaviors:
- Example: **IF (Interrupt Flag)** — if cleared to 0, keyboard input is ignored by the processor.
- Covered in Chapters 11 and 15.

---

## 3.3 Organization of the PC

Hardware is controlled by software. To understand the PC fully, you also need to understand the software layer.

---

## 3.3.1 The Operating System

The **operating system (OS)** coordinates all devices in the computer. Functions:
1. Reading and executing user commands
2. Performing I/O operations
3. Generating error messages
4. Managing memory and other resources

The IBM PC's OS is **DOS (Disk Operating System)** — also called PC DOS or MS DOS.

DOS was designed for 8086/8088 machines, so it:
- Can only manage **1 MB of memory**
- Does **not support multitasking**
- Can run on 80286/386/486 only when they're in **real address mode**

### Files

Disk data is organized into **files**. Each file has:
- **File name** — 1 to 8 characters
- **File extension** — optional, period + 1 to 3 characters (identifies file type)

Example: `COMMAND.COM` → name is `COMMAND`, extension is `.COM`

### DOS structure

DOS is not one program — it's a **collection of service routines**. The routine handling user commands is **COMMAND.COM**, which generates the `C>` prompt.

Two types of commands:
- **Internal commands** — already loaded in memory, execute immediately
- **External commands** — routines on disk not yet loaded, or application programs

### BIOS (Basic Input/Output System)

Because DOS lives on disk, something must run before DOS loads. That's **BIOS** — stored in **ROM** (survives power-off).

BIOS vs DOS:
| | DOS | BIOS |
|---|---|---|
| Works across | Entire PC family | Machine-specific |
| Stored in | Disk (RAM when loaded) | ROM |
| Role | High-level OS services | Direct hardware I/O |

DOS I/O operations are **ultimately carried out by BIOS**. BIOS also does circuit checking and loads DOS at startup.

**Interrupt vectors** — addresses of BIOS and DOS routines, stored starting at address `00000h`, so programs can call them.

IBM compatibles use their own BIOS — compatibility depends on how well it matches the IBM BIOS.

---

## 3.3.2 Memory Organization of the PC

Total addressable memory: **1 MB** (00000h to FFFFFh). But not all is available to your programs.

### Memory Map of the IBM PC

The book partitions the 1 MB into **16 disjoint segments** of 64 KB each (segment 0000h, 1000h, 2000h … F000h):

| Address Range | Contents |
|---|---|
| 00000h–003FFh | Interrupt vectors (first 1 KB) |
| 00400h onwards | BIOS and DOS data |
| Above that | DOS itself |
| Large middle area | **Application program area** (your programs run here) |
| A0000h–BFFFFh | Video display memory (what's shown on screen) |
| C0000h–E0000h | Reserved |
| F0000h–FFFFFh | BIOS routines (ROM, not RAM) + ROM BASIC |

Only the **first 10 disjoint segments** (0000h to 9000h) = **640 KB** are available for DOS and application programs.

That famous "640 KB limit" of DOS — this is where it comes from.

---

## 3.3.3 I/O Port Addresses

The 8086/8088 supports **64 KB of I/O ports**. Some common port addresses from Table 3.1:

| Port Address | Device |
|---|---|
| 20h–21h | Interrupt controller |
| 60h–63h | Keyboard controller |
| 200h–20Fh | Game controller |
| 2F8h–2FFh | Serial port COM2 |
| 320h–32Fh | Hard disk |
| 378h–37Fh | Parallel printer port |
| 3C0h–3CFh | EGA (video) |
| 3D0h–3DFh | CGA (video) |
| 3F8h–3FFh | Serial port COM1 |

Note: direct I/O port programming is **not recommended** because port addresses vary between PC models. Use BIOS/DOS routines instead.

---

## 3.3.4 Start-up Operation

What happens the moment you power on the PC:

1. Processor enters **reset state**: CS = FFFFh, IP = 0000h
2. First instruction is at **FFFFOh** — this is in ROM
3. That instruction jumps to the start of **BIOS routines**
4. BIOS checks for system/memory errors
5. BIOS initializes **interrupt vectors** and BIOS data area
6. BIOS loads the **boot program** from disk
7. Boot program loads the actual **DOS routines**
8. Finally, **COMMAND.COM** takes control → you see the `C>` prompt

Why two steps (boot program then DOS)? The boot program isolates BIOS from changes made to DOS, and keeps BIOS smaller.

---

That's the complete Chapter 3, Riyad. This chapter is arguably the most important one for your course — the segment:offset addressing model and the register layout will come up in literally every assembly program you write. Ready for the next chapter?