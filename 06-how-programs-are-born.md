# How Programs Are Born: The Process API

> Module 1 · Post 6 of 13

## 1. The fork() System Call

The OS needs a way to create new processes. The question is how.

One approach would be to give programmers a system call that creates a brand new, empty process from scratch and lets them fill it in. It is a simple idea, but UNIX took a completely different and more elegant approach.

The UNIX approach is to copy an existing process. The new process starts as an almost perfect duplicate of the one that created it. The system call that performs this copy is called `fork()`.

### 1.1 What fork() Actually Does

When a process calls `fork()`, the OS creates a new process, called the child, and copies almost everything from the parent into it: the entire address space, meaning the code, the static data, the heap, and the stack; all register values; the Program Counter, pointing to the exact same instruction; and all open file descriptors. The child is then given a new, unique pid of its own. The strangest part of the whole mechanism is what happens next: `fork()` returns twice, once inside the parent and once inside the child.

### 1.2 fork() Returns Twice, The Most Confusing Part

Look at this code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    printf("hello world (pid: %d)\n", getpid());

    int rc = fork();

    if (rc < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        printf("I am the child (pid: %d)\n", getpid());
    } else {
        printf("I am the parent of %d (pid: %d)\n", rc, getpid());
    }

    return 0;
}
```

When this runs, the output is:

```
hello world (pid: 29146)
I am the parent of 29147 (pid: 29146)
I am the child (pid: 29147)
```

Let's trace through exactly what happens.

Before `fork()` runs, only one process exists. Its pid is 29146, its Program Counter is sitting at the `fork()` call, and the first `printf()` has already executed, so `"hello world"` has been printed exactly once.

When `fork()` is called, the OS creates a child process that is a copy of the parent. Both processes now exist simultaneously, and both have their Program Counter pointing at the same place: the line right after `fork()`, where the return value is being assigned to `rc`. The one thing that differs between them is the value `fork()` actually returns:

```
Parent process:   rc = 29147   (the child's pid, a positive number)
Child process:    rc = 0       (always zero in the child)
```

After `fork()` returns, the parent has `rc = 29147` and falls into the final `else` branch, printing `"I am the parent of 29147 (pid: 29146)"`. The child has `rc = 0` and falls into the `else if (rc == 0)` branch, printing `"I am the child (pid: 29147)"`.

The `"hello world"` line printed only once, because `fork()` was called after it had already run. The child came into existence only after that line had executed, so it does not re-run `main()` from the beginning; it starts executing from exactly the point where `fork()` returned.

### 1.3 The Return Value Is the Signal

The different return value is how each process knows its own role:

```
rc < 0    → fork failed (not enough memory, too many processes, etc.)
rc == 0   → I am the child
rc > 0    → I am the parent, and rc is my child's pid
```

This is elegant: one system call and one line of code give you two processes, each one knowing exactly who it is.

### 1.4 The Non-Determinism Problem

Look at the output again:

```
hello world (pid: 29146)
I am the parent of 29147 (pid: 29146)    ← parent ran first
I am the child (pid: 29147)
```

But it could just as easily have come out like this:

```
hello world (pid: 29146)
I am the child (pid: 29147)              ← child ran first
I am the parent of 29147 (pid: 29146)
```

Both outputs are valid. Which process runs first after `fork()` is entirely up to the CPU scheduler. The parent might run first, or the child might run first, and you cannot know in advance which it will be. This is called non-determinism: the same program can produce different output orderings on different runs, purely because of scheduling decisions outside the program's control.

This non-determinism is not a bug; it is a fundamental property of concurrent systems. It does, however, create problems when a program depends on a specific ordering of events, which is exactly why `wait()` exists.

## 2. The wait() System Call

After `fork()`, the parent and child run independently. Sometimes the parent needs to know when the child is done before it can continue. Without any coordination, the parent might finish and exit before the child even starts printing, or it might try to use results that the child has not computed yet.

`wait()` gives the parent a way to pause itself until its child finishes.

### 2.1 What wait() Actually Does

When the parent calls `wait()`, one of two things happens. If the child is still running, the parent blocks: it stops executing and waits. Once the child exits, the OS wakes the parent back up, and `wait()` returns the pid of the child that exited. The parent then continues executing from there.

Look at this code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    printf("hello world (pid: %d)\n", getpid());
    int rc = fork();

    if (rc < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        printf("I am the child (pid: %d)\n", getpid());
    } else {
        int wc = wait(NULL);
        printf("I am the parent of %d (wc: %d) (pid: %d)\n",
               rc, wc, getpid());
    }
    return 0;
}
```

Output is now always:

```
hello world (pid: 29266)
I am the child (pid: 29267)
I am the parent of 29267 (wc: 29267) (pid: 29266)
```

The child always prints first now, and it is worth understanding why that holds regardless of which process the scheduler happens to run first. If the child runs first naturally, it prints its message, exits, and only then does the parent's `wait()` return so the parent can print. If the parent runs first instead, it reaches `wait(NULL)` before the child has printed anything and immediately blocks, entering the Sleeping state until the child exists to wake it. The child then gets the CPU, prints its message, and exits; the OS notices the exit, wakes the parent, and `wait()` returns so the parent can print.

Either way, the child always prints before the parent. The non-determinism from the previous section has been eliminated: the output is now fully deterministic.

### 2.2 What wait() Returns

`wait(NULL)` returns the pid of the child that exited. In the example above, that is 29267, the child's own pid, which is why the output shows `wc: 29267`: the variable `wc` is holding the return value of `wait()`.

You can also pass a pointer instead of `NULL` to get the child's exit status:

```c
int status;
int wc = wait(&status);
// status now contains the child's exit code
// WEXITSTATUS(status) extracts the actual exit code number
```

This is how the parent finds out whether the child succeeded or failed, and it is the entire reason the zombie state exists in the first place. The child's exit code has to be kept somewhere after the child dies so that the parent can collect it through `wait()`, and the child's PCB is exactly what serves that purpose until `wait()` is called.

## 3. The exec() System Call

### 3.1 The Problem exec() Solves

`fork()` creates a copy of the current process, but what if you want to run a completely different program, not a copy of yourself, but a new program loaded fresh from disk? That is the problem `exec()` solves.

### 3.2 What exec() Actually Does

When a process calls `exec("wc", args)`, the OS first finds the executable file `wc` on disk. It then completely replaces the calling process's address space: the code segment is overwritten with `wc`'s code, the static data is overwritten with `wc`'s static data, and the heap and stack are both reset. What `exec()` does not touch is just as important. The file descriptor table, `ofile[]`, survives untouched, along with the pid, the parent pointer, and any signal handlers the process had set up. Once the replacement is complete, the OS sets the Program Counter to `wc`'s entry point, which is `wc`'s own `main()`, and the process starts running `wc`.

The key point is that `exec()` does not create a new process. The same process continues running, only now it is running a completely different program: the old code, stack, and heap are gone, replaced entirely by the new program's own. This is also why a successful `exec()` never returns to the calling code. There is no code left to return to; it has been overwritten.

Look at this example:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    printf("hello world (pid: %d)\n", getpid());
    int rc = fork();

    if (rc < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child process
        printf("I am the child (pid: %d)\n", getpid());
        char *myargs[3];
        myargs[0] = strdup("wc");     // program to run
        myargs[1] = strdup("p3.c");   // argument: file to count
        myargs[2] = NULL;             // end of arguments
        execvp(myargs[0], myargs);    // run wc, replaces the entire program
        printf("this should never print");  // exec never returns
    } else {
        // parent process
        int wc = wait(NULL);
        printf("I am the parent of %d (wc: %d) (pid: %d)\n",
               rc, wc, getpid());
    }
    return 0;
}
```

Output:

```
hello world (pid: 29383)
I am the child (pid: 29384)
      29     107    1030 p3.c
