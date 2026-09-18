# CPU Scheduling: Introduction

> Module 4 · Post 10 of 13

## 1. Picking Up the Thread

The previous post connected everything covered so far, processes, memory translation, and I/O, into one picture. The OS takes messy, limited, shared hardware and makes it feel clean, unlimited, and private to every program running on top of it. This post picks up one thread from that picture and studies it in depth: how does the OS decide which process gets the CPU at any given moment?

Every time you type a character in your text editor and it appears instantly, every time you switch between applications and they respond immediately, every time a background download continues while you browse, the OS's scheduler is making decisions behind the scenes. By the end of this post you will understand the core scheduling algorithms that real operating systems have used and still use today: FIFO, Shortest Job First, Shortest Time to Completion First, and Round Robin.

## 2. What Is Scheduling and Why It Exists

The CPU can only execute one instruction at a time. On a single core machine, only one process is literally running at any given nanosecond.

But your computer has hundreds of processes, and all of them want CPU time. The OS must decide, constantly, thousands of times per second, which one gets to run right now. This decision-making system is called the scheduler, and the rules it follows are called the scheduling policy or scheduling discipline.

The scheduler is a pure policy component. It does not do the actual switching; that is the context switch mechanism covered in earlier posts. The scheduler only answers one question: which process should run next? Everything in this post is about answering that question in different ways.

## 3. The Workload: What We Are Scheduling

Before designing a scheduler, it helps to think clearly about what exactly is being scheduled. The collection of processes waiting to run is called the workload.

To understand scheduling algorithms clearly, it is useful to start with simplified assumptions about the workload and relax them one at a time. This post starts with five assumptions:

1. Each job runs for the same amount of time.
2. All jobs arrive at the same time.
3. Once started, each job runs to completion.
4. Jobs only use the CPU; none of them perform I/O.
5. The runtime of each job is known in advance.

These assumptions are unrealistic. Real programs have different lengths, arrive at different times, get preempted, perform I/O, and their runtimes are unknown to the OS in advance. Starting with these assumptions, though, makes it possible to understand each algorithm clearly before adding that complexity back in. Each algorithm below drops one of these assumptions, and by the end of the post, all five will be gone.

## 4. Scheduling Metrics: How We Measure "Good"

Before comparing algorithms, it helps to agree on what "good" means. This requires metrics, ways of measuring performance.

### 4.1 Turnaround Time

The turnaround time of a job is defined as the time at which the job completes minus the time at which the job arrived in the system.

```
T_turnaround = T_completion − T_arrival
```

If a job arrives at time 0 and finishes at time 30, its turnaround time is 30.

Turnaround time is a performance metric. Lower is better. A scheduling algorithm that minimizes average turnaround time gets jobs done faster overall.

### 4.2 Response Time

Response time measures how long a job waits before it first gets any CPU time at all. It is defined as the time of first run minus the time of arrival.

```
T_response = T_firstrun − T_arrival
```

If a job arrives at time 0 and first runs at time 10, its response time is 10.

Response time is an interactivity metric. Lower is better. A scheduling algorithm that minimizes response time makes the system feel fast and responsive to the user.

### 4.3 Why Turnaround and Response Time Conflict

Consider two opposite scheduling philosophies. In the first, the scheduler focuses on finishing jobs quickly. It might pick short jobs first, so that each one finishes fast once it starts. The problem is that a long job sitting at the back of the queue might have to wait a very long time before it even starts. Jobs finish quickly once they begin, which is good for turnaround time, but some jobs wait a long time just to begin, which is bad for response time.

In the second philosophy, the scheduler focuses on starting everything quickly. It gives every job a small amount of CPU time right away, the way round robin does, so that every job starts almost immediately. Everyone starts quickly, which is good for response time, but each job now takes longer to fully finish, since the CPU keeps switching between jobs instead of running one to completion before moving to the next. That is bad for turnaround time.

