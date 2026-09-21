---
---

> Module 4 · Post 11 of 13

## 1. Picking Up the Thread

The previous post built four scheduling algorithms from scratch and ran into a wall. FIFO suffered from the convoy effect. SJF and STCF fixed that, but only by assuming the scheduler already knows how long each job will run, which is not information the OS actually has. Round Robin gave every job fast response time, but at a serious cost to turnaround time. Every algorithm optimized one goal at the cost of the other, and none of them could do both without cheating.

The Multi-Level Feedback Queue (MLFQ) was invented to solve exactly this problem. It was first described by Fernando Corbato in 1962, work that eventually earned him the Turing Award, the highest honor in computer science, and it has been refined over sixty years since. Some version of it runs inside BSD UNIX, Solaris, Windows NT, and most modern operating systems in use today.

By the end of this post you will understand exactly how MLFQ is structured, all five of its rules, the three problems that a naive version of it suffers from, and how each problem gets fixed. You will also understand why the algorithm running in your operating system right now does not need to know how long your programs will run, which is one of the most elegant ideas in all of systems design.

## 2. Why MLFQ Exists

There are two goals a scheduler might want. The first is good turnaround time, which means running short jobs first, the way SJF and STCF do. The problem is that this requires knowing job length in advance, and the OS has no way of knowing that. The second goal is good response time, which means giving every job a turn quickly, the way Round Robin does. The problem there is that Round Robin gives terrible turnaround time.

No single algorithm covered so far achieves both goals, and no algorithm can implement SJF without knowing job lengths ahead of time. MLFQ is the answer to this dilemma. It does not require knowing job lengths at all. Instead, it watches how jobs behave while they run and uses that observation to make smart decisions.

The core insight behind MLFQ is simple: if a job uses its full time slice, it is probably a long, CPU-hungry job, and if a job gives up the CPU before its time slice ends, it is probably short or interactive. MLFQ uses this past behavior to predict future behavior, and treats each job accordingly.

## 3. The Structure: Multiple Queues With Different Priorities

MLFQ organizes jobs into multiple queues, each with a different priority level. A higher queue means higher priority, which means it gets the CPU first.

```
┌─────────────────────────────────┐
│  Queue 8 (HIGHEST PRIORITY)     │  ← jobs here run first
├─────────────────────────────────┤
│  Queue 7                        │
├─────────────────────────────────┤
│  Queue 6                        │
├─────────────────────────────────┤
│  Queue 5                        │
├─────────────────────────────────┤
│  Queue 4                        │
├─────────────────────────────────┤
│  Queue 3                        │
├─────────────────────────────────┤
│  Queue 2                        │
├─────────────────────────────────┤
│  Queue 1 (LOWEST PRIORITY)      │  ← jobs here run last
└─────────────────────────────────┘
```

A job sits in exactly one queue at any given time, and the scheduler always runs the job from the highest non-empty queue. The number of queues shown here is just an example. To keep the diagrams readable, every example later in this post uses a simplified three-queue system: Q2 as the highest priority, Q1 in the middle, and Q0 as the lowest. The same rules apply no matter how many queues a real implementation uses.

## 4. The Basic Rules

Two simple rules govern which job runs at any instant:

```
Rule 1: If Priority(A) > Priority(B) → A runs, B does not
Rule 2: If Priority(A) = Priority(B) → A and B take turns using Round Robin
```

These two rules alone do not make MLFQ special. Plenty of priority schedulers work this way. What makes MLFQ special is how priorities change over time. A job does not stay in the same queue forever; its priority rises or falls based on how it behaves.

## 5. How Jobs Enter and Move Through Queues

### 5.1 New Jobs Start at the Top

```
Rule 3: When a job enters the system, place it at the highest priority queue
```

Why start at the top? Because MLFQ assumes every new job might be short and interactive, and interactive jobs deserve a fast response. So every new job gets the benefit of the doubt, top priority, and is treated as if it were a short interactive job. If it turns out to be a long job instead, it will prove that over time and get demoted. MLFQ starts optimistic and corrects itself as evidence comes in.

### 5.2 How Priority Changes Based on Behavior

```
Rule 4a: If a job uses its ENTIRE time slice → reduce priority (move down one queue)
Rule 4b: If a job gives up the CPU BEFORE the time slice ends → keep the same priority
```

