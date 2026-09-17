# The Illusion of Private Memory

> Module 2 · Post 8 of 13

## 1. The Question Behind the Address Space

In an earlier post we learned that a process has an address space: four regions of memory, code, static data, heap, and stack. We said each process gets its own chunk of RAM. But we never asked the deeper question: how does each process get its own private memory when all processes share the same physical RAM?

There is only one RAM in your computer, yet Process A, Process B, and Process C all think they have their own private memory starting from address 0. How can three processes all think they start at address 0 when there is only one address 0 in physical RAM?

That is the question this post answers. The answer is virtual memory. Let's build everything up from absolute zero, one concept at a time, before we get there.

## 2. Physical RAM: What It Actually Is

Your computer has physical RAM chips soldered onto the motherboard. These chips are arrays of millions or billions of tiny circuits called memory cells. In the most common kind of RAM, DRAM, each cell is made of one capacitor, which stores a charge and is the actual data, and one transistor, which acts like a switch controlling access to it. Each tiny capacitor holds a charge or no charge, so each cell stores exactly one bit.

A capacitor is a tiny battery that cannot hold a charge forever the way a perfect battery would; the stored charge slowly disappears over time because of tiny imperfections in the material. Left alone, data stored in RAM would fade away automatically, and your program's memory would turn to garbage after a short time. To prevent this, the system rewrites the same value again and again, a process called refreshing, and it happens automatically, driven entirely by the memory controller hardware.

### 2.1 How a Bit Becomes a Byte

A single bit is not useful for storing real data like the number 65 or the letter A, so computers group 8 bits together, giving 256 possible values, from 0 to 255. That range of 256 is enough to represent every number from 0 to 255, every printable character (A through Z, a through z, 0 through 9, and the common symbols), and the basic control characters besides. Early computer designers settled on 8 bits as the natural, useful grouping, and they called it a byte. The decision stuck and became the universal standard.

### 2.2 Physical Addresses

Physical RAM has a fixed number of locations, and each location has a unique number called its physical address. A machine with 8 GB of RAM has exactly 8,589,934,592 byte-sized locations, addressed from 0 to 8,589,934,591:

```
8 GB = 8 × 1024 × 1024 × 1024
     = 8,589,934,592 bytes
```

A simplified view of physical RAM looks like this:

```
Physical Address    Contents
────────────────    ────────
0x00000000          byte of data
0x00000001          byte of data
0x00000002          byte of data
...
0x1FFFFFFF          byte of data  (last byte of 512MB RAM)
```

Organized down to the level of individual bits, a few bytes of physical RAM might look like this, where each address points to a group of 8 capacitors, or bits, that are always read and written together as one unit:

| Address | 8 capacitors (bits) | Value stored |
|---|---|---|
| 0x0000 | 0 1 0 0 0 0 0 1 | 65 (letter A) |
| 0x0001 | 0 1 0 0 0 0 1 0 | 66 (letter B) |
| 0x0002 | 0 1 0 0 0 0 1 1 | 67 (letter C) |
| 0x0003 | 0 1 0 0 0 1 0 0 | 68 (letter D) |

This is real, physical silicon; there is nothing virtual about it. When the CPU reads physical address `0x00320000`, it sends electrical signals to the RAM chips, which return the byte stored there.

### 2.3 What Lives in Physical RAM

At any given moment, physical RAM holds a mix of everything currently active on the machine:

```
Physical RAM contents:
┌──────────────────────────────────────┐ 0x00000000
│  OS kernel code and data              │
│  (the OS itself lives here)           │
├──────────────────────────────────────┤
│  Process A's memory                   │
│  (its code, heap, stack)              │
├──────────────────────────────────────┤
│  Process B's memory                   │
├──────────────────────────────────────┤
│  Process C's memory                   │
├──────────────────────────────────────┤
│  Page tables (translation data)       │
│  (explained later in this post)       │
├──────────────────────────────────────┤
│  Free space (unused)                  │
└──────────────────────────────────────┘ 0x1FFFFFFF
```

Everything, the OS, every running process, all translation data, shares this one physical RAM. There is only one, and all processes compete for it.

### 2.4 VRAM: A Common Confusion

VRAM stands for Video RAM, and despite the similar name, it has nothing to do with virtual memory. VRAM is a separate pool of memory physically located on your graphics card, the GPU, dedicated to storing the frame buffer, which holds the pixels currently on screen; textures, the images used in 3D rendering; shader programs, the GPU's own rendering code; and 3D geometry data, meaning vertices, meshes, and models.

VRAM is fast memory optimized for the GPU's parallel access patterns. When you play a game and the GPU renders a frame, it reads textures and geometry from VRAM and writes the finished frame to the frame buffer, also in VRAM, which then gets sent to your monitor.

## 3. Virtual Memory: The Core Illusion

### 3.1 The Fundamental Problem

You have three processes running simultaneously, and each one needs memory. But physical RAM is finite and shared, and three processes cannot all have the address range 0 to 16 KB in physical RAM; they would overlap and corrupt each other. If Process A is sitting at address 320 KB and Process B is sitting right next to it at 192 KB, there is nothing stopping Process A from reading or writing into Process B's memory, or into the OS's memory, or anywhere else in RAM. This is catastrophic: one buggy program can corrupt another program's data, one malicious program can read another program's passwords, and a crashing program can take down the entire OS.

### 3.2 The Lie Every Process Is Told

To isolate processes from each other, the solution is to lie to each one, consistently and under strict control. Each process is told: you have your own private memory starting from address 0, you can use address 0 through however much you need, and nobody else can touch your memory.

This lie is called a virtual address space. The addresses a process uses are virtual addresses; they do not correspond directly to physical locations in RAM. The OS and the hardware translate them silently, on every single access.

### 3.3 Virtual vs Physical Addresses

```
What Process A sees (virtual):        What exists in physical RAM:
──────────────────────────────        ──────────────────────────────
Address 0x0000 = my code              Physical 0x5000 = A's code
Address 0x1000 = my heap              Physical 0x6000 = A's heap
Address 0x3000 = my stack             Physical 0x8000 = A's stack

What Process B sees (virtual):
──────────────────────────────
Address 0x0000 = my code              Physical 0xA000 = B's code
Address 0x1000 = my heap              Physical 0xB000 = B's heap
Address 0x3000 = my stack             Physical 0xD000 = B's stack
```

Both A and B use virtual address `0x0000`, but they map to completely different physical locations. The OS ensures their translations never overlap, so neither process can reach the other's physical memory.

Three things are needed to make this work: a way to place each process somewhere in physical RAM, a way to translate virtual addresses into physical addresses, and a way to stop processes from accessing each other's memory. Base and bounds is the simplest possible solution to all three at once.

