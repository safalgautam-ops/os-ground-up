---
---

> Module 1 · Post 4 of 13

## 1. What Is a Program?

A program is a passive, dead thing. It is nothing more than a file sitting on disk: a collection of instructions and data that exists as information, and nothing else. It does not run on its own, and it does not do anything until something else brings it to life.

When you install an application, you are simply placing a program file on your disk. That file sits there, unused, until you decide to run it. It is much like a recipe written on paper: the recipe itself does not cook anything. It is only information, waiting to be acted on.

```
Your Disk
─────────
chrome.exe        ← just a file, sitting here, doing nothing
python.exe        ← just a file, sitting here, doing nothing
myapp.exe         ← just a file, sitting here, doing nothing
```

A program has no memory allocated to it, no registers, and no CPU time assigned to it. It is completely passive.

## 2. What Is a Process?

A process is what happens the moment you take a program and actually run it. The operating system picks up that dead file from disk, loads it into RAM, allocates memory for it, sets up its registers, and starts the CPU executing its instructions.

At that moment, the program becomes a process. It is now alive.

```
Disk              OS loads it          RAM (active, running)
─────────         ──────────────►      ──────────────────────
chrome.exe                             Process (pid=1042)
                                        - code loaded into memory
                                        - stack created
                                        - heap created
                                        - registers set up
                                        - CPU executing it
```

The key distinction is this: a program is like a recipe on paper, passive and sitting on disk. A process is like a chef actually cooking using that recipe, active, living in memory, and using the CPU.

This distinction matters in practice. You can open Chrome twice, and both windows will be running the exact same program file, `chrome.exe`, yet they are two completely separate processes. Each one has its own memory, its own registers, its own stack, and its own heap. None of that is shared between them.

```
Disk              Running in RAM
─────────         ──────────────────────────────
                   Process 1042 (first Chrome window)
chrome.exe   ──►   Process 1043 (second Chrome window)
                   Process 1044 (third Chrome window)
```

One program. Three processes. Three completely independent running instances.

## 3. Process Memory Layout (Process Address Space)

When your program runs, the operating system creates a virtual address space for it. This is not the actual physical RAM in your machine; it is an abstraction that makes the program feel like it owns a large, continuous block of memory entirely to itself, even though many programs are actually sharing the same physical RAM underneath.

Within this virtual address space, memory is divided into distinct regions, each with a specific purpose. These regions are arranged in a fixed order, from low addresses at the bottom to high addresses at the top, and they interact with each other in well-defined ways during execution. When the OS creates a process, it assigns this structured block of memory, known as the process's address space, and this becomes the complete working environment for the program.

```
High addresses
┌───────────────────────┐
│         STACK          │  ← grows downward
│  local vars, params    │
│  return addresses      │
├───────────────────────┤
│           ↓             │
│      (free space)       │
│           ↑             │
├───────────────────────┤
│         HEAP           │  ← grows upward
│  malloc'd memory       │
│  dynamic structures    │
├───────────────────────┤
│         BSS            │  ← uninitialized globals/statics
│  zeroed at startup     │  (int x; / int x = 0;)
├───────────────────────┤
│         DATA           │  ← initialized globals/statics
│  copied from disk      │  (int x = 5;)
├───────────────────────┤
│         TEXT           │  ← machine code instructions
│      (read-only)       │
└───────────────────────┘
Low addresses
```

### 3.1 Text Segment: The Foundation

At the very bottom of the address space sits the Text segment, also called the Code segment. This is where the compiled machine code of your program lives. When you write a function in C, compile it, and run it, the actual CPU instructions produced by that compilation are loaded here.

The Text segment is loaded into memory by the OS loader when your program first starts, before your program does anything at all. It is read-only during execution: the program cannot modify its own instructions while running. This is a deliberate protection. If the Text segment were writable, a bug or malicious code could overwrite instructions and change the program's behavior in unpredictable ways.

The CPU has a special register called the Program Counter (PC), also called the Instruction Pointer. At every clock cycle, the CPU looks at the Program Counter, fetches the instruction at that address in the Text segment, executes it, and moves the Program Counter forward to the next instruction. Your entire program's execution is, at its core, the CPU walking through the Text segment one instruction at a time. Everything else in memory exists to support this process.

### 3.2 Data Segment: Initialized Static Storage

