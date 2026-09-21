---
---

> Module 1 · Post 7 of 13

## 1. Picking Up the Context-Switch Thread

Your computer is running hundreds of programs simultaneously. Your browser, your music player, background system services, all of them apparently alive at once. Yet your CPU is a fundamentally sequential machine: it executes exactly one instruction at a time. So how does it pull off this illusion? The answer lies in a beautifully interlocked set of mechanisms, and this post builds them up from zero, piece by piece, until the full picture clicks into place.

In the last post we saw that the OS pauses and resumes processes thousands of times per second, a technique called the context switch, and that to resume a process correctly, the OS must save three registers: the Program Counter, the Stack Pointer, and the Frame Pointer. But why exactly these three? What makes them special, and what breaks if even one of them is missing? Answering that requires going one level deeper, into the CPU itself, into the stack, and into how function calls actually work. That is what this post does.

## 2. Memory: The Foundation

### 2.1 What Memory Actually Is

Before anything else, you need a clear mental picture of what memory actually is. RAM, Random Access Memory, is, at its most fundamental level, an enormous sequence of storage locations. Each location holds a small, fixed amount of data, typically one byte, which is 8 bits, and each location has a unique number called its address.

Think of it exactly like a street of numbered houses. House 100 is distinct from house 101, which is distinct from house 102, and you can go directly to any house if you know its number. That is where the "Random Access" in RAM comes from: you can jump to any address instantly, without having to scan from the beginning. A tiny slice of memory addressed this way might look like this:

```
Address    Contents
100        LOAD  rax ← 5
101        LOAD  rbx ← 3
102        ADD   rcx ← rax + rbx
103        CALL  square
104        STORE result ← rcx
105        HALT
...
200        MUL   rcx ← rcx * rcx
201        RETURN
```

Your program, before it runs, is just a file sitting on disk. When you launch it, the operating system copies its instructions into memory, assigning each instruction an address exactly like the ones shown above. From that moment, the CPU reads instructions out of memory one by one and executes them.

**Definition:** Memory (RAM) is a large array of byte-sized storage cells, each identified by a unique numeric address. It holds both the instructions of a running program and the data that program reads and writes.

### 2.2 The Memory Hierarchy

Memory sits in a hierarchy. The closer storage is to the CPU, the faster it is, but also the smaller and more expensive it becomes:

| Level | Location | Typical size | Access speed |
|---|---|---|---|
| Registers | Inside CPU chip | ~64 bytes total | ~0.3 nanoseconds |
| L1 Cache | Inside CPU chip | 32-64 KB | ~1 nanosecond |
| L2 Cache | Near CPU | 256 KB - 1 MB | ~4 nanoseconds |
| RAM | On motherboard | 8-64 GB | ~100 nanoseconds |
| SSD | Storage drive | 500 GB+ | ~100,000 nanoseconds |

This hierarchy matters deeply, because it shapes every decision the CPU makes about where to keep data. The closer to the CPU, the better, which is why the CPU has its own tiny storage locations built right into the chip itself. Those are called registers.

## 3. Registers

### 3.1 General-Purpose Registers

If memory is the desk where your work is spread out, registers are your hands: they are the only place where the CPU can actually do work. The CPU cannot add two numbers that are sitting in RAM, and it cannot compare values that are in RAM either. Before any operation can happen, the data must first be moved from memory into a register, and after the operation, the result is stored back into memory.

A modern 64-bit CPU has around 16 to 32 general-purpose registers, each holding 64 bits, or 8 bytes, of data. On x86-64, the architecture in most laptops and desktops, these registers are named `rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, and `r8` through `r15`. While any of them can technically hold anything, convention gives most of them a typical role:

| Register | Common use (not strict, but typical) |
|---|---|
| `rax` | Return value of functions |
| `rbx` | General storage (callee-saved) |
| `rcx` | Counter (loops, shifts) |
| `rdx` | Extra data / multiplication / I/O |
| `rsi` | Source index (data copy) |
| `rdi` | Destination index / 1st argument |

`r8` through `r15` are pure general-purpose registers, used for anything the compiler needs at the moment. To see why registers matter at all, consider what actually happens at the CPU level when you write `int c = a + b;` in C:

```
; In C you write:  int c = a + b;
; The CPU actually does this:

