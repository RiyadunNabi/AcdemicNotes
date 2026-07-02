Full beginner-friendly breakdown of Chapter 5, Riyad.

---

# Chapter 5 — The Processor Status and the FLAGS Register

---

## Overview

What makes a computer different from a simple calculator machine? **Decision-making.** The CPU can look at the outcome of an operation and decide what to do next. It does this using **9 individual bits called flags**, stored inside a special register called the **FLAGS register**.

Two categories:
- **Status flags** — reflect the result of the last computation (e.g. was it zero? negative? did it overflow?)
- **Control flags** — turn certain CPU behaviors on/off

This chapter also introduces **DEBUG** — a tool to actually watch flags change in real time as your program runs.

---

## 5.1 The FLAGS Register

Think of FLAGS as a row of 16 switches (bits), numbered 0 to 15. Only some of these switches actually mean something:

- **Status flags** live in bits: 0, 2, 4, 6, 7, 11
- **Control flags** live in bits: 8, 9, 10
- The rest of the bits are unused/meaningless

You don't need to memorize which bit number belongs to which flag — you just need to know the flag **names and what they represent**.

| Bit | Name | Symbol | Type |
|---|---|---|---|
| 0 | Carry flag | CF | Status |
| 2 | Parity flag | PF | Status |
| 4 | Auxiliary carry flag | AF | Status |
| 6 | Zero flag | ZF | Status |
| 7 | Sign flag | SF | Status |
| 11 | Overflow flag | OF | Status |
| 8 | Trap flag | TF | Control |
| 9 | Interrupt flag | IF | Control |
| 10 | Direction flag | DF | Control |

This chapter focuses on the **status flags** — they're the ones instructions like ADD and SUB use to report their results.

---

### Understanding Each Status Flag

Let's build intuition for each one before formal definitions.

**Carry Flag (CF)**

Think of this like the "carry" you do in elementary school addition — when a column adds up to more than 9 and you carry a 1 to the next column. In binary, the same thing happens, except now it's about whether a bit "spills over" past the last bit position (the **most significant bit**, or **msb**).

> **CF = 1** if there's a carry OUT of the msb on addition, or a borrow INTO the msb on subtraction. Otherwise CF = 0.

This flag matters for **unsigned** number arithmetic — it tells you if the "true" answer didn't fit in the register.

**Parity Flag (PF)**

This just counts bits. Look at the **low byte** of the result. Count how many bits are `1`.

> **PF = 1** if that count is **even**. **PF = 0** if it's **odd**.

Example: result's low byte has 7 ones (odd) → PF = 0.

This flag is rarely used in normal programming — historically used for error-checking in serial communication.

**Auxiliary Carry Flag (AF)**

Same idea as CF, but instead of watching the boundary between bit 7/8 (byte boundary), it watches the boundary between bit 3/4 (a **nibble** boundary — remember, a nibble = 4 bits).

> **AF = 1** if there's a carry out of bit 3 on addition, or a borrow into bit 3 on subtraction.

This flag exists specifically to support **BCD (Binary-Coded Decimal)** arithmetic, covered later in Chapter 18. You can mostly ignore it for now.

**Zero Flag (ZF)**

The simplest one.

> **ZF = 1** if the result is exactly 0. **ZF = 0** otherwise.

This is hugely important — it's how programs check "did this equal zero?" (used constantly in loops and conditionals, covered in Chapter 6).

**Sign Flag (SF)**

Recall from Chapter 1: in two's complement, the **msb** (leftmost bit) tells you if a number is negative (1) or positive (0), *when interpreted as signed*.

> **SF = 1** if the msb of the result is 1 (i.e., result looks negative in signed interpretation). **SF = 0** if msb is 0.

**Overflow Flag (OF)**

The trickiest one — gets its own full section (5.2) because it needs real explanation.

> **OF = 1** if **signed overflow** occurred. Otherwise 0.

---

## 5.2 Overflow

### The Core Problem

Every register has a limited size. A 16-bit word can only hold so many distinct values. If a computation's *true* mathematical answer is bigger than what fits, you get **overflow** — the stored result is simply wrong.

Ranges to remember:

| Type | Size | Range |
|---|---|---|
| Signed byte | 8 bits | −128 to 127 |
| Unsigned byte | 8 bits | 0 to 255 |
| Signed word | 16 bits | −32768 to 32767 |
| Unsigned word | 16 bits | 0 to 65535 |

### The Critical Insight: Overflow depends on interpretation

