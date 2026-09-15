# How the OS Never Forgets, Inside the PCB

> Module 1 · Post 5 of 13

## 1. Picking Up Where We Left Off

Part 1 ended with a question: how does the OS keep track of hundreds of simultaneous processes correctly? When the OS pauses a process and resumes it later, how does it know where that process was, what files it had open, what its registers were, and what state it was in?

The answer is a data structure called the Process Control Block, one per process, maintained by the OS and updated constantly as the process runs. The PCB is a kernel data structure: it lives in the OS kernel's own memory, not in the process's own address space. The process itself cannot see or touch its own PCB. Only the kernel manages it.

## 2. What the PCB Contains

The PCB is essentially a complete snapshot of everything the OS needs to know about a process. It holds several categories of information, all connected to each other.

### 2.1 pid: Process ID

When the OS creates a process, it assigns it a unique integer called the process ID, or pid. This number is the process's identity for its entire lifetime. No two running processes ever share the same pid. The pid connects to everything else in the PCB. When the OS needs to find a specific process's PCB, to send it a signal, to check its state, or to modify its scheduling priority, it uses the pid as the key to look it up.

### 2.2 state: Current Process State

Every process is always in exactly one state at any given moment. The state field in the PCB records this, and the scheduler reads this field constantly to make decisions about who gets the CPU.

RUNNING means this process currently has the CPU and is executing instructions right now. On a single-core system, only one process can be in this state at any moment. On a multi-core system, one process per core can be Running simultaneously.

RUNNABLE, also called Ready, means the process is fully capable of running (it has everything it needs) but another process currently has the CPU. It is waiting in the scheduler's queue. The moment the CPU becomes available, a Runnable process can be promoted to Running instantly.

SLEEPING, also called Blocked, means the process is waiting for something external. It might have requested a disk read and the data has not arrived yet. It might be waiting for a network packet, or for a timer to expire. It cannot use the CPU even if you gave it one, because the thing it needs does not exist yet, so the OS will not schedule a Sleeping process. When the event it is waiting for occurs, whether the disk signals completion via an interrupt, a network packet arrives, or a timer fires, the OS transitions it from Sleeping to Runnable.

ZOMBIE is the state a process enters after it has exited but before its parent has acknowledged the exit. The process is dead: it is no longer executing any code, its memory has been freed, and its resources have been released. But its PCB still exists, preserved in the zombie state, holding the exit status code that the parent needs to collect. Once the parent calls `wait()` and collects that exit code, the OS destroys the PCB entirely and the pid is released.

The state field is what makes the scheduler efficient. Without it, the scheduler would have to examine every process in detail to figure out which ones are eligible to run. With it, the scheduler simply scans for processes in the Runnable state and picks among them. Blocked processes are invisible to the scheduler until an interrupt wakes them up.

### 2.3 context: Saved Register Values

This is the field that makes time sharing possible. Without it, multitasking as we know it could not exist.

When the OS decides to pause a running process, whether because its time slice expired, because it blocked on I/O, or because a higher-priority process became ready, the CPU is in the middle of doing something. All the CPU registers contain values that belong to that process's computation. The Program Counter holds the address of the next instruction to execute. The Stack Pointer holds the current top of the stack. The general-purpose registers hold intermediate values: partial results of arithmetic, addresses being used, loop counters, anything the process was working with.

If the OS simply wiped these out and let another process use the CPU, that computation would be destroyed. When the OS tried to resume the original process later, it would have no idea where it was or what it was doing.

The context field solves this completely. The moment the OS decides to pause a process, it saves every single CPU register into the context field of that process's PCB: the Program Counter, the Stack Pointer, the Frame Pointer, all general-purpose registers, the status flags, everything. The CPU's complete state at that exact moment is frozen into the PCB.

Later, when the OS decides to resume that process, it does the reverse. It reads every value out of the context field and loads them back into the actual CPU registers. The CPU is now in exactly the same state it was in when the process was paused. The process resumes from exactly the instruction it was about to execute, with exactly the values it had in every register. From the process's own perspective, nothing happened; it has no awareness of ever having been paused.

This save-and-restore operation is called a context switch. It happens thousands of times per second on a busy system. The context field in the PCB is the mechanism that makes it lossless: no computation is ever destroyed by the act of switching.

### 2.4 mem and sz: Memory Location and Size

The `mem` field stores the starting address of the process's memory in physical RAM. The `sz` field stores how large that memory region is.