Directly above the Text segment sits the Data segment, also called the Initialized Data segment. It holds global variables and static variables that the programmer has given an explicit initial value.

When you write `int x = 5;` at the global level, the value `5` is stored inside the executable file itself, on disk. When the OS loads your program, it copies this value directly from the executable file into the Data segment in RAM. From that moment, the variable `x` lives at a fixed address in the Data segment for the entire lifetime of the program, from before `main()` runs until the program exits completely.

This is what makes global and static variables fundamentally different from local variables. They are not created and destroyed as functions are called; they exist continuously. Any function in your program can access them at any time, because their address is fixed and known ahead of time. The compiler bakes these addresses directly into the machine code in the Text segment. When a function needs to read `x`, the instruction says, in effect, "go to address 0x601020 and read the value there," and that address points into the Data segment.

### 3.3 BSS Segment: Uninitialized Static Storage

Sitting directly above the Data segment is the BSS segment, short for Block Started by Symbol. It serves the same general purpose as the Data segment: it holds global and static variables for the entire lifetime of the program. The difference is that BSS holds variables that were not given an explicit, non-zero initial value by the programmer. When you write `int count;` or `int total = 0;` at the global level, these go into BSS, not the Data segment.

This distinction matters for a concrete reason. The Data segment must store actual values, such as `5` or `3.14`, inside the executable file on disk. But BSS variables are all zero by definition, and there is no point storing thousands of zeros in the executable file. That would waste both disk space and load time for no benefit. So the executable file only records how large the BSS section needs to be, a single size value, and nothing else.

When the OS loads the program, before `main()` ever runs, the runtime system reads that size, allocates the right amount of memory, and fills the entire region with zeros. This happens automatically and invisibly. It is also why, in C, uninitialized global variables are guaranteed to be zero: this is not a coincidence or a compiler courtesy, it is the deliberate behavior of the BSS mechanism.

From the program's perspective during execution, Data and BSS feel identical. Both live at fixed addresses, both persist for the lifetime of the program, and both are accessible from anywhere in the code. The distinction between them only matters at the level of the executable file format and the loading process.

### 3.4 Heap: Dynamic Runtime Memory

Above BSS begins a fundamentally different kind of region: the Heap. Everything below the Heap, Text, Data, and BSS, is determined entirely at compile time. Their sizes are fixed before the program ever runs. The Heap is different. It exists to handle memory whose size and lifetime cannot be known until the program is actually running.

When your program calls `malloc(n)` in C, it is asking the operating system, indirectly, for `n` bytes of memory on the Heap. The allocator finds a free block of the right size and gives your program a pointer to it. That memory now belongs to your program and persists until you explicitly call `free()` on that pointer. It does not matter whether the function that allocated it has already finished executing; the memory remains alive regardless. The Heap grows upward, toward higher addresses, as more memory is allocated.

This is the critical behavioral difference from everything below it. Data and BSS variables are born when the program starts and die when the program exits; their lifetime is tied to the program itself. Heap memory behaves very differently. It is not tied to any function or scope automatically. Instead, it is controlled manually by the programmer, which leads to an important idea: a function can allocate memory, return, and disappear from execution, and yet the memory it allocated continues to exist. The only thing keeping that memory useful is a pointer, a variable that stores its address. As long as at least one valid pointer to it exists, you can still access and free that memory later.

This power comes with a direct cost. If you allocate heap memory and then lose every pointer to it, whether by returning from a function without saving the pointer, or by overwriting the pointer with something else, the memory is still allocated and still belongs to your process, but you can no longer reach it. You cannot free it. It stays occupied until the program exits. This is called a memory leak, and in long-running programs it causes the process to slowly consume more and more RAM until the system runs out.

The Heap is where complex data structures live: linked lists, trees, hash tables, and dynamically sized arrays. These structures are created at runtime, may grow or shrink, and may need to outlive the functions that created them, which is exactly what the Heap is designed for.

When you allocate heap memory, you get back a pointer, an address:

```c
int *p = malloc(sizeof(int));
```

At this point, memory has been allocated somewhere in RAM, and `p` stores the address of that memory. You can use it and later release it:

```c
free(p);
```

So far, everything is fine. Now consider this variation:

```c
int *p = malloc(sizeof(int));
p = NULL;
```

