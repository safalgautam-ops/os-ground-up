# os-ground-up

**A free, from-scratch operating systems course covering what actually happens when you run a program, traced from `./a.out` down to the CPU.**

![Posts](https://img.shields.io/badge/posts-13-blue)
![Read time](https://img.shields.io/badge/read%20time-~4%20hrs-informational)
![License](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey)
![Status](https://img.shields.io/badge/status-in%20progress-yellow)

Most computer science students can define a process. They can recite what a semaphore does. Yet when asked what actually happens between running a program and seeing it appear on screen, most go quiet.

This series is written to close that gap. It presents a single continuous account, traced from one keypress down through processes, memory, the CPU, scheduling, and concurrency. Nothing is assumed beyond basic C and a general understanding that a CPU executes instructions.

**[Start here](./01-structures-complete-guide.md)**

---

## Roadmap

| # | Post | Module |
|---|------|--------|
| 1 | [Structures: Complete Guide](./01-structures-complete-guide.md) | Module 0: C Foundations |
| 2 | [Pointers and Arrays in C](./02-pointers-and-arrays.md) | Module 0: C Foundations |
| 3 | [Null, Dangling, Void, and Wild Pointers](./03-pointer-types.md) | Module 0: C Foundations |
| 4 | [The Process - Most Important Abstraction in Your Computer](./04-the-process.md) | Module 1: What Is a Process? |
| 5 | [How the OS Never Forgets, Inside the PCB](./05-process-control-block.md) | Module 1: What Is a Process? |
| 6 | [How Programs Are Born: The Process API](./06-how-programs-are-born.md) | Module 1: What Is a Process? |
| 7 | [Inside the CPU: Stack Frames and Registers](./07-inside-the-cpu.md) | Module 1: What Is a Process? |
| 8 | [The Illusion of Private Memory](./08-illusion-of-private-memory.md) | Module 2: Memory |
| 9 | [From CPU to Disk](./09-from-cpu-to-disk.md) | Module 3: The Big Picture |
| 10 | [CPU Scheduling: Introduction](./10-cpu-scheduling-introduction.md) | Module 4: Scheduling |
| 11 | [The Scheduler That Learns: MLFQ](./11-the-scheduler-that-learns.md) | Module 4: Scheduling |
| 12 | [Concurrency: An Introduction](./12-concurrency-introduction.md) | Module 5: Concurrency |
| 13 | [Interlude: Thread API](./13-interlude-thread-api.md) | Module 5: Concurrency |

---

## Source and credit

These posts are rewritten in plain language, for beginners, from two books that I read and worked through:

- **The operating systems lessons (Modules 1 to 5, posts 4 to 13)** are based on *Operating Systems: Three Easy Pieces* (OSTEP) by Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau.
- **The C foundations (Module 0, posts 1 to 3)** are based on *The C Programming Language*, 2nd Edition, by Brian W. Kernighan and Dennis M. Ritchie.

The topics and core ideas come from those books; the wording, walkthroughs and diagrams here are my own rewriting for readers who are new to the subject.

This series is not a replacement for either book. For the full and authoritative treatment, read the originals. OSTEP is free to read online at [pages.cs.wisc.edu/~remzi/OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/), and Kernighan and Ritchie is available in print. Any mistakes in these posts are mine, not the authors.

---

## Who this is for

This series is intended for students who know the vocabulary but do not yet have the underlying mental model. If you have memorized that a process is a running program without ever seeing what that means in memory, this material is written for you.

**Prerequisites:** basic C syntax. Nothing further. No assembly and no prior operating systems coursework are required.

**No compiler available?** [Compiler Explorer](https://godbolt.org) and [Replit](https://replit.com) will run everything here directly in a browser.

---

## How to read it

The posts are meant to be read in order. Each one closes on a question that the following post answers, and later modules rely on vocabulary established earlier. Beginning at Module 4 will technically work, but you will end up reconstructing context that the series has already provided.

Every post opens with a breadcrumb such as `Module 1 · Post 4 of 13` so your position in the series is always clear, and closes with a short recap and a direct link forward.

---

## Using this in your own notes or classes

Please do, as that is what it is for. You are welcome to take notes from it, quote it, use it in a study group, build a class around it, or translate it. The only request is that you credit me and link back to this repository so that others can find the complete series.

---

## Contributing

If you find something unclear, incorrect, or assuming knowledge it should not, that is the most useful feedback you can offer. Please open an issue. Corrections, improved explanations, and reports of the form "this paragraph lost me" are all genuinely welcome.

Discussions are open for questions on any individual post.

---

## License

Written content is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), and code snippets under the [MIT License](https://opensource.org/licenses/MIT). See [LICENSE.md](./LICENSE.md).

Suggested credit: *"os-ground-up" by Safal Gautam, CC BY 4.0.*