This pair of rules is the heart of MLFQ. The scheduler watches what a job does with the time it is given. If a job uses its full time slice, the scheduler concludes that it needed all the time it got, so it is probably a long, CPU-hungry job, and moves it down to lower priority. If a job gives up the CPU early instead, because it is doing I/O, waiting for input, or otherwise blocking, the scheduler concludes that it did not even need its full slice, so it is probably short or interactive, and keeps it at the same priority so it continues to get a fast response.

## 6. Walking Through MLFQ With Examples

### 6.1 Example 1: A Single Long-Running Job

Suppose one long job arrives, with three queues, Q2 highest, Q1 in the middle, Q0 lowest, and a time slice of 10ms.

```
Time 0: Job enters → placed at Q2 (top queue, Rule 3)
Q2: [Job A]
Q1: []
Q0: []
Scheduler: run Job A from Q2
```

After 10ms, Job A has used its entire time slice, so Rule 4a applies: it moves down one queue, from Q2 to Q1. After another 10ms, Job A again uses its entire time slice, so Rule 4a applies again, and it moves from Q1 down to Q0.

```
Rule 4a applies again: move down
Job A moves from Q1 to Q0
```

From that point on, Job A stays at Q0 forever. It is clearly a long, CPU-intensive job, it deserves low priority, and it runs in the background from then on. The chart below shows how little time Job A spends at each of the higher queues before sinking to the bottom, where it then stays for the rest of its run:

```
Q2  ▓▓                                                 (0-10ms, first slice)
Q1    ▓▓                                               (10-20ms, second slice)
Q0      ████████████████████████████████████████████  (20-200ms, sinks here and stays)
    0         50         100        150        200   (time in ms)
```

Long jobs naturally sink to the bottom. MLFQ figured this out by itself, without ever being told the job's length in advance.

### 6.2 Example 2: A Short Job Arrives While a Long Job Is Running

Now the more interesting case. Job A, the long job from before, has been running for a while and is sitting at Q0. Job B arrives at time 100.

```
Time 100: Job B arrives → placed at Q2 (top queue)
Q2: [Job B]   ← new job, starts at top
Q1: []
Q0: [Job A]   ← long job sitting at bottom
```

The scheduler's decision is immediate: Q2 has a job, and Q2 is the highest priority, so Job B runs.

```
Rule 1: Priority(B) > Priority(A) → B runs, A does not

B runs at Q2 for 10ms → uses full slice → demoted to Q1
B runs at Q1 for 10ms → B finishes

Scheduler concludes after the fact:
"That was a short job. It finished before reaching Q0."
```

Reading through this step by step: from time 0 to 100, Job A runs at Q0, since it had already sunk to the bottom in the previous example. At time 100, Job A pauses because Job B arrives and takes priority. Job B runs at Q2 from roughly 100 to 110ms, uses its full slice, and gets demoted to Q1. It then runs at Q1 from roughly 110 to 120ms, uses its full slice there too, but finishes at that point, having completed 20ms of total work, and it never reaches Q0 at all. From time 120 onward, Job A resumes at Q0.

```
Q2              ▓▓                                     (Job B, ~100-110ms)
Q1                ▓▓                                   (Job B, ~110-120ms, finishes here)
Q0  ████████████████  ░░  ██████████████████████████   (Job A: 0-100, paused, resumes 120-200)
    0         50         100        150        200    (time in ms)
```

MLFQ gave Job B excellent response time and excellent turnaround time, just as SJF would have, without ever knowing in advance that B was a short job. B got high priority simply because it was new, and it finished before it could be demoted all the way down. This is how MLFQ approximates SJF without knowing job lengths ahead of time.

### 6.3 Example 3: An I/O-Intensive Job

Now consider two jobs running at the same time. Job A is a long-running, CPU-intensive batch job: it needs lots of CPU time, never performs I/O, and has already sunk to Q0. Job B is an interactive, I/O-intensive job: it needs the CPU for only 1ms at a time, then does I/O, then needs the CPU again for another 1ms, repeating this pattern constantly.

Job B's behavior repeats in a cycle: it gets the CPU at Q2, runs for 1ms, and then needs to do I/O, so it gives up the CPU voluntarily, well before using its full 10ms time slice. Rule 4b applies, since it gave up the CPU before the time slice ended, so it stays at Q2. Once its I/O completes, it goes back to the ready queue at Q2, gets the CPU again, runs for another 1ms, and does I/O again, giving up the CPU once more. Rule 4b applies again, and it stays at Q2 again. This repeats indefinitely: Job B never uses its full time slice, so it never gets demoted, and it stays at Q2 permanently.