A virtual address is just a binary number, and in a real system its size is fixed: on a 32-bit system, every virtual address is exactly 32 bits, and on a 64-bit system, every virtual address is exactly 64 bits. You never drop bits. For the examples in this post, we will use a simplified 14-bit address system, where every address is a 14-digit binary number:

```
Position: 13 12 11 10 9 8 7 6 5 4 3 2 1 0
Value:      0  1  1  0 0 0 0 0 0 0 0 0 0 0
```

## 4. Base and Bounds: The Simplest Translation

### 4.1 The Hardware Registers

The CPU has two special hardware registers dedicated specifically to memory translation, distinct from general-purpose registers like `RAX` or `RBX`:

```
CPU (inside):
┌─────────────────────────────┐
│  RAX  = 42                  │  ← general purpose
│  RBX  = 7                   │  ← general purpose
│  RSP  = 0x1008              │  ← stack pointer
│  RBP  = 0x1000              │  ← frame pointer
│                              │
│  BASE   = 320KB              │  ← where THIS process starts in physical RAM
│  BOUNDS = 16KB                │  ← how large THIS process's space is
└─────────────────────────────┘
```

The OS sets `BASE` and `BOUNDS` every time it switches to a different process. When Process A runs, `BASE = 320KB` and `BOUNDS = 16KB`; when Process B runs, `BASE = 192KB` and `BOUNDS = 32KB`. The hardware uses whatever currently sits in these registers to translate every memory access. As part of loading a process to run, the OS restores its saved registers and loads its `BASE` and `BOUNDS` into these CPU hardware registers.

`BASE` is the starting physical address of the process in RAM, and the OS chooses it based on which physical memory is currently free, how large the process's `BOUNDS` requirement is, and whatever allocation strategy the OS uses, such as first-fit or best-fit. `BOUNDS` is the total size, or end address, of the process's virtual address space; when a program starts, the OS looks at its executable file and works out the sizes of its sections, heap, and stack, and decides on `BOUNDS` based on program size, expected runtime needs, system policies, and available memory.

### 4.2 Translating an Address, Step by Step

Let's say Process A is running with `BASE = 320KB` and `BOUNDS = 16KB`. From A's own perspective, its virtual address space looks like this:

```
Process A thinks it has:
┌──────────────────┐ virtual address 0KB
│ code              │
├──────────────────┤ virtual address 1KB
│ heap              │
│                    │
│ (free space)       │
│                    │
│ stack              │
└──────────────────┘ virtual address 16KB
```

When Process A accesses virtual address `0x1000`, which is 4KB, the hardware does the following. First, it checks the bounds: is 4KB greater than or equal to 16KB? No, so the address is within bounds and translation proceeds. Second, it translates the address: `physical address = BASE + virtual address = 320KB + 4KB = 324KB`. Third, it accesses RAM: it goes to physical address 324KB, reads the data there, and returns it to the process. Process A asked for virtual address 4KB and got data from physical address 324KB, with no idea any translation happened at all; it was completely invisible.

### 4.3 Catching a Violation

Now suppose Process A tries to access virtual address `0x8000`, which is 32KB. The bounds check asks: is 32KB greater than or equal to 16KB? Yes, so this is a violation, an address outside A's space. The hardware raises an exception immediately, the OS takes control, and the OS kills Process A, which sees this as a segmentation fault. Process A tried to reach outside its 16KB boundary, and the hardware caught it before any actual RAM access happened. Process B's memory at 192KB was never touched. Protection worked.

### 4.4 Base and Bounds Across the Process Lifecycle

When a process is created, the OS finds a free slot in physical memory, sets the base and bounds registers, and records these values in the PCB. During a context switch, the OS saves the current process's base and bounds into its PCB, then loads the next process's base and bounds from its own PCB; this step is critical, because if the OS forgot to update these registers, the newly running process would translate its addresses using the previous process's base, reading and writing entirely the wrong memory. When a process terminates, its memory goes back onto the free list so other processes can use it.

### 4.5 The Problem With Base and Bounds

Base and bounds treats the entire process as one solid chunk in physical RAM, and this creates a serious waste problem. Consider what a process actually uses:

```
Process A's virtual address space (16KB total):
┌──────────────────┐ 0KB
│ code (1KB)        │  ← actually used
├──────────────────┤ 1KB
│ heap (2KB)        │  ← actually used
│                    │
│                    │
│  EMPTY SPACE       │  ← 12KB of nothing
│  (12KB)            │
│                    │
│                    │
│ stack (1KB)        │  ← actually used
└──────────────────┘ 16KB
```

Only 4KB is actually used; 12KB is empty space between the heap and the stack. But in physical RAM under base and bounds:

```
Physical RAM:
├──────────────────┤ 320KB
│ A's code (1KB)     │  ← real data
│ A's heap (2KB)     │  ← real data
│                    │
│ EMPTY (12KB)       │  ← WASTED physical RAM
│                    │    this space is allocated but empty
│ A's stack (1KB)    │  ← real data
├──────────────────┤ 336KB
```

The entire 16KB physical chunk is reserved for Process A even though 12KB of it is empty, and that empty space cannot be used by any other process. It is physically allocated but logically empty: wasted. This is called internal fragmentation, waste space inside an allocated region. With 100 processes each wasting 12KB, that is 1.2MB of RAM wasted on nothing, and with larger processes the waste grows enormously.

## 5. Segmentation

### 5.1 The Core Idea

The problem with base and bounds is that it treats the process as one chunk, forcing the empty space between the heap and the stack to be part of that chunk. What if, instead, the process were split into separate pieces, one for code, one for heap, one for stack, and each piece placed independently in physical RAM? That is segmentation.

Each piece is called a segment, and each segment gets its own base and bounds. The OS no longer needs to allocate one big contiguous chunk that includes the empty space between heap and stack; that empty space does not need to exist in physical RAM at all.

### 5.2 The Segment Table

Instead of a single pair of base and bounds registers, the hardware now has a segment table, a small table with one row per segment:

| Segment | Base (physical) | Size |
|---|---|---|
| Code | 320KB | 1KB |
| Heap | 400KB | 2KB |
| Stack | 600KB | 1KB |

Each segment is placed independently somewhere in physical RAM: code is at 320KB, heap is at 400KB, a completely different location, not adjacent to code, and stack is at 600KB, further away still. The table simply says, for each segment, where that segment lives in physical memory. Physical RAM now looks like this:

```
Physical RAM:
├──────────────────┤ 320KB
│ A's code (1KB)    │  ← segment 1
├──────────────────┤ 321KB
│ (something else)  │  ← can be used by other processes
...
├──────────────────┤ 400KB
│ A's heap (2KB)    │  ← segment 2
├──────────────────┤ 402KB
│ (something else)  │
...
├──────────────────┤ 600KB
│ A's stack (1KB)   │  ← segment 3
├──────────────────┤ 601KB
```

