Got it, Riyad. Full beginner-friendly breakdown of Chapter 4 — every section, every concept explained from scratch.

---

# Chapter 4 — Introduction to IBM PC Assembly Language

---

## Overview

This chapter teaches you how to actually **write, assemble, and run** an assembly program. By the end you can write real programs. It covers:
- Syntax rules (how to write statements)
- How to define data (variables, constants)
- Basic instructions (move data, do arithmetic)
- How to do input/output using DOS
- How to build and run a program

---

## 4.1 Assembly Language Syntax

The assembler used in this book is **MASM (Microsoft Macro Assembler)**. Assembly is **not case sensitive** — `MOV`, `mov`, and `Mov` all mean the same thing. The book uses uppercase to make code stand out.

### What is a Statement?

Every line in an assembly program is a **statement**. There are two kinds:

- **Instruction** — gets translated into actual machine code. The CPU executes it.
- **Assembler directive** (also called pseudo-op) — a command *to the assembler*, not the CPU. It doesn't become machine code. Examples: defining a variable, declaring a segment.

Every statement can have up to **four fields**, always in this order:

```
name    operation    operand(s)    comment
```

Fields are separated by at least one **space or tab**. They don't need to line up in specific columns, but the order must be respected.

Example instruction:
```asm
START:    MOV CX, 5    ;initialize counter
```
- Name = `START:` (a label)
- Operation = `MOV`
- Operands = `CX, 5`
- Comment = `;initialize counter`

Example directive:
```asm
MAIN    PROC
```
- Name = `MAIN`
- Operation = `PROC` (create a procedure)

---

## 4.1.1 Name Field

The name field is used for **labels**, **procedure names**, and **variable names**. The assembler converts names into memory addresses.

**Rules for names:**
- 1 to 31 characters long
- Can contain: letters, digits, and `? . @ _ $ %`
- No embedded spaces
- If a period is used, it must be the **first** character
- Cannot begin with a digit
- Case doesn't matter (`COUNTER` = `counter`)

**Legal names:**
```
COUNTER1    @character    SUM_OF_DIGITS    $1000    DONE?    .TEST
```

**Illegal names:**
```
TWO WORDS   → has a space
2abc        → starts with a digit
A45.28      → period is not first character
YOU&ME      → & is not allowed
```

---

## 4.1.2 Operation Field

For **instructions**: contains the **opcode** — a symbolic name for the operation. Examples: `MOV`, `ADD`, `SUB`. The assembler translates this into binary machine code.

For **directives**: contains a **pseudo-op** — tells the assembler to do something. Example: `PROC` creates a procedure. Pseudo-ops are **never** turned into machine code.

---

## 4.1.3 Operand Field

Specifies the **data** the operation works on. An instruction can have zero, one, or two operands:

```asm
NOP              ; zero operands — does nothing
INC AX           ; one operand — adds 1 to AX
ADD WORD1, 2     ; two operands — adds 2 to WORD1
```

In a **two-operand instruction**:
- First operand = **destination** — where the result goes
- Second operand = **source** — what is being used

The source is usually unchanged by the instruction.

---

## 4.1.4 Comment Field

Starts with a **semicolon** `;`. The assembler ignores everything after it. Comments are optional but assembly is so low-level that **good comments are essential** — almost every line should have one.

Bad comment (just states the obvious):
```asm
MOV CX, 0    ;move 0 to CX
```

Good comment (explains *why*):
```asm
MOV CX, 0    ;CX counts terms, initially 0
```

A whole line can be a comment:
```asm
;initialize registers
MOV AX, 0
MOV BX, 0
```

---

## 4.2 Program Data

The CPU only works in binary. But in assembly source code, you can write data in three ways — the assembler converts it all to binary.

### Numbers

**Binary:** bit string + letter `B`
```
1010B       → binary
11011B      → binary
```

**Decimal:** just digits, optional `D` suffix
```
64223       → decimal
-21843D     → decimal
```

**Hexadecimal:** must start with a decimal digit (0–9) and end with `H`
```
1B4DH       → hex (legal — starts with 1)
0ABCH       → hex (legal — starts with 0)
FFFFH       → ILLEGAL — starts with F (looks like a name)
0FFFFH      → hex (legal — starts with 0)
1B4D        → ILLEGAL — no H at end
```

Why must hex start with a digit? Because if it started with a letter like `ABCH`, the assembler couldn't tell if it's a variable named `ABCH` or a hex number. So the rule is: **always prefix hex with 0 if it starts with A–F**.