```
Q2  ▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕▏▕   (Job B: repeated 1ms bursts, never demoted)
Q1                                            (empty)
Q0  ████████████████████████████████████████  (Job A: fills the remaining CPU time, stays at the bottom)
    0         50         100        150        200   (time in ms)
```

MLFQ's goal is exactly this outcome: treat interactive jobs well by keeping them at high priority, and let long jobs run efficiently in the background without starving the interactive ones of CPU time on every turn.

## 7. The Three Problems With This Basic MLFQ

The rules covered so far work, but the basic version of MLFQ has three serious flaws.

### 7.1 Problem 1: Starvation

If there are too many interactive jobs, they can fill up the high priority queues permanently.

```
Q2: [Interactive1] [Interactive2] [Interactive3] [Interactive4]...
Q1: []
Q0: [LongJobA]  ← never runs, forever waiting
```

Rule 1 says the scheduler always runs the highest priority job. If the high queues are always full, LongJobA never gets the CPU at all. It starves, waiting indefinitely and making no progress. Starvation means a process waits so long that it effectively never runs.

### 7.2 Problem 2: Gaming the Scheduler

A clever program can exploit Rule 4b to stay at high priority forever. Honest behavior looks like this: run for the full 10ms time slice, then get demoted to lower priority. Gaming behavior looks like this instead: run for 9.9ms, then perform a tiny fake I/O operation, such as writing to a file the program does not actually care about, giving up the CPU just before the time slice ends. Rule 4b applies, since the job gave up the CPU early, so it keeps its high priority. On the next turn, it runs for 9.9ms again, performs another fake I/O, and stays at high priority once more, indefinitely. This lets a dishonest program monopolize the CPU at the expense of every other job, simply by doing a tiny fake I/O operation just before each time slice would otherwise expire.

### 7.3 Problem 3: Job Behavior Changes Over Time

Real programs are not static. A program might start out CPU-intensive, such as while compiling code, and then become interactive, such as while waiting for the user to type something.

```
Phase 1: Job A compiles code → heavy CPU usage → sinks to Q0
Phase 2: Job A finishes compiling → now waits for user input → interactive
```

But Job A is stuck at Q0 because of its past behavior. The basic version of MLFQ never reconsiders a job once it has been placed, so Job A gets terrible response time in its interactive phase, even though it is now behaving exactly like an interactive job.

## 8. Fixing Starvation and Behavior Change: Priority Boost

The fix for both starvation and the behavior-change problem is the same: periodically give every job a fresh start.

```
Rule 5: After some time period S, move ALL jobs in the system
        to the topmost queue
```

Every S milliseconds, perhaps every second, every single job in the system, regardless of where it currently sits, gets moved back to Q2, the top queue.

```
Before boost:
Q2: [Interactive jobs running here]
Q1: []
Q0: [LongJobA stuck here, starving]

After boost (every S ms):
Q2: [Interactive jobs] [LongJobA]  ← A gets a turn
Q1: []
Q0: []
```

This solves starvation directly: LongJobA gets a fresh start every S milliseconds and gets CPU time even while high-priority jobs exist. It also solves the behavior-change problem. If LongJobA has become interactive since its last boost, it will now give up the CPU early under Rule 4b and stay at high priority, so MLFQ adjusts to its new behavior naturally, without needing any special-case logic.

```
Without priority boost:            With priority boost every 50ms
Q2: [interactive jobs running]     Q2: [A gets boost] [interactive]
Q1: []                             Q1: [A starts sinking]
Q0: [A stuck, never runs]          Q0: [A sinks here, then boost again]
Long job starves.                  Long job makes progress
```

### 8.1 The Voodoo Constant Problem

What should S actually be set to? If S is too large, say every 10 minutes, long jobs starve for 10 minutes before each boost, which is basically the same as having no boost at all. If S is too small, say every 1ms, everything gets boosted constantly, so no process is ever demoted in any meaningful way, and the scheduler degenerates into plain Round Robin. S has to be "just right," and there is no formula that tells you what that value is.