### 4.4 Fairness

Fairness measures whether all processes get a reasonable share of the CPU. A scheduler that makes one process wait forever while others run constantly is unfair, even if the overall turnaround time looks good.

Performance and fairness are often at odds. Better performance generally means running shorter jobs first, which makes longer jobs wait, which is unfair. Better fairness generally means giving every job an equal share of time, which lengthens the average turnaround time for everyone. There is no single correct balance; different systems prioritize different metrics depending on their purpose.

## 5. FIFO: First In, First Out

The most basic scheduling algorithm is First In, First Out (FIFO), sometimes called First Come, First Served (FCFS). It is simple and easy to implement: run jobs in the order they arrive. Whoever arrives first runs first, and when that job finishes, the next one in line runs.

Suppose three jobs, A, B, and C, arrive in the system at roughly the same time (T_arrival = 0). Because FIFO has to put some job first, assume that although they all arrived essentially simultaneously, A arrived just a hair before B, which arrived just a hair before C. Assume also that each job runs for 10 seconds.

```
Jobs: A, B, C all arrive at time 0
      Each runs for 10 seconds

Timeline:
│  A (10s)  │  B (10s)  │  C (10s)  │
0           10          20          30
```

Calculating turnaround times:

```
A: finished at 10, arrived at 0 → turnaround = 10 - 0 = 10
B: finished at 20, arrived at 0 → turnaround = 20 - 0 = 20
C: finished at 30, arrived at 0 → turnaround = 30 - 0 = 30

Average turnaround = (10 + 20 + 30) ÷ 3 = 20 seconds
```

This looks fine. But now relax assumption 1: jobs are no longer the same length.

### 5.1 The Convoy Effect

Suppose instead:

```
Job A: arrives at 0, runs for 100 seconds
Job B: arrives at 0, runs for 10 seconds
Job C: arrives at 0, runs for 10 seconds
```

FIFO runs them in arrival order, and A arrived first, so A goes first:

```
│         A (100s)         │  B (10s)  │  C (10s)  │
0                         100         110         120
```

Turnaround times:

```
A: 100 - 0 = 100
B: 110 - 0 = 110
C: 120 - 0 = 120

Average = (100 + 110 + 120) ÷ 3 = 110 seconds
```

B and C each only need 10 seconds of work, but they have to wait 100 seconds because A is ahead of them. Their turnaround times, 110 and 120, are ten times worse than their actual work time.

This is called the convoy effect. Short jobs get stuck behind one long job and suffer enormously. It is much like standing in a grocery checkout line where the person in front has three full carts and you have one item.

## 6. SJF: Shortest Job First

The fix for the convoy effect is straightforward: run the shortest jobs first. The SJF rule is to always run the job with the shortest runtime next, and when that finishes, pick the next shortest remaining job.

Using the same three jobs but now with SJF:

```
Jobs: A (100s), B (10s), C (10s) - all arrive at time 0
SJF runs B and C first (both 10s), then A (100s)

│  B (10s)  │  C (10s)  │         A (100s)         │
0           10          20                         120
```

Turnaround times:

```
B: 10 - 0 = 10
C: 20 - 0 = 20
A: 120 - 0 = 120

Average = (10 + 20 + 120) ÷ 3 = 50 seconds
```

Average turnaround dropped from 110 to 50 seconds, more than double the performance, just by running the shorter jobs first.

### 6.1 Where SJF Breaks Down: Late Arrivals

Now relax assumption 2: jobs no longer arrive at the same time.

```
Job A: arrives at time 0,  runs for 100 seconds
Job B: arrives at time 10, runs for 10 seconds
Job C: arrives at time 10, runs for 10 seconds
```

At time 0, only A is available, so SJF must run A; there is no other choice. At time 10, B and C arrive, but A is already running. SJF as defined here is non-preemptive: once a job starts, it runs to completion. So B and C must wait for A to finish.