The memory is still allocated, but you have just destroyed the only pointer that knew where it was. You no longer know the address, you cannot access the memory, and you cannot call `free()` on it. That memory is lost until the program ends.

A related mistake looks like this:

```c
void func() {
    int *p = malloc(sizeof(int));
}
```

Here, `p` is created inside the function, and memory is allocated for it. But the moment the function ends, `p` disappears along with the rest of its stack frame. The allocated memory is still sitting in the Heap, but no variable anywhere in your program points to it anymore. Once again, the memory is lost.

### 3.5 Stack: The Engine of Function Execution

At the top of the address space, at the highest addresses, sits the Stack. Unlike the Heap, which grows upward, the Stack grows downward, toward lower addresses. This means the Stack and Heap grow toward each other. In practice, the OS ensures there is enough space between them, but in theory, if both grew large enough, they would collide. This is called a stack-heap collision, and it results in a crash.

The Stack is the engine that makes function calls work. Every time a function is called, the CPU automatically creates a stack frame for it: a block of memory on the Stack containing the function's local variables, the parameters passed to it, and the return address, which is the address in the Text segment where execution should resume once this function finishes.

Local variables on the Stack do not persist. The moment a function returns and its stack frame is destroyed, those variables are gone. If you return a pointer to a local variable from a function, you are handing back a pointer to memory that no longer meaningfully belongs to anyone. This is a dangling pointer, and accessing it afterward is undefined behavior.

## 4. How a Process Is Created, Step by Step

When you double-click a program icon or type a command in the terminal, the OS creates a process. This does not happen instantaneously or all at once; it happens in a specific, well-defined sequence of steps.

### 4.1 Step 1: The OS Reads the Program from Disk

The OS opens the executable file from disk. The file follows a specific format, ELF on Linux, PE on Windows, and the OS reads this file to understand what it contains: where the code is, where the static data is, and how much memory the program needs.

```
Disk
──────────────
myprogram.exe  ← OS reads this file
               ← finds: code section, data section, size info
```

### 4.2 Step 2: The OS Allocates Memory and Loads Code and Static Data

The OS allocates a chunk of RAM for the process's address space. It copies the program's instructions into the code segment, and it copies the initialized global variables into the static data segment.

```
RAM
──────────────────────
code segment   ← instructions copied from disk
static data    ← global variables copied from disk
heap           ← empty for now
stack          ← empty for now
```

### 4.3 Step 3: Eager vs Lazy Loading

Older systems used eager loading: copy everything from disk into RAM before the program starts running at all. This approach is simple, but it makes programs slow to start, and it wastes memory on parts of the program that may never actually be used.

Modern systems use lazy loading instead: only load the parts of the program that are actually needed, when they are needed. If a function is never called, its code never gets loaded into RAM in the first place. Memory is managed in fixed-size chunks called pages, usually 4 KB each. So instead of handling the whole program as one unit, the OS deals with it page by page, loading only a handful of pages at first, just enough to begin execution.

When the CPU tries to access memory that has not been loaded yet, the following sequence happens. The OS is notified through an event called a page fault, and the program is briefly paused. The OS then locates where that piece of code exists on disk, inside the executable file, and loads just that one page into RAM. Only the needed piece is loaded, not the entire program.

### 4.4 Step 4: The OS Creates and Sets Up the Stack

The OS allocates the stack region of memory and initializes it with the program's startup information. Specifically, it pushes `argc`, the count of command line arguments, and `argv`, the actual argument strings, onto the stack, so that `main()` can access them the moment it starts.

```c
int main(int argc, char *argv[]) {
    // argc and argv are already on the stack, placed there by the OS
}
```

### 4.5 Step 5: The OS Allocates an Initial Heap

When your program starts, the operating system sets aside a range of virtual memory for the heap. This region is reserved, but no memory has actually been given to your program from it yet. The program has not asked for anything. It is like having an empty warehouse: the space exists, but nothing is inside it yet. As the program calls `malloc()`, the heap grows, and the OS can expand it further if needed using a system call such as `brk()` or `mmap()`.

At this point, there is a boundary at the top of the Heap region, and this boundary has a specific name: the program break, sometimes called the brk pointer. Everything below the program break is memory that belongs to the Heap region. Everything above it is unmapped; it does not exist as far as the process is concerned. If your program tried to access memory above the program break, the OS would immediately kill it with a segmentation fault.