These exist because the OS must know precisely where each process's memory lives. When the process is Running, the CPU needs to know which physical memory addresses correspond to this process's virtual address space. When the OS switches from one process to another during a context switch, it must also switch the memory mapping, so the CPU starts seeing the new process's memory instead of the old process's. The `mem` and `sz` fields tell the OS what that mapping should be.

These fields also serve as boundaries. If a process tries to access a memory address that falls outside its own region, either into another process's memory or into completely unmapped space, the OS catches this using `mem` and `sz` and delivers a segmentation fault. This is how process isolation is enforced. One process cannot read or corrupt another process's memory, because the OS uses these fields to verify that every memory access stays within legitimate bounds.

When the process exits, `sz` tells the OS exactly how much memory to free and return to the system's pool of available physical memory.

In modern systems, this concept is implemented through page tables, a more sophisticated mechanism that provides fine-grained control over memory mapping, but `mem` and `sz` represent the fundamental idea that the OS must track where each process's memory lives.

### 2.5 kstack: Kernel Stack

Every process actually has two stacks, not one. The user stack is the one described in the previous post: the Stack segment in the process's own virtual address space, holding local variables, function parameters, and return addresses for the process's own code. The kernel stack is completely separate. It lives in protected kernel memory, invisible and inaccessible to the process itself.

The kernel stack exists because the OS frequently needs to execute code on behalf of a process. When a process calls `read()`, `write()`, `open()`, or `malloc()`, or makes any system call at all, execution transfers from the process's own code into the OS kernel. The kernel code that handles the system call needs a stack of its own for its local variables, its own function call chain, and its own return addresses. It cannot use the process's user stack for this, for a critical reason: the process's user stack lives in memory that the process itself controls. A buggy or malicious program could have corrupted it, set it up in a way meant to trick the kernel, or tried to manipulate it while the kernel is executing. The kernel cannot trust user memory for its own execution.

So each process has a small, private kernel stack allocated in kernel-controlled memory. When the process makes a system call, the CPU switches from user mode to kernel mode, and the stack pointer switches from the user stack to this kernel stack. The kernel does its work using the kernel stack entirely. When the system call completes and returns to the process, the stack pointer switches back to the user stack, and the process resumes its own code on its own stack as if nothing unusual happened.

The kernel stack is also used when an interrupt occurs. If a timer interrupt fires while a process is running, the OS interrupt handler needs a stack to execute on, and it uses the current process's kernel stack for this. This is why every process needs its own kernel stack: interrupts and system calls can happen to any process at any time, and each needs isolated kernel stack space.

### 2.6 parent: Pointer to Parent Process

Every process except pid 1 was created by another process. The `parent` field in the PCB stores a pointer directly to the PCB of the process that created this one.

In Unix, the only way to create a new process is through the `fork()` system call. When a process calls `fork()`, the OS creates a new process that is an almost exact copy of the parent, with the same memory contents, the same open files, and the same state, only with a new pid. The child's PCB has its `parent` field pointing to the parent's PCB. This creates the process tree, a hierarchy where every process knows its creator.

The parent pointer is essential for the zombie and wait system to function correctly. When a process exits, it does not just disappear. It needs to communicate its exit status, whether it succeeded or failed and what its return code was, to the process that created it. This is how shell scripts know whether a command succeeded. The child's PCB is preserved in zombie state, holding the exit status, and the OS uses the parent pointer to know which process is the intended recipient of it. The OS sends a signal to the parent, telling it a child has exited. The parent then calls `wait()`. The OS uses the parent pointer to find the relevant zombie child, extracts the exit status, hands it to the parent, and only then destroys the child's PCB completely.

Without the parent pointer, the OS would not know who to notify when a process dies, and orphaned zombie processes would accumulate with no one to clean them up. In practice, if a parent process dies before its children, the OS reassigns those children's `parent` field to pid 1, which is `init` or `systemd`, and that process continuously calls `wait()` to clean up any orphaned zombies.

### 2.7 ofile[]: Open File Descriptor Table

`ofile[]` is an array stored inside the PCB. Each slot in this array corresponds to one file descriptor. A file descriptor is just a small non-negative integer, such as 0, 1, 2, or 3, that the process uses as a handle to refer to an open file or other resource.

When a process opens a file by calling `open()`, the OS does several things. It finds the file on disk, sets up internal kernel structures to track the reading position and access mode, and then finds the lowest available slot in `ofile[]` and places a reference to that internal structure there. It returns the index of that slot, the file descriptor number, back to the process. From that point on, whenever the process wants to read from or write to that file, it passes the file descriptor number to the OS, which looks it up in `ofile[]` and finds the actual file information.

