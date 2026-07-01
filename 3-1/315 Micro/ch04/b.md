
---
High and Low Bytes of a Word

WORD1    DW    1234H

    Low byte (34h) → at address WORD1
    High byte (12h) → at address WORD1+1

This is because the 8086 stores the low byte first (little-endian). 
---
why? why not at a single address as word is of 2 byte =16bits. isn't 16 bit enough to store 1234h in binary?
---
---
Excellent question, Riyad. This is one of the biggest conceptual hurdles when learning computer architecture.

The confusion comes from mixing up **memory** and **registers**.

---

## Step 1: Is `1234h` really 16 bits?

Yes.

```text
1234h
= 0001 0010 0011 0100₂
```

That's exactly **16 bits (2 bytes)**.

So you're right:

> "A word is 16 bits. Why not store it at one address?"

The answer is...

> **Because memory addresses point to bytes, not words.**

---

## Step 2: What does one memory address hold?

In the 8086,

**One address = One byte (8 bits)**

Imagine memory like this:

| Address | Stores |
| ------- | ------ |
| 1000    | 8 bits |
| 1001    | 8 bits |
| 1002    | 8 bits |
| 1003    | 8 bits |

Notice that there is **no address that stores 16 bits.**

Each address stores exactly **1 byte**.

---

## Step 3: Then how is a 16-bit word stored?

Suppose

```asm
WORD1 DW 1234h
```

The assembler splits it into two bytes.

```
1234h

High byte = 12h
Low byte  = 34h
```

Now it stores

| Address | Value |
| ------- | ----- |
| 1000    | 34h   |
| 1001    | 12h   |

because the 8086 is **little-endian**.

---

## Step 4: But the CPU is 16-bit!

Yes!

The **CPU registers** are 16 bits.

For example,

```text
AX

+--------+--------+
|  AH    |  AL    |
+--------+--------+
 12h       34h
```

AX can directly contain

```
1234h
```

because AX is a **16-bit register**.

Registers are different from memory.

---

## Step 5: So what happens when we do

```asm
MOV AX, WORD1
```

Memory contains

| Address | Value |
| ------- | ----- |
| 1000    | 34h   |
| 1001    | 12h   |

The CPU reads **both bytes**.

First byte = 34h

Second byte = 12h

Then combines them into

```
AX = 1234h
```

The programmer doesn't need to combine them manually.

---

## Step 6: Why doesn't one address hold 16 bits?

Think of apartment numbers.

```
Room 1000
Room 1001
Room 1002
```

Each room can hold **one box**.

If you have a large object needing two boxes, you place

```
Room1000 -> first box
Room1001 -> second box
```

You don't suddenly create a "Room1000½" that holds twice as much.

Memory works the same way.

Each address is fixed at **1 byte**.

---

## Step 7: Why little-endian?

The CPU designers could have chosen either

### Little-endian (8086)

| Address | Value |
| ------- | ----- |
| 1000    | 34h   |
| 1001    | 12h   |

or

### Big-endian

| Address | Value |
| ------- | ----- |
| 1000    | 12h   |
| 1001    | 34h   |

Both represent the same 16-bit value.

Intel simply chose **little-endian** as its storage convention. Once that choice was made, every Intel processor has followed it for compatibility.

---

### The key takeaway

You're thinking:

> "A word is 16 bits, so one address should hold a word."

But in the 8086 architecture:

* **Memory is byte-addressable** → **1 address = 1 byte**
* A **word = 2 consecutive bytes**
* Registers like `AX` are **16 bits**, so they can hold the entire word after reading those two bytes.

Once you separate **memory (byte-addressable)** from **registers (16-bit storage)**, the whole picture becomes much clearer.