The empty space between code, heap, and stack is gone from physical RAM. Only the actual data takes physical space: 3KB used instead of 16KB, a real improvement.

### 5.3 Encoding the Segment Inside the Address

When a process accesses a virtual address, the hardware, the CPU together with the MMU inside it, needs to know two things: which segment this address refers to, code, heap, or stack, and how far into that segment it is. Only then can it translate the virtual address into a physical one, using the settings the OS has set up.

Consider this C code:

```c
int *p = malloc(100);
p[3] = 10;
```

The line `p[3]` becomes `address = p + (3 * sizeof(int))`. Suppose `p = 0x17F4`; then `0x17F4 + 0xC = 0x1800`. So `0x1800` is simply a computed virtual address, meaning "go to byte number `0x1800` in my virtual address space." Given a segment table with `code → 320 KB`, `heap → 400 KB`, and `stack → 600 KB`, the CPU must decide whether `0x1800` falls inside code, heap, or stack. Without knowing which segment, the hardware cannot do the translation at all.

The solution hardware designers settled on is to encode the segment number directly inside the virtual address itself, splitting the address into two parts: which segment, and what offset within it.

```
[ segment | offset ]
[  code   | 0x1800 ]
```

Let's convert `0x1800` to binary. Each hex digit becomes 4 binary bits: `1 = 0001`, `8 = 1000`, `0 = 0000`, `0 = 0000`, giving `0001 1000 0000 0000`, which is 16 bits. Since we are using 14-bit addresses in this example, we drop the leading two zero bits, giving `01 1000 0000 0000`. Splitting this into the top 2 bits and the bottom 12 bits:

```
TOP 2 BITS:      01
BOTTOM 12 BITS:  100000000000
```

The top 2 bits identify the segment: `00` means code, `01` means heap, and `10` means stack. Our top 2 bits are `01`, so the hardware immediately knows this address is inside the heap.

The bottom 12 bits give the offset, how many bytes into the heap we are going. Converting `100000000000` to decimal, with the leftmost bit at position 11, gives `1 × 2^11 = 2048`. So the offset is 2048, meaning we want to go 2048 bytes into the heap.

### 5.4 Translating a Segmented Address, Step by Step

The rule for a bounds check within a segment is that the offset must be strictly less than the segment's size. Here, 2048 must be less than 2048, and it is not, so this particular address triggers a segmentation fault.

Now suppose the address had been `0x1400` instead. Converting to binary: `1 = 0001`, `4 = 0100`, `0 = 0000`, `0 = 0000`, giving `0001 0100 0000 0000`, or as 14 bits, `01 0100 0000 0000`. Splitting gives top bits `01`, the heap segment again, and bottom bits `010000000000`. Converting the bottom bits to decimal: `1 × 2^10 = 1024`, so the offset is 1024 bytes, or 1KB.

The bounds check now asks whether 1024 is less than 2048, and it is, so this address is within bounds and translation proceeds: `physical = heap base + offset = 400KB + 1KB = 401KB`. The hardware accesses physical address 401KB. Done, with no violation.

### 5.5 GrowsPositive: Handling a Segment That Grows Backward

`GrowsPositive` is a single bit, 0 or 1, stored per segment in the segment table, that tells the hardware which direction that segment expands in memory. Code and heap grow toward higher addresses, so `GrowsPositive = 1`; when you allocate more heap memory, the heap pointer moves toward bigger addresses. The stack grows toward lower addresses instead, so `GrowsPositive = 0`; every time a function is called, the stack pointer moves down, toward smaller addresses, which is simply how the stack works physically on x86 architecture.

This matters for translation because the offset calculation is different for a segment that grows backward. For code and heap, the formula is simply `physical = base + offset`. For the stack, it is instead `negative_offset = offset - max_segment_size`, followed by `physical = base + negative_offset`.

As a concrete example, suppose virtual address 15KB maps to the stack segment, with top bits `11`, and the offset within the segment works out to 3KB. If the maximum segment size is 4KB, the negative offset is `3KB - 4KB = -1KB`, so the physical address is `28KB + (-1KB) = 27KB`. Without this bit, the hardware would treat the stack's offset as positive and compute the wrong physical address every single time. That is the entire purpose of `GrowsPositive`: it is a flag that switches the hardware between two different translation formulas.

### 5.6 Protection Bits

The segment table already has a base, where a segment lives in physical memory, and a size, how big it is. Protection bits are simply a fourth column added to that same table, answering a new question: what is this memory allowed to do?

| Segment | Base | Size | GrowsPositive? | Protection |
|---|---|---|---|---|
| Code | 32KB | 2KB | 1 | Read-Execute |
| Heap | 34KB | 2KB | 1 | Read-Write |
| Stack | 28KB | 2KB | 0 | Read-Write |

The code segment is marked Read-Execute: you can read from it and run it as instructions, but you cannot write to it. The heap and stack are marked Read-Write: you can read and modify them, but you cannot execute them as code.

Without protection bits, nothing would stop a buggy or malicious program from doing something like this:

```c
// Your program's code starts at virtual address 0
// The code segment is at physical 32KB
// Without protection, you could do:
char *code = (char *)0;   // point to your own code
code[0] = 0x90;           // overwrite your own instruction with NOP
```

Here the program is writing into its own code segment and changing its own instructions. This can happen by accident, a bug that writes to the wrong address, or intentionally, a security exploit injecting malicious code; either way, it is catastrophic, since the program's behavior becomes unpredictable. Protection bits let the hardware catch this the moment it happens: the hardware checks not just whether an address is within bounds, but whether this particular type of access is allowed on this particular segment. If the answer is no, it raises an exception, and the OS kills the process immediately.

### 5.7 The Problem Segmentation Creates: External Fragmentation

Segmentation solves the internal-waste problem, but it creates a new one: external fragmentation. Each individual segment is contiguous, not broken into pieces, but different segments, even of the same process, can be placed in completely different locations. Holes appear because segments from many processes are mixed together in physical memory, and when a process exits, its code, heap, and stack segments each disappear, leaving a hole where each one used to be. Memory ends up looking like `FREE → USED → FREE → USED → FREE`, rather than one big chunk.

Imagine the holes are all small: 8KB free at address 64KB, 3KB free at 73KB, and 5KB free at 80KB, for 16KB of free memory in total. A new process needs one contiguous 10KB segment. Hole 1, at 8KB, is too small. Hole 2, at 3KB, is too small. Hole 3, at 5KB, is too small. There is 16KB free in total, but no single hole is big enough, so the process cannot be loaded even though enough total free space exists. The free space is fragmented, broken into pieces that are individually too small to use. This is external fragmentation: free space exists but is scattered in unusable pieces outside the allocated regions.

