---
---

> Module 5 · Post 12 of 13

## 1. Picking Up the Thread

Every post so far in this series has assumed a single process has exactly one thing executing instructions at a time: one program counter, one set of registers, one stack. Scheduling, the subject of the last two posts, was entirely about deciding which of these single-threaded processes gets the CPU next. That model is about to change.

This post builds concurrency from absolute zero. It is a long topic, but every piece connects to the next, so it helps to read it in order rather than skipping ahead.

## 2. What Is a Thread?

### 2.1 Starting From What We Know

So far, every process in this series has run one instruction, then the next, then the next, in a single straight line of execution. This is called a single-threaded process.

But what if a program needs to do multiple things at the same time? Not the OS running multiple programs simultaneously, since that is already familiar from earlier posts, but one single program doing multiple things simultaneously. A web browser might need to download a file, render a web page, and play a video all at once. A web server might need to handle a thousand client connections at the same time. A word processor might need to check spelling while you type. All of these require a single program to have multiple simultaneous points of execution, and that is exactly what a thread provides.

### 2.2 The Definition

A thread is an independent point of execution within a process. A single-threaded process has one thread, and a multi-threaded process has two or more threads.

Each thread has its own program counter, tracking which instruction it is executing, its own registers, holding its own computation state, and its own stack, holding its own function call history and local variables. But all threads within the same process share the same code segment, so every thread runs from the same program code, the same heap, so every thread can allocate from and read the same dynamic memory, the same static data, so every thread sees the same global variables, and the same file descriptors, so every thread sees the same open files. This sharing is exactly what makes threads powerful, and exactly what makes them dangerous, as the rest of this post will show.

### 2.3 Thread vs Process: The Key Difference

A process has its own address space, completely private from every other process, along with its own code, heap, stack, and static data. It cannot accidentally access another process's memory. Creating a process is expensive, since the OS has to set up an entire new address space, and switching to it requires switching page tables.

A thread, by contrast, shares its address space with every other thread in the same process, including the code, heap, and static data. It has only its own stack and its own registers, but it can access all of the shared memory directly. Creating a thread is cheap, since no new address space is needed, and a context switch between threads does not need to switch page tables at all. This is why threads are sometimes called lightweight processes: they are much cheaper to create and to switch between than full processes.

### 2.4 The Address Space With Multiple Threads

With a single thread, a process's address space looks like this:

```
Single-threaded process address space:
┌──────────────────┐ 0KB
│ Code             │
├──────────────────┤ 1KB
│ Heap             │ ← grows downward
│                  │
│ (free space)     │
│                  │
│ Stack            │ ← grows upward, ONE stack
└──────────────────┘ 16KB
```

With two threads, each thread needs its own stack, but the code and heap remain shared between them:

```
Multi-threaded process address space (2 threads):
┌──────────────────┐ 0KB
│ Code             │ ← SHARED by both threads
├──────────────────┤ 1KB
│ Heap             │ ← SHARED by both threads
│                  │
│ (free space)     │
│                  │
│ Stack (Thread 2) │ ← Thread 2's private stack
│                  │
│ (free space)     │
│                  │
│ Stack (Thread 1) │ ← Thread 1's private stack
└──────────────────┘ 16KB
```

The address space now has two stacks, one per thread. Each thread's local variables, function calls, and return addresses live in its own stack, and they do not interfere with each other. This private region is sometimes called thread-local storage: data that belongs to one specific thread and no other.

### 2.5 The Thread Control Block

Just as the OS tracks processes using a Process Control Block (PCB), it tracks threads using a Thread Control Block (TCB). The TCB stores everything that is specific to one thread:

```
Thread Control Block (TCB):
┌──────────────────────────────┐
│ Thread ID                    │ ← unique identifier
│ Program Counter              │ ← where this thread is executing
│ Stack pointer                │ ← top of this thread's stack
│ Register values              │ ← this thread's computation state
│ State (running/ready/blocked)│ ← what the thread is doing now
└──────────────────────────────┘
```

Notice what is not in the TCB: the address space, the file descriptors, the heap. Those are shared resources that belong to the process as a whole and live in the process's own PCB instead.