LOAD  rax ← [address of a]     ; bring 'a' from RAM into register rax
LOAD  rbx ← [address of b]     ; bring 'b' from RAM into register rbx
ADD   rcx ← rax + rbx          ; add them, result stays in rcx
STORE [address of c] ← rcx     ; write the result back to RAM
```

Nothing in that sequence happens directly in RAM. Every single operation has to go through a register first.

### 3.2 The Three Special-Purpose Registers

Beyond the general-purpose registers, there are three special-purpose registers that are absolutely critical to understand, because they control where the CPU is executing, how functions are called, and how the stack is organized.

```
1. Program Counter (PC), also called the Instruction Pointer (IP)
2. Stack Pointer (SP)
3. Frame Pointer (FP), also called the Base Pointer (BP)
```

These three registers are so important that the operating system must save and restore them every time it switches from one running program to another. We will see exactly why as we go deeper.

## 4. The Program Counter

### 4.1 What the PC Does

Of all the registers, the Program Counter, called the Instruction Pointer on Intel chips and held in the register named `rip`, is the single most important one. It answers the most fundamental question the CPU asks every moment of its life: which instruction should I execute right now?

The PC is just a number, nothing more. That number is a memory address, and the CPU reads whatever instruction is stored at that address, executes it, and then, in the normal case, automatically adds one instruction's worth to the PC, so it points to the very next instruction.

**Definition:** The Program Counter (PC) is a register that holds the memory address of the next instruction the CPU will fetch and execute. It auto-increments after each instruction, moving the CPU forward through the program one step at a time.

The significance of this becomes clear the moment you think about the operating system pausing your program to run someone else's. If the OS pauses your process while the PC holds the value 4,827,392, and later restores your process, it puts 4,827,392 back into the PC. The CPU then continues from that exact instruction, as if nothing happened. The PC is your program's bookmark: save it, and you can resume anywhere.

### 4.2 The Fetch-Decode-Execute Cycle

The CPU does not think, and it does not plan. It blindly executes one loop, billions of times per second, until the machine is turned off. That loop is called the fetch-decode-execute cycle.

Fetch: the CPU looks at the PC, held in `rip`. It goes to that address in memory, say `rip = 0x400100`, reads the instruction stored there, for example `mov rax, 5`, and brings it into the CPU.

Decode: the CPU figures out what the binary instruction means, breaking the bits into parts: the operation is `mov`, the destination is `rax`, and the value is `5`. This is done entirely by hardware logic; the CPU has circuitry that decodes the raw bits into a recognized operation, whether it is an `ADD`, a `LOAD`, a `JUMP`, or a `CALL`. At the end of decoding, the CPU knows exactly what it has to do: put the value `5` into the register `rax`.

Execute: the CPU carries out the operation. It might add two numbers, as in `add rax, rbx`, move data around, or compare values, as in `cmp rax, 10`. Whatever the instruction says to do, the actual computation or data movement happens at this step.

Update PC: in the normal case, the CPU increments the PC by one instruction's worth, so `rip = rip + instruction_size`, pointing it at the next instruction. In special cases, such as a `JUMP`, a `CALL`, or a `RETURN`, the CPU sets the PC to a completely different address instead.

Repeat, forever: the CPU goes back to the fetch step and never stops.

This cycle is so fundamental that everything your computer does, playing video, running a machine learning model, rendering a web page, is ultimately just this loop executing over and over, billions of times per second. The sophistication is in what the instructions tell it to do, not in some magical intelligence belonging to the CPU itself.

## 5. JUMP, CALL, and RETURN: How Control Flow Changes

Normal execution is linear: the PC goes 100, 101, 102, 103, and so on. But programs are not linear. They have `if` statements, loops, and functions, and all of these require the CPU to change direction, to set the PC to a completely different address. Three instructions handle this.

### 5.1 JUMP: A One-Way Leap

A `JUMP` instruction simply sets the PC to a new address. It carries no memory of where it came from and no way back; it is strictly a one-way trip. This is exactly what makes an `if` statement work at the CPU level:

```
; C code:  if (x > 0) { doSomething(); }

300  CMP  rax, 0                  ; compare x with 0
301  JUMP_IF_NOT_GREATER 400      ; if x <= 0, skip to 400
302  ...                          ; doSomething() body runs here
303  ...
400  ...                          ; code continues after the if block
```

If the condition is false, the CPU sets `PC = 400`, completely skipping instructions 302 and 303. This is also how loops work: a `JUMP` placed at the bottom of the loop body sends the PC backward to the top, repeating the body again.

### 5.2 CALL: A Jump That Leaves a Note

When you call a function, you need to come back once it is done. `CALL` is a `JUMP` that first saves the return address, the address of the instruction right after the `CALL`, so the CPU knows exactly where to resume once the function finishes. Here is what happens, step by step, when the CPU executes a `CALL`:

```
; PC is at 200. Instruction: CALL square (which is at address 800)

Step 1, save the return address onto the stack:
Stack ← push 201     ; "when done, come back to 201"

Step 2, jump to the function:
PC ← 800              ; start executing square() at address 800

; Now the CPU runs 800, 801, 802, ... until it hits RETURN
```

### 5.3 RETURN: Coming Home Using the Saved Note

At the end of the function, `RETURN` does the opposite of `CALL`. It picks up the saved return address and sets the PC to it:

```
; PC is at 850 (the last instruction of square). Instruction: RETURN

Step 1, retrieve the return address from the stack:
return_addr ← pop Stack    ; the stack gives back 201

Step 2, jump back to the caller:
PC ← 201                    ; resume exactly where CALL was made