Here's the key beginner trap to avoid: **the same bit pattern can be valid as unsigned but invalid as signed, or vice versa.** The CPU doesn't know if you intend a number to be signed or unsigned — so it computes **both kinds of overflow simultaneously**, and gives you two separate flags:

- **CF** → tells you about **unsigned overflow**
- **OF** → tells you about **signed overflow**

You, the programmer, know which interpretation you care about, so you check the flag that matches.

### Example: Unsigned overflow WITHOUT signed overflow

```
AX = FFFFh, BX = 0001h
ADD AX, BX
```

Binary math:
```
1111 1111 1111 1111
+ 0000 0000 0000 0001
---------------------
1 0000 0000 0000 0000
```

The true sum needs 17 bits, but AX only holds 16. The stored result is `0000h`.

**Unsigned check:** FFFFh = 65535, +1 = 65536. But 65536 doesn't fit in an unsigned word (max 65535). **Unsigned overflow occurred → CF = 1.**

**Signed check:** FFFFh as signed = −1. 0001h = +1. −1 + 1 = 0. The stored result (0000h = 0) is **correct**! **No signed overflow → OF = 0.**

Same operation, two different verdicts depending on how you read the numbers.

### Example: Signed overflow WITHOUT unsigned overflow

```
AX = 7FFFh, BX = 7FFFh
ADD AX, BX
```

Binary math:
```
0111 1111 1111 1111
+ 0111 1111 1111 1111
----------------------
1111 1111 1111 1110  = FFFEh
```

**Unsigned check:** 7FFFh = 32767 each. Sum = 65534. Fits fine in unsigned range (0–65535). **No unsigned overflow → CF = 0.**

**Signed check:** 7FFFh = +32767 (the largest positive signed word). Adding two positives should give a positive result. But FFFEh, interpreted as signed, is **−2** — a negative number! Sign flipped unexpectedly. **Signed overflow occurred → OF = 1.**

### How does the CPU actually detect this? (The simple rule)

You don't need to redo the "does it make sense" reasoning every time. There's a mechanical shortcut the hardware uses:

> **OF = 1** if the carry **INTO** the msb and the carry **OUT** of the msb **don't match** (one happened, the other didn't). If they match (both happen or neither happens), OF = 0.

### Simplified rules for addition/subtraction overflow:

**Unsigned overflow (→ CF):**
- Addition: happens when there's a carry OUT of the msb (true answer is bigger than max unsigned value)
- Subtraction: happens when there's a borrow INTO the msb (true answer is less than 0)