```
│              A (100s)              │  B  │  C  │
0                                  100   110   120
```

Turnaround times:

```
A: 100 - 0  = 100
B: 110 - 10 = 100
C: 120 - 10 = 110

Average = (100 + 100 + 110) ÷ 3 = 103.33 seconds
```

Even though B and C are short jobs, they suffer because A started before they arrived and cannot be interrupted. The convoy problem reappears, this time caused by late arrivals rather than job order.

## 7. STCF: Shortest Time-to-Completion First

The fix is to let the scheduler preempt a running job when a shorter one arrives. This requires relaxing assumption 3, that jobs must run to completion once started.

Preemption means the scheduler can stop a currently running job mid-execution and switch to a different job. The stopped job goes back to the ready queue and resumes later.

The STCF rule: whenever a new job arrives, compare its remaining time to the remaining time of the currently running job, and run whichever has less time remaining.

Applying STCF to the same three jobs:

```
Job A: arrives at 0,  needs 100s
Job B: arrives at 10, needs 10s
Job C: arrives at 10, needs 10s

Time 0:   Only A available → run A
Time 10:  B and C arrive
          A has 90s remaining, B needs 10s, C needs 10s
          B is shortest → preempt A, run B
Time 20:  B finishes
          A has 90s remaining, C needs 10s
          C is shortest → run C
Time 30:  C finishes
          Only A remains → run A
Time 120: A finishes
```

```
│ A │    B    │    C    │              A              │
0   10        20        30                           120
```

Turnaround times:

```
A: 120 - 0  = 120
B: 20  - 10 = 10
C: 30  - 10 = 20

Average = (120 + 10 + 20) ÷ 3 = 50 seconds
```

Compared to non-preemptive SJF with late arrivals (103.33 seconds), STCF drops the average turnaround time to 50 seconds. B and C get excellent turnaround times because they preempt A the moment they arrive. STCF is optimal for turnaround time when jobs can arrive at any time: when a short job arrives, it immediately cuts in front of the long job.

### 7.1 The New Problem STCF Creates

STCF is excellent for turnaround time, but think about response time instead. In the example above, C waits from time 10 until time 20 before it first runs. If ten short jobs all arrived at time 10, the last one would wait a long time before seeing any CPU time at all.

STCF does not eliminate the convoy problem so much as shift it. Instead of long jobs blocking short ones, short jobs now block each other, and the last short job in line still waits.

There is also a more fundamental issue: STCF assumes the scheduler knows job runtimes in advance, which was assumption 5. In reality, the OS has no idea how long a process will run, so true STCF cannot actually be implemented in a general-purpose system.

## 8. Round Robin: Fair Time Sharing

### 8.1 Why Response Time Needs Its Own Algorithm

Up to this point, the focus has been on turnaround time. But for interactive systems, where users are sitting at a terminal waiting for responses, turnaround time is the wrong metric to optimize.

Consider typing a command and wanting to see output quickly. You do not care when the job "finishes" in some batch sense; you care when it first responds to you. That is response time.

With SJF and three jobs arriving at the same time:

```
Jobs A, B, C all arrive at time 0, each needs 5 seconds
SJF runs them sequentially: A → B → C

A response time: 0  - 0 = 0   (runs immediately)
B response time: 5  - 0 = 5   (waits for A)
C response time: 10 - 0 = 10  (waits for A and B)

Average response time = (0 + 5 + 10) ÷ 3 = 5 seconds
```

C waits 10 seconds just to get any CPU time at all. If C is a user typing at a keyboard, that delay is unacceptable. A different algorithm is needed, one that gets to every job quickly rather than one that finishes jobs quickly.

### 8.2 The Round Robin Rule

Round Robin (RR) solves the response time problem with one simple idea: do not run any job to completion. Instead, give every job a small fixed slice of time, then switch to the next job, cycling through all jobs repeatedly. That small fixed time slice is called a time quantum or scheduling quantum.

