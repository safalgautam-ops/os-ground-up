---
---

> Module 3 · Post 9 of 13

## 1. Picking Up the Thread: From CPU and Memory to I/O

You have spent a long time understanding how the OS manages the CPU, through processes, scheduling, and context switching, and how it manages memory, through virtual address spaces, segmentation, and paging. In both cases, the pattern was the same: the OS creates an illusion. Each process believes it owns the entire CPU and its own private memory, when in reality the OS is secretly sharing and managing everything underneath.

I/O follows the exact same philosophy, but now the resource being managed is not the CPU or RAM, it is the devices. Disk, keyboard, network card, USB, all of these are physical hardware that the OS must control, share between processes, and hide the complexity of. When your program calls `read()`, it has no idea whether the data is on an SSD, a spinning disk, or a USB drive. The OS and its drivers handle all of that invisibly, just as they handled memory translation invisibly through paging.

There is also a direct connection to interrupts, which you already met during CPU virtualization. When a timer interrupt fires, the OS regains control of the CPU and can switch processes. The exact same interrupt mechanism is reused here: when a disk finishes reading, it fires an interrupt to wake up the process that was waiting on it. One mechanism, used everywhere.

## 2. Why I/O Exists At All

A computer with only a CPU and RAM is useless in isolation. It cannot save anything, cannot receive input, and cannot display anything. I/O, Input/Output, is how the CPU talks to everything else: keyboard, screen, disk, network, USB drives, all of it.

The OS is the middleman for every one of these interactions. No user program is allowed to talk to a device directly; the OS manages every device interaction on its behalf, and this post explains exactly how.

## 3. System Architecture: How Devices Connect to the CPU

It helps to think of the following diagram as a hierarchy of roads: some are highways, fast, expensive, and short, and some are local streets, slow, cheap, and numerous.

```
CPU ──── Memory
    └── Memory Bus (fastest, proprietary, very short)
         │
         ├── General I/O Bus (PCI), medium speed
         │        └── Graphics card (needs high bandwidth)
         │
         └── Peripheral I/O Bus (SCSI, SATA, USB), slowest
                  └── Disks, mice, keyboards, USB drives
```

### 3.1 Why Three Levels? Physics and Cost

There are two reasons for this layered design: physics and cost. Physically, the faster a bus needs to be, the shorter it has to be. A memory bus running at high speed can only be a few centimeters long on the motherboard, so you cannot plug 20 devices into it; there simply is not room. Cost-wise, engineering a fast bus is extremely expensive, and you do not want to waste that expensive fast connection on something as slow as a keyboard.

So the design puts the things that need speed, graphics and memory, close to the CPU on fast buses, and puts slow things, disks and mice, further away on cheap, slow buses. This lets the system attach many slow devices without wasting expensive fast-bus capacity on them.

## 4. What a Device Looks Like From the Inside

Every device, no matter how complex internally, presents the OS with two parts.

### 4.1 The Interface: Three Registers

The interface is what the OS actually sees and uses, and it typically consists of three registers. The status register is what the OS reads to ask "are you busy, done, or in an error state?", much like checking whether a printer has a paper jam. The command register is what the OS writes to, to tell the device what to do, such as "read sector 42" or "print this page." The data register is what the OS uses to send data to the device or to receive data back from it.

### 4.2 The Internals: Hidden From the OS

Beneath that interface sits everything the OS never needs to see: a micro-controller, essentially a tiny CPU inside the device itself; its own memory, DRAM or SRAM; and whatever other hardware-specific chips the device needs. The OS only ever touches the interface, the three registers, and never needs to know what is happening underneath. This is the same idea as a TV remote: you press a button, the interface, and you never think about the electronics that make it work inside.

## 5. The Basic Protocol: How the OS Talks to a Device

### 5.1 The Four Steps

The OS follows the same basic sequence to use almost any device. First, it checks the status register in a loop, waiting until the device is not busy. Second, it writes the data it wants to send into the data register. Third, it writes a command into the command register, which starts the device working. Fourth, it checks the status register in a loop again, this time waiting until the device reports that it is done. In code, it looks something like this:

```c
while (STATUS == BUSY)
    ;                            // keep checking, do nothing else

write(DATA_REGISTER, myData);
write(COMMAND_REGISTER, READ);   // tell device to start

while (STATUS == BUSY)
    ;                            // wait until it finishes
```

This works, and it is simple, but it has a serious problem.

### 5.2 The Problem: Polling Wastes the CPU

Steps one and four both sit in a loop doing nothing but repeatedly checking the status register. This is called polling, and it is a bit like standing at a bus stop constantly glancing left to see if the bus is coming, instead of sitting down and reading a book while you wait.

While the CPU is polling, it is doing zero useful work, and devices, especially disks, are slow. A disk read might take 10 milliseconds, and a modern CPU can execute around 10 million instructions in that same 10 milliseconds. That is 10 million instructions completely wasted, just waiting.