; CPU continues: 201, 202, 203, ...
; The function call was completely transparent.
```

### 5.4 Comparing the Four Kinds of Control Flow

It helps to see all four kinds of instruction side by side, since each one treats the PC and the stack a little differently:

| Instruction | Changes PC to | Uses stack? | Can return? |
|---|---|---|---|
| Normal execution | PC + 1 (auto) | No | — |
| JUMP | Any address | No | No, one way |
| CALL | Function address | Yes, pushes return address | Yes, via RETURN |
| RETURN | Popped return address | Yes, pops return address | That is the point |

## 6. The Stack

In the discussion above, we have been referring to "the stack" without fully defining it. Time to fix that.

### 6.1 What the Stack Is

The stack is a dedicated region of a process's memory used to manage function calls. It operates on one strict rule: you can only add to the top, and you can only remove from the top. This is called Last In, First Out, or LIFO, and it behaves exactly like a stack of plates: you can put a plate on top, and when you need one, you take it from the top as well.

**Definition:** The stack is a LIFO region of memory where the CPU automatically saves function call information, such as return addresses, local variables, and parameters. Adding to it is called a push, and removing from it is called a pop.

### 6.2 The Stack Frame

Every time a function is called, a chunk called a stack frame is pushed onto the stack. A stack frame holds everything that function needs in order to run: the return address, which is where to go once the function finishes; the parameters, which are the arguments passed into the function; the local variables declared inside the function's body; and any saved registers, meaning old register values that must be restored once the function returns.

### 6.3 Why Registers Must Be Saved and Restored

Registers are shared by every function in the program. If one function changes them freely, it can break another function that was relying on their previous values. That is why a function saves a register onto the stack before it modifies that register. Because the stack remembers the old values, each function effectively gets "its own version" of the registers it uses, even though physically the same handful of registers are being reused safely across every function call.

When a function returns, its memory on the stack is no longer in use. The stack pointer moves back up, the frame disappears, and the stack shrinks: it had grown downward while the function was active, and now it shrinks back upward as that function's frame is released. The previous function's frame, having sat exposed underneath the whole time, becomes active again. This is beautifully automatic: the CPU does not need to manually track which function called which. The stack structure itself encodes the entire call history.

### 6.4 A Call Chain on the Stack: main → cook → chop

Consider this call chain: `main()` calls `cook()`, which in turn calls `chop()`. While `chop()` is running, the stack looks like this:

```
High memory addresses (stack base)
┌────────────────────────────────┐
│  main() frame                   │  ← oldest, at the bottom
│  local: int minutes = 30        │
│  return addr: OS entry          │
├────────────────────────────────┤
│  cook() frame                   │
│  local: int temp = 200          │
│  return addr: main+5            │
├────────────────────────────────┤
│  chop() frame                   │  ← newest, at the top
│  local: int pieces = 4          │
│  return addr: cook+3            │
├────────────────────────────────┤  ← SP points here
Low memory addresses (stack grows this way ↓)
```

The SP's exact position relative to these addresses can vary by system architecture; what matters is the relationship between the frames, not the specific numbers.

When `chop()` returns, its frame disappears, the SP moves up, and execution resumes inside `cook()` right where it left off. When `cook()` returns, the same thing happens, sending control back to `main()`. The stack is the physical record of how you got here.

### 6.5 Why This Matters: Stack Traces

When your program crashes and you see a stack trace in an error message, that is literally the contents of the stack being printed out: the chain of function calls that led to the crash, from the most recent call back to `main()`. A stack trace is a snapshot of history.

## 7. The Stack Pointer and the Frame Pointer

The CPU needs to know two things about the stack at all times: where the top currently is, and where the current function's local data begins. Two dedicated registers track exactly this.

### 7.1 The Stack Pointer, Always Tracking the Top

The Stack Pointer, `RSP` on x86-64, is a register that always holds the memory address of the most recently pushed item, the current top of the stack. As frames are pushed and the stack grows, `RSP` moves to a lower address. As frames are popped and the stack shrinks, `RSP` moves to a higher address.

**Definition:** The Stack Pointer (SP) is a register that always points to the current top of the stack. It moves automatically on every push and pop.

### 7.2 What PUSH Actually Does

Let's say `RSP` starts at `0x1000` and `rax` holds the value `42`.

```
Before push:
Address    Value
───────    ─────
0x1000     (empty)    ← RSP points here (current top)
0x0FF8     (empty)
0x0FF0     (empty)
```

When you write `push rax`, two things happen, in this exact order. First, `RSP` moves down by 8. Why 8? Because on a 64-bit system, every register holds 8 bytes of data, so every data push needs 8 bytes of space.

```
RSP = 0x1000 - 8 = 0x0FF8
```

```
Address    Value
───────    ─────
0x1000     (empty)    ← was the top before
0x0FF8     (empty)    ← RSP now points here (new top)
0x0FF0     (empty)
```

Second, `rax`'s value, `42`, is written at the new `RSP` location:

```
Memory[0x0FF8] = 42
```

```
Address    Value
───────    ─────
0x1000     (empty)
0x0FF8     42        ← RSP points here, rax is stored here
0x0FF0     (empty)
```

### 7.3 What POP Actually Does

`pop rax` is the exact reverse. First, the value is read from wherever `RSP` currently points:

```
rax = Memory[RSP]
rax = Memory[0x0FF8]
rax = 42
```

Second, `RSP` moves up by 8:

```
RSP = RSP + 8
RSP = 0x0FF8 + 8 = 0x1000
```

The stack shrank by one item, and `RSP` went back up. It helps to keep a small glossary of these terms straight, since words like "top," "bottom," and "grows" can be misleading if you picture the stack the wrong way around:

| Concept | Correct meaning |
|---|---|
| Top of stack | The address pointed to by `RSP` |
| Bottom of stack | The higher address (the start, or base, of the stack frame) |
| Stack grows | Toward lower addresses |
| Push | `RSP` decreases, then the value is stored at the new `RSP` |
| Pop | The value is read from `RSP`, then `RSP` increases |

### 7.4 What Is RBP?

`RBP` stands for Register Base Pointer. It is just a register, a tiny storage location inside the CPU itself, exactly like `RSP`, `RAX`, `RBX`, and all the others. It holds one thing: a memory address. Sitting alongside the other registers inside the CPU, it might look like this at some given moment:

```
CPU (inside)
┌──────────────────────────────────────────────┐
│  RAX = 42       ← general purpose, holds data │
│  RBX = 7        ← general purpose, holds data │
│  RSP = 0x0FF8   ← always tracks top of stack  │
│  RBP = 0x1008   ← holds a chosen reference addr│
│  PC  = 0x4020   ← tracks current instruction  │
└──────────────────────────────────────────────┘
```

What makes `RBP` special is not the register itself, but a convention: an agreement that programmers and compilers follow. When a function starts, its code copies `RSP`'s current value into `RBP`, and then never changes `RBP` again for the entire duration of that function.

That is the entire secret. `RBP` is just a register that we voluntarily agree to keep still while `RSP` moves around freely.

## 8. The Basic Situation: Functions Call Functions

Programs are never flat. Functions call other functions, like this:

```c
void main() {
    func();
}