I am the parent of 29384 (wc: 29384) (pid: 29383)
```

Here is what happened, step by step:

```
1. Main process prints "hello world"       (pid 29383)
2. fork() creates the child                (pid 29384)
3. Child prints "I am the child"
4. Child calls execvp("wc", ["wc", "p3.c", NULL])
5. OS replaces the child's entire address space with wc's code and data
6. wc runs, counts lines/words/bytes in p3.c, and prints the result
7. wc exits, so the child process (pid 29384) exits
8. Parent's wait() returns
9. Parent prints its message
```

The line `printf("this should never print")` never executes, because `execvp()` replaced the entire program. Once `exec()` succeeds, the old code simply does not exist anymore.

## 4. A Few C Concepts Before We Continue

Before going further into how the `execvp()` call above actually builds its argument array, it is worth pausing on a handful of C concepts this section leans on.

### 4.1 What Is a Pipe?

A pipe is a one-way channel for data: data goes in one end and comes out the other.

```
write end ──────────────────► read end
          data flows this way
```

Think of it like a physical pipe. You pour water in one end, and it comes out the other end; the pipe itself is just the channel in between. In the OS, a pipe is a small buffer, a region of memory maintained inside the kernel. One process writes into it, and another process reads from it. This concept will matter more once we look at how shells connect commands together, but it is worth having the picture in mind now.

### 4.2 What Is a String in C?

In C, a string is not a single, self-contained object. It is a sequence of characters stored in memory one after another, ending with a special character called the null terminator, `\0`.

So the string `"wc"` in memory looks like this:

```
Address    Value
───────    ─────
0x5000     'w'
0x5001     'c'
0x5002     '\0'    ← null terminator, marks the end
```

That is three bytes in total: two actual characters plus the one terminator.

### 4.3 What Is a Pointer to a String?

A `char *`, a character pointer, is simply a variable that holds the starting address of a string in memory. So `char *p = "wc";` means the following:

```
p = 0x5000    ← p holds the address where 'w' lives