### 2.6 Context Switching Between Threads

Context switching between threads is similar to context switching between processes, but simpler in one important way.

Between two processes, the OS saves process A's registers to A's PCB, switches page tables so the CPU sees an entirely different memory view, loads process B's registers from B's PCB, and resumes process B. Between two threads in the same process, the OS saves thread 1's registers to thread 1's TCB, loads thread 2's registers from thread 2's TCB, and resumes thread 2, with no page table switch at all, since both threads already live in the same address space. The page table switch is the expensive part of a process context switch, and threads skip it entirely, which is why thread context switches are significantly faster than process context switches.

## 3. Creating Threads: A Concrete Example

### 3.1 The Code

```c
#include <stdio.h>
#include <assert.h>
#include <pthread.h>

// takes a generic pointer, typecasts it into a char *, and returns NULL
void *mythread(void *arg) {
    printf("%s\n", (char *) arg);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p1, p2;
    int rc;
    printf("main: begin\n");

    // stores the thread ID, NULL means use default thread attributes,
    // and "A" is the argument passed to the thread function
    rc = pthread_create(&p1, NULL, mythread, "A");
    // checks whether the previous pthread function succeeded
    assert(rc == 0);
    rc = pthread_create(&p2, NULL, mythread, "B");
    assert(rc == 0);
    rc = pthread_join(p1, NULL);
    assert(rc == 0);
    rc = pthread_join(p2, NULL);
    assert(rc == 0);
    printf("main: end\n");
    return 0;
}
```

### 3.2 Breaking Down the Code

`pthread_t p1, p2` declares two variables of type `pthread_t`, a type that represents a thread. `p1` and `p2` will hold handles to the two threads, much like a `pid_t` holds a handle to a process.

`pthread_create(&p1, NULL, mythread, "A")` creates a new thread and takes four arguments: `&p1` is where the thread handle gets stored, `NULL` means the thread should use default attributes, `mythread` is the function this new thread will run, and `"A"` is the argument passed into that function. The moment this line runs, a new thread is created and starts running `mythread("A")` independently, while the main thread continues on to the next line at the same time.

`mythread(void *arg)` is the function each thread runs. It receives one argument, the string `"A"` or `"B"`, prints it, and returns. When a thread's function returns, that thread terminates.

`pthread_join(p1, NULL)` makes the calling thread, in this case main, wait until thread `p1` finishes, much like `wait()` does for a child process. The main thread blocks here until thread 1 completes, then moves on to the next line.

### 3.3 The Non-Determinism Problem

Here is the most important thing to understand about threads: you cannot predict the order in which they run.

After both calls to `pthread_create` return, three separate things exist at once: the main thread, continuing to execute the rest of `main()`, thread 1, running `mythread("A")`, and thread 2, running `mythread("B")`. The scheduler decides who actually runs when, and that produces several different possible orderings of the same program.

In one possible ordering, main creates both threads and then they run afterward:

```
Trace 1 - Main creates both, then threads run:
Main:     starts → prints "main: begin" → creates T1 → creates T2 → waits for T1
Thread 1:                                              → runs → prints "A" → exits
Thread 2:                                                                         → runs → prints "B" → exits
Main:     T1 done → waits for T2 → T2 done → prints "main: end"

Output:
main: begin
A
B
main: end
```

In another possible ordering, thread 1 runs immediately as soon as it is created, before thread 2 even exists:

```
Trace 2 - Thread 1 runs immediately after creation:
Main:     creates T1
Thread 1: immediately runs → prints "A" → exits
Main:     creates T2
Thread 2: immediately runs → prints "B" → exits
Main:     both already done when join is called → prints "main: end"

Output:
main: begin
A
B
main: end
```

In a third possible ordering, thread 2 happens to run before thread 1, even though thread 1 was created first:

```
Trace 3 - Thread 2 runs before Thread 1:
Main:     creates T1 → creates T2
Thread 2: runs first → prints "B" → exits
Thread 1: runs second → prints "A" → exits
Main:     joins complete → prints "main: end"

Output:
main: begin
B
A
main: end
```

