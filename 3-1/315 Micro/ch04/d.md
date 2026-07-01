This is talking about **how your program is organized in memory**. Since you're learning the 8086, let's build the intuition first.

---

# Why does an assembly program have segments?

Imagine you're writing a C program:

```c
#include <stdio.h>

int x = 10;      // data

int main()       // code
{
    int a;       // stack
    printf("%d", x);
}
```

Even though you didn't explicitly separate them, your program already has three different kinds of memory:

* **Instructions (code)** → `main()`
* **Variables (data)** → `x`
* **Function/local variables (stack)** → `a`

The 8086 simply makes this separation **explicit**.

---

# The three segments

## 1. Code Segment (CS)

Contains the instructions the CPU executes.

Example:

```asm
MOV AX, 5
ADD AX, 2
INT 21H
```

These instructions are stored in the **Code Segment**.

Think of it as:

> 📖 "The recipe."

---

## 2. Data Segment (DS)

Contains variables.

Example:

```asm
NUM DB 10
MSG DB 'HELLO','$'
```

These are **not instructions**.

They're just bytes stored in memory.

Think of it as:

> 📦 "The ingredients."

---

## 3. Stack Segment (SS)

Stores temporary information.

For example:

* function calls
* return addresses
* local variables
* saved registers

Think of it as:

> 📚 "A pile of plates."

The last plate placed on top is the first one removed.

---

# Why are they called segments?

The 8086 can access **1 MB** of memory.

Instead of treating it as one huge block, Intel divided it into **segments**.

For example:

```
Memory

+-------------------------+
| Code Segment            |
| instructions            |
+-------------------------+
| Data Segment            |
| variables               |
+-------------------------+
| Stack Segment           |
| stack                   |
+-------------------------+
```

Each segment can be up to **64 KB**.

---

# Then what is `.MODEL`?

The assembler asks:

> "How big do you expect your program to become?"

You answer using

```asm
.MODEL SMALL
```

or

```asm
.MODEL LARGE
```

---

## SMALL model

```
.MODEL SMALL
```

means

```
Code:
+----------------+
| one segment    |
+----------------+

Data:
+----------------+
| one segment    |
+----------------+
```

One code segment.

One data segment.

Since each segment is at most **64 KB**,

* Code ≤ 64 KB
* Data ≤ 64 KB

This is enough for most beginner programs.

---

## MEDIUM

```
.MODEL MEDIUM
```

means

```
Code:
+-------+
|Seg1   |
+-------+
|Seg2   |
+-------+
|Seg3   |
+-------+

Data:
+-------+
|One Seg|
+-------+
```

If your program has lots of code but only a little data.

---

## COMPACT

Opposite of MEDIUM.

```
Code:
+-------+
|One Seg|
+-------+

Data:
+-------+
|Seg1   |
+-------+
|Seg2   |
+-------+
```

Useful when your program stores lots of data but doesn't have much code.

---

## LARGE

```
Code:
Seg1
Seg2
Seg3
...

Data:
Seg1
Seg2
Seg3
...
```

Both code and data can span multiple segments.

---

## HUGE

Same as LARGE, **except** one array is allowed to be larger than 64 KB.

Normally,

```c
char arr[100000];
```

wouldn't fit into one 64 KB segment.

The HUGE model supports such arrays by allowing them to span multiple segments.

---

# Why does everyone use SMALL?

Because beginner programs look like this:

```asm
.MODEL SMALL
.STACK 100H
.DATA
MSG DB "HELLO$"
.CODE
MAIN PROC
...
MAIN ENDP
END MAIN
```

The program is tiny.

64 KB of code is enormous for simple exercises.

64 KB of data is also plenty.

So **SMALL** is the simplest and most common model for learning.

---

## One thing to correct from the text

The book says:

> "Every assembly program has three segments."

Conceptually, yes—**code**, **data**, and **stack** are the standard organization. However, not every assembly program literally has all three explicitly defined. Very small or specialized programs (such as DOS `.COM` programs) can use a **single segment** for everything. The three-segment structure is the typical model used for `.EXE` programs and in most educational examples.

So when your teacher says "every assembly program has code, data, and stack," think of it as the **standard way to organize an 8086 program**, not an absolute rule with no exceptions.