Memory:
0x5000    'w'
0x5001    'c'
0x5002    '\0'
```

`p` does not hold the string itself. It holds only the address of where the string starts. That is what makes it a pointer.

### 4.4 What Is execvp()?

Suppose you want to run the `wc` program, which counts words, lines, and bytes in a file, from inside your own C code. Normally you would just type `wc p3.c` in the terminal. `execvp()` is the way to do that same thing from inside a C program: you are telling the OS to stop running your current program and start running `wc` instead, with a given set of arguments.

`execvp()` needs two things to do this: the name of the program to run, and an array of strings holding the program name followed by all of its arguments. That array has to look exactly like what you would have typed in the terminal:

```
Terminal:    wc     p3.c

Array:      [0]     [1]     [2]
            "wc"   "p3.c"   NULL
```

The `NULL` at the end acts as a signal that the list of arguments is finished. Without it, the OS would have no way of knowing where the array ends, and it would keep reading past it into whatever garbage happens to sit in memory next.

### 4.5 Where Does the String Literal "wc" Live?

When you write `"wc"` directly in your source code, the compiler places it into a special region of memory called the read-only data segment. This region is fixed at program start, it cannot be modified while the program runs, and it is shared across the whole program.

```
Read-only memory (set by the compiler, cannot change):
Address    Value
───────    ─────
0x5000     'w'
0x5001     'c'
0x5002     '\0'
```

If you try to modify this memory, for example by writing a different character into it, the program crashes with a segmentation fault. The OS actively protects this region.

### 4.6 What Does strdup() Do?

`strdup("wc")` does three things in sequence. First, it allocates fresh memory on the heap, large enough to hold the string:

```
Heap (new allocation):
Address    Value
───────    ─────
0x9000     ???    ← fresh memory, 3 bytes allocated
0x9001     ???
0x9002     ???
```

Second, it copies the characters from the original string into this new memory:

```
Heap (after copy):
Address    Value
───────    ─────
0x9000     'w'    ← copied from read-only memory
0x9001     'c'    ← copied
0x9002     '\0'   ← copied
```

Third, it returns the address of this new copy, `0x9000`. So now there are two separate copies of the same text living in two different places:

```
Read-only memory (original, untouched):
0x5000    'w'
0x5001    'c'
0x5002    '\0'