All three outputs are valid. The scheduler makes these ordering decisions, and the program itself cannot assume any particular one will happen. This is already unsettling, but it is about to get much worse.

## 4. Shared Data: The Real Problem

### 4.1 The Dangerous Example

```c
static volatile int counter = 0;

void *mythread(void *arg) {
    printf("%s: begin\n", (char *) arg);
    int i;
    for (i = 0; i < 1e7; i++) {
        counter = counter + 1;
    }
    printf("%s: done\n", (char *) arg);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p1, p2;
    printf("main: begin (counter = %d)\n", counter);

    Pthread_create(&p1, NULL, mythread, "A");
    Pthread_create(&p2, NULL, mythread, "B");

    Pthread_join(p1, NULL);
    Pthread_join(p2, NULL);
    printf("main: done with both (counter = %d)\n", counter);
    return 0;
}
```

The capitalized `Pthread_create` and `Pthread_join` here are simple wrapper functions used throughout OS textbooks. They call the real `pthread_create` and `pthread_join` underneath and automatically assert that the call succeeded, saving the explicit `assert(rc == 0)` lines from the previous example. `counter` is declared `volatile` so the compiler does not cache its value in a register across loop iterations; this keeps the compiler from optimizing the loop in a way that would hide the bug, but volatile does not make the increment safe, as the rest of this section shows.

Two threads run at once, and each one adds 1 to `counter` ten million times. Both run to completion. The expected final value is simple: 10,000,000 plus 10,000,000 equals 20,000,000. But the actual output looks like this:

```
Run 1: counter = 20000000   ← correct
Run 2: counter = 19345221   ← WRONG
Run 3: counter = 19221041   ← WRONG AND DIFFERENT
```

Not only is the answer wrong, it is different on every run. How is this possible?

### 4.2 Why This Happens: The CPU Level

The key is understanding what `counter = counter + 1` actually is at the CPU level. It looks like a single operation, but it is not. The compiler translates it into three separate CPU instructions:

```
Instruction 1: mov 0x8049a1c, %eax
               Read counter's value from memory into register eax
Instruction 2: add $0x1, %eax
               Add 1 to eax
Instruction 3: mov %eax, 0x8049a1c
               Write eax back to counter's memory location
```

Three separate instructions, and the OS can interrupt a thread between any two of them.

### 4.3 The Race Condition, Step by Step

Suppose `counter` is currently 50, and thread 1 is running. Thread 1 runs instruction 1, reading counter's value of 50 into its own `eax` register; at this point, `eax` in thread 1 is 50, and `counter` in memory is still 50. Thread 1 then runs instruction 2, adding 1 to get 51 in its register; `eax` in thread 1 is now 51, but `counter` in memory has not been written back yet, it is still 50.

At this exact moment, an interrupt fires. The OS's timer goes off and triggers a context switch to thread 2. The OS saves thread 1's state, its program counter pointing at instruction 3 and its `eax` value of 51, into thread 1's TCB. Thread 1 is now paused, still holding 51 in its saved register, but that value has not reached memory.

Thread 2 now starts running. It runs instruction 1, reading `counter` from memory, which is still 50, since thread 1 never wrote its 51 back. Thread 2's `eax` is now 50. It runs instruction 2, adding 1 to get 51 in its own register. It runs instruction 3, writing 51 back to `counter` in memory.

Now another interrupt fires, and the OS switches back to thread 1, restoring its saved state: program counter at instruction 3, `eax` still 51, exactly as it was saved. Thread 1 resumes and runs instruction 3, writing its own 51 back to `counter` in memory.

The final result is `counter = 51`. But both thread 1 and thread 2 incremented the counter, so the correct answer should have been 52. One increment was lost.

### 4.4 The Full Trace Diagram