### 5.8 Why Compaction Is Not a Real Solution

The OS could fix fragmentation by compacting memory, moving all allocated segments together to one end and collecting all the free space into one large hole:

```
Before compaction:
│ OS │ free 8KB │ A code │ free 3KB │ B heap │ free 5KB │ C stack │ free │

After compaction:
│ OS │ A code │ B heap │ C stack │           FREE (16KB)           │
```

Now all 16KB is contiguous, and the new 10KB process can fit. But compaction has serious problems. It is extremely slow: moving A's code from address 72KB to address 64KB means copying 1KB of data byte by byte to the new location and updating A's segment table so the code base changes from 72KB to 64KB, and this has to be done for every segment of every process, which takes significant time when there are many processes with large segments. Processes must also be stopped: you cannot move a process's memory while it is running, so the process must be paused during compaction, and every process is unresponsive while this happens. Worst of all, it keeps happening: as soon as compaction finishes, processes start and stop again, fragmentation builds back up, and compaction has to run again, making it a never-ending and expensive maintenance task.

Compaction is a band-aid; it treats the symptom, not the cause. The root cause is that segments have variable sizes. If one segment needs 1KB and another needs 7KB, the holes they leave behind are 1KB and 7KB, awkward sizes that may not match what the next process needs.

### 5.9 Base-and-Bounds vs Segmentation

Base and bounds is simple and fast, one addition and one comparison, but it wastes physical RAM on the empty space between heap and stack, and it requires one large chunk to be contiguous in physical RAM. Segmentation eliminates that internal waste, since each segment is placed independently, but it introduces external fragmentation as holes of different sizes accumulate, compaction is too slow to be practical, and variable segment sizes make placement decisions genuinely complex.

The solution is to stop requiring contiguous memory entirely, which is exactly what paging does, by breaking everything into fixed-size pieces that can be scattered anywhere in physical RAM and reassembled transparently.

## 6. Free Space Management

### 6.1 Why This Belongs Here

This topic often feels like a separate subject, but it is not; it is the direct consequence of the problems segmentation created. When memory uses base and bounds, one block per process, management is simple: RAM is split into equal-sized slots, a process needing memory gets a slot, and a finished process gives it back. There is no fragmentation between slots because all slots are the same size, and the free list just tracks which slots are free.

Once we switch to segmentation, each process has three segments of different sizes that can go anywhere in RAM, and over time memory ends up looking like this:

```
[OS][free 8KB][Process A code 2KB][free 5KB][Process B heap 4KB][free 11KB][Process C stack 3KB]
```

Now there are many free holes of different sizes spread across memory. Suppose a new process needs 10KB; there is 24KB free in total, but no single block reaches 10KB, so the OS cannot allocate the memory even though enough total memory exists. This is exactly the problem free space management solves: the OS must figure out how to use these irregular holes effectively.

### 6.2 Two Levels Where It Operates

Free space management actually operates at two different levels, using the same underlying ideas. At the OS level, it exists because of external fragmentation: the OS must track which regions of physical RAM are free, choosing where to allocate with a strategy such as first-fit or best-fit, splitting blocks when needed, and merging, or coalescing, blocks to reduce fragmentation; when a segment is freed because a process exits, the OS puts that region back on the free list.

At the process level, inside your own heap, the OS has already given your process one segment for the whole heap, but within that segment, your program keeps allocating and freeing individual objects through `malloc()` and `free()`. The heap has its own mini free list tracking which parts of it are available, and `malloc` uses these same kinds of algorithms internally to decide which part of the heap to hand you. Free space management is essentially the same problem at two different scales: the OS managing physical RAM, and `malloc` managing the heap inside a single process.

### 6.3 The Free List

The free list is a linked list where each node represents one contiguous free chunk of memory, storing the starting address of that chunk and its length. The OS, or `malloc`, walks this list whenever it needs to allocate memory. For example, after some allocations in a 30-byte heap, the free list might look like this:

```
[addr:0, len:10] → [addr:20, len:10] → NULL
```

The 10 bytes at addresses 10 through 19 are in use, allocated, while bytes 0 through 9 and 20 through 29 are free. The allocator walks this list to find space when a request comes in.

### 6.4 Splitting

A request comes in for 1 byte. The allocator finds a free chunk of 10 bytes, which is bigger than needed, so it splits the chunk: it gives byte 20 (1 byte) to the requester, and keeps bytes 21 through 29 (9 bytes) on the free list. After splitting:

```
[addr:0, len:10] → [addr:21, len:9] → NULL
```

Splitting happens automatically whenever a request is smaller than the chunk that satisfies it.

### 6.5 Coalescing

Now suppose that 1-byte allocation is freed. The allocator simply adds it back onto the list:

```
[addr:20, len:1] → [addr:0, len:10] → [addr:21, len:9] → NULL
```

This is a mess: three separate entries, even though bytes 20 through 29 are all free and adjacent to each other. Without coalescing, a request for 5 bytes would fail to find the 9 contiguous bytes at 21 through 29, since the list only records individual entries of 1 and 9 bytes.

Coalescing means that when a chunk is freed, the allocator looks at the chunks immediately before and after it in memory, and if either of them is also free, it merges them all into one bigger chunk. After coalescing `addr:20 (len:1)` with `addr:21 (len:9)`:

```
[addr:0, len:10] → [addr:20, len:10] → NULL
```

And if these two entries are also adjacent, they merge as well:

```
[addr:0, len:30] → NULL
```

The full heap is free again, in one single piece.

### 6.6 How malloc Headers Work

`free(ptr)` takes only a pointer; it does not take a size. But the allocator needs to know the size of the block in order to put it back on the free list correctly. It knows this because `malloc()` does not give you back exactly the number of bytes you asked for; when you call `malloc(20)`, it secretly allocates a small header just before your 20 bytes and stores the size there:

```
Memory layout after malloc(20):
[header: size=20, magic=1234567][your 20 bytes]
↑                                ↑
ptr - sizeof(header)             ptr  ← this is what malloc returns to you
```

When you call `free(ptr)`, the allocator steps backward from your pointer to find that header:

```c
header_t *hptr = ptr - sizeof(header_t);   // step back to find the header
int size = hptr->size;                      // read the size
// now I know this block is 28 bytes total (8 header + 20 data)
// put it back on the free list
```

The magic number stored in the header acts as a sanity check: if it has been corrupted, for example because you wrote past the end of your own buffer, the allocator can detect that and catch the bug.

### 6.7 The Four Allocation Strategies

Even if you split blocks perfectly and merge them perfectly, you can still create fragmentation depending on where you choose to allocate from, which is exactly the question these strategies answer: given a free list with multiple holes, which hole should this request use?