**Signed overflow (→ OF):**
- **Adding two numbers with the SAME sign:** overflow happens if the result has a **different** sign than expected
- **Subtracting numbers with different signs:** treated like adding same-sign numbers (since A − (−B) = A + B) — same overflow rule applies
- **Adding numbers with DIFFERENT signs:** overflow is **impossible** (a positive + negative always fits, since it's essentially a subtraction of two in-range numbers)
- **Subtracting numbers with the SAME sign:** overflow is **impossible** for the same reason

This is a useful shortcut: if the signs you're combining are guaranteed to "cancel out" the magnitude, overflow simply cannot happen.

---

## 5.3 How Instructions Affect the Flags

Not every instruction touches every flag. Here's the reference table for instructions you already know from Chapter 4:

| Instruction | Affects flags |
|---|---|
| MOV / XCHG | **None** — doesn't touch any flag |
| ADD / SUB | **All** status flags |
| INC / DEC | All **except CF** (Carry Flag is deliberately skipped) |
| NEG | All (CF = 1 unless result is 0; OF = 1 only if the operand was 8000h word or 80h byte) |

**Why does INC/DEC skip CF?** This is a deliberate design choice — INC/DEC are often used as loop counters, and you don't want a counter's carry behavior to accidentally mess up a carry flag you're using for something else in the same loop.

### Walking through worked examples (exactly as the book does)

**Example 5.1 — ADD AX, BX** where AX = FFFFh, BX = FFFFh

```
  FFFFh
+ FFFFh
--------
1 FFFEh   → stored result in AX = FFFEh
```

Binary version:
```
1111 1111 1111 1111
+1111 1111 1111 1111
--------------------
1 1111 1111 1111 1110
```

- **SF = 1** — msb of FFFEh (`1111 1111 1111 1110`) is 1 → looks negative
- **PF = 0** — low byte `1111 1110` has 7 ones → odd count
- **ZF = 0** — result isn't zero
- **CF = 1** — there was a carry out of the msb
- **OF = 0** — carry into msb AND carry out of msb both happened (they match) → no signed overflow

**Example 5.2 — ADD AL, BL** where AL = 80h, BL = 80h

```
  80h
+ 80h
------
1 00h   → stored result in AL = 00h
```

Binary version:
```
1000 0000
+1000 0000
---------
1 0000 0000
```

- **SF = 0** — result 00h, msb is 0
- **PF = 1** — all bits are 0 → even count of ones (zero is even)
- **ZF = 1** — result is exactly 0
- **CF = 1** — carry out of msb happened
- **OF = 1** — we added two negative numbers (80h as signed = −128 each) but got a **positive-looking** result (00h). Sign flipped unexpectedly → signed overflow

**Example 5.3 — SUB AX, BX** where AX = 8000h, BX = 0001h

```
  8000h
- 0001h
--------
  7FFFh
```

Binary version:
```
1000 0000 0000 0000
-0000 0000 0000 0001
--------------------
0111 1111 1111 1111
```

- **SF = 0** — msb of 7FFFh is 0
- **PF = 1** — 8 ones in low byte (even)
- **ZF = 0** — nonzero
- **CF = 0** — no borrow needed (we're subtracting a smaller unsigned number from a bigger one)
- **OF = 1** — signed-wise, 8000h is negative and we're subtracting a positive (0001h), which is like adding two negatives. We'd expect a negative result, but got 7FFFh (positive) → sign flipped unexpectedly → signed overflow

**Example 5.4 — INC AL** where AL = FFh

```
  FFh
+  1h
------
1 00h  → stored result in AL = 00h
```

Binary version:
```
1111 1111
+0000 0001
----------
1 0000 0000
```

- **SF = 0, PF = 1, ZF = 1**
- **CF is unaffected by INC** — stays whatever it was before (if it was 0 before, it's still 0 — even though there technically WAS a carry out!) This is the special INC/DEC exception.
- **OF = 0** — we added FFh (−1 signed) and 1h (+1 signed), which are **different signs** → signed overflow is impossible by the rule above

**Example 5.5 — MOV AX, −5**

Result stored: FFFBh (which is −5 in two's complement). **No flags change at all** — MOV never touches flags.

Binary version: `1111 1111 1111 1011` (FFFBh). No arithmetic is performed, so there is no add/subtract step to show.

**Example 5.6 — NEG AX** where AX = 8000h

```
8000h = 1000 0000 0000 0000
one's complement (flip all bits) = 0111 1111 1111 1111
add 1 → 1000 0000 0000 0000 = 8000h
```

Binary version:
```
1000 0000 0000 0000
one's complement → 0111 1111 1111 1111
+0000 0000 0000 0001
--------------------
1000 0000 0000 0000
```

Interesting — negating 8000h gives you **8000h again**! This is a special quirk: 8000h (as signed, −32768) has no positive counterpart in two's complement range (the range is −32768 to +32767 — there's no +32768 to flip to). So it "wraps around" to itself.

- **SF = 1, PF = 1, ZF = 0**
- **CF = 1** — for NEG, CF is always 1 *unless* the result is 0
- **OF = 1** — because negating should normally flip the sign, but here it didn't (still negative) → signed overflow

---

## 5.4 The DEBUG Program

**DEBUG** is a DOS tool that lets you **watch your program execute one instruction at a time** and see exactly how registers and flags change. This is incredibly useful for actually understanding what's happening (instead of just trusting the theory).

### Demo Program (PGM5_1.ASM)

```asm
TITLE PGM5_1:CHECK FLAGS
;used in DEBUG to check flag settings
.MODEL SMALL
.STACK 100H
.CODE
MAIN PROC
    MOV AX, 4000H    ;AX = 4000h
    ADD AX, AX       ;AX = 8000h
    SUB AX, 0FFFFH   ;AX = 8001h
    NEG AX           ;AX = 7FFFh
    INC AX           ;AX = 8000h
    MOV AH, 4CH
    INT 21H          ;DOS exit
MAIN ENDP
    END MAIN
```

### Launching DEBUG

Assemble and link normally to get `PGM5_1.EXE`. Then load it into DEBUG:

```
C>DEBUG A:PGM5_1.EXE
```

DEBUG shows a `-` prompt and waits for commands.

### The "R" command — display Registers

```
-R
AX=0000 BX=0000 CX=001F DX=0000 SP=000A BP=0000 SI=0000 DI=0000
DS=0ED5 ES=0ED5 SS=0EE5 CS=0EE6 IP=0000 NV UP DI PL NZ NA PO NC
0EE6:0000 B80040    MOV    AX,4000
```

Breaking this down:
- First two lines show all **register values** in hex
- `0EE6:0000` is the **segment:offset** of the *next instruction to execute*
- `B80040` is the raw **machine code** of that instruction
- `MOV AX,4000` is DEBUG's disassembly (translation back to readable assembly) of that machine code

### DEBUG's flag symbols

DEBUG doesn't print "CF=1" — it uses **short letter codes**. Two letters per flag, one for "set" (1) and one for "clear" (0):

| Flag | Set (1) | Clear (0) |
|---|---|---|
| CF | CY (carry) | NC (no carry) |
| PF | PE (even parity) | PO (odd parity) |
| AF | AC (auxiliary carry) | NA (no auxiliary carry) |
| ZF | ZR (zero) | NZ (nonzero) |
| SF | NG (negative) | PL (plus) |
| OF | OV (overflow) | NV (no overflow) |
| DF | DN (down) | UP (up) |
| IF | EI (enable interrupts) | DI (disable interrupts) |

The flags always print in this fixed order: **OF, DF, IF, SF, ZF, AF, PF, CF**

At the very start, all flags are cleared by DEBUG: `NV UP DI PL NZ NA PO NC`

### The "T" command — Trace (execute one instruction at a time)

Each time you type `T`, DEBUG executes **just the next instruction** and shows you the updated state.

**Trace 1 — MOV AX, 4000h:**
```
AX=4000 ... IP=0003 NV UP DI PL NZ NA PO NC
0EE6:0003 03C0    ADD    AX,AX
```
AX becomes 4000h. Flags unchanged (MOV never touches flags).

**Trace 2 — ADD AX, AX** (4000h + 4000h = 8000h):
```
AX=8000 ... OV UP DI NG NZ NA PE NC
```
- **SF = NG** (negative) — msb of 8000h is 1
- **OF = OV** (overflow!) — we added two positive numbers (4000h + 4000h, both positive signed) and got a negative-looking result (8000h) → signed overflow
- **PF = PE** (even) — low byte 00h has zero ones (even)

**Trace 3 — SUB AX, 0FFFFh** (8000h − FFFFh = 8001h):
```
AX=8001 ... NV UP DI NG NZ AC PO CY
```
- **OF = NV** (no overflow now) — subtracting numbers of the same sign can't cause signed overflow
- **CF = CY** (carry/borrow happened) — unsigned overflow: we subtracted a bigger unsigned number (FFFFh) from a smaller one (8000h), requiring a borrow

**Trace 4 — NEG AX** (negating 8001h → 7FFFh):
```
AX=7FFF ... NV UP DI PL NZ AC PE CY
```
- **CF = CY** — for NEG, CF is always 1 unless result is 0
- **OF = NV** — result isn't the special case 8000h, so no overflow

**Trace 5 — INC AX** (7FFFh + 1 = 8000h):
```
AX=8000 ... OV UP DI NG NZ AC PE CY
```
- **OF = OV** — added two positives (7FFFh and 1) and got a negative-looking result
- **CF stays CY** — unchanged from before, because **INC never touches CF**, even though technically no carry happened here

### Finishing up

- **`G`** (go) — runs the rest of the program to completion:
```
-G
Program terminated normally
```

- **`Q`** (quit) — exits DEBUG back to DOS:
```
-Q
C>
```

---

## Summary (from the book)

- FLAGS register = 16 bits; 6 are status flags, 3 are control flags
- Status flags reflect the result of the last operation: **CF, PF, AF, ZF, SF, OF**
- **CF** = 1 on carry-out (addition) or borrow-in (subtraction) at the msb
- **PF** = 1 if result has an even number of 1-bits
- **AF** = 1 on carry/borrow at bit 3 (used for BCD math)
- **ZF** = 1 if result is exactly 0
- **SF** = 1 if msb of result is 1
- **OF** = 1 if the correct *signed* result doesn't fit in the destination
- Overflow happens when the true result falls outside the representable range
- CF tracks unsigned overflow; OF tracks signed overflow — **independent of each other**
- Mechanical rule: OF = 1 when carry-into-msb and carry-out-of-msb **disagree**
- Same-sign addition / different-sign subtraction → signed overflow possible
- Different-sign addition / same-sign subtraction → signed overflow **impossible**
- MOV/XCHG touch **no** flags; INC/DEC touch all **except CF**
- DEBUG commands: **R** (registers), **T** (trace one instruction), **G** (run to completion), **Q** (quit)

---

That's the full Chapter 5, Riyad. This chapter is the foundation for Chapter 6 (jumps/branches/loops) — every conditional jump instruction you'll learn next is literally just "check a flag and decide where to go." Ready for Chapter 6?