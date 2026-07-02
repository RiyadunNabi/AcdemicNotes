# Chapter 2: Processes and Threads — Full Study Notes

> Built from your teacher's slide deck + Tanenbaum's *Modern Operating Systems* Ch.2, explained from zero assumed knowledge. Slide order is kept as the backbone; extra explanation is added wherever the slide alone wouldn't make sense.

---

## 0. Why this chapter matters (the big picture)

Before processes/threads, think about what your laptop is actually doing right now: Pop!_OS is running GNOME (Wayland), Brave with your profile shortcuts, maybe a terminal, maybe Spotify, background stuff like NetworkManager, systemd services, etc. — probably 200+ things "running" — but your CPU almost certainly has just 4, 6, or 8 physical cores. So how does one core run hundreds of things?

Answer: it doesn't, really. It runs **one thing at a time**, switches to another after a few milliseconds, switches again, and so on — so fast that it *looks* like everything is happening simultaneously. This illusion-of-parallelism trick is what the "process" abstraction exists to manage. This chapter is about **what a process is**, **how the OS keeps track of it**, and **why we also need something lighter called a thread**.

---

## 1. What is a Program?

A **program** is just a file sitting on disk (e.g., `a.out`, or `/usr/bin/firefox`). It's dead, inert data until you run it. It contains:

- **Code** — machine instructions (what to do)
- **Data** — variables that will be stored and manipulated in memory:
  - initialized global variables
  - dynamically allocated variables (`malloc`, `new`)
  - stack variables (function parameters, local variables in C/C++)

A program by itself does *nothing*. It's a recipe, not a meal.

### Process ≠ Program

This is the single most important distinction in this chapter. Example the slide gives: if you open **two windows of Firefox**, that's the *same program* (same code on disk) but **two separate processes** — each with its own memory, its own state, its own PID. If one crashes, the other is unaffected.

You can verify this yourself right now on Pop!_OS:
```bash
pgrep -fl brave    # or firefox — lists every running instance as a separate PID
```
Each PID = one process, even though they all loaded the same executable.

### The Cake-Baking Analogy (very useful, memorize this)

| Analogy | Real term |
|---|---|
| The recipe | The **program** (algorithm written down) |
| The baker | The **processor (CPU)** |
| Flour, eggs, sugar | The **input data** |
| The *activity* of reading the recipe, fetching ingredients, and baking | The **process** |

Now imagine the baker's kid runs in crying — stung by a bee. The baker **remembers exactly where he was in the recipe** (this is called *saving state*), switches to "administering first aid" (a totally different "program" — the first aid manual), finishes that, and then goes back to baking **exactly where he left off**. This is literally what a CPU does when it switches between processes — it's called a **context switch**, and we'll get to the formal mechanics of it in section 6.

**Key takeaway:** A *program* is static (sits on disk, does nothing). A *process* is dynamic — an activity with a program, input, output, and a **state** (where exactly it currently is in its execution).

---

## 2. From Program to Process: Process in Memory

When you double-click an icon or type a command, the OS has to physically bring that program to life. This is where **loading** happens.

### 2.1 How a Program Becomes a Process

```
Executable (a.out, all 1s and 0s on disk)
        │
        ▼
      Loader ──► reads it into memory
        │
        ▼
        OS:
  1. Loads the executable file into memory
  2. Creates a kernel data structure for the process (its PCB — see §3)
  3. Initializes data (globals get their starting values, stack/heap set up)
  4. Starts execution from an entry point (main())
```

So "process address space" = the full set of memory addresses this specific process is allowed to touch. **No other process can see into it** (this is enforced by the OS + hardware — it's why one crashing app usually doesn't take down your whole system).

### 2.2 The Layout of a Process's Memory (Address Space)

This is a diagram you should be able to redraw from memory — it comes up constantly in OS courses (memory management, virtual memory, buffer overflows, etc.):