Three file descriptors exist by default for every process, already filled in when the process starts. File descriptor 0 is standard input, the place where the process reads keyboard input from. File descriptor 1 is standard output, where the process writes its normal output. File descriptor 2 is standard error, where the process writes error messages. These three slots in `ofile[]` are pre-populated by the OS before the process ever runs. When you write `printf()` in C, it eventually writes to file descriptor 1. When your program crashes and prints an error, it goes to file descriptor 2.

Every subsequent `open()`, `socket()`, `pipe()`, or similar call adds another entry to `ofile[]`, filling the next available slot. If a process opens ten files, slots 3 through 12 in `ofile[]` are occupied, each pointing to a different internal kernel file structure.

The entries in `ofile[]` do not point directly to the file on disk. They point to a kernel data structure called an open file description, sometimes called a file table entry. This intermediate structure exists for an important reason: the file table entry records two critical things, the current file offset, meaning how many bytes into the file you have read so far, and the access mode, meaning whether the file was opened for reading, writing, or both.

The reason this intermediate layer exists is that multiple file descriptors can point to the same file table entry. When a process calls `fork()` to create a child process, the child gets a copy of the parent's `ofile[]` array. Both the parent's file descriptor and the child's file descriptor now point to the same file table entry, meaning they share the same file offset. If the parent reads 100 bytes, the offset advances, and the child will pick up reading from byte 101. This sharing is deliberate and essential for how Unix pipes and process communication work.

From the OS's perspective, when a process dies, everything that process was holding needs to be released. Memory gets unmapped: the OS looks at the memory information in the PCB and frees all the pages. The pid is released back into the pool for future processes. CPU registers no longer need saving. But none of that is the tricky part.

The tricky part is resources that exist outside the process's own memory, things in the kernel or in the physical world. Open files are exactly this kind of resource.

When a file is open, the OS has internal structures tracking it. The file may be locked, preventing other processes from writing to it. A network socket being open means a connection is being maintained with a remote machine: packets are being expected, and state is being held on both ends. A pipe being open means another process on the other end is waiting. None of these things disappear automatically when the process's memory is freed. They live in the kernel and in the network, and if the OS did nothing with them, they would persist indefinitely.

So when a process exits, the OS goes to the PCB, finds `ofile[]`, and walks through every slot. For each slot that contains an open file descriptor, it performs the proper close operation: flushing any buffered writes to disk, releasing file locks, sending the appropriate network termination signals to close TCP connections cleanly, and notifying any process waiting on the other end of a pipe that the pipe is now closed. Only after doing this for every entry does the OS fully tear down the PCB and consider the process gone.

This is why you can write programs that crash halfway through, forget to call `close()` or `fclose()` on files, and still find that the files are not corrupted or permanently locked afterward. The OS cleaned up through `ofile[]`. It is a safety net built into the process lifecycle.

### 2.8 cwd: Current Working Directory

The `cwd` field stores the directory the process is currently considered to be inside. Every process has one at all times.

When a process opens a file using a relative path, a path that does not start with `/`, the OS needs to know what directory to start looking from. The `cwd` is that starting directory. If `cwd` is `/home/user/projects` and the process calls `open("data.txt")`, the OS looks for `/home/user/projects/data.txt`. If `cwd` is `/tmp` and the same call is made, the OS looks for `/tmp/data.txt`. Same code, different `cwd`, completely different file.

The `cwd` is inherited from the parent process through `fork()`. When a shell runs a program, the child process starts with the same `cwd` as the shell. This is why programs launched from a directory can naturally find files in that directory by name alone. The shell's `cd` command works by calling a system call named `chdir()` on itself, updating its own `cwd` field in its PCB. Every subsequent program the shell launches inherits that updated `cwd`.

The `cwd` is also used for resolving other relative paths, not just file opens. Creating a file, deleting a file, creating a directory, or listing files: any operation that takes a path resolves relative paths against `cwd`. An absolute path starting with `/` ignores `cwd` entirely and is resolved from the filesystem root, but relative paths always go through `cwd`, making it a silent but constant presence in nearly every file operation a process performs.

## Conclusion

Together, these fields are everything. A process does not exist as a physical thing; it exists as this record in the OS's memory, plus the address space it points to. Delete the PCB and the process is gone, even if its code and data are still sitting in RAM.

But knowing how the OS tracks processes raises the next natural question. How does a process actually come into existence in the first place? The PCB has a `parent` field, which means every process has a parent, and that means every process was created by another process. How does that creation actually work? What system calls are involved? What exactly happens at the OS level when you type a command in your terminal and hit enter?

**Next:** [How Programs Are Born: The Process API →](./06-how-programs-are-born.md)