### Characters

Enclosed in single or double quotes. The assembler converts them to their ASCII codes automatically.

```asm
'A'    →  41h    (same thing)
"hello" → 5 bytes: 68h, 65h, 6Ch, 6Ch, 6Fh
```

Inside a string, case matters — `'abc'` is different from `'ABC'`.

You can mix characters and numbers in one definition:
```asm
MSG DB 'HELLO', 0AH, 0DH, '$'
```
This is a string followed by a line feed, carriage return, and dollar sign.

[See Details](a.md)
---

## 4.3 Variables

Variables in assembly play the same role as in high-level languages — named memory locations that store data.

**Data-defining pseudo-ops:**

| Pseudo-op | Stands for | Size |
|---|---|---|
| DB | Define Byte | 1 byte |
| DW | Define Word | 2 bytes (16 bits) |
| DD | Define Doubleword | 4 bytes |
| DQ | Define Quadword | 8 bytes |
| DT | Define Tenbytes | 10 bytes |

This chapter focuses on DB and DW.

---

## 4.3.1 Byte Variables

```asm
name    DB    initial_value
```

Example:
```asm
ALPHA   DB    4        ; byte named ALPHA, initialized to 4
BYT     DB    ?        ; byte named BYT, uninitialized (? means don't care)
```

Valid initial value range: **−128 to 127** (signed) or **0 to 255** (unsigned) — anything that fits in 8 bits.

---

## 4.3.2 Word Variables

```asm
name    DW    initial_value
```

Example:
```asm
WRD     DW    -2       ; word named WRD, initialized to -2
SUM     DW    ?        ; uninitialized word
```

Valid range: **−32768 to 32767** (signed) or **0 to 65535** (unsigned).

---

## 4.3.3 Arrays

An array in assembly is just **consecutive bytes or words** in memory.

**Byte array:**
```asm
B_ARRAY    DB    10H, 20H, 30H
```

If the assembler puts `B_ARRAY` at address `200h`, memory looks like:

| Symbol | Address | Contents |
|---|---|---|
| B_ARRAY | 200h | 10h |
| B_ARRAY+1 | 201h | 20h |
| B_ARRAY+2 | 202h | 30h |

**Word array:**
```asm
W_ARRAY    DW    1000, 40, 29887, 329
```

Words take 2 bytes each, so addresses go in steps of 2:

| Symbol | Address | Contents |
|---|---|---|
| W_ARRAY | 0300h | 1000 |
| W_ARRAY+2 | 0302h | 40 |
| W_ARRAY+4 | 0304h | 29887 |
| W_ARRAY+6 | 0306h | 329 |

### High and Low Bytes of a Word

```asm
WORD1    DW    1234H
```

- Low byte (34h) → at address `WORD1`
- High byte (12h) → at address `WORD1+1`

This is because the 8086 stores the **low byte first** (little-endian).

[See Details](b.md)

### Character Strings

```asm
LETTERS    DB    'ABC'
```
Is exactly the same as:
```asm
LETTERS    DB    41H, 42H, 43H
```

The assembler converts characters to ASCII automatically.

---

## 4.4 Named Constants

Sometimes you want to use a name for a fixed value to make code readable. Use **EQU** (equates).

```asm
name    EQU    constant
```

Example:
```asm
LF    EQU    0AH    ; LF = line feed character (ASCII 0Ah)
```

Now anywhere you write `LF`, the assembler substitutes `0AH`. So:
```asm
MOV DL, 0AH
```
and
```asm
MOV DL, LF
```
produce **identical machine code**.

EQU can also be a string:
```asm
PROMPT    EQU    'TYPE YOUR NAME'
MSG       DB      PROMPT
```

**Important:** EQU allocates **no memory**. It's just a text substitution for the assembler.

---

## 4.5 A Few Basic Instructions

The 8086 has over 100 instructions. This section introduces the most essential 6.

---

## 4.5.1 MOV and XCHG

### MOV — Move (copy) data

```asm
MOV    destination, source
```

MOV copies the source value into the destination. The source is **unchanged**.

Examples:
```asm
MOV AX, WORD1    ; copy contents of memory WORD1 into AX
MOV AX, BX       ; copy BX into AX (BX unchanged)
MOV AH, 'A'      ; put 41h (ASCII of 'A') into AH
```

Think of MOV as **photocopying** — the original stays, a copy goes to the destination.

### XCHG — Exchange (swap) data