Best fit searches for the smallest hole that is still big enough, minimizing leftover waste after the allocation, but it requires searching the entire list every single time. Worst fit does the opposite, finding the largest hole and using part of it, on the theory that the leftover chunk will also be large and useful for future requests; in practice this performs badly, since it fragments memory into medium-sized chunks that fit neither small nor large requests well. First fit simply uses the first hole that is big enough and stops searching, which is fast, but the beginning of the free list tends to accumulate many small leftover chunks over time, since small allocations keep picking holes near the start of the list. Next fit works like first fit but remembers where it left off last time and continues from there on the next request, spreading allocations more evenly across the list so the beginning does not get overloaded with tiny fragments.

### 6.8 Buddy Allocation

Buddy allocation is a full memory management scheme in its own right, not just a placement strategy. The challenge with coalescing in a regular free list is knowing which adjacent chunks are free, which normally requires searching or maintaining extra pointers, making coalescing slow and complicated. Buddy allocation sidesteps this by enforcing one strict rule: every memory block must be a power-of-2 size, such as 64KB, 32KB, 16KB, or 8KB.

Suppose you have 64KB of total memory. Whenever you need smaller pieces, you split a block into two equal halves, called buddies: `64KB → 32KB + 32KB`, then `32KB → 16KB + 16KB`, then `16KB → 8KB + 8KB`, and so on. You never split unevenly. If you need 7KB, but the rule only allows power-of-2 sizes, the system keeps splitting step by step until it produces an 8KB block, wasting 1KB as internal fragmentation.

When you later free that 8KB block, instead of searching all of memory, the allocator simply checks its buddy. Every block has exactly one buddy, and the buddy's address differs from the block's own address by exactly one bit, so finding it is instant. If the buddy is also free, the two merge back into a 16KB block; the allocator then checks whether that 16KB block's buddy is free too, and if so, merges again into 32KB, and so on, potentially all the way back up to the full 64KB if everything nearby happens to be free.

## 7. Paging: The Modern Solution

### 7.1 Fixed-Size Pages and Frames

Instead of dividing memory into variable-sized segments, paging divides both virtual memory and physical memory into fixed-size chunks. The standard size on most systems is 4KB, or 4096 bytes.

```
Virtual address space divided into pages:
┌──────────────────┐ virtual 0KB
│ virtual page 0    │ 4KB
├──────────────────┤ virtual 4KB
│ virtual page 1    │ 4KB
├──────────────────┤ virtual 8KB
│ virtual page 2    │ 4KB
├──────────────────┤ virtual 12KB
│ virtual page 3    │ 4KB
└──────────────────┘ virtual 16KB

Physical RAM divided into page frames:
┌──────────────────┐ physical 0KB
│ frame 0           │ 4KB
├──────────────────┤ physical 4KB
│ frame 1           │ 4KB
├──────────────────┤ physical 8KB
│ frame 2           │ 4KB
│ ...                │
└──────────────────┘
```

Each virtual page maps to exactly one physical frame. The OS keeps track of which virtual page maps to which physical frame using a data structure called the page table. It helps to see how this compares with what came before:

| System | Virtual memory | Physical RAM |
|---|---|---|
| Segmentation | Contiguous (per segment) | Contiguous (per segment) |
| Paging | Contiguous | Shattered / scattered |

Even parts of a single segment can end up scattered across physical RAM once paging is involved, but they always remain clean and contiguous from the process's own point of view, in virtual memory.

### 7.2 Why Fixed Size Eliminates External Fragmentation

With pages, every free chunk of physical RAM is exactly one page in size. Any free frame can hold any virtual page from any process, so there are no awkward holes of mismatched sizes:

```
Physical RAM frames:
Frame 0: OS
Frame 1: Process A page 0 (code)
Frame 2: FREE
Frame 3: Process B page 0 (code)
Frame 4: FREE
Frame 5: Process A page 1 (heap)
Frame 6: FREE
Frame 7: Process C page 0 (code)
```

Any of the free frames, 2, 4, or 6, can immediately hold the next page any process needs. No compaction is required, and there is no fragmentation problem in the sense segmentation had one.

### 7.3 A Process Spans Multiple Pages

When a label like "Process A page 0 (code)" appears above, it does not mean the entire code section fits in one page, or that the entire heap fits in one page; it means each page contains one piece of the code or the heap. A process's code, heap, and stack are almost never small enough to fit into a single page; they span multiple pages.

Suppose a process has 3KB of code, 6KB of heap, and 2KB of stack, for 11KB total, with a 4KB page size. The OS divides everything into 4KB chunks. The code, at 3KB, fits entirely in one page, with 1KB of that page left unused, which is internal fragmentation. The heap, at 6KB, needs `6KB ÷ 4KB = 1.5` pages, rounded up to 2: the first page holds the first 4KB of heap, and the second page holds the remaining 2KB, with 2KB of that second page unused. The stack, at 2KB, fits in one page, with 2KB unused. In total, this process needs `1 + 2 + 1 = 4` pages.

These 4 pages can be placed in any 4 free frames anywhere in physical RAM:

```
Physical RAM frames:
Frame 0: OS
Frame 1: Process A code page      ← code lives here
Frame 2: FREE
Frame 3: Process A heap page 1    ← first half of heap
Frame 4: Process B something
Frame 5: Process A heap page 2    ← second half of heap
Frame 6: FREE
Frame 7: Process A stack page     ← stack lives here
```

Frames 1, 3, 5, and 7 are scattered all over physical RAM, but the process sees them as one continuous virtual address space; the page table stitches them together. As the heap grows dynamically through calls to `malloc()`, every time it needs another 4KB, the OS finds any free frame in physical RAM, adds a new entry to the page table mapping the new virtual page to that physical frame, and returns the new memory to the process. The new frame does not have to be adjacent to the previous heap frame; it can be anywhere at all, and the page table tracks where everything is.

### 7.4 Why Free Space Management Still Matters Inside the Heap

It might seem that since paging uses fixed-size pages, free space management no longer applies. That is mostly true at the OS level: the OS just keeps a free list of page frames, and since every frame is the same size, any free frame works for any request, so no complex placement algorithm is needed there.

But free space management absolutely still applies inside the heap of your own process. Even in a paging system, `malloc` and `free` are managing the heap segment with variable-sized allocations of their own. Every time you call `malloc(13)` or `malloc(200)` inside your C program, the heap allocator is doing splitting and coalescing using one of the strategies described earlier. The heap is backed by whole pages, which the OS hands over as a unit, but within those pages, `malloc` manages individual bytes, and that is exactly where variable-size free space management continues to live.

### 7.5 Internal Fragmentation, Paging's New Problem

When a segment does not perfectly fill its last page, the leftover space in that page is wasted; this is internal fragmentation, waste inside a page. For example, 3KB of code with a 4KB page size needs one page, and the diagram looks like this:

```
┌────────────────┐
│ code (3KB)      │  ← used
│                  │
│ wasted (1KB)     │  ← empty, cannot be used by anyone else
└────────────────┘
1 page (4KB)
```

That 1KB inside the page is permanently wasted for as long as this process is alive; no other process can use it.

### 7.6 Paging vs Segmentation: The Tradeoff

Segmentation's waste is external fragmentation: holes between allocated segments, of any size, that get worse over time and are hard to fix, since compaction is slow. Paging's waste is internal fragmentation: wasted space inside the last page of each segment, with a maximum waste per segment of one byte less than a full page, 4095 bytes with 4KB pages, which is predictable and bounded.

Internal fragmentation from paging is considered far more manageable than external fragmentation from segmentation, for a concrete reason. In the worst case, internal fragmentation wastes almost one full page per segment, so a process with 3 segments wastes at most roughly `3 × (4KB - 1) ≈ 12KB`, a bounded and predictable amount. External fragmentation, on the other hand, has no such bound in the worst case: the entire RAM could be free yet split into pieces too small to use for anything, and this only gets worse over time as more processes come and go.

### 7.7 Why Smaller Pages Reduce Internal Fragmentation

If the page size is 4KB, the maximum waste per segment is 4095 bytes, roughly 4KB. If the page size were instead 256 bytes, the maximum waste per segment would be only 255 bytes, roughly 256 bytes. Smaller pages mean less internal fragmentation. But smaller pages also mean more pages per process, which means larger page tables, more page table entries to store, more memory spent just tracking pages, and more TLB misses, since the TLB, discussed shortly, can hold fewer useful entries at a time. This is why real systems settle on 4KB pages: it is a balance between internal-fragmentation waste and page-table overhead. Some systems support larger pages, 2MB or even 1GB, called huge pages, for specific workloads where that tradeoff favors them.

## 8. Page Tables: The Translation Map

### 8.1 What a Page Table Is

A page table is a data structure stored in RAM that maps virtual page numbers to physical frame numbers, and every process has its own. Because it is a structure the hardware has to read on every single memory access, it needs to live somewhere the hardware can reach instantly, and RAM is that place.

| Virtual Page | Physical Frame | Valid? |
|---|---|---|
| page 0 | frame 1 | YES (code is here) |
| page 1 | frame 5 | YES (heap is here) |
| page 2 | frame 9 | YES (stack is here) |
| page 3 | (none) | NO (not in RAM) |

The Valid bit tells the hardware whether this virtual page is currently loaded into physical RAM. If it is not valid and the process tries to access it, a page fault occurs, which is explained in section 10.

### 8.2 Splitting a Virtual Address Into Page Number and Offset

When your program runs, it never directly uses real physical memory addresses; instead it uses virtual addresses, the illusion the OS creates. So when your program effectively says "load the value from address `0x00001800`," that `0x00001800` is a virtual address, not a real RAM address. The question the hardware must answer is: which page is this address in, and how far into that page?

Since each page is 4KB, or 4096 bytes, and pages are numbered starting from 0, virtual addresses 0 through 4095 fall inside page 0, addresses 4096 through 8191 fall inside page 1, addresses 8192 through 12287 fall inside page 2, and addresses 12288 through 16383 fall inside page 3. Any virtual address automatically tells you both things you need: which page, by dividing the address by the page size, and how far in, by taking the remainder of that division.

Take virtual address `0x00001800` as an example. Converting to decimal: `0x1800 = 1×16^3 + 8×16^2 = 4096 + 2048 = 6144`. Dividing by the page size, `6144 ÷ 4096 = 1` remainder `2048`. So the page number is 1, meaning this address is inside page 1, and the offset is 2048, meaning it is 2048 bytes into that page.

### 8.3 Why Binary Makes This Instant

The hardware does this exact same division, but in binary, which makes it essentially free. Since the page size is `4KB = 4096 = 2^12`, dividing by `2^12` in binary is just a matter of splitting off the top bits; the bottom 12 bits automatically become the remainder, the offset, with no actual division ever taking place. Writing `0x00001800` out as 32-bit binary:

```
0000 0000 0000 0000 0001 1000 0000 0000
─────────────────────────────────────────
Bits 31 down to 12:        Bits 11 down to 0:
0000 0000 0000 0000 0001   1000 0000 0000
= VPN (Virtual Page Number) = Offset within page
= 1                          = 0x800 = 2048
```

The hardware reads the top 20 bits as the page number, giving `VPN = 1`, and the bottom 12 bits as the offset, giving 2048 bytes into the page. No math is needed, only reading bits; this is why 4KB pages and binary addresses work so elegantly together, since `4KB = 2^12` makes the split land exactly on a bit boundary.

### 8.4 The Full Translation, Step by Step

When a process accesses memory, the full sequence runs like this: the program generates a virtual address, such as `0x00001800`; the CPU sends it to the Memory Management Unit, the MMU; the MMU splits the address into a page number and an offset, the distance inside that page, and uses the page number to look up the page table; it finds the corresponding physical address in RAM; and finally, the actual memory is accessed.

Take virtual address `0x00001800` and walk through it. First, split the virtual address:

```
0x00001800 in binary (32 bits):
0000 0000 0000 0000 0001 | 1000 0000 0000
─────────────────────────  ───────────────
VPN = 1                    Offset = 0x800
```

Second, look up the VPN in Process A's page table:

| VPN | Physical Frame | Valid |
|---|---|---|
| 0 | frame 1 | YES (code) |
| 1 | frame 5 | YES (heap) ← we are looking up this one |
| 2 | frame 3 | YES (heap) |
| 3 | frame 7 | YES (stack) |

`VPN 1 → Frame 5, Valid = YES`.

Third, compute the physical address:

```
Physical address = (frame number × page size) + offset
                 = (5 × 4096) + 2048
                 = 20480 + 2048
                 = 22528
                 = 0x5800
```

Fourth, access physical RAM at `0x5800`: frame 5 starts at `20KB = 20480 bytes` and ends at `24KB = 24576 bytes`, and we access the byte at `20480 + 2048 = 22528 = 0x5800`. This checks out.

It is worth noticing that the offset, `0x800`, or 2048, is identical on both the virtual and physical sides: virtual page 1, offset `0x800`, is byte 2048 within page 1, and physical frame 5, offset `0x800`, is byte 2048 within frame 5. Only the page or frame number changes; the offset stays the same, which makes sense, since the offset simply describes where within a 4KB chunk you are, and that position does not change just because the chunk itself moved to a different physical location.

### 8.5 How Big Is One Page Table?