#### 4.5.1 What malloc() Actually Does

The program break is not something you normally think about as a programmer. It is hidden beneath `malloc()`, but it is the real mechanism underneath. When you call `malloc(100)` in your program, you are not talking directly to the OS. You are talking to a memory allocator, a library that lives inside your program's own process, provided by the C standard library, called libc on Linux. This allocator is a fairly sophisticated piece of software sitting between your code and the OS.

The allocator maintains its own internal data structures: essentially a record of which parts of the Heap region are in use and which are free. It helps to picture this concretely. Suppose the allocator has already obtained some memory from the OS:

```
Total heap it controls: 1000 bytes
```

Inside that block, the situation might look like this:

```
[ used 200 ][ free 300 ][ used 100 ][ free 400 ]
```

The allocator remembers this layout using its own internal data structures, such as lists of blocks, without needing to ask the OS anything every time. When you call `malloc(100)`, the allocator first checks this internal layout and asks: do I already have 100 free bytes available within the memory I already control? Looking at the picture above, the answer is yes, since one of the free blocks is 300 bytes. The allocator carves 100 bytes out of that free block, marks it as used in its internal records, and returns a pointer to you. No system call happens, and there is no interaction with the OS at all. This path is fast.

If the answer had been no, meaning none of the free blocks were large enough, the allocator would have run out of usable space in its current Heap region, and it would need to go to the OS and ask for more memory. This is where `brk()` and `mmap()` come in.

#### 4.5.2 brk(): Expanding the Heap by Moving the Boundary

`brk()` is a system call: a direct request from your process to the OS kernel. Its job is simple and specific: move the program break upward, expanding the Heap region.

When the allocator needs more memory from the OS, it calls `brk()` with a new, higher address. The OS looks at this request, verifies that the process is allowed to have that much memory, updates its internal records about the process's memory layout, and moves the program break to the new address. The memory between the old program break and the new one is now part of the process's Heap region: it is mapped, and it exists. The allocator can now use this new space to satisfy the original `malloc()` request.

A closely related call is `sbrk()`, which works the same way but takes an increment rather than an absolute address. Instead of saying "move the break to address X," you say "move the break forward by N bytes." Internally, both achieve the same thing.

The key characteristic of `brk()` is that it extends the heap continuously and contiguously. The heap is one single, unbroken region of memory that simply gets a larger boundary. This is clean and simple, but it has a limitation: the heap can only grow in one direction, and it must remain one continuous block. This works well for most general-purpose allocation, but there are situations where it is not the right tool.

#### 4.5.3 mmap(): A Completely Different Mechanism

`mmap()` stands for memory map, and it is a much more powerful and general system call than `brk()`. Instead of extending the heap boundary, `mmap()` asks the OS to map a new, independent region of memory somewhere in the process's virtual address space, not necessarily adjacent to the existing heap at all. The OS finds a suitable gap in the virtual address space, maps a fresh region of memory there, and returns a pointer to the start of it. This region is completely separate from the heap region managed by `brk()`.

The allocator typically uses `mmap()` in two situations. First, when you request a very large block of memory, typically larger than a threshold such as 128 KB, `malloc()` bypasses the heap entirely and uses `mmap()` to create a dedicated region just for that one allocation. When you later call `free()` on it, the allocator calls `munmap()` to release that region back to the OS immediately. This is efficient for large allocations, because the dedicated region can be cleanly returned without leaving fragmentation behind in the main heap.

Second, `mmap()` is also used to map files directly into memory, to share memory between processes, and for other more advanced operations that have nothing to do with `malloc()`. Those are separate use cases outside the scope of this post.

### 4.6 Step 6: The OS Sets Up File Descriptors

Before the program even runs, the OS opens three I/O channels for the process automatically:

```
fd 0 = stdin   ← connected to keyboard by default
fd 1 = stdout  ← connected to screen by default
fd 2 = stderr  ← connected to screen by default
```

These are available immediately when `main()` starts. The program can use `printf()` without ever opening anything, because `printf()` writes to fd 1, which is already open by the time your code runs.

### 4.7 Step 7: The OS Jumps to main()

The OS sets the Program Counter to the address of `main()`, and the process begins executing. From this moment, the process is alive and running.

```
PC ← address of main()
CPU starts fetch-decode-execute cycle
Process is now running
```