```asm
XCHG    destination, source
```

Swaps the contents of both operands.

```asm
XCHG AH, BL     ; AH gets BL's value, BL gets AH's value
XCHG AX, WORD1  ; swap AX and memory word WORD1
```

### Restrictions on MOV and XCHG

**Cannot move directly between two memory locations:**
```asm
MOV WORD1, WORD2    ; ILLEGAL
```

Workaround — use a register as a middleman:
```asm
MOV AX, WORD2
MOV WORD1, AX
```

Also: you cannot move a **constant directly into a segment register**. Always go through a general register first.

---

## 4.5.2 ADD, SUB, INC, and DEC

### ADD and SUB

```asm
ADD    destination, source    ; destination = destination + source
SUB    destination, source    ; destination = destination - source
```

Examples:
```asm
ADD WORD1, AX    ; WORD1 = WORD1 + AX  (AX unchanged)
SUB AX, DX      ; AX = AX - DX  (DX unchanged)
ADD BL, 5       ; BL = BL + 5
```

**Cannot add/subtract directly between two memory locations:**
```asm
ADD BYTE1, BYTE2    ; ILLEGAL
```
Fix:
```asm
MOV AL, BYTE2
ADD BYTE1, AL
```

### INC and DEC

Add or subtract 1 from a register or memory location.

```asm
INC    destination    ; destination = destination + 1
DEC    destination    ; destination = destination - 1
```

Examples:
```asm
INC WORD1    ; WORD1 = WORD1 + 1
DEC BYTE1    ; BYTE1 = BYTE1 - 1
```

---

## 4.5.3 NEG

Negates (flips the sign of) a value using **two's complement**.

```asm
NEG    destination
```

Example:
```asm
NEG BX    ; if BX was 0002h, it becomes FFFEh (which is -2)
```

### Type Agreement Rule

Both operands in a two-operand instruction must be the **same size** — both bytes, or both words.

```asm
MOV AX, BYTE1    ; ILLEGAL — AX is word, BYTE1 is byte
```

But the assembler is smart about constants:
```asm
MOV AH, 'A'    ; AH is byte → moves 41h (byte)
MOV AX, 'A'    ; AX is word → moves 0041h (word)
```

---

## 4.6 Translation of High-Level Language to Assembly

The book shows how common assignment statements translate. A and B are word variables.