```
PC    eax   counter
─────────────────────────────────────────────────────
Start         100   0     50
Thread 1:
  mov          105   50    50    ← T1 reads 50 into eax
  add          108   51    50    ← T1 adds 1, eax=51
  INTERRUPT → save T1 state (PC=108, eax=51)
  restore T2 state (PC=100, eax=0)
Thread 2:
  mov          105   50    50    ← T2 reads 50 (counter still 50!)
  add          108   51    50    ← T2 adds 1, eax=51
  mov          113   51    51    ← T2 writes 51 to counter
  INTERRUPT → save T2 state
  restore T1 state (PC=108, eax=51)
Thread 1 resumes:
  mov          113   51    51    ← T1 writes 51 to counter (again!)
Final counter = 51  ← SHOULD BE 52. We lost one increment.
```

Two threads both incremented the counter, yet it only went from 50 to 51 instead of 52. When this same sequence of events happens across ten million iterations on two threads, millions of increments can get lost this way, and the final count can land anywhere between 10,000,000 and 20,000,000, depending entirely on how the scheduler happened to interleave the two threads on that particular run.

## 5. The Key Vocabulary

### 5.1 Race Condition

A race condition occurs when the result of a program depends on the timing and ordering of thread execution. In the example above, two threads race to update `counter`, whichever one writes last wins, and the loser's increment is lost. The outcome depends entirely on who happened to run when, so different runs produce different results. This is a race condition, named for the idea of threads racing each other to access shared data, where the outcome depends on who wins the race.

### 5.2 Critical Section

A critical section is a piece of code that accesses shared data and must not be executed by more than one thread at the same time.

```c
// THIS IS A CRITICAL SECTION:
counter = counter + 1;
// (the three-instruction sequence that reads, modifies, and writes counter)

// If two threads execute this simultaneously, a race condition results.
// Only one thread should be inside this code at any given time.
```

Any code that reads or modifies shared variables is a critical section, and the OS must provide some way to protect critical sections from being entered by more than one thread at once.

### 5.3 Mutual Exclusion

Mutual exclusion is the property that guarantees only one thread can be inside a critical section at any given time. If thread 1 enters a critical section and thread 2 then tries to enter the same one, thread 2 is blocked and must wait; only once thread 1 exits can thread 2 enter. Mutual exclusion is what prevents race conditions, and it is the solution to the critical section problem. The word comes from "mutually exclusive": if thread 1 is in, thread 2 is excluded, and if thread 2 is in, thread 1 is excluded. They cannot both be there at the same time.

### 5.4 Indeterminate Program

A program with one or more race conditions produces indeterminate results, meaning its output varies from run to run and cannot be predicted in advance. A deterministic program, which is what programmers normally assume they are writing, always produces the same output for the same input. An indeterminate program, caused by race conditions, produces different output for the same input on different runs, is extremely difficult to debug, and its bugs can be very rare and hard to reproduce. This is exactly why concurrency bugs are so dangerous: the program works correctly most of the time and fails only occasionally, precisely when the scheduler happens to interrupt at the wrong moment.

## 6. The Wish for Atomicity

### 6.1 What Atomicity Means

The core problem is that `counter = counter + 1` is really three instructions, and a thread can be interrupted between any of them. The ideal solution would be to make it a single, uninterruptible instruction instead.

Atomic means "as a unit," all or nothing. An atomic operation either completes entirely before any other thread can interfere with it, or it has not started at all; it cannot be interrupted halfway through.

```
Non-atomic (three instructions):
  read counter  ← can interrupt here
  add 1         ← can interrupt here
  write counter ← can interrupt here

Atomic (one uninterruptible step):
  add 1 to counter ← cannot be interrupted, happens all at once
```

If there were a single atomic instruction that performed all three steps, read, add, and write, together, the problem would disappear entirely: no thread could ever observe the counter in an intermediate state. Modern CPUs do provide special atomic instructions for simple cases like this, such as atomic increment, compare-and-swap (CAS), test-and-set, and fetch-and-add, all implemented directly in hardware.

### 6.2 Why We Cannot Just Add One Instruction for Everything

For simple cases, incrementing an integer, swapping a pointer, updating a flag, some CPUs do provide atomic instructions directly. But consider a more complex operation, such as updating a balanced binary tree: find the right node, update that node's value, rebalance the tree, which might touch fifty other nodes, and update parent and sibling pointers along the way. This can involve dozens of memory locations, many branches, and many instructions. There is no realistic way to build a single CPU instruction for every complex operation like this; that would require millions of special instructions, one for every data structure and algorithm ever invented, which is simply impossible.