## 6. Interrupts: The Fix for Polling

### 6.1 Polling vs Interrupts, Side by Side

Under polling, the picture looks like this, where `1` marks Process 1 running and `p` marks the CPU polling uselessly:

```
(polling, bad):
CPU:  [1][1][1][1][1][p][p][p][p][p][1][1][1][1][1]
Disk:                [1][1][1][1][1]
```

Process 1 runs for a while, then issues a disk request. The CPU spends the entire disk operation just polling, completely idle, and only once the disk finishes does Process 1 resume.

Under interrupts, the picture looks like this, where `2` marks Process 2 running instead:

```
(interrupts, good):
CPU:  [1][1][1][1][1][2][2][2][2][2][1][1][1][1][1]
Disk:                [1][1][1][1][1]
```

Process 1 issues the disk request, and the OS puts it to sleep, since it has nothing to do until the disk responds. The OS immediately runs Process 2 on the CPU instead. While Process 2 runs, the disk does its own work in parallel, and when the disk finishes, it sends an interrupt signal to the CPU. The OS then wakes Process 1 back up and resumes it. Both the CPU and the disk are working simultaneously, and that overlap is the entire win.

### 6.2 How Interrupts Work Mechanically

When the disk finishes, it electrically signals the CPU that it is done. The CPU stops whatever it is doing, saves its current state, and jumps to a pre-registered piece of OS code called an interrupt handler, also known as an Interrupt Service Routine, or ISR.

The interrupt handler reads the data from the device if this was a read request, marks the waiting process as ready to run, and then returns control to whatever was running before the interrupt arrived. From there, the OS scheduler picks the next process to run, which might well be the now-ready Process 1.

### 6.3 When Interrupts Are Not Better

Interrupts are not free either. They add overhead of their own: saving and restoring state, switching processes, and running the handler itself. If a device is very fast, this overhead can actually cost more than it saves.

Imagine a device that finishes its work in 50 microseconds, while a context switch costs around 10 microseconds. In that case, it might genuinely be faster to just poll for those 50 microseconds than to pay the cost of a full context switch. As a rule of thumb, a slow device, such as a disk or a network card, should use interrupts, letting the CPU do other work in the meantime. A very fast device, such as some SSDs or purely in-memory operations, is often better served by polling, avoiding the context switch overhead entirely. When the device's speed is unknown ahead of time, a hybrid approach works well: poll briefly first, and only switch over to interrupts if the operation has not finished by then.

### 6.4 Livelock and Coalescing

Interrupts introduce their own failure mode, called livelock. If a web server gets flooded with network packets, each one triggers its own interrupt, and the CPU can end up spending all of its time handling interrupts, never actually getting around to processing any requests. The fix is to poll temporarily under heavy load, handling a batch of packets together instead of interrupting for each one individually.

Coalescing is a related optimization that works even under normal load. Instead of the device interrupting the CPU for every single completed operation, it waits a short moment to collect several completions together and delivers them all in one interrupt. This reduces interrupt overhead, at the cost of a slightly increased latency for any individual completion.

## 7. DMA: Solving the Data Movement Problem

### 7.1 Programmed I/O: The CPU as a Copy Machine

Even once interrupts solve the waiting problem, there is still the problem of actually moving the data. When the OS wants to write 4KB to disk, it has to copy that data, word by word, from RAM into the device's data register; the disk cannot reach directly into RAM and grab it itself. The CPU has to perform each of these copies manually. This approach is called Programmed I/O, or PIO:

```
CPU:  [1][1][1][1][1][c][c][c][2][1][2][2][1][2][2][1][1][1][1]
Disk:                         [1][1][1][1][1]
```

The `c` slots above are the CPU doing this copying, one chunk at a time, purely mechanical work that wastes its time. Even though interrupts already solved the pure waiting problem, the CPU here is still tied up manually shuttling the data over before the disk can even begin its own work; only once the copying is done does the disk's own busy period start. After that, the CPU goes back to alternating between other process work and handling the eventual completion interrupt from the disk, but the copying itself was still pure overhead.

### 7.2 DMA: A Dedicated Copying Engine

DMA, Direct Memory Access, is a dedicated hardware engine on the motherboard whose only job is moving data between RAM and devices, entirely without CPU involvement. The OS simply tells the DMA controller "copy 4KB from RAM address X to the disk," and it is done; the DMA engine handles all of the actual copying independently, and the CPU is free immediately.

```
CPU:  [1][1][1][1][1][2][2][c][c][2][2][c][1][2][2][1][1][1][1]
DMA:                     [c][c][c]
Disk:                            [1][1][1][1][1]
```

Here the DMA engine does the copying, marked `c`, while the CPU keeps running Process 2 the whole time. When the DMA engine finishes copying, it raises an interrupt to tell the OS, the disk then does its own work, and when the disk itself is done, it raises another interrupt in turn. The CPU is free for actual computation the entire time; it only gets involved briefly at the start, to tell the DMA engine what to do, and briefly at the end, to run the interrupt handler.