## 5. File Descriptors

When a process wants to interact with any I/O resource, whether a file, a network connection, a keyboard, or a screen, it must go through the OS. The OS does the actual work and hands back a small integer called a file descriptor. That integer becomes the process's handle to that resource.

Think of it like a coat check at a restaurant. You hand your coat to the attendant, which is the equivalent of asking the OS to open a resource. They give you a ticket with a number on it, which is the file descriptor. Whenever you want your coat back, you show the ticket number. You never deal with the coat storage system directly; the attendant does that on your behalf.

```
Process asks: "open this file"
OS finds the file, sets up internal tracking
OS returns: 3   ← this is the file descriptor
Process uses 3 to read and write to the file
```

### 5.1 The Three Automatic File Descriptors

Every process gets three file descriptors automatically the moment it is created, before it runs a single instruction:

```
fd 0 = stdin  (standard input)
       Default: connected to keyboard
       Used for: reading user input
       Example: scanf() reads from fd 0

fd 1 = stdout (standard output)
       Default: connected to terminal screen
       Used for: normal program output
       Example: printf() writes to fd 1

fd 2 = stderr (standard error)
       Default: connected to terminal screen
       Used for: error messages
       Example: fprintf(stderr, "error!") writes to fd 2
```

This is why `printf("hello")` works without you ever opening anything yourself. Under the hood, `printf` writes to fd 1, which the OS already set up and connected to your terminal before `main()` ever ran.

### 5.2 What Happens When You Open More Files

When your program calls `open()` or `fopen()`, the OS finds the file on disk, sets up an internal structure that tracks the file's position and permissions, and picks the next available integer starting from 3, since 0, 1, and 2 are already taken. It then returns that integer to your program.

```c
int fd = open("data.txt", O_RDONLY);
// fd is now 3 (or 4, 5, etc., whatever is next available)

read(fd, buffer, 100);   // read 100 bytes using the descriptor
close(fd);                // close it when done, fd 3 is now available again
```

### 5.3 File Descriptors Are Per Process

Each process has its own separate file descriptor table. If Process A has fd 3 pointing to `data.txt`, and Process B also has fd 3, these are completely different things; they point to different resources entirely. The number 3 only has meaning inside the specific process that owns it.

### 5.4 Why the OS Tracks This in the PCB

The PCB stores `ofile[]`, an array of all open file descriptors for that process. When a process exits normally or is killed, the OS looks at `ofile[]` and closes every file descriptor that is still open. Without this cleanup step, files would stay locked, network connections would stay open, and resources would leak forever.

## 6. CPU Virtualization and Time Sharing

### 6.1 The Core Problem

Your computer has maybe 4 or 8 CPU cores, yet it runs hundreds of processes simultaneously: your browser, your music player, system services, background updaters, and more. How is that possible?

The CPU is a fundamentally sequential machine. Each core executes exactly one instruction at a time, and it cannot actually run 200 things at once. Something has to give, and the answer is an illusion. The OS creates the appearance of many CPUs by rapidly switching between processes, giving each one a tiny slice of time on the real CPU, one at a time, switching so fast that it appears simultaneous to us.

This technique is called CPU virtualization.

### 6.2 Time Sharing

Time sharing means giving one process the CPU for a little while, then taking it away and handing it to another process, then another, cycling through them all rapidly.

```
Real CPU (one core)
──────────────────────────────────────────────────────────►  time
│ P1 │ P2 │ P3 │ P1 │ P2 │ P3 │ P1 │ P2 │ P3 │ P1 │ ...

What each process experiences:
P1 thinks: ───────────────────────────►  (I have the CPU always)
P2 thinks: ───────────────────────────►  (I have the CPU always)
P3 thinks: ───────────────────────────►  (I have the CPU always)
```

Each process believes it has the CPU entirely to itself. In reality, each one gets a tiny slice, maybe 10 milliseconds, and is then paused while another process runs. The switching happens thousands of times per second, and humans cannot perceive individual 10-millisecond slices, so the whole system feels smooth and simultaneous.

### 6.3 Space Sharing

Space sharing is the counterpart to time sharing. Instead of dividing a resource by time, you divide it by space: each user gets their own permanent piece of it.

Disk storage is space-shared. When a file owns a block of disk, that block belongs to it, and no other file gets that block until the original file is deleted. The blocks are divided in space, not in time.