Consider a 32-bit virtual address space with a 4KB, or `2^12`-byte, page size. A 32-bit address gives `2^32` unique addresses, and since each address in most systems points to one byte, that is `2^32` bytes, or 4 GB, of addressable virtual memory. Dividing that by the page size gives the number of virtual pages: `2^32 / 2^12 = 2^20`, which is 1,048,576 virtual pages on a 32-bit processor.

Each of those virtual pages needs one entry in the page table. Each page table entry stores the physical frame number, needing 20 bits to identify where the page sits in RAM, a valid bit indicating whether the page is actually in memory, permission bits for read, write, and execute, needing 3 bits, and a handful of other flags such as dirty and accessed bits; rounded up, this totals about 4 bytes per entry.

The full page table for one process therefore comes to `1,048,576 entries × 4 bytes per entry = 4,194,304 bytes = 4MB`. One process needs a 4MB page table. With 100 processes, that is `100 × 4MB = 400MB` of physical RAM spent just on storing page tables, a massive waste, especially since most of those entries describe pages the process never actually uses.

### 8.6 Segmentation vs Paging, One More Time

It is worth making the contiguity distinction completely explicit before going further, since segmentation and paging are easy to blur together. In base and bounds, a process gets exactly one continuous chunk of physical memory, and translation is simply `physical = base + offset`; that single chunk must be contiguous in RAM. Segmentation breaks that one chunk into several independent pieces, code, heap, and stack, each with its own base and its own bounds; each individual segment must still be contiguous in physical RAM, but the segments do not need to sit next to each other, and a virtual address in segmentation is really a pair, `(segment number, offset)`, rather than one flat range. Paging goes further still: the virtual address space is one single, continuous range, so code and heap do appear adjacent from the process's own point of view, but that continuous range is chopped into fixed-size pages, and physically, those pages can be scattered anywhere at all in RAM.

### 8.7 The Core Problem: Most Entries Are Empty

Consider what a typical process actually uses out of its full 4GB address space on a 32-bit system:

```
Process virtual address space (4GB total):
┌──────────────────────────────┐ 0GB
│ Code (maybe 1MB)               │ ← actually used
├──────────────────────────────┤
│                                │
│                                │
│   ENORMOUS EMPTY GAP           │ ← 3.99GB of nothing
│   (never used)                 │
│                                │
│                                │
├──────────────────────────────┤
│ Stack (maybe 8MB)              │ ← actually used
└──────────────────────────────┘ 4GB
```

The process is allowed to use any address in this 4GB range if it ever needs to, but a real process might only use around 10MB out of 4GB, and yet the page table still has entries for the full 4GB range, meaning over 99% of those entries are empty and simply wasting space. Every process on a 32-bit system faces this same 4GB range of possible addresses; it is a bit like every house on a street being allowed a door number from 0 to 4 billion, when most houses will only ever use a handful of them.

### 8.8 The Solution: Multi-Level Page Tables

Instead of one giant flat table with an entry for every possible page, the solution is a tree structure, allocating table memory only for pages that actually exist. In short, group pages into chunks, and only track the chunks that are actually used.

A flat page table is one huge table with a million entries, always sitting fully in RAM. A multi-level page table instead has a small top-level table that points to smaller sub-tables, which are only created when they are actually needed.

Concretely, split the 4GB virtual space into 1024 large regions of 4MB each (`4GB ÷ 1024 = 4MB` per region): region 0 covers `0` to `4MB`, region 1 covers `4MB` to `8MB`, region 2 covers `8MB` to `12MB`, and so on. The Level 1 table has 1024 entries, and each one says either "this region is used, here is a pointer to its Level 2 table" or "this region is empty, NULL."

```
Region 0    → Code  → used   → create L2
Region 1    → Heap  → used   → create L2
Region 2    → Gap   → unused → NULL
Region 3    → Gap   → unused → NULL
...
Region 1023 → Stack → used   → create L2
```

For the unused regions, nothing is created at all; the entry is simply `NULL`. A Level 2 table is like a mini page table for just one region, holding entries only for the pages that exist inside it, and it is only created if that region is actually used. This two-level structure does not remove the gap itself; it simply avoids wasting memory describing that gap in fine-grained detail.

## 9. The TLB: Making Translation Fast

### 9.1 The Performance Problem

Every time the CPU accesses memory under paging, the naive sequence is: the CPU gets a virtual address, looks up the page table in RAM, which is one memory access, gets the physical frame, and then accesses the actual data in RAM, a second memory access. RAM is slow compared to the CPU, so this doubles the cost of every memory access, which would completely defeat the purpose of virtual memory.

### 9.2 The Translation Lookaside Buffer

The TLB, the Translation Lookaside Buffer, is a tiny, extremely fast cache built directly into the CPU chip that stores recent virtual-to-physical translations:

| VPN | Physical Frame | Valid |
|---|---|---|
| page 0 | frame 1 | YES |
| page 1 | frame 5 | YES |
| page 2 | frame 9 | YES |

When the CPU needs to translate a virtual address, it first checks the TLB for that VPN. On a TLB hit, which happens roughly 99% of the time, the cached translation is used instantly, with no RAM access needed for translation at all, so the total cost is just the one RAM access for the actual data. On a TLB miss, which happens the remaining roughly 1% of the time, the CPU has to look up the page table in RAM, which is slow, and it then stores that result in the TLB for next time, so the total cost is two RAM accesses this time, but future accesses to the same page will be fast. Because programs tend to access the same pages repeatedly, a pattern called locality of reference, the TLB hit rate is typically 99% or higher, so the vast majority of translations end up costing nothing extra at all.

### 9.3 The TLB and Context Switches

When the OS switches between processes, the TLB has to be handled carefully, because Process A's translations are meaningless for Process B: virtual page 0 means completely different physical frames for each of them. There are two ways to handle this.

The first approach is to flush the TLB on every context switch, clearing all its entries whenever the OS switches processes. This is simple, but expensive: the next process suffers a wave of TLB misses until its own working set gets cached again. The second, more modern approach tags each TLB entry with an Address Space ID, or ASID: every entry also records which process it belongs to, so on a context switch the OS does not need to flush anything, it just changes the current ASID, and the TLB can hold entries from multiple processes at once.

| ASID | VPN | Frame | Valid |
|---|---|---|---|
| 1 | page 0 | frame 1 | YES ← Process A's entry |
| 1 | page 1 | frame 5 | YES ← Process A's entry |
| 2 | page 0 | frame 3 | YES ← Process B's entry |
| 2 | page 2 | frame 7 | YES ← Process B's entry |

When Process A, ASID 1, is running, only ASID 1 entries match; when Process B, ASID 2, is running, only ASID 2 entries match. No flushing is needed, and both processes' translations coexist peacefully in the same TLB. This is the approach used in modern ARM and x86-64 systems.

## 10. Page Faults: When a Page Is Not in RAM

### 10.1 The Valid Bit