Heap (new writable copy):
0x9000    'w'
0x9001    'c'
0x9002    '\0'

myargs[0] = 0x9000    ← points to the heap copy
```

### 4.7 Why Not Just Write myargs[0] = "wc" Directly?

You could actually write `myargs[0] = "wc";` directly, and it would work perfectly fine for `execvp()`. The difference between the two approaches comes down to where the string ends up living. Writing `myargs[0] = "wc"` makes `myargs[0]` point to read-only memory at `0x5000`; you cannot modify that string afterward, but that is fine here, because `execvp()` only ever reads it. Writing `myargs[0] = strdup("wc")` instead makes `myargs[0]` point to a writable heap copy at `0x9000`, so you could modify the string later if you needed to, and it still works just as well for `execvp()`.

`strdup()` is used here simply as good practice. It builds the habit of keeping argument arrays in writable memory, because in real programs you often do need to construct or modify those strings dynamically. That flexibility is not strictly needed in this particular example, but it comes up often enough in the general case that it is worth forming the habit early.

### 4.8 The Real Problem With Read-Only Memory

String literals like `"wc"` are stored in read-only memory, so trying to modify them crashes the program:

```c
char *p = "wc";   // points to read-only memory
p[0] = 'l';       // crashes: segmentation fault, writing to read-only memory
```

The OS protects read-only memory at the hardware level, so any write attempt like this one causes an immediate crash. `strdup()` gives you a way around this, because it hands back writable memory instead:

```c
char *p = strdup("wc");   // points to writable heap memory
p[0] = 'l';                // works fine
p[1] = 's';                // works fine
// p now contains "ls" instead of "wc"
```

You now have a writable copy that you fully control. You can change any character in it, and you can build strings dynamically at runtime.

### 4.9 Now the Array Makes Complete Sense

`char *myargs[3];` creates an array of three slots, and each slot holds a `char *`, a pointer to a string. Right now all three slots contain garbage values, since nothing has been assigned to them yet.

The line `myargs[0] = strdup("wc");` makes `strdup` create a heap copy of `"wc"` somewhere, say at address `0x9000`, and that address is what gets stored in slot 0:

```
myargs[0] = 0x9000   →   memory at 0x9000 contains: 'w' 'c' '\0'
myargs[1] = garbage
myargs[2] = garbage
```

The next line, `myargs[1] = strdup("p3.c");`, does the same thing for `"p3.c"`, placing its heap copy at, say, address `0x9010`:

```
myargs[0] = 0x9000   →   'w' 'c' '\0'
myargs[1] = 0x9010   →   'p' '3' '.' 'c' '\0'
myargs[2] = garbage
```

Finally, `myargs[2] = NULL;` sets the last slot to zero. `NULL` here simply means "nothing here," and it is the signal that tells `execvp()` to stop reading: the argument list ends at this slot.

```
myargs[0] = 0x9000     →   'w' 'c' '\0'
myargs[1] = 0x9010     →   'p' '3' '.' 'c' '\0'
myargs[2] = NULL (0)   →   nothing, end of list
```

So `myargs` itself never holds the strings. It holds pointers to the strings, which actually live elsewhere in heap memory. Each slot in the array is just a starting address, and the CPU follows that address to find the actual characters when it needs them.

### 4.10 The Full Picture in One Diagram

```
myargs array (3 slots, each holds an address):
┌──────────┬──────────┬──────────┐
│  0x9000  │  0x9010  │   NULL   │
└──────────┴──────────┴──────────┘
     │            │
     ▼            ▼
  "wc\0"      "p3.c\0"
  (heap)       (heap)