```
 max address
┌─────────────┐
│    stack    │  ← grows DOWNWARD (toward lower addresses)
│      ↓      │
│             │
│   (free)    │  ← unused space between stack and heap
│             │
│      ↑      │
│    heap     │  ← grows UPWARD (toward higher addresses)
├─────────────┤
│    data     │  ← global/static variables (fixed size, set at load time)
├─────────────┤
│    text     │  ← the actual machine code (usually read-only)
└─────────────┘
 address 0
```

- **text**: your compiled code. Usually read-only (so a bug can't accidentally overwrite the program's own instructions).
- **data**: global variables, e.g. `int counter = 0;` at file scope in C++.
- **heap**: dynamic memory — everything from `malloc()`/`new`. Grows upward as you allocate more.
- **stack**: function call frames — local variables, function parameters, return addresses. Grows downward. Every function call pushes a new "stack frame"; returning pops it off.

The stack and heap grow *toward each other* in the middle gap — if they ever collide, you get a **stack overflow** (this is literally why that term exists — it's not just a website name!).

This layout matters a lot later for your AI course too, indirectly — e.g. deep recursion in your N-Puzzle A* search (if you did it recursively) eats stack space; that's this diagram in action.

---

## 3. Process Data Structures: the PCB

The OS obviously needs to keep track of every process somehow — where its memory is, what its registers currently hold, whether it's running or waiting, etc. It stores all of this in a structure called the **Process Control Block (PCB)**, also called a **process table entry**.

Think of the PCB as a process's "ID card + medical record" combined — everything the OS needs to know to pause it, resume it, or manage it.

| Category | Example fields |
|---|---|
| **Process management** | Registers, Program Counter, Program status word, Stack pointer, Process state, Priority, Scheduling parameters, Process ID (PID), Parent process, Process group, Signals, Start time, CPU time used, Children's CPU time, Time of next alarm |
| **Memory management** | Pointer to text segment, Pointer to data segment, Pointer to stack segment |
| **File management** | Root directory, Working directory, File descriptors, User ID, Group ID |

Every process in the system has one of these sitting in kernel memory. On Linux, you can peek at a live approximation of this via:
```bash
cat /proc/<pid>/status
```
That file is basically a human-readable window into (part of) the kernel's PCB for that process.

---

## 4. Process Creation

The OS doesn't spontaneously create processes. There are exactly **four principal events** that trigger process creation:

1. **System initialization** — at boot, dozens of background processes ("daemons") start automatically: e.g. on your Pop!_OS system, things like `systemd`, `NetworkManager`, `gnome-shell`, `pulseaudio`/`pipewire` etc. all spin up before you even log in.
2. **A running process executes a process-creation system call** — e.g. your shell creating a new process when you run a command.
3. **A user explicitly requests it** — you type a command in the terminal, or double-click an app icon.
4. **Initiation of a batch job** — mostly relevant to old-school mainframe/queued job systems, less common today outside HPC clusters.

### How this actually works under the hood (Linux specifics — relevant to you)

The slide keeps this abstract, but since you're on Linux daily, it's worth knowing concretely:

- **UNIX/Linux** has exactly one core system call for this: **`fork()`**. It creates an *exact clone* of the calling process — same memory image, same open files, same everything, just a new PID. The clone (**child**) then typically calls **`execve()`** (or a wrapper like `execvp`) to *replace* its memory image with a completely different program. This two-step dance (`fork` then `exec`) is why, e.g., when your shell runs `ls`, it first clones itself (fork), and the clone then overwrites itself with the `ls` program (exec) — leaving your original shell process untouched and waiting.
- **Windows** does this in one shot with a single `CreateProcess()` call — no fork/exec split.
- Modern Linux doesn't actually copy all of the parent's memory on fork (that would be slow and wasteful) — it uses **copy-on-write**: parent and child *share* the same physical memory pages until one of them tries to *modify* something, at which point only that specific page gets copied. This is a classic OS optimization trick worth remembering.

---

## 5. Process Termination

"Nothing lasts forever, not even processes." A process ends for one of four reasons:

| Type | Voluntary? | Example |
|---|---|---|
| **Normal exit** | Voluntary | Program finishes its job and calls `exit()` |
| **Error exit** | Voluntary | Program detects an error itself and quits (e.g., "file not found", so it exits cleanly) |
| **Fatal error** | Involuntary | Crash — e.g. segmentation fault, divide-by-zero |
| **Killed by another process** | Involuntary | Something else sends it a kill signal (`kill -9 <pid>`, or the OOM-killer ending a process that's eating too much RAM — you've probably seen this if your laptop overheated/froze during heavy compiles) |

---

## 6. Lifecycle of a Process (Process States)

As a process exists, it moves between five states:

```
        admitted
 new ─────────────► ready ◄────────────────┐
                       │  \                 │
              scheduler │   \  I/O or event │
              dispatch  │    \ completion   │
                       ▼      \             │
                    running    \            │
                       │  \     \           │
                interrupt│   \    ┌─────────┴──┐
                       │    \  I/O or event   │
                       │     ▶│    waiting      │
                     exit     │   (blocked)     │
                       │      └─────────────────┘
                       ▼
                  terminated
```

In plain words:

- **new**: process is being created (OS is setting up its PCB, address space, etc.)
- **ready**: process is fully set up and *could* run, but the CPU is currently busy with something else — it's waiting its turn.
- **running**: it currently has the CPU and its instructions are actively executing.
- **waiting / blocked**: it's paused because it's waiting on something external — disk I/O, keyboard input, a network response, a timer — and literally *cannot* proceed even if given the CPU right now.
- **terminated**: finished (see §5).

The transitions matter:
- **ready → running**: the OS *scheduler dispatches* it (gives it the CPU).
- **running → ready**: an **interrupt** happens (e.g., its time slice ran out) — it's forced to give up the CPU even though it wasn't done.
- **running → waiting**: it voluntarily blocks on I/O or some event (e.g. it called `read()` on the keyboard and there's no input yet).
- **waiting → ready**: the I/O/event it was waiting for completes — but note it goes back to **ready**, *not* straight to running. It still has to wait its turn again.

**Common exam trap:** a process can *never* go directly from waiting → running. It must pass through ready first.

---

## 7. The Process Model & Multiprogramming

This is the conceptual model that makes all of the above sane to reason about.

- There is (for now, assume) **only one physical CPU**, and therefore only **one physical program counter (PC)**.
- But conceptually, we pretend there are many independent processes, each with its **own logical PC**, running "in parallel."
- **Multiprogramming** = the OS rapidly switching the real, physical PC/registers back and forth between processes' saved logical states.

```
(a) Only one physical PC, four programs sitting in memory
(b) Conceptually: four independent processes, each imagining it has its own PC
(c) Reality over time: only ONE process is ever actually running at any single instant,
    but zoomed out over a second, all of them have made progress.
```

This gives the *illusion* of parallelism even on a single core — Tanenbaum calls this **pseudoparallelism**, to contrast with **true parallelism** on a **multiprocessor/multicore** system (where two+ CPUs really do execute simultaneously, sharing the same physical memory). Since you're on an HP Victus with a multicore CPU, you actually get *some* true parallelism too — but even with, say, 8 cores and 300 running processes, most of that "everything is happening at once" feeling is still pseudoparallelism underneath.

**Important consequence:** because switching timing is unpredictable and not reproducible, you must never write code that silently assumes a certain timing (e.g. "this loop will always take about 2ms so I'll use it as a delay") — the OS could preempt you mid-loop for an arbitrary amount of time.

---

## 8. Context Switching

This ties §2 (PCB) and §6 (states) together mechanically.

**Setup:** while a process runs, all its live state (PC, stack pointer, general-purpose registers) sits *inside the actual CPU*, being actively modified.

**When the OS decides to switch from process P0 to P1:**

1. **Save** P0's current register values into **P0's own PCB** (freezing its exact state).
2. **Load** P1's saved register values from **P1's PCB** into the CPU.
3. P1 resumes executing exactly where it left off last time.

```
process P0        operating system         process P1
executing  │
    ▼      │──interrupt/syscall──►  save state into PCB₀
   idle    │                              ⋮
    │      │                        reload state from PCB₁ ──────►  executing
    │      │                                                            ▼
    │      │◄──interrupt/syscall────  save state into PCB₁            idle
    │      │                              ⋮
executing◄─┼────────────────────  reload state from PCB₀
```

This whole save-then-load procedure is called a **context switch**. Two things to note:
- It's **pure overhead** — no useful work happens *during* the switch itself, it's bookkeeping.
- It's **very machine-dependent** (different CPU architectures have different register sets, so the exact save/restore code differs per hardware).

### Why creating/switching a process is expensive

- Building a new PCB: relatively cheap.
- **Setting up a whole new address space**: expensive (allocating memory, setting up page tables, etc.)
- A full process context switch also has to flush/reload memory-management hardware state (cache, TLB — Translation Lookaside Buffer, segmentation buffers) because the *new* process has a completely different address space than the old one.

This expense is exactly the motivation for the next big topic.

---

## 9. Enter Threads: "We need something lighter"

**The problem:** processes are heavyweight to create and heavyweight to switch between, mainly because of the *address-space* setup/teardown and memory-management flushing. But often you don't actually need a totally separate, isolated program — you just want **several independent streams of execution that cooperate closely and share the same data**, without paying the full process cost every time.

**The solution:** most modern OSes support *two* entities:

| Entity | Defines |
|---|---|
| **Process** | The address space + general resources/attributes (the "container") |
| **Thread** | A sequential execution stream *within* a process (the "worker") — like a mini-process |

- A thread is always bound to exactly one process.
- A single process can contain **many threads**.
- **Threads are the unit that actually gets scheduled** by the CPU scheduler — *processes* are just containers that threads execute inside of.

### The one new ability threads need

Threads within the *same* process **share the entire address space and all its data** — recall from §2 that separate *processes* never share address spaces (each process's memory is isolated). Threads deliberately break that isolation *within* one process, on purpose, because they're assumed to be cooperating, not competing.

### Multithreaded Processes — visual

```
single-threaded process:                 multithreaded process:
┌───────────────────────┐                 ┌────────────────────────────────┐
│  code   data   files   │                │        code   data   files      │
├───────────┬────────────┤                ├──────────┬──────────┬──────────┤
│ registers │   stack     │                │registers │registers │registers │
│           │             │                ├──────────┼──────────┼──────────┤
│           │             │                │  stack   │  stack   │  stack   │
│    thread → 〜           │                │   〜      │   〜      │   〜  ← thread│
└───────────────────────┘                 └────────────────────────────────┘
```

Notice: **code, data, and open files are shared** (one copy for the whole process) but **each thread gets its own registers and its own stack**.

---

## 10. The Classical Thread Model

### 10.1 Shared vs. Private state

| **Shared across all threads in a process (per-process)** | **Private to each thread (per-thread)** |
|---|---|
| Address space (code, data, heap) | Program Counter |
| Open files, I/O | Registers |
| Global variables | Execution stack |
| Child processes | State (ready/running/blocked) |
| Accounting info | |

The intuition: everything that describes the process's *identity and resources* is shared; everything that describes what a specific line of execution is currently *doing* is private.

### 10.2 Why does each thread need its own stack?

Because each thread is independently calling different functions, at different points in time, with different call depths — they each have a **different execution history**. If threads shared one stack, thread A's function-call frames would get interleaved with (and corrupt) thread B's — you'd have no reliable way to know whose local variables/return addresses are whose. So: separate stack per thread is not optional, it's required for correctness.

```
        ┌────────── Process ──────────┐
        │   T1↑    T2↑     T3↑        │
        │  [T1's] [T2's]  [T3's]      │
        │  stack   stack   stack      │
        └──────────────────────────────┘
```

### 10.3 Thread states

Just like processes (§6), a thread can be **running**, **ready**, **blocked**, or **terminated** — same diagram, same transitions, just scoped to the thread instead of the whole process.

---

## 11. Thread Context Switch (cheaper than process context switch)

Same basic idea as §8 (save registers, load registers) — **but** since threads in the same process already share the address space, switching between them **does not require touching memory management at all**. No new page tables, no cache/TLB flush.

**Direct comparison (this table is exam gold):**

| | **Process Context Switch (PCS)** | **Thread Context Switch (TCS)** |
|---|---|---|
| User→kernel mode switch? | Required | Not required (for user-level threads) |
| Address space change? | Yes | No — stays the same |
| Memory addresses flushed? | Yes (all of them get flushed) | No, they remain saved |
| Memory-management overhead (cache, TLB, segmentation buffer flush/reload)? | Yes | No |
| What's actually switched | PC, registers, stack pointers **+** full memory-management state | Just PC, registers, stack pointers |

This is *the* reason threads exist: **cheap creation, cheap switching**, while still letting multiple things happen "at once" within one program.

---

## 12. Concurrent vs. Parallel Execution (don't mix these up)

- **Concurrent execution (single core):** the CPU rapidly time-slices between threads. Only one is *actually* executing at any instant, but they all make progress over time — this is the same pseudoparallelism idea from §7, just applied to threads instead of processes.
```
single core:  T1 | T2 | T3 | T4 | T1 | T2 | T3 | T4 | T1 | ...
```
- **Parallel execution (multicore):** with multiple cores, threads can genuinely run *at the exact same instant* on different cores — real, simultaneous execution.
```
core 1:  T1 | T3 | T1 | T3 | T1 | ...
core 2:  T2 | T4 | T2 | T4 | T2 | ...
```

Your Victus's CPU has multiple cores, so a well-written multithreaded program on your machine gets genuine parallel speedup, not just the illusion of it.

---

## 13. Why Threads? — Concrete Use Cases

Beyond "it's cheaper," threads are genuinely useful for structuring programs:

- **Simplify coding** for programs with multiple loosely-independent activities happening logically at once.
- **Better CPU utilization** — while one thread blocks on I/O, another can keep the CPU busy.
- **Better responsiveness** — e.g. a UI thread stays responsive to clicks while a background thread does heavy work.
- **Cheaper create/switch** than full processes (§11).
- **Exploit multicore parallelism** for real speedups (§12).

### Worked example: Word Processor

```
      Keyboard  ───►  ┌──────────────────────────┐  ───►  Disk
                       │  Kernel                    │
                       │   〜thread    〜thread       │
                       │  (typing/     (formatting/  │
                       │   display)     autosave)    │
                       └──────────────────────────┘
```

A word processor can have one thread handling your live typing/screen redraw and another thread doing something slow like reformatting a huge document or auto-saving to disk. **Question the slide poses: what if it were single-threaded?** Answer: while it's busy reformatting or saving, it would completely freeze and ignore your keystrokes until that operation finishes — exactly the "app not responding" freeze you've probably experienced.

### Other classic examples (from the book, good to know even though not explicitly on the slide)

- **Web server**: one thread accepts incoming requests and immediately hands each one to a worker thread, so a slow disk-bound request for one client doesn't block every other client.
- **Producer–consumer / pipeline processing**: e.g. one thread reads a big file in from disk, another thread processes the data, a third thread writes results back out — all overlapping instead of happening strictly one after another.

---

## 14. Thread Implementation

Where does the actual scheduling logic for threads live? Two fundamentally different approaches, plus a hybrid:

### 14.1 User-Level Threads

```
                Run-time     Thread
                system       table
                   ↓           ↓
 User space:  ┌─────────────────────┐  ┌─────────────┐
              │  Process (3 threads) │  │  Process     │
              │     〜  〜  〜         │  │    〜  〜      │
              └─────────────────────┘  └─────────────┘
 ─────────────────────────────────────────────────────
 Kernel space:              Kernel
                                          [Process table]
```

- The kernel is **completely unaware** that multiple threads exist — as far as the kernel is concerned, it just sees one process.
- A **user-level runtime library** does all the scheduling itself, using a thread table it maintains entirely in user space.
- Each thread = just a PC, registers, stack, and a small control block — all managed at user level.
- Creating a thread, switching threads, synchronizing threads — **all happen without ever entering the kernel** (no system call, no trap).

**Advantages:**
- **Very fast context switching** — it's just a local procedure call in user space; no trap to kernel, no memory flush.
- **Customizable scheduling** — the application can implement whatever scheduling policy it wants, since the kernel isn't involved.

**Disadvantages:**
- **Blocking problem**: if *any* one user-level thread makes a blocking system call (e.g. `read()` on the keyboard), the **entire process** blocks — because the kernel only sees one process, it has no idea other threads inside it could otherwise run. (This defeats a lot of the point of using threads in the first place!)
- **No protection**: threads must voluntarily be "polite" and yield the CPU to each other. A buggy or greedy thread can simply hog the CPU forever, since there's no kernel-level enforcement.

### 14.2 Kernel-Level Threads

```
 User space:      Process              Process
                   〜  〜  〜             〜  〜  〜  〜
 ──────────────────────────────────────────────────
 Kernel space:   Kernel
                 [Process table] [Thread table]
```

- The **kernel itself** knows about and manages each individual thread — it maintains the thread table (not the application).
- Every thread operation (create, switch, destroy) requires a **system call** (trap into the kernel).

**Advantages:**
- If one kernel-level thread blocks (e.g. on I/O), the kernel can immediately schedule another thread from the *same* process to run — no more "one thread blocks everything" problem.

**Disadvantages:**
- Context switching is **more expensive** than user-level, since every operation needs a trap into the kernel, plus the kernel has to do extra validation/checking (it can't blindly trust user code).

### 14.3 Hybrid Implementation

```
        Multiple user threads
           on a kernel thread
                 ↓   ↓
 User space:  ┌───────────┐
              │  〜  〜  〜   │
              └───────────┘
 ──────────────────────────
 Kernel space:    〜  ← kernel thread
                 Kernel
```

- Multiplexes many user-level threads on top of a (usually smaller) number of kernel-level threads.
- The kernel only sees and schedules the **kernel-level** threads.
- Each kernel thread has a set of user-level threads that take turns running "inside" it, scheduled entirely at user level — same as §14.1 but now with kernel awareness at a coarser grain, avoiding the total-blocking problem while keeping most switches cheap.
- This is genuinely the best-of-both-worlds approach and is closer to what real-world thread libraries (like `pthreads`/NPTL on Linux) conceptually aim for.

### Quick comparison table

| | **User-Level Threads** | **Kernel-Level Threads** |
|---|---|---|
| Managed by | Application (runtime library) | Kernel |
| Kernel aware? | No | Yes |
| Context switch cost | Cheap | Expensive |
| Max number | As many as you want | Limited by kernel resources |
| Ease of use | Needs careful handling | Simpler (kernel handles blocking correctly) |

---

## 15. Quick self-test (try answering before checking the slide/book)

1. Two terminal windows both running `bash` — same program, same process, or different processes? *(Answer: same program, different processes — just like the Firefox example.)*
2. Why does the stack grow downward and the heap upward in a process's address space?
3. Name all four events that can trigger process creation.
4. Can a process go directly from **waiting** to **running**? Why or why not?
5. What's the one memory-management-related step a process context switch does that a thread context switch does *not* need to do?
6. Why does a buggy user-level thread have the power to freeze an entire process, but a buggy kernel-level thread doesn't have that same effect on other threads?

---

## 16. Where this connects to your other coursework

- The **address space diagram (§2.2)** is foundational for anything involving pointers, dynamic memory, and recursion depth in C++ — directly relevant to your CSE 318 A* N-Puzzle work if you're using recursive search or deep STL containers.
- **fork/exec (§4)** underlies how your shell (bash, on Pop!_OS) runs every single command you type — worth trying `strace -f -e trace=fork,execve <command>` sometime if you're curious to literally watch it happen.
- Later in this course you'll build on the **thread model** (§9–14) directly into **race conditions, synchronization, semaphores, and the classical IPC problems** (producer-consumer, dining philosophers, readers-writers) — those all assume you're solid on "threads share memory, therefore they can step on each other," which is exactly why §10.1's "shared state" column matters so much.