```
Jobs A, B, C all arrive at time 0, each needs 5 seconds
RR with time quantum = 1 second

Timeline:
│A│B│C│A│B│C│A│B│C│A│B│C│A│B│C│
0 1 2 3 4 5 6 7 8 9 ...       15
```

Every second, the scheduler switches to the next job, so every job gets a turn every 3 seconds.

```
A: first runs at time 0 → response time = 0
B: first runs at time 1 → response time = 1
C: first runs at time 2 → response time = 2

Average response time = (0 + 1 + 2) ÷ 3 = 1 second
```

Compared to SJF's average response time of 5 seconds, RR's average response time of 1 second is dramatically better.

### 8.3 The Cost: RR Is Terrible for Turnaround Time

Now calculate turnaround time for RR with the same three jobs:

```
A finishes at time 13
B finishes at time 14
C finishes at time 15

Average turnaround = (13 + 14 + 15) ÷ 3 = 14 seconds
```

Compare this to SJF's turnaround time with the same jobs, which was A at 5, B at 10, C at 15, for an average of 10 seconds. RR is worse than SJF for turnaround, and considerably so.

The reason is that RR stretches every job out as long as possible. Instead of finishing A in 5 seconds and moving on, RR keeps interrupting A and making it take 13 seconds instead. Every job ends up finishing later than it strictly needs to.

### 8.4 The Time Quantum Tradeoff

The length of the time quantum is a critical design decision, and it trades response time against overhead:

| Quantum length | Response time | Context-switch overhead | Overall behavior |
|---|---|---|---|
| Short (around 1 ms) | Better, jobs get the CPU very frequently | Worse, switches happen constantly and eat a large share of total time | Very responsive, but wasteful |
| Long (around 1 second) | Worse, jobs wait longer between turns | Better, little time lost to switching | Approaches FIFO behavior; an extremely long quantum never switches at all |

Real systems typically use time quantums in the range of 10 to 100 milliseconds, long enough that context switch overhead stays small, short enough that interactive responses feel instant.

### 8.5 The Hidden Cost of Context Switching

Context switching is not free. Beyond simply saving and restoring registers, switching processes destroys the CPU's warm state in several ways.

The CPU caches are filled with data from the process that was just running, so after a switch they are cold and filled with the wrong process's data, causing many cache misses until the new process warms them up again. The TLB is filled with address translations for the current process, so after a switch its entries may be invalid for the new process, causing TLB misses until it warms up. The branch predictor has learned the branching patterns of the current process, so after a switch its predictions are wrong until it adapts to the new process's code.

These hidden costs mean context switching is more expensive than it first appears, which is another reason time quantums should not be made too short.

## 9. Incorporating I/O

### 9.1 What Is a CPU Burst

Until now, the discussion assumed jobs only use the CPU, which was assumption 4. Real programs constantly perform I/O: reading files, writing to the network, waiting for user input.

A program does not use the CPU continuously from start to finish. Instead, it alternates between using the CPU and waiting for I/O to complete, in a repeating pattern:

```
CPU work → wait (I/O) → CPU work → wait (I/O) → CPU work → ...
```

Each of these "CPU work" segments is called a CPU burst: a period of time during which a process is actively using the CPU, between one I/O wait and the next. For example, a process might follow this pattern:

```
[CPU burst] → [Disk read] → [CPU burst] → [Keyboard input] → [CPU burst]
```

A concrete process might look like this:

```
Process A timeline:
  1. Do calculation   → CPU burst
  2. Read from disk   → wait (blocked)
  3. Process data      → CPU burst
  4. Write to disk     → wait (blocked)
  5. Continue          → CPU burst
```

Each CPU burst is a separate stretch of pure computation, bounded on either side by an I/O operation that takes the process off the CPU.

### 9.2 The Problem With Ignoring I/O