```

When `execvp()` reads this array, it walks through it slot by slot: slot 0 sends it to address `0x9000`, where it reads `"wc"`, the program name; slot 1 sends it to address `0x9010`, where it reads `"p3.c"`, the first argument; and slot 2 is `NULL`, telling it to stop, since the list is done.

`execvp()` then finds the `wc` program on disk and runs it with the argument `p3.c`, exactly as if you had typed `wc p3.c` in the terminal yourself.

## 5. Why This Design? The Separation of fork() and exec()

### 5.1 The Problem Being Solved

Imagine you are building a shell, the program that runs when you open a terminal. Its job is simple to describe: show a prompt, wait for the user to type a command, run that command, show the prompt again, and repeat forever. The hard part is the third step. How does the shell actually run a command?

### 5.2 The Naive Approach, One System Call

A natural first idea is to imagine a single system call, something like `spawn("wc", "p3.c")`, that just creates a new process running `wc` directly. Many operating systems actually do exactly this; Windows, for instance, has `CreateProcess()`, which creates a new process and starts a program running in one single step.

The problem with this approach is timing. By the time the process exists, the program is already running, so you never get a chance to set anything up beforehand. The new process is already going, and the window in which you could have configured it has already closed.

### 5.3 What UNIX Does Instead, Two Separate Steps

UNIX splits process creation into two completely separate operations:

```
Step 1: fork()   → create the new process (an exact copy of the parent)
Step 2: exec()   → transform that process into a different program
```

Between these two steps there is a gap: a moment in time where the child process exists but has not yet become the new program. During this gap, the child is free to do any setup work it needs. This gap is everything. It is the reason UNIX shells are as powerful as they are.

### 5.4 Scenario 1, Running a Simple Command

Consider what happens when you type `wc p3.c` in the terminal.

```
Shell (parent, pid=100)
│
├── fork()
│   Creates child (pid=101, an exact copy of the shell)
│
│   Child (pid=101):
│   ├── [gap: no setup needed this time]
│   └── exec("wc", ["wc", "p3.c", NULL])
│       Child becomes wc
│       wc runs, prints its output to the terminal
│       wc exits
│
└── wait()
    Shell waits for pid 101 to finish
    Shell prints the prompt again
```

This is the simple case: no setup was needed in the gap, and it works exactly as you would expect.

### 5.5 Scenario 2, Output Redirection

Now consider `wc p3.c > newfile.txt`. The `>` symbol means: do not print to the screen, write the output to `newfile.txt` instead. The catch is that `wc` has no idea any of this redirection is happening. `wc` always writes to fd 1, standard output, and it cannot be told to write somewhere else, since it is a separate program whose code you cannot modify.

So how does the shell make `wc` write to a file instead of the screen? The answer lies entirely in the gap between `fork()` and `exec()`: the child manipulates its own file descriptors before it becomes `wc`.

#### 5.5.1 Watching the File Descriptors Change, Step by Step

Starting state, the shell is running:

```
CPU REGISTERS               ofile[] for the shell process
──────────────              ─────────────────────────────
PC = shell code             [0] → stdin  (keyboard)
                             [1] → stdout (terminal screen)
                             [2] → stderr (terminal screen)