Recall the Valid bit from the page table:

| Virtual Page | Physical Frame | Valid? |
|---|---|---|
| page 0 | frame 1 | YES |
| page 1 | frame 5 | YES |
| page 2 | (none) | NO ← not in RAM |

Page 2 here has `Valid = NO`, meaning the OS has not loaded this page into physical RAM yet. This might be because the program just started and this code has never been needed, an idea called lazy loading, or because the page was swapped out to disk earlier to free up RAM.

### 10.2 What Happens on a Page Fault

When Process A accesses a virtual address on page 2, the hardware first checks the TLB, does not find an entry, and then checks the page table, finding `Valid = NO`. This causes the hardware to raise a page fault exception, and the CPU transfers control to the OS's page fault handler.

The handler first asks whether this is even a valid virtual address for this process at all, meaning whether the process's address space is supposed to include page 2 in the first place. If not, this is a genuinely invalid address, and the OS kills the process with a segmentation fault. If it is a valid address, and the page is simply not in RAM right now, the handler continues: it finds a free physical frame, possibly by evicting an existing page to make room, discussed in the next section; loads the page from disk into that free frame, which is the slow part, since disk access takes milliseconds; updates the page table so that page 2 now points to the new frame with `Valid = YES`; updates the TLB with the new translation; and finally restarts the instruction that originally caused the fault.

Process A simply continues, with no idea any of this happened. It was paused during the disk read, and from its own perspective, the memory access just took a little longer than usual. The OS handled the entire thing invisibly.

## 11. Swapping: When RAM Is Full

### 11.1 The Problem

Physical RAM is finite. If you have 8GB of RAM but the running processes need 12GB in total, something has to give. The OS solves this by swapping: moving pages that have not been used recently out of RAM and onto disk, freeing up physical frames for pages that are needed right now.

```
Physical RAM (8GB, getting full):
┌──────────────────────────────┐
│ OS                             │
│ Process A pages (active)       │
│ Process B pages (active)       │
│ Process C pages (not used      │ ← these pages have not been
│                 recently)      │    accessed in a while
│ Process D pages (active)       │
└──────────────────────────────┘

OS decides to swap out Process C's unused pages:

Step 1: Copy Process C's pages to swap space on disk
Step 2: Mark those page table entries as Valid = NO
Step 3: Physical frames are now free

Physical RAM after swap:
┌──────────────────────────────┐
│ OS                             │
│ Process A pages                │
│ Process B pages                │
│ [FREE FRAMES]                  │ ← available for other processes
│ Process D pages                │
└──────────────────────────────┘

Disk (swap partition):
┌──────────────────────────────┐
│ Process C pages                │ ← safely stored here
└──────────────────────────────┘
```

When Process C later needs one of those swapped-out pages, a page fault occurs exactly as in the previous section, the OS reads the page back from disk into a free frame, and execution continues. The process never knew its pages had been sitting on disk at all.

### 11.2 Page Replacement Policies

When RAM is full and a page fault occurs, the OS has to choose a page to evict, and which one to pick is a policy decision, just like deciding which process runs next in scheduling.

FIFO, First In First Out, evicts whichever page has been in RAM the longest. It is simple, but it often ends up evicting pages that are still being used frequently. LRU, Least Recently Used, evicts whichever page has gone the longest without being accessed; this performs better in practice, since recently used pages are likely to be needed again soon, but implementing it perfectly is expensive, since it requires tracking every single access, which is difficult to do efficiently in hardware. The Clock algorithm offers a practical middle ground: instead of tracking exact usage times, it keeps a single reference bit per page, where `R = 0` means the page was used recently and `R = 1` means it was not; this is efficient and is widely used in real operating systems.

The choice of replacement policy has a real effect on performance, since evicting the wrong page means it will likely be needed again almost immediately, causing another page fault and another slow round trip to disk.

## 12. Copy-on-Write After fork()

Recall from an earlier post that `fork()` creates an almost exact copy of the parent process. Copying all of that memory up front would be expensive, so modern operating systems use a technique called copy-on-write, or COW, to make `fork()` fast.

Immediately after `fork()`, the child's page table entries point to the exact same physical frames as the parent's, so both processes share the same physical pages, and every one of those shared pages is marked read-only:

```
Parent page table    Physical RAM    Child page table
page 0 → frame 1  ←─── frame 1 ───→  page 0 → frame 1
page 1 → frame 5  ←─── frame 5 ───→  page 1 → frame 5
page 2 → frame 9  ←─── frame 9 ───→  page 2 → frame 9
(read-only)          (shared)          (read-only)
```

When the parent tries to write to page 1, the hardware detects a write to a read-only page and raises a fault. The OS intercepts this, and because it recognizes the page as a copy-on-write page rather than a genuine protection violation, it does not treat this as a segfault. Instead it copies frame 5 into a brand new frame, say frame 12, points the parent's page 1 at this new, now-writable frame 12, and leaves the child's page 1 pointing at the original frame 5, unchanged:

```
Parent page 0 → frame 1  ←─── frame 1 ───→  Child page 0 → frame 1
Parent page 1 → frame 12       frame 5        Child page 1 → frame 5
Parent page 2 → frame 9  ←─── frame 9 ───→  Child page 2 → frame 9
```

Pages are only ever copied when they are actually modified. If the child immediately calls `exec()`, which replaces its address space entirely, no copying ever happens at all, and `fork()` ends up being nearly instant. Once a page has been split apart by a copy-on-write, no further step keeps the two copies in sync; from that point on, the parent and child are completely independent for that particular page.

Copy-on-write makes `fork()` fast by letting the parent and child share the same physical memory at first, and only paying the cost of an actual copy the moment one of them tries to modify a shared page.

## Conclusion

We now have the complete picture of how the OS gives each process its own private memory: through the virtual address space, page tables, TLB translation, demand paging, and copy-on-write. Together with the earlier posts, the OS has now created two illusions: a CPU that seems to belong to every process, and memory that seems private to each one.

But there is a piece of this story that we have quietly used without ever opening up. Whenever a page fault occurred, or a page was swapped out, the OS read from or wrote to the disk, and we treated that as a black box: the data simply arrived, and the process carried on. Physical RAM is fast but small and loses everything when the power goes off, so sooner or later every program has to reach a real device.

This raises an immediate question. How does the CPU actually talk to a disk, a keyboard, or a network card? How does it avoid sitting idle while a slow device does its work? And how does one operating system cope with thousands of different devices from hundreds of manufacturers? These are the questions of I/O, and the OS answers them with the same strategy it used for the CPU and for memory: hide the messy hardware behind a clean, uniform interface.

That is the subject of the next post: how the CPU reaches the disk.

**Next:** [From CPU to Disk →](./09-from-cpu-to-disk.md)