When a process initiates I/O, it cannot use the CPU. It is blocked waiting for the I/O to complete. Leaving the CPU idle during this time is wasteful.

Consider two jobs: job A needs 50ms of CPU time total, but every 10ms of computation it does a 10ms disk read, while job B needs 50ms of CPU time with no I/O at all. A naive scheduler that runs A completely before starting B wastes the CPU during every one of A's disk reads:

```
CPU:  │A│A│A│A│A│     B     │
Disk:       │A│A│A│A│A│
Time: 0     10    20    30    40    50    60    70    80    90   100   110   120   140
```

### 9.3 The Fix: Treat Each CPU Burst as a Separate Job

When A starts a disk read, it is blocked, and the scheduler should immediately give the CPU to B. When A's disk read completes, A becomes ready again and competes for the CPU like any other job. Treating each of A's CPU bursts as an independent job that can be interleaved with B produces a much better schedule:

```
CPU:  │A│B │A│B │A│B │A│B │A│B │
Disk:    │A│  │A│  │A│  │A│  │A│
Time: 0  10  20  30  40  50  60  70  80  90  100
```

A and B now overlap. While A is waiting for the disk, B uses the CPU. Total time drops from 140ms to roughly 100ms, and CPU utilization is much higher.

The key insight is that when a process blocks on I/O, it voluntarily gives up the CPU, and the scheduler should immediately hand that CPU to another ready process. When the I/O completes, the blocked process becomes ready again and rejoins the queue. This is exactly the running, blocked, ready, running cycle covered earlier in this series when process states were introduced. That cycle is precisely why scheduling and I/O are so closely connected: every time a process blocks on I/O, it creates an opportunity for the scheduler to make a decision.

## 10. The Oracle Problem: Not Knowing Job Length

Every algorithm discussed so far assumes the scheduler knows how long each job will run, which was assumption 5. That assumption is what let SJF and STCF always pick the shortest job. In reality, the OS has absolutely no idea how long a process will run. When a process starts, the OS cannot know whether it is a 1 millisecond job or a 10 hour job without actually running it to see. There is no oracle to consult.

This leaves the algorithms covered in this post in an unresolved position. SJF and STCF optimize turnaround time well, but they require knowing job lengths in advance, which is not available in practice. Round Robin optimizes response time well, but it does so at a serious cost to turnaround time. None of the algorithms covered so far achieve both goals without knowing job lengths ahead of time.

The solution that real operating systems actually use is the Multi-Level Feedback Queue (MLFQ). Instead of requiring foreknowledge of job length, it observes how processes behave over time and uses that history to predict future behavior. Processes that have been running for a long time are likely long jobs. Processes that repeatedly yield the CPU are likely interactive. Yielding the CPU means a process voluntarily gives up its turn before its time slice fully expires, effectively saying "I am done for now, let something else run," instead of being forced off by the scheduler. MLFQ adjusts each process's priority dynamically based on this observed behavior.

## Conclusion

This post worked through the core tradeoffs of CPU scheduling by relaxing one simplifying assumption at a time. FIFO is simple but suffers from the convoy effect. SJF fixes that when all jobs arrive together, but breaks down once jobs can arrive late. STCF fixes that by allowing preemption, and is optimal for turnaround time, but it can leave individual jobs waiting a long time before they first run, and it depends on knowing job lengths in advance. Round Robin fixes response time by giving every job frequent small turns, but at a real cost to turnaround time, and its time quantum has to be tuned carefully against context switch overhead. Bringing I/O into the picture showed that a process's CPU bursts, not the whole process, are really what get scheduled, since a process should give up the CPU the moment it blocks.

All of this still leaves one open problem: the OS never actually knows how long a job will run. That is the subject of the next post: the Multi-Level Feedback Queue, a scheduler that learns a process's behavior over time instead of needing to know it in advance.

**Next:** [The Scheduler That Learns: MLFQ →](./11-the-scheduler-that-learns.md)