This is what systems textbooks call a voodoo constant: a value that has no mathematical derivation and must be tuned by experience and experimentation, where setting it too high or too low causes real problems, and the right value depends on the workload, which varies from system to system. In practice, Solaris uses approximately 1 second, and Linux uses similar values. These are educated guesses refined over years of real-world use, not values derived from first principles.

## 9. Fixing Gaming: Better Accounting

The gaming problem exists because Rules 4a and 4b, as originally stated, only look at a single time slice at a time. A job can reset the clock over and over by performing fake I/O just before each slice ends.

The fix is to track the total CPU time a job has used at each queue level across all of its time slices at that level, not just its current slice. Once a job's total usage at a given level reaches that level's time allotment, for example 10ms at Q2, it gets demoted, regardless of whether that time was used in one continuous burst or accumulated across many small pieces. This reframes the question the scheduler asks. Instead of "what did you do this time slice," it becomes "how much CPU have you used overall at this level."

```
Example with time allotment = 10ms at Q2:

Honest job:
  Uses 10ms in one go → total at Q2 = 10ms → demote to Q1

Gaming job (old rules):
  Uses 9.9ms → fake I/O → stays at Q2 (clock reset!)
  Uses 9.9ms → fake I/O → stays at Q2 (reset again!)
  Stays at Q2 forever

Gaming job (new rule):
  Uses 9.9ms → fake I/O → total at Q2 = 9.9ms → stays (not yet at 10ms)
  Uses 0.1ms more → total at Q2 = 10ms → DEMOTE to Q1
  Cannot escape demotion by doing fake I/O
```

With this fix, a gaming job can no longer monopolize the high-priority queue. It gets demoted based on total CPU consumption at that level, not on how it happened to slice up that consumption.

## 10. Configuring MLFQ: The Real Parameters

Understanding how MLFQ works, multiple queues, priority changes based on behavior, priority boosts, and anti-gaming accounting, is one problem. Knowing how to configure it is a completely different one. MLFQ has several parameters that must be decided before it can run at all: how many queues should exist, how long the time slice should be at each queue, and how often the priority boost should happen. None of these has a mathematically correct answer. They depend entirely on the workload, meaning what kinds of programs are running, how interactive they are, how long they run, and how much I/O they perform, and that workload changes from system to system and from moment to moment.

| Parameter | Too small | Too large | Notes |
|---|---|---|---|
| Number of queues | Too blunt, no gradual learning | Too complex to manage | Solaris uses 60, found through experimentation |
| Time slice per queue | High queues get short slices for fast response | Low queues get long slices for efficient throughput | High-priority slices around 10ms, low-priority slices 100ms or more |
| Priority boost interval | MLFQ degenerates into plain Round Robin | Long jobs starve, behavior changes get ignored | Solaris uses approximately 1 second |

### 10.1 Time Slice per Queue

High priority queues use short slices, around 10ms, because interactive jobs live there. They need fast response, several interactive jobs typically share a queue via Round Robin, and short slices mean each one gets frequent turns. Low priority queues use long slices, 100ms or more, because batch jobs live there. Batch jobs need throughput rather than response time, and long slices reduce how often the CPU has to pay the cost of a context switch, letting these jobs run efficiently in large chunks.

### 10.2 All Three Parameters Are Voodoo Constants

None of these three parameters has a mathematical derivation. All of them are found through experience and experimentation, their default values are set by the operating system's developers, they are rarely changed by system administrators, and every system essentially hopes its defaults happen to fit its actual workload.

### 10.3 Real Implementations

| System | Approach |
|---|---|
| Solaris | Table-based, 60 queues, time slices from 20ms to 300ms, priority boost roughly every second |
| FreeBSD | Formula-based, priority decays over time as a function of CPU usage |

Both approaches prevent starvation and both handle jobs that change behavior over time, but they reach those goals through different mechanisms.

### 10.4 Reserved Priorities and the nice Command

Many real schedulers reserve the very highest priority levels exclusively for OS kernel work, such as interrupt handlers, critical kernel threads, and real-time OS tasks, and user programs can never reach those levels. On the other end, the `nice` command lets a user hint at their own program's priority, which the OS may or may not choose to follow.

## 11. Varying Time Slices Per Queue Level

Putting the time-slice idea from section 10.1 together into a concrete example, a real MLFQ implementation might assign a 10ms slice at Q2, a 20ms slice at Q1, and a 100ms slice at Q0:

```
Q2: time slice = 10ms  (fast response for interactive jobs)
Q1: time slice = 20ms  (moderate)
Q0: time slice = 100ms (efficient for long CPU-bound jobs)

A job's timeline as it sinks through the queues:
│ Q2: 10ms │ Q1: 20ms │ Q0: 100ms | 100ms | 100ms... │
0          10         30          130     230     330...
```

This is an elegant result. Interactive jobs sitting at the top get snappy response because their slices are short, and long jobs that sink to the bottom run efficiently in large chunks because their slices are long, which keeps context-switch overhead low exactly where it would otherwise add up the most.

## 12. The Complete Set of MLFQ Rules

Putting every fix together, the complete rule set for a working MLFQ scheduler is:

```
Rule 1: If Priority(A) > Priority(B) → A runs, B does not

Rule 2: If Priority(A) = Priority(B) → A and B run in Round Robin

Rule 3: New jobs start at the highest priority queue

Rule 4: Once a job uses up its total time allotment at a given level
        (regardless of how many separate slices that time was spread across),
        its priority is reduced, and it moves down one queue

Rule 5: After time period S, move ALL jobs to the highest priority queue
```

## 13. Putting It All Together: A Full Example

Consider three queues, Q2, Q1, and Q0, with time allotments of 10ms at Q2, 20ms at Q1, and unlimited at Q0, and a priority boost every 100ms. Three jobs are involved: Job A, a long CPU-intensive job arriving at time 0, Job B, a short interactive job arriving at time 50ms, and Job C, an I/O-intensive job arriving at time 75ms.

```
Time 0: A enters → Q2
Q2: [A]
A runs for 10ms → uses full allotment → demoted to Q1

Time 10: A at Q1
Q1: [A]
A runs for 20ms → uses full allotment → demoted to Q0

Time 30: A at Q0, now running in 100ms slices
Q0: [A]
A keeps running...

Time 50: B arrives → Q2
Q2: [B]   ← higher priority than A
Q0: [A]

Rule 1: B has higher priority → B runs immediately, A is preempted
B runs for 8ms → does I/O (gives up the CPU before its 10ms allotment)
Rule 4: B used only 8ms of its 10ms allotment → stays at Q2

Time 58: B is doing I/O, CPU is free
Q2: []
Q0: [A]
A resumes

Time 60: B's I/O finishes → B returns to Q2
Q2: [B]
Q0: [A]
B preempts A again, runs for 2ms → total used at Q2 = 10ms → demoted to Q1

Time 62: B at Q1
Q1: [B]
Q0: [A]
B now runs with a 20ms allotment, and keeps doing I/O, staying under that allotment

Time 75: C arrives → Q2
Q2: [C]
Q1: [B]
Q0: [A]
C is highest priority → C runs
C is very I/O-intensive → keeps giving up the CPU early → stays at Q2 indefinitely

Time 100: Priority boost fires
ALL jobs move to Q2:
Q2: [A] [B] [C]
Q1: []
Q0: []
```

After the boost, A, B, and C all share Q2 through Round Robin. A will quickly use up its allotment and sink back down toward Q0, since it is genuinely CPU-bound, while B and C will stay high as long as they continue behaving interactively. This is the entire MLFQ system working together: new jobs get the benefit of the doubt, long jobs sink naturally without needing to declare themselves, gaming is blocked by tracking total usage rather than single slices, and the periodic boost keeps the whole system fair over time.

## Conclusion

MLFQ solves the problem that closed out the previous post: getting good turnaround time and good response time without ever knowing a job's length in advance. It does this by watching behavior instead of requiring foreknowledge. Jobs that use their full time slice are demoted, jobs that give up the CPU early are kept at high priority, and both starvation and jobs that change behavior over time are handled by the periodic priority boost. Gaming the naive per-slice rule is closed off by tracking total time used at each level instead. What remains are three voodoo constants, the number of queues, the time slice at each level, and the boost interval, that every real system has to tune by experience rather than by formula.

Everything covered from the very first post through this one has assumed a single CPU with one process running on it at a time, switching in and out. The next module drops that assumption. Modern programs often need to do more than one thing at once inside the same process, and that raises an entirely new set of problems: concurrency, the subject of the next post.

**Next:** [Concurrency: An Introduction →](./12-concurrency-introduction.md)