## 8. How the OS Communicates With Device Registers

There are two ways the OS actually reads and writes those status, command, and data registers.

### 8.1 Explicit I/O Instructions

In this first method, devices live in a separate address space called I/O ports, and the CPU has special instructions dedicated purely to talking to them. On x86, these are `in`, to read from a device, and `out`, to write to one. You specify a port number that identifies which device register you are targeting, and conceptually it looks like this: the CPU issues a special instruction, which goes to an I/O port, which reaches the device register.

```c
outb(0x1F7, IDE_CMD_READ);  // write READ command to IDE disk
```

These instructions are privileged; only the OS, running in kernel mode, can execute them. If any user program could directly command the disk, it could read or overwrite anyone's data, and the result would be chaos.

### 8.2 Memory-Mapped I/O

In the second method, device registers are mapped onto specific memory addresses, and the OS reads and writes them using ordinary load and store instructions. Some addresses are not connected to real RAM at all; the hardware intercepts accesses to those addresses and redirects them to the corresponding device instead.

```c
// Address 0xFE000000 is mapped to the graphics card's control register
*((volatile int *)0xFE000000) = COMMAND_DRAW;  // normal write, goes to device
```

No special instructions are needed here; the same `mov` instruction you would use for ordinary memory works, and the hardware routing handles the redirection transparently. Both methods are used in modern systems, and neither is strictly better than the other; they coexist.

## 9. Device Drivers: Hiding the Mess

Every device is different. An old IDE disk has one set of registers, a USB drive speaks a totally different protocol, and an NVMe SSD is different again entirely. If the OS had to handle each device directly, on its own terms, it would become unmanageably complex. The solution is device drivers.

### 9.1 The Layered Stack

A device driver is a piece of OS code that knows everything about one specific device. It translates the OS's generic requests, such as "read block 42," into the specific sequence of register reads and writes that particular device requires.

```
Application (your program)
    │  uses POSIX API: open(), read(), write(), close()
    ▼
File System (ext4, NTFS, etc.)
    │  thinks in terms of files and directories
    │  issues generic "read block N" requests
    ▼
Generic Block Interface
    │  a standard contract: block read/write
    ▼
Generic Block Layer (OS kernel)
    │  routes requests to the right driver
    ▼
Specific Block Interface
    │  protocol-specific: SCSI protocol, ATA protocol, etc.
    ▼
Device Driver (SCSI driver, ATA driver, etc.)
    │  knows register addresses, command sequences, error codes
    ▼
Physical Hardware (the actual disk)
```

The elegant part is that the file system does not know or care whether the disk underneath is SCSI, SATA, USB, or NVMe; it simply says "give me block 42" to the generic block layer, which figures out which driver to call, and that driver handles all of the hardware-specific details. This layering means that when a new type of disk is invented, only a new driver needs to be written; the file system, the generic block layer, and every application built on top all keep working unchanged. Plug in a new USB drive, install its driver, and everything just works.

### 9.2 The Cost of Uniformity

This uniformity comes at a price: going through a generic interface loses device-specific detail. SCSI disks, for example, have very detailed error reporting and can tell you precisely what went wrong, but the generic block interface only offers a generic "I/O error" code, and all of that finer detail disappears along the way. The file system only ever sees `EIO`, a generic I/O error, never the specific SCSI error underneath it.

It is a sobering fact that over 70% of Linux kernel code is device drivers. The OS itself, scheduling, memory management, file systems, is a comparatively small fraction of the whole; most of the OS is simply code written to talk to specific hardware. And because drivers are often written by the device manufacturers themselves rather than kernel experts, they tend to contain the most bugs and cause the most kernel crashes of any part of the system.

## Conclusion

I/O completes the picture. The CPU and disk can now work in parallel thanks to interrupts. Data moves without wasting the CPU thanks to DMA. And the entire hardware zoo, thousands of different devices from hundreds of manufacturers, is tamed by the layered driver model, which presents a clean, uniform interface upward so that file systems and applications never need to care about the specifics underneath.

What you have built, across this checkpoint and the two posts before it, is a mental model of the entire OS: from how a single process runs and gets scheduled, to how its memory is translated on every instruction, to how its data reaches the physical disk and comes back again. Every piece connects. The OS is not a collection of unrelated features; it is one coherent system, built around a single idea: take messy, limited, shared hardware, and make it feel clean, unlimited, and private to every program running on top of it.

There is still one piece of that picture left unopened. Whenever more than one process wants the CPU at the same time, something has to decide who actually goes first, and for how long. That decision is what the next post takes on: CPU scheduling.

**Next:** [CPU Scheduling: Introduction →](./10-cpu-scheduling-introduction.md)