### 6.3 Synchronization Primitives

Instead of giant atomic instructions for every possible operation, hardware gives programmers a few simple atomic tools, and the OS and programmers build safe concurrency mechanisms on top of those tools. These mechanisms are called synchronization primitives, the building blocks of concurrent programs.

The most important synchronization primitives are locks, also called mutexes, which allow only one thread into a critical section at a time:

```c
lock();
counter++;
unlock();
```

Condition variables allow threads to sleep and wake each other up. Locks solve the problem of not accessing shared data simultaneously, but a second problem remains: waiting until something happens. Condition variables solve that second problem, allowing threads to sleep, signal each other, and wake each other up as needed.

Semaphores generalize locks by allowing up to N threads in at once, where a plain mutex only ever allows one. Semaphores are especially useful for connection pools, producer-consumer systems, and general resource management. Locks, condition variables, and semaphores will each be covered in detail in later posts in this series.

## 7. The Second Problem: Waiting for Another Thread

The counter example above is about threads interfering with each other while accessing shared data. But there is a second, completely different type of concurrency problem: one thread waiting for another thread to do something.

Thread A might be computing a result that thread B needs before it can continue, so thread B must wait for thread A to finish. A thread might request disk I/O and be unable to continue until the disk responds, so it must sleep and wait to be woken up. A thread might be waiting for user input and be unable to continue until the user types something, so it must sleep until that input arrives.

This sleeping-and-waking interaction between threads requires a different mechanism than locks provide. Locks prevent simultaneous access to shared data, while condition variables allow threads to communicate, with one thread signaling another to wake it up. Condition variables will be covered in detail in a later post.

## 8. Why Is This Taught in an OS Course?

Threads might seem like purely a programming problem, so it is fair to ask why the OS itself cares about any of this. The answer is history: the OS was the first concurrent program.

The OS manages many processes, many files, and many devices, all simultaneously, and it has its own shared data structures to go along with that: the process list, the page tables, the file system metadata, the device queues. Multiple parts of the OS run concurrently and access these same shared structures.

Consider two processes that both call `write()` at the same time, both wanting to append data to the same file. Both have to find a free disk block, update the file's inode, update the file size, and write the data, and all of these operations touch shared OS data structures. If two threads, acting on behalf of these two processes, do this simultaneously without any synchronization, both might find the same "free" block, both might write to that same block, one write overwrites the other, and the file system ends up corrupted.

OS developers had to solve the concurrency problem from the very beginning, long before multi-threaded applications existed at all. Every major OS data structure, page tables, process lists, file system inodes, buffer caches, must be accessed with proper synchronization. Application programmers later inherited both the problem and the solutions that OS developers had already invented to deal with it.

## Conclusion

This post replaced the single-threaded model used throughout the rest of the series with a more realistic one. A single-threaded process has one program counter, one stack, and one set of registers, giving sequential, predictable behavior that is simple but cannot do more than one thing at a time. A multi-threaded process has multiple program counters, multiple stacks, and multiple register sets, all sharing the same code, heap, static data, and file descriptors, which lets it do multiple things simultaneously at the cost of far greater complexity, and far subtler ways for things to go wrong.

The core danger is captured in one line: shared data, combined with multiple threads, combined with a context switch that can land at any point in the code, produces race conditions, which produce indeterminate results, which produce bugs that appear randomly and are hard to reproduce. The vocabulary introduced along the way, critical section, race condition, indeterminate, mutual exclusion, atomic, names each part of this problem precisely. And the solution direction is now clear in outline: hardware provides a small set of simple atomic primitives, the OS builds synchronization primitives such as locks and condition variables on top of them, and programmers use those primitives to protect critical sections, so that properly synchronized code produces deterministic results even though it is running concurrently.

What has not been covered yet is how those synchronization primitives are actually used in practice: the real pthread API for creating and coordinating threads, and how locks and condition variables get applied to real code. That is the subject of the next post.

**Next:** [Interlude: The Thread API →](./13-interlude-thread-api.md)