void func() {
    int a = 10;
    int b = 20;
}
```

`main` runs first. Then it calls `func`. Then `func` runs. Then `func` finishes and returns back to `main`, which continues. This is completely normal, but it creates a problem that needs to be solved carefully.

### 8.1 Every Function Needs Its Own Workspace

When `main` is running, it has local variables that live on the stack, and `main` needs a fixed anchor, `RBP`, to find them reliably. So while `main` is running:

```
RBP = 0x2000    ← main's anchor, pointing to main's workspace
```

Now `main` calls `func`. `func` also has local variables, `a` and `b`, and it also needs its own `RBP` anchor to find them. But here is the critical problem.

### 8.2 There Is Only One RBP Register

The CPU has exactly one `RBP` register. Not two, not one per function, just one.

```
CPU (inside)
┌───────────────┐
│  RBP = 0x2000  │  ← only ONE exists, currently belongs to main
└───────────────┘
```

When `func` starts and needs to set up its own anchor, it must put a new value into `RBP`, something like:

```
RBP = 0x1008    ← func's new anchor
```

But the moment `0x1008` is written into `RBP`, the old value, `0x2000`, which belonged to `main`, is gone from the register forever.

```
CPU (inside)
┌───────────────┐
│  RBP = 0x1008  │  ← main's 0x2000 is destroyed, gone
└───────────────┘
```

And `main` is not finished. When `func` returns, `main` still needs to access its own local variables using `RBP`. If `0x2000` is gone, `main` is broken. So before that value is destroyed, it must be saved somewhere safe.

### 8.3 Where Do We Save It?

We save it on the stack. The stack is exactly the right place for temporary storage like this, and this is what the single instruction `push rbp` does: it saves the current value of `RBP` onto the stack.

It helps to keep two very different-looking numbers straight here. An address is the slot number in RAM; a value is the number stored in that slot. They look similar because they are both just numbers written in hexadecimal, but they are two completely different things. The return address, `0x4008`, is a number that represents a location in the program's code; specifically, it means "when the function finishes, jump to instruction `0x4008` in the program."

Let's trace it physically. Before `func` starts:

```
Address     Value
--------    --------
0x1010      0x4008      ← return address (where to go back in main's code)
0x1008      (empty)
0x1000      (empty)

RSP = 0x1010
RBP = 0x2000   ← main's anchor value, still sitting in the register
```

The rule that governs everything in this section is simple: only save something if it will be destroyed and it is needed again later. Once `func` has finished setting up, here is exactly what that rule says about each value involved:

| Value | Stored where | Gets destroyed? | Needed again? | Must save? |
|---|---|---|---|---|
| Main's RBP (0x2000) | RBP register | Yes, `func` overwrites it | Yes, `main` needs it back | YES |
| Func's RBP (0x1008) | RBP register | Yes, the next function will overwrite it | No, nobody needs `func`'s anchor after `func` ends | NO |
| Return address (0x4008) | Pushed by `call` | Yes, `ret` consumes it | Yes, to get back to `main` | YES, already saved by `call` |

`func`'s own `RBP`, `0x1008`, is never saved anywhere, because once `func` returns, that value is irrelevant: the frame is gone, and the address means nothing anymore. Only `main`'s value needs preserving, and only because `main` is still alive and waiting to resume.

### 8.4 Registers vs RAM: Two Separate Things

Registers are completely separate from RAM. They are tiny storage locations physically inside the CPU chip itself; they are not part of RAM at all, even though a register's value can happen to look like a RAM address. Laid out side by side, right before `push rbp` runs, the picture looks like this:

```
REGISTERS (inside CPU)                RAM (physical memory chips)
────────────────────────              ────────────────────────────────
RSP = 0x1010                          Address     Value        What it is
RBP = 0x2000                          --------    --------     ─────────────────────────────
PC  = 0x4000                          0x1010      0x4008       ← return address (RSP here)
RAX = 42                              0x1008      (empty)      ← nothing useful here yet
                                       0x1000      (empty)      ← nothing useful here yet
                                       0x0FF8      (empty)      ← nothing useful here yet
```

These are two completely separate physical things, connected by wires but not the same thing. `RSP` is a register that holds the value `0x1010`, and that value happens to be a RAM address, so `RSP` is said to be pointing at RAM slot `0x1010`:

```
RSP = 0x1010
         │
         └──────────────► RAM slot 0x1010 contains 0x4008
```

`RSP` does not contain `0x4008`. `RSP` contains `0x1010`, and `0x1010` happens to be the address of a RAM slot that itself contains `0x4008`. This relationship, a register holding an address that points to something in RAM, is exactly what a pointer is.

`RBP` holds `0x2000` in this same picture. That address is somewhere else in RAM, specifically `main`'s stack frame, further up in memory; it is not shown here because it is not relevant to this particular trace.

### 8.5 Tracing push rbp, Step by Step

When a function starts, two instructions run in sequence:

```
push rbp          ; save the caller's rbp value, to restore it later
mov  rbp, rsp      ; copy rsp's current value into rbp
```

The single instruction `push rbp` actually does two steps internally:

```
RSP = RSP - 8           ; make space, growing the stack downward by 8 bytes
Memory[RSP] = RBP        ; go to the address now in RSP, and write RBP's value there
```

Step 1 changes only the register; RAM is not touched at all:

```
REGISTERS               RAM
──────────               ──────────────────────────
RSP = 0x1008              0x1010    0x4008    ← untouched
RBP = 0x2000               0x1008    (empty)   ← RSP now points here
                          0x1000    (empty)
```

`RSP` changed from `0x1010` to `0x1008`. That is the entire step; nothing has been written to RAM yet.

Step 2 writes `RBP`'s value into the RAM slot that `RSP` is now pointing at. Since `RSP = 0x1008` and `RBP = 0x2000`, the CPU writes `0x2000` into RAM slot `0x1008`:

```
REGISTERS               RAM
──────────               ──────────────────────────
RSP = 0x1008              0x1010    0x4008    ← return address, untouched
RBP = 0x2000               0x1008    0x2000    ← RSP points here, main's RBP saved
                          0x1000    (empty)
```

`0x2000` now exists in two places at once: it is still sitting in the `RBP` register, and it has also been copied into RAM at address `0x1008`. The register still holds it for now, but this copy in RAM is what will keep it safe once `func` overwrites the register.

The next instruction, `mov rbp, rsp`, runs completely separately, after `push rbp` is done. At this moment, `RSP = 0x1008`, `RBP` still holds the old value `0x2000`, and `RAM[0x1008]` holds the copy, `0x2000`, that was just saved. The instruction does exactly one thing: it copies the value inside `RSP` into `RBP`.

```
RBP = RSP = 0x1008
```

RAM is not touched at all here; only the register changes:

```
REGISTERS                     RAM
──────────                     ──────────────────────────
RSP = 0x1008                    0x1010    0x4008    ← return address
RBP = 0x1008  (changed)         0x1008    0x2000    ← still 0x2000, untouched
                                0x1000    (empty)
```

After this step, the `RBP` register holds `0x1008`; the old value `0x2000` is gone from the register, overwritten. But `RAM[0x1008]` still holds `0x2000`, the saved copy, completely untouched.

Both `RSP` and `RBP` now hold `0x1008`, and both point to the same RAM slot, which itself contains `0x2000`. There is nothing contradictory about this; the difference is in what happens next. `RSP` will keep changing as the very next instructions move it, while `RBP` stays frozen at `0x1008` for as long as `func` is running.

`RBP = 0x1008` means, from this point forward, that `func`'s stack frame starts at address `0x1008` in RAM. That fact is not lost when `func` eventually returns; it is already physically encoded in the stack itself. `RAM[0x1008]` simply is the slot at address `0x1008`; the address does not need to be stored anywhere separately, because the slot is physically there in RAM and does not disappear on its own.

### 8.6 What Actually Needs Saving, and Why

Looking back over everything that just happened, only one thing genuinely needed saving. `main`'s old `RBP` value, `0x2000`, lived only in the `RBP` register with no other copy anywhere, so the moment `func` was about to overwrite `RBP` with `0x1008`, that value would have disappeared for good unless it was saved to RAM first, which is exactly what `push rbp` accomplished.

Nothing else needed saving. The new value, `0x1008`, does not need to be preserved anywhere, because when `func` finishes and runs `pop rbp`, the CPU simply reads `0x2000` back out of `RAM[0x1008]` and restores it into `RBP`, while `RSP` automatically moves back up to `0x1010`. The address `0x1008` itself becomes irrelevant the instant `func`'s frame disappears; nobody needs to remember it afterward.

### 8.7 Why Copy RSP Into RBP Specifically?

The reason is that `RSP` is pointing at the top of the stack at exactly the right moment: the boundary between what was already on the stack, the return address and the saved old `RBP`, and what is about to be created, the function's own local variables. It is the perfect dividing line.

```
(above RBP) → return address, parameters     ← already existed
              ↑
              RBP frozen here  ← the dividing line
              ↓
(below RBP) → local variables                ← about to be created
```

The final setup instruction, `sub rsp, 16`, moves `RSP` down by 16 to make room for two local variables, while `RBP` does not move at all:

```
RSP = 0x1008 - 16 = 0x0FF8
```

```
CPU REGISTERS                  RAM
─────────────                  ──────────────────────────
RSP = 0x0FF8   ← changed        Address    Value
RBP = 0x1008   ← unchanged      ───────    ─────
PC  = 0x5000                    0x1018     (empty)
                                0x1010     0x4008    ← return address
                                0x1008     0x2000    ← saved main's RBP  ← RBP
                                0x1000     (empty)   ← space for variable a
                                0x0FF8     (empty)   ← space for variable b  ← RSP
```

Now `RSP = 0x0FF8` and `RBP = 0x1008`. They have diverged again: `RSP` moved to make room for the new local variables, while `RBP` stayed exactly where it was set.

### 8.8 The Full Trace, From Call to Return

Putting the whole sequence together, for this code:

```c
void main() {
    func();
}

void func() {
    int a = 10;
    int b = 20;
}
```

the complete life cycle looks like this:

```
WHILE main IS RUNNING:
─────────────────────
RBP register = 0x2000    (main owns RBP, pointing to main's frame)

MAIN CALLS FUNC:
────────────────
call func runs
  → return address (0x4008) pushed to RAM[0x1010]
  → PC jumps to func

FUNC STARTS:
────────────
push rbp runs
  → 0x2000 (main's RBP) saved to RAM[0x1008]
  → RBP register still = 0x2000

mov rbp, rsp runs
  → RBP register = 0x1008  (func now owns RBP)
  → 0x2000 is safely in RAM[0x1008]
  → 0x2000 is GONE from the register (overwritten)

sub rsp, 16 runs
  → space made for func's local variables

FUNC RUNS NORMALLY:
────────────────────
RBP = 0x1008 throughout
RSP moves up and down freely
local variables found at [RBP - 8], [RBP - 16], etc.

FUNC RETURNS:
─────────────
mov rsp, rbp runs
  → RSP = RBP = 0x1008  (local variables discarded)

pop rbp runs
  → RBP = RAM[0x1008] = 0x2000   ← main's value RESTORED to register
  → RSP = 0x1010

ret runs
  → PC = RAM[0x1010] = 0x4008    ← return address used
  → RSP = 0x1018

BACK IN MAIN:
─────────────
RBP register = 0x2000    (main owns RBP again, exactly as before)
```

## 9. Why the Frame Pointer Exists: A Deeper Look

Now we switch to a deeper example to understand why the frame pointer exists, what happens with three nested calls, and why the OS must save all three registers during a context switch. The new code is:

```c
void main() {
    int a = 1;
    foo();
}

void foo() {
    int b = 2;
    bar();
}

void bar() {
    int c = 3;
}
```

### 9.1 Finding a Variable Through a Moving RSP

Let's build this understanding from scratch using `foo()` as the example. When `foo()` starts, its stack frame is set up: `RBP` is locked in place, and `RSP` is free to move.

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x2FF8                 Address    Value
RBP = 0x3008                 ───────    ─────────
                             0x3010     0x4008    ← return address   [RBP+8]
                             0x3008     0x4000    ← saved main's RBP [RBP+0] ← RBP
                             0x3000     (empty)   ← space for b      [RBP-8]
                             0x2FF8     (empty)   ← RSP here
```

Now, inside `foo()`, suppose the CPU does some temporary work and saves a couple of registers before doing calculations:

```
push rax    → RSP = 0x2FF8 - 8 = 0x2FF0
push rbx    → RSP = 0x2FF0 - 8 = 0x2FE8
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x2FE8   ← moved       Address    Value
RBP = 0x3008   ← unmoved     ───────    ─────────
                             0x3010     0x4008    ← return address
                             0x3008     0x4000    ← saved main's RBP  ← RBP STILL HERE
                             0x3000     (empty)   ← variable b
                             0x2FF8     (empty)
                             0x2FF0     saved rax
                             0x2FE8     saved rbx  ← RSP moved here
```

Now try finding variable `b` using `RSP` as the reference point:

```
Before the pushes:  b = RAM[RSP + 8]  = RAM[0x2FF8 + 8]  = RAM[0x3000]  ✓
After one push:     b = RAM[RSP + 16] = RAM[0x2FF0 + 16] = RAM[0x3000]  ✓ but the offset changed
After two pushes:   b = RAM[RSP + 24] = RAM[0x2FE8 + 24] = RAM[0x3000]  ✓ but the offset changed again
```

Variable `b` is still at the same RAM address, `0x3000`, every time. But its distance from `RSP` keeps changing: 8, then 16, then 24. Every single push and pop shifts the offset, so the compiler would have to track every push and pop happening anywhere in the function and recalculate the offset every time it needed to reach a variable. That is impossible to manage reliably in practice.

### 9.2 Finding the Same Variable Through a Frozen RBP

Now find `b` using `RBP` instead:

```
Before the pushes:  b = RAM[RBP - 8] = RAM[0x3008 - 8] = RAM[0x3000]  ✓
After one push:      b = RAM[RBP - 8] = RAM[0x3008 - 8] = RAM[0x3000]  ✓ same
After two pushes:    b = RAM[RBP - 8] = RAM[0x3008 - 8] = RAM[0x3000]  ✓ still the same
```

`RBP` never moved, so the offset is always `-8`, no matter what `RSP` is doing elsewhere in the function. This is exactly why `RBP` exists: it is a frozen reference point that keeps variable access reliable, regardless of how many temporary pushes and pops happen around it.

## 10. The Full Call Chain: main → foo → bar

Now let's trace all three functions building their frames one by one.

### 10.1 main Sets Up Its Frame

Starting state, as `main` begins:

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x5018                 Address    Value
RBP = 0x6000                 ───────    ─────────
PC  = 0x4000                 0x5018     (empty)   ← RSP here
                             0x5010     (empty)
                             0x5008     (empty)
                             0x5000     (empty)

(RBP = 0x6000 is main's own anchor from whatever called it, somewhere
 further up in RAM; it is not shown here because it is not relevant.)
```

`main` runs its prologue and stores its local variable, `a = 1`:

```
push rbp        → RSP = 0x5010, RAM[0x5010] = 0x6000  (saved caller's RBP)
mov rbp, rsp     → RBP = 0x5010
sub rsp, 8       → RSP = 0x5008
store a = 1      → RAM[RBP-8] = RAM[0x5010-8] = RAM[0x5008] = 1
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x5008                 Address    Value
RBP = 0x5010                 ───────    ─────────
PC  = 0x4000                 0x5018     (empty)
                             0x5010     0x6000    ← saved caller's RBP  ← RBP
                             0x5008     1         ← variable a = [RBP-8]  ← RSP
```

`main`'s frame is now fully built, and `RBP = 0x5010` is `main`'s anchor.

### 10.2 main Calls foo

```
call foo runs
RSP = 0x5008 - 8 = 0x5000
RAM[0x5000] = 0x4010    ← return address (back into main after foo finishes)
PC jumps to foo
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x5000   ← changed     Address    Value
RBP = 0x5010   ← unchanged   ───────    ─────────
PC  = 0x6000   ← jumped      0x5018     (empty)
                             0x5010     0x6000    ← saved caller's RBP  ← RBP
                             0x5008     1         ← variable a
                             0x5000     0x4010    ← return address  ← RSP
```

### 10.3 foo Sets Up Its Frame

`foo` runs its own prologue. It must save `main`'s `RBP`, `0x5010`, before overwriting it with its own:

```
push rbp        → RSP = 0x5000-8 = 0x4FF8, RAM[0x4FF8] = 0x5010  (saved main's RBP)
mov rbp, rsp     → RBP = 0x4FF8  (foo's anchor locked here)
sub rsp, 8       → RSP = 0x4FF0
store b = 2      → RAM[RBP-8] = RAM[0x4FF8-8] = RAM[0x4FF0] = 2
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x4FF0                 Address    Value
RBP = 0x4FF8                 ───────    ─────────
PC  = 0x6000                 0x5010     0x6000    ← main's saved caller RBP  ← main's RBP anchor
                             0x5008     1         ← main's variable a
                             0x5000     0x4010    ← return address (back to main)
                             0x4FF8     0x5010    ← saved main's RBP  ← foo's RBP anchor
                             0x4FF0     2         ← foo's variable b  ← RSP
```

`foo`'s frame is now fully built, and `RBP = 0x4FF8` is `foo`'s anchor.

### 10.4 foo Calls bar

```
call bar runs
RSP = 0x4FF0 - 8 = 0x4FE8
RAM[0x4FE8] = 0x6010    ← return address (back into foo after bar finishes)
PC jumps to bar
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x4FE8   ← changed     Address    Value
RBP = 0x4FF8   ← unchanged   ───────    ─────────
PC  = 0x7000   ← jumped      0x5010     0x6000    ← main's saved RBP  ← main's anchor
                             0x5008     1         ← main's a
                             0x5000     0x4010    ← main's return address
                             0x4FF8     0x5010    ← foo's saved RBP  ← foo's anchor
                             0x4FF0     2         ← foo's b
                             0x4FE8     0x6010    ← return address  ← RSP
```

### 10.5 bar Sets Up Its Frame

`bar` runs its own prologue and saves `foo`'s `RBP`, `0x4FF8`, before overwriting it:

```
push rbp        → RSP = 0x4FE8-8 = 0x4FE0, RAM[0x4FE0] = 0x4FF8  (saved foo's RBP)
mov rbp, rsp     → RBP = 0x4FE0  (bar's anchor locked here)
sub rsp, 8       → RSP = 0x4FD8
store c = 3      → RAM[RBP-8] = RAM[0x4FE0-8] = RAM[0x4FD8] = 3
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x4FD8                 Address    Value
RBP = 0x4FE0                 ───────    ─────────
PC  = 0x7000                 0x5010     0x6000    ← main's saved RBP     ─┐
                             0x5008     1         ← main's a              │ main's frame
                             0x5000     0x4010    ← main's return addr   ─┘
                             0x4FF8     0x5010    ← foo's saved RBP      ─┐ points back to main
                             0x4FF0     2         ← foo's b               │ foo's frame
                             0x4FE8     0x6010    ← foo's return addr    ─┘
                             0x4FE0     0x4FF8    ← bar's saved RBP      ─┐ points back to foo
                             0x4FD8     3         ← bar's c  ← RSP       ─┘ bar's frame  ← RBP
```

`bar`'s frame is now fully built. `RBP = 0x4FE0` is `bar`'s anchor, and `RSP = 0x4FD8` is the current top of the stack.

### 10.6 The Chain of Saved RBP Values

This is the complete picture of three nested function frames sitting on the stack simultaneously. Each frame holds its own saved `RBP`, pointing back to the previous frame, and together they form a chain:

```
RBP (0x4FE0) → RAM[0x4FE0] = 0x4FF8 → RAM[0x4FF8] = 0x5010 → RAM[0x5010] = 0x6000
    bar's anchor          foo's anchor          main's anchor        caller's anchor
```

This chain is exactly what a debugger reads to show you a stack trace. It walks backward through this chain of saved `RBP` values to reconstruct the entire history of calls that led to the current point.

### 10.7 bar Returns, Step by Step

`bar: mov rsp, rbp` discards `bar`'s local variable `c` by bringing `RSP` back up to `RBP`:

```
RSP = RBP = 0x4FE0
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x4FE0   ← changed     Address    Value
RBP = 0x4FE0   ← unchanged   ───────    ─────────
                             0x4FE0     0x4FF8    ← bar's saved RBP  ← RSP and RBP here
                             0x4FD8     3         ← dead (below RSP now)
```

`bar`'s variable `c` is now dead; `RSP` has moved past it.

`bar: pop rbp` reads `RAM[RSP] = RAM[0x4FE0] = 0x4FF8`, puts it into `RBP`, and moves `RSP` up by 8:

```
RBP = RAM[0x4FE0] = 0x4FF8    ← foo's RBP restored to the register
RSP = 0x4FE0 + 8 = 0x4FE8
```

```
CPU REGISTERS               RAM
──────────────               ──────────────────────────
RSP = 0x4FE8   ← changed     Address    Value
RBP = 0x4FF8   ← RESTORED    ───────    ─────────
                             0x4FF8     0x5010    ← foo's saved RBP  ← RBP now here
                             0x4FF0     2         ← foo's b
                             0x4FE8     0x6010    ← foo's return addr  ← RSP here
```

`RBP = 0x4FF8`, `foo`'s anchor, is back in the register, and `foo`'s entire frame is fully accessible again: `foo`'s `b` is at `[RBP-8] = [0x4FF8-8] = [0x4FF0] = 2`, correct.

`bar: ret` reads the return address from `RAM[RSP] = RAM[0x4FE8] = 0x6010`, jumps there, and moves `RSP` up:

```
PC  = 0x6010    ← back inside foo's code
RSP = 0x4FE8 + 8 = 0x4FF0
```

Execution is now back inside `foo()`. `RBP = 0x4FF8`, and `foo`'s variable `b` is at `[RBP-8] = [0x4FF0] = 2`, perfectly accessible.

### 10.8 foo Returns

`foo` returns the same way, using the same three instructions:

```
mov rsp, rbp   → RSP = 0x4FF8
pop rbp        → RBP = RAM[0x4FF8] = 0x5010  (main's anchor restored)
                 RSP = 0x5000
ret            → PC = RAM[0x5000] = 0x4010  (back into main)
                 RSP = 0x5008
```

Execution is now back in `main`. `RBP = 0x5010`, and `main`'s variable `a` is at `[RBP-8] = [0x5008] = 1`, perfectly accessible.

## 11. The Context Switch: Why All Three Registers Must Be Saved

### 11.1 Process A Is Paused Mid-Function

Now suppose the OS decides to pause Process A while it is executing inside `bar()`, in order to run Process B for a while. At this exact moment, Process A looks like this:

```
CPU REGISTERS               RAM (Process A's stack)
──────────────               ──────────────────────────
RSP = 0x4FD8                 Address    Value
RBP = 0x4FE0                 ───────    ─────────
PC  = 0x7042                 0x5010     0x6000    ← main's saved RBP
                             0x5008     1         ← main's a
                             0x5000     0x4010    ← main's return addr
                             0x4FF8     0x5010    ← foo's saved RBP
                             0x4FF0     2         ← foo's b
                             0x4FE8     0x6010    ← foo's return addr
                             0x4FE0     0x4FF8    ← bar's saved RBP  ← RBP
                             0x4FD8     3         ← bar's c  ← RSP
```

Process A is sitting inside `bar()`, about to execute instruction `0x7042`, and its entire call chain, `main → foo → bar`, is sitting on the stack in RAM.

The OS saves Process A's three critical registers into its PCB:

```
PROCESS A's PCB (saved in OS memory)
─────────────────────────────────────
saved PC  = 0x7042    ← exact instruction bar() was about to run
saved RSP = 0x4FD8    ← top of stack, bar's c is here
saved RBP = 0x4FE0    ← bar's anchor, c is at [RBP-8]
```

The OS then loads Process B's saved registers and runs Process B for a while. Process A sits paused, its entire state frozen inside its PCB, with its stack sitting completely untouched in RAM.

### 11.2 Process A Is Resumed

Later, the OS restores Process A by loading the PCB values straight back into the CPU registers:

```
CPU REGISTERS (restored from the PCB)
────────────────────────────────────
PC  = 0x7042    ← CPU resumes exactly here inside bar()
RSP = 0x4FD8    ← entire stack intact, bar → foo → main chain preserved
RBP = 0x4FE0    ← bar's anchor restored, c findable at [RBP-8]
```

Process A resumes at instruction `0x7042` inside `bar()`. Variable `c` is at `[RBP-8] = [0x4FE0-8] = [0x4FD8] = 3`, correct. The entire call chain from `bar` back through `foo` to `main` is still intact on the stack. Process A has no idea it was ever paused.

### 11.3 What Breaks If Any One Register Is Not Saved

It is worth walking through exactly what would go wrong if the OS skipped saving any single one of these three registers:

| Register | If not saved | What breaks |
|---|---|---|
| PC | The CPU has no idea which instruction to run. | It jumps to a garbage address; the process crashes immediately. |
| RSP | The stack top is lost, and the entire call chain becomes unreachable. | `bar` cannot find `c`, cannot return to `foo`, and `foo` cannot return to `main`. Everything breaks. |
| RBP | `bar`'s anchor is gone, so `bar` cannot find its local variable `c`. | Reading `c = RAM[RBP-8]` reads from the wrong address, a silent wrong answer, and `bar` also cannot restore `foo`'s anchor correctly when it tries to return. |

All three together form a complete snapshot of exactly what Process A was doing. Lose any one of them and the process is broken beyond recovery. Save all three, and the process resumes perfectly, as if it had never been paused at all.

## Conclusion

We started with a simple question: why does the OS save exactly three registers during a context switch, the PC, the RSP, and the RBP? The answer is now concrete. The PC is the bookmark that tells the CPU which instruction to resume from. The RSP is the pointer to the top of the stack; lose it, and the entire call chain of every function currently in progress becomes unreachable. The RBP is the fixed anchor inside the current function; lose it, and that function can no longer find any of its own local variables. Together, these three registers are the minimum complete description of what the CPU was doing at any given moment.

The previous post looked at how the OS manages processes from the outside: creating them, scheduling them, tracking them in the PCB. This post has looked at the inside of what the OS is actually preserving every time it performs a context switch. Together, the two perspectives give the complete picture of how a modern operating system runs many programs on a single machine.

The next post turns to a different kind of illusion entirely: how every process on your machine believes it owns the whole of your computer's memory, even though hundreds of other processes are sharing that same physical RAM at the very same time.

**Next:** [The Illusion of Private Memory, How Every Process Thinks It Owns Your Computer →](./08-illusion-of-private-memory.md)