```

`fork()` runs, and the child is created. The child is an exact copy of the shell, so it inherits everything, including the file descriptor table:

```
Parent (shell):                     Child (pid 101):
ofile[]                             ofile[]  ← exact copy
[0] → stdin  (keyboard)             [0] → stdin  (keyboard)
[1] → stdout (terminal)             [1] → stdout (terminal)
[2] → stderr (terminal)             [2] → stderr (terminal)
```

At this point, both the parent and the child have fd 1 pointing to the terminal screen.

The child now enters the gap, and its setup begins. It is still running shell code; `exec()` has not been called yet, so it is free to do anything it wants to its own process state.

The child runs `close(STDOUT_FILENO)`, which closes fd 1 in the child's own `ofile[]` table. Only the child's table is affected; the parent's fd 1 still points to the terminal.

```
Parent (shell):                     Child (pid 101):
ofile[]                             ofile[]
[0] → stdin  (keyboard)             [0] → stdin  (keyboard)
[1] → stdout (terminal)             [1] → CLOSED  ← fd 1 is now free
[2] → stderr (terminal)             [2] → stderr (terminal)
```

The child then runs `open("./newfile.txt", O_CREAT | O_WRONLY | O_TRUNC)`. The OS opens `newfile.txt` and needs to assign it a file descriptor number, so it uses the lowest number that is currently free. fd 0 is taken, fd 1 is free, and fd 2 is taken, so the OS assigns fd 1 to `newfile.txt`.

```
Parent (shell):                     Child (pid 101):
ofile[]                             ofile[]
[0] → stdin  (keyboard)             [0] → stdin  (keyboard)
[1] → stdout (terminal)             [1] → newfile.txt  ← fd 1 now points here
[2] → stderr (terminal)             [2] → stderr (terminal)
```

The child's fd 1 now points to a file instead of the terminal, and the parent remains completely unaffected by any of this.

The child then runs `execvp("wc", myargs)`. It transforms into `wc`: `wc`'s code and data replace the child's memory entirely. The file descriptor table, however, survives `exec()`. `ofile[]` is not reset; it stays exactly as the child left it.

```
Child (pid 101) is now wc:
ofile[]
[0] → stdin  (keyboard)
[1] → newfile.txt   ← inherited from the setup done in the gap
[2] → stderr (terminal)
```

`wc` starts running and calls `printf()` to print its results. `printf()` writes to fd 1, and fd 1 now points to `newfile.txt`, so all of its output goes into the file. `wc` has no idea any of this happened. It simply does what it always does, writing to fd 1; it was the child's setup during the gap that made fd 1 point somewhere different. That is the entire trick.

Once `wc` finishes, the parent continues: `wc` exits, the parent's `wait()` returns, and the shell prints its prompt again. `newfile.txt` now contains `wc`'s output, and the terminal screen shows nothing from `wc` at all, because fd 1 never pointed there during its execution.

### 5.6 Why a Single spawn() Call Cannot Do This

Now imagine trying to build this with a single call like `spawn("wc", "p3.c", redirect_to="newfile.txt")`. Three problems appear immediately. The system call would need a separate parameter for every possible setup operation someone might want, redirecting stdout, redirecting stdin, redirecting stderr, setting environment variables, changing the working directory, closing specific file descriptors, setting up pipes, and the parameter list would grow enormous. Every new feature would require modifying the system call itself, which means modifying the OS every time a new capability is needed. And complex combinations of these operations would become awkward or outright impossible without an ever-growing set of specialized variants.

The UNIX approach needs none of this, because the gap between `fork()` and `exec()` handles everything on its own. Redirecting stdout is `close(1)`, `open("file")`, then `exec()`. Redirecting stdin is the same idea with fd 0. Changing directory is `chdir("/new/path")` before `exec()`. Setting an environment variable is `setenv("VAR", "value")` before `exec()`. Closing a descriptor is just `close(fd)` before `exec()`. Doing all of these together is simply doing all of them, in whatever order, and then calling `exec()` once at the end.

The shell uses this exact same pair, `fork()` and `exec()`, for every one of these cases. No new system calls are needed, and no changes to the OS are needed either. The gap absorbs every possible setup operation on its own.

## 6. kill() and Signals

### 6.1 What Is a Signal?

A signal is a software interrupt: an asynchronous notification sent to a process. The process does not have to be checking for it in any way; the OS can deliver it at any moment, interrupting whatever the process happens to be doing at the time.

It helps to think of it like a tap on the shoulder. The process is busy doing its own work, and then, without warning, it is interrupted and must stop to handle whatever arrived.

### 6.2 Who Sends Signals?

Three different sources can send a signal to a process: the user, by pressing something like Ctrl+C in the terminal; another process, by calling `kill(pid, signal)`; or the OS itself, when something goes wrong internally, such as a division by zero or a memory violation.

### 6.3 What Happens When a Signal Arrives?

When a signal arrives at a process, one of three things happens: by default, for most signals, the process dies immediately; if the process has set up a handler function, that handler runs instead; or, if the process has chosen to ignore that particular signal, nothing happens at all. Which of these actually occurs depends on the specific signal type and on whether the process has registered a handler for it.

### 6.4 The Default Behavior, the Process Just Dies

If a process has not done anything special to prepare, most signals simply kill it outright.

```
OS sends SIGTERM to process 1042
Process 1042 was in the middle of some calculation
OS interrupts it
Process 1042 dies immediately
```

No warning is given, no cleanup happens, and the process is simply gone.

### 6.5 kill(), Sending a Signal

The `kill()` system call, and the `kill` command that wraps it, is how one process asks the OS to send a signal to another process.

```c
kill(1042, SIGKILL);
```

| Command | Effect |
|---|---|
| `kill 1042` | sends SIGTERM (15) to process 1042; a polite request to stop |
| `kill -9 1042` | sends SIGKILL (9) to process 1042; forces termination |
| `kill -SIGSTOP 1042` | sends SIGSTOP to process 1042; freezes it |
| `kill -SIGCONT 1042` | sends SIGCONT to process 1042; resumes it |

Calling `kill(1042, SIGKILL)` directly sends the SIGKILL signal to process 1042, and the OS terminates it immediately, with no chance for cleanup.

### 6.6 Signal Handlers, the Process Catches the Signal

A process can tell the OS: when signal X arrives, do not kill me immediately, run this function of mine first. That function is called a signal handler:

```c
void my_handler(int sig) {
    printf("caught signal %d, cleaning up...\n", sig);
    exit(0);
}
```

This is just an ordinary C function that takes one argument, the number of the signal that was received. Inside it, you can do essentially anything: save your work, close open files, print a final message, and then exit cleanly on your own terms.

The OS does not know about this function until you register it:

```c
signal(SIGTERM, my_handler);
```

This line tells the OS that when SIGTERM arrives for this process, it should not kill it immediately, and should run `my_handler()` first instead.

Before a handler is registered, the sequence when SIGTERM arrives is short:

```
1. Process is running normally
2. SIGTERM arrives
3. OS kills the process immediately
```

Once the handler has been registered, the sequence looks quite different:

```
1. Process registers signal(SIGTERM, my_handler)
2. Process continues running normally
3. SIGTERM arrives
4. OS interrupts the process
5. OS runs my_handler() instead of killing it immediately
6. Inside my_handler: printf() runs, then exit(0) runs
7. Process exits cleanly, on its own terms
```

The process got a genuine chance to clean up before dying. That is the entire point of a signal handler.

### 6.7 Common Signals and What They Do by Default

| Signal | Number | Default action | Common cause |
|---|---|---|---|
| SIGTERM | 15 | Kill the process | `kill` command; a polite request to stop |
| SIGKILL | 9 | Kill immediately | `kill -9`; a forced stop |
| SIGINT | 2 | Kill the process | user presses Ctrl+C |
| SIGSTOP | 19 | Freeze the process | user presses Ctrl+Z |
| SIGCONT | 18 | Resume a frozen process | `fg` or `bg` command in the shell |
| SIGCHLD | 17 | Notify the parent | a child process exited |

### 6.8 Ctrl+C and Ctrl+Z, Signals You Use Every Day

Pressing Ctrl+C in the terminal sends SIGINT, signal 2, to the running process. Its default behavior is to kill the process, though many programs catch it deliberately so they can clean up before dying, since the process is always free to define its own handler and decide what to do.

Pressing Ctrl+Z sends SIGSTOP, signal 19, to the running process instead. The process freezes completely: it stays in memory with all of its data intact, but it is not scheduled at all, and the OS simply ignores it until further notice.

Typing `fg` or `bg` afterward has the shell send SIGCONT, signal 18, to that frozen process. The process unfreezes, goes back into the ready queue, and continues running from exactly the point where it was frozen.

SIGSTOP cannot be caught or ignored, and neither can SIGKILL; the OS freezes or kills the process regardless of what it is doing at the time. This is the real distinction that matters among all the signals covered here: some of them, like SIGINT and SIGTERM, are polite requests that the process can catch and respond to on its own terms, while others, like SIGKILL and SIGSTOP, are forced and cannot be intercepted at all.

## 7. Inspecting Processes from the Outside

### 7.1 getpid() and getppid()

```c
pid_t my_pid    = getpid();   // returns this process's pid
pid_t my_parent = getppid();  // returns the parent process's pid
```

Every process knows its own pid and its parent's pid at any time, since both are stored directly in its PCB.

### 7.2 ps and top, Observing Processes

`ps aux` takes a snapshot of every running process right now and prints it as a table, showing each process's pid, the user who owns it, its CPU usage, its memory usage, and its current state, where R means running, S means sleeping, Z means zombie, and T means stopped.

`ps` by itself only shows processes belonging to you in the current terminal. The `aux` flags change that: `a` shows processes from all users, not just your own; `u` displays the detailed, user-oriented columns; and `x` includes processes that are not attached to any terminal at all. Put together, `aux` means show everything, from every user, in full detail.

`top` does the same job as `ps aux`, except instead of taking a single snapshot, it keeps refreshing automatically, usually every two or three seconds, giving you a live view of processes and resource usage as they change.

### 7.3 /proc, the Window Into the OS

On Linux, every process has a special directory at `/proc/pid/` containing detailed, live information about that process. If Chrome is running with pid 1042, for instance, that directory is `/proc/1042/`, and it exists for exactly as long as the process is alive; the moment the process dies, the directory disappears with it.

```
/proc/1042/status    ← state, memory, uid, pid, ppid
/proc/1042/fd/       ← all currently open file descriptors
/proc/1042/maps      ← the memory map, where each region of the address space sits
/proc/1042/cmdline   ← the exact command that started this process
```

None of this is a real directory sitting on your disk; there are no actual files stored on your hard drive at `/proc/1042/`. When you try to read a file from inside it, the OS intercepts the read request and generates the data on the spot, directly from that process's PCB and the OS's own internal memory. It is a live window into the OS's internal data structures, disguised as an ordinary file system, which is exactly why it is called a virtual file system: it looks like files and directories, but what you are actually reading is the OS's internal state, presented in a familiar shape.

## Conclusion

Together, `fork()`, `exec()`, `wait()`, and `kill()` are the entire process API that UNIX gives you, and everything a shell does, launching commands, redirecting output, managing background jobs, comes down to combinations of these four calls plus a bit of file descriptor bookkeeping in the gap between `fork()` and `exec()`. You have also seen how a process can be inspected entirely from the outside, through `ps`, `top`, and the live window that `/proc` opens into the OS's own data structures.

There is still a layer underneath all of this that has been taken for granted so far. Every time a function is called, whether it is `main()` calling `fork()` or `wc` calling `printf()`, the CPU has to do real work: save a return address, set up space for local variables, and keep straight where in memory the current function's world begins and ends. That mechanism, the stack frame, and the registers that drive it, is what the next post takes apart.

**Next:** [Inside the CPU: Stack Frames and Registers →](./07-inside-the-cpu.md)