```
TIME SHARING (CPU)           SPACE SHARING (Disk)
────────────────────         ──────────────────────
P1 gets CPU now               File A owns blocks 1-10
P2 gets CPU next              File B owns blocks 11-20
P3 gets CPU after             File C owns blocks 21-30
P1 gets CPU again             (these blocks never switch)
(same CPU, rotating)          (each file keeps its own blocks)
```

RAM is partly space-shared as well: each process gets its own region of memory that belongs only to it.

### 6.4 The Cost of Time Sharing

Time sharing is not free. Each switch between processes takes time, since the OS must save the current process's state, load another process's state, and resume it. These switches add overhead.

Each individual process also runs slower than it would if it had the entire CPU to itself, because it keeps getting paused and having to wait for its next turn. The tradeoff is worth it regardless: being able to run many programs simultaneously is vastly more useful than running only one program at full speed.

## 7. Mechanisms vs Policies

### 7.1 The Core Idea

The OS needs to make two completely different kinds of decisions when managing processes. The first kind asks how the OS actually stops one process and starts another: what are the exact low-level steps? This is a mechanism. The second kind asks which process should run next: should it be the one that has been waiting longest, the one with the highest priority, or the one that is most interactive? This is a policy.

These two things are deliberately kept separate in OS design.

### 7.2 Mechanism: The How

A mechanism is the low-level machinery that implements a capability. It describes the exact sequence of steps needed to perform an operation.

The context switch is a mechanism. It describes precisely:

```
1. Save current process's PC into its PCB
2. Save current process's RSP into its PCB
3. Save current process's RBP into its PCB
4. Save all other registers into its PCB
5. Mark current process as Ready
6. Pick next process to run (this is policy, not mechanism)
7. Load next process's registers from its PCB into CPU
8. Jump to the loaded PC
9. Next process is now Running
```

Steps 1 through 5 and 7 through 9 are the mechanism. They never change. The OS always saves and restores registers this same way, regardless of which process it is switching to or from.

### 7.3 Policy: The Which

A policy is the decision-making algorithm that sits on top of the mechanism. It decides which process gets to run next, without caring about the low-level details of how that switch actually happens.

The scheduling policy is one example. Different policies answer the same question differently:

```
FIFO policy:      run whichever process has been waiting longest
Priority policy:  run whichever process has the highest priority number
Round Robin:      give each process equal time slices in rotation
Shortest Job:     run whichever process will finish soonest
```

Each of these policies produces a different answer to "which process runs next," but they all use the exact same context switch mechanism to actually perform the switch.

### 7.4 Why Separating Them Matters

If mechanisms and policies were mixed together, changing one would require rewriting the other, which is poor engineering. By keeping them separate, changing which process runs next only requires changing the scheduling policy, while the context switch mechanism stays untouched. Likewise, optimizing how context switches happen only requires changing the mechanism, and every scheduling policy built on top of it keeps working unchanged.

This separation is called modularity, a fundamental software engineering principle: keep components independent so that you can change one without breaking the others.

Linux is a good real-world example. It has used several different scheduling policies over the years, including the O(1) scheduler and the Completely Fair Scheduler (CFS), while the context switch mechanism itself stayed essentially the same throughout. Only the policy changed.

## Conclusion

You now have the foundation. You understand what a process actually is: not just a running program, but a living entity with its own address space, its own memory regions, and its own slice of CPU time. You understand how the OS loads a program from disk and transforms it into a process, step by step. You understand how time sharing creates the illusion that one CPU is running many programs at once. And you understand the design principle that makes the OS elegant: keeping the low-level machinery of context switching completely separate from the high-level decision of which process runs next.

But one question is still sitting unanswered. The OS is juggling hundreds of processes at the same time, and to do that correctly, it needs a complete record of each one: where its memory is, what state it is in, which files it has open, and what its registers were the last time it was paused. Without this record, resuming a paused process correctly would be impossible.

That record is called the Process Control Block. It is the OS's complete dossier on every process, updated every time a process changes state, saved every time a process is paused, and restored every time a process is resumed. Understanding every field inside the PCB is what turns a vague sense of what a process is into a precise, mechanical understanding of it.

**Next:** [Part 2: How the OS Never Forgets, Inside the Process Control Block →](./05-process-control-block.md)