| High-level | Assembly translation |
|---|---|
| `B = A` | `MOV AX, A` then `MOV B, AX` (can't do direct memory-to-memory) |
| `A = 5 - A` | `MOV AX, 5` → `SUB AX, A` → `MOV A, AX` OR shorter: `NEG A` → `ADD A, 5` |
| `A = B - 2×A` | `MOV AX, B` → `SUB AX, A` → `SUB AX, A` → `MOV A, AX` |

The general technique: **do the arithmetic in a register, then store the result**.

---

## 4.7 Program Structure

Every assembly program has three segments: code, data, stack. These map directly to memory segments from Chapter 3.

---

## 4.7.1 Memory Models

You declare the program's size with `.MODEL`:

```asm
.MODEL    memory_model
```

| Model | Code | Data |
|---|---|---|
| SMALL | 1 segment | 1 segment |
| MEDIUM | multiple segments | 1 segment |
| COMPACT | 1 segment | multiple segments |
| LARGE | multiple segments | multiple segments (arrays ≤ 64 KB) |
| HUGE | multiple segments | multiple segments (arrays can exceed 64 KB) |

For almost everything you'll write: **use SMALL**. `.MODEL` must come before any segment declaration.

---

## 4.7.2 Data Segment

Declared with `.DATA`. Contains all variable and constant definitions.

```asm
.DATA
WORD1    DW    2
WORD2    DW    5
MSG      DB    'THIS IS A MESSAGE'
MASK     EQU   10010010B
```

---

## 4.7.3 Stack Segment

Sets aside memory for the runtime stack. Must always be declared.

```asm
.STACK    size
```

Example:
```asm
.STACK    100H    ; 256 bytes for the stack
```

If you omit the size, 1 KB is used by default. 100H bytes is enough for most programs.

---

## 4.7.4 Code Segment

Contains all instructions, organized into **procedures**.

```asm
.CODE
MAIN    PROC
    ; your instructions here
MAIN    ENDP
```

`PROC` marks the start, `ENDP` marks the end. The procedure name must match on both lines.

---

## 4.7.5 Putting It Together — The Complete Program Template

```asm
.MODEL SMALL
.STACK 100H
.DATA
    ; variable definitions go here
.CODE
MAIN    PROC
    ; instructions go here
MAIN    ENDP
    ; other procedures go here
END MAIN
```

The **last line** must be `END MAIN` — tells the assembler where the program starts executing.

---

## 4.8 Input and Output Instructions

Two ways to do I/O:
1. **Direct I/O** — using `IN` and `OUT` instructions. Fast but port addresses vary per machine. Not recommended for general use.
2. **BIOS/DOS service routines** — easier, portable. This is what the book uses.

### The INT Instruction

To call a DOS or BIOS routine:

```asm
INT    interrupt_number
```

`INT` = interrupt. It hands control to a specific service routine.

`INT 21h` is the main DOS interrupt — a giant collection of functions. You tell it which function you want by putting a **function number in AH** before calling it.

---

## 4.8.1 INT 21h Functions

### Function 1 — Read a single key from keyboard

```
Input:   AH = 1
Output:  AL = ASCII code of the key pressed
         AL = 0 if a non-character key (arrow key, F1, etc.)
```

```asm
MOV AH, 1    ; function 1
INT 21H      ; now AL contains the key's ASCII code
```

The processor **waits** for the user to press a key. The character is also automatically displayed on screen (echoed).

---

### Function 2 — Display a single character

```
Input:   AH = 2
         DL = ASCII code of character to display
Output:  AL = the character's ASCII code
```

```asm
MOV AH, 2       ; function 2
MOV DL, '?'     ; character to display
INT 21H          ; display it
```

After display, the cursor moves to the next position.

Function 2 also handles **control characters** — if you put a control code in DL, it performs that function instead of displaying:

| ASCII hex | Symbol | Function |
|---|---|---|
| 07h | BEL | Beep |
| 08h | BS | Backspace |
| 09h | HT | Tab |
| 0Ah | LF | Line feed (move down one line) |
| 0Dh | CR | Carriage return (move cursor to start of current line) |

**Important:** Function 2 **changes AL**. If you read a character and then want to display something else using function 2, you must save the original character somewhere else first (e.g. in BL).

---

## 4.9 A First Program — Echo Program

The book walks through building a program step by step. Goal: display `?`, read a character, then display it on the next line.

Step 1 — Display `?`:
```asm
MOV AH, 2
MOV DL, '?'
INT 21H
```

Step 2 — Read a character:
```asm
MOV AH, 1
INT 21H          ; character now in AL
MOV BL, AL       ; SAVE it in BL because function 2 will overwrite AL
```

Step 3 — Move cursor to next line (carriage return + line feed):
```asm
MOV AH, 2
MOV DL, 0DH     ; carriage return
INT 21H
MOV DL, 0AH     ; line feed
INT 21H
```

Step 4 — Display the character:
```asm
MOV DL, BL      ; retrieve saved character
INT 21H          ; display it
```

Complete program (PGM4_1.ASM):
```asm
TITLE PGM4_1: ECHO PROGRAM
.MODEL SMALL
.STACK 100H
.CODE
MAIN    PROC
    ; display prompt
    MOV AH, 2
    MOV DL, '?'
    INT 21H
    ; input a character
    MOV AH, 1
    INT 21H
    MOV BL, AL          ; save it
    ; go to new line
    MOV AH, 2
    MOV DL, 0DH
    INT 21H
    MOV DL, 0AH
    INT 21H
    ; display character
    MOV DL, BL
    INT 21H
    ; return to DOS
    MOV AH, 4CH
    INT 21H
MAIN    ENDP
END MAIN
```

Note — no `.DATA` segment because no variables were used.

### Terminating a program

Every program must return control to DOS when it finishes:
```asm
MOV AH, 4CH    ; DOS exit function
INT 21H         ; exit to DOS
```

---

## 4.10 Creating and Running a Program

Four steps to go from source code to running program:

**Step 1 — Write the source file**
Use any text editor. Save with `.ASM` extension. Example: `PGM4_1.ASM`

**Step 2 — Assemble**
Run MASM to translate `.ASM` → `.OBJ` (machine code, but not yet runnable):
```
C:MASM PGM4_1;
```
MASM checks for syntax errors and reports line numbers. If no errors, produces `PGM4_1.OBJ`.

Optional files MASM can also produce:
- **.LST file** — line-numbered listing showing both assembly code and corresponding machine code side by side. Very useful for debugging.
- **.CRF file** — cross-reference file: lists every name in the program and all line numbers where it appears. Useful for large programs.

**Step 3 — Link**
Run LINK to convert `.OBJ` → `.EXE` (runnable file):
```
C:LINK PGM4_1;
```

Why is linking needed? Two reasons:
1. When assembling, the final memory addresses aren't known yet — LINK fills them in.
2. A large program may be split across multiple `.OBJ` files — LINK combines them.

**Step 4 — Run**
```
PGM4_1
```
Type the filename (with or without `.EXE`) at the DOS prompt. The program runs.

---

## 4.11 Displaying a String

### INT 21h Function 9 — Display a string

```
Input:   AH = 9
         DX = offset address of the string
         String MUST end with '$' character
```

The `$` marks the end — it is not displayed.

### The LEA Instruction

To get the address of a variable into a register, use **LEA (Load Effective Address)**:

```asm
LEA    destination, source
```

Where destination is a general register and source is a memory location.

```asm
LEA DX, MSG    ; put the offset address of MSG into DX
```

### The Program Segment Prefix (PSP) Problem

When DOS loads your program into memory, it places a **256-byte PSP (Program Segment Prefix)** before your code. DOS also sets DS to point to the PSP — not to your data segment.

So if you have variables, DS is wrong when your program starts. You must fix it at the beginning:

```asm
MOV AX, @DATA    ; @DATA = the segment number of your .DATA segment
MOV DS, AX       ; point DS to your data segment
```

Why two instructions? Because you **cannot move a number directly into a segment register** — it must go through a general register first.

### Complete string display program (PGM4_2.ASM):

```asm
TITLE PGM4_2: PRINT STRING PROGRAM
.MODEL SMALL
.STACK 100H
.DATA
MSG    DB    'HELLO!$'
.CODE
MAIN    PROC
    ; initialize DS
    MOV AX, @DATA
    MOV DS, AX
    ; display message
    LEA DX, MSG
    MOV AH, 9
    INT 21H
    ; return to DOS
    MOV AH, 4CH
    INT 21H
MAIN    ENDP
END MAIN
```

Running this outputs: `HELLO!`

---

## 4.12 A Case Conversion Program

Combines everything learned so far. Goal: prompt user for a lowercase letter, display it in uppercase on the next line.

Sample run:
```
ENTER A LOWERCASE LETTER: a
IN UPPER CASE IT IS: A
```

### Key trick — lowercase to uppercase conversion

In ASCII:
- Lowercase `a` = 61h
- Uppercase `A` = 41h
- Difference = 20h

So subtracting 20h from any lowercase letter gives the uppercase:
```asm
SUB AL, 20H    ; convert lowercase to uppercase
```

### Data definitions

```asm
CR    EQU    0DH
LF    EQU    0AH

MSG1    DB    'ENTER A LOWERCASE LETTER: $'
MSG2    DB    CR, LF, 'IN UPPER CASE IT IS: '
CHAR    DB    ?, '$'
```

The trick with MSG2 and CHAR: they are declared **consecutively** in memory. MSG2 starts with CR and LF, so when displayed, it automatically moves to the next line. MSG2 doesn't end with `$`, so INT 21h function 9 keeps going and also displays whatever is in CHAR — which the program fills with the converted letter.

### Complete program (PGM4_3.ASM):

```asm
TITLE PGM4_3: CASE CONVERSION PROGRAM
.MODEL SMALL
.STACK 100H
.DATA
MSG1    DB    'ENTER A LOWER CASE LETTER: $'
MSG2    DB    0DH, 0AH, 'IN UPPER CASE IT IS: '
CHAR    DB    ?, '$'
.CODE
MAIN    PROC
    ; initialize DS
    MOV AX, @DATA
    MOV DS, AX
    ; print user prompt
    LEA DX, MSG1
    MOV AH, 9
    INT 21H
    ; input character and convert to upper case
    MOV AH, 1
    INT 21H             ; read lowercase letter into AL
    SUB AL, 20H         ; convert to uppercase
    MOV CHAR, AL        ; store it in CHAR
    ; display on next line
    LEA DX, MSG2
    MOV AH, 9
    INT 21H             ; displays MSG2 + CHAR (continuous in memory)
    ; DOS exit
    MOV AH, 4CH
    INT 21H
MAIN    ENDP
END MAIN
```

---

That's the complete Chapter 4, Riyad. This chapter is the one that ties everything together — after this you can actually write and run real 8086 assembly programs. Ready for the next chapter?