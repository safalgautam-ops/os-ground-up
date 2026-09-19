# Interlude: Thread API

> Module 5 · Post 13 of 13

## 1. Before Anything: What Problem Are We Solving

Imagine writing a web server. When a thousand users send requests at the same time, you want to handle all of them simultaneously, not one after another. Or imagine a video editor that compresses a video in the background while you keep editing. Or a music player that loads the next song while the current one is still playing. All of these require one program to do multiple things at the same time, and that is exactly what threads enable.

The previous post introduced what a thread is and why sharing memory between threads creates race conditions. This post is the practical companion to that one: the actual pthread API used to create threads, wait for them, protect shared data with locks, and coordinate threads with condition variables.

## 2. What Is a Thread, Really?

### 2.1 The Single-Threaded World You Already Know

Every program written so far in this series has one path of execution. It starts at `main()`, runs line by line, calls functions, returns from functions, and eventually ends.

```
main() starts
  │
  ▼
line 1 executes
  │
  ▼
line 2 executes
  │
  ▼
function A() called
  │
  ▼
function A() returns
  │
  ▼
line 3 executes
  │
  ▼
program ends
```

One instruction at a time, always in order, completely predictable. This is a single thread.

### 2.2 Adding a Second Thread

A thread is simply a second, independent path of execution inside the same program.

```
Thread 1 (main):          Thread 2 (new):
main() starts
creates Thread 2    →     starts executing myfunction()
continues running         running simultaneously
waits for Thread 2        finishes myfunction()
Thread 2 done
main continues
```

Both threads run at the same time, or at least appear to, through rapid switching by the scheduler. Both are part of the same program, and both share the same memory.

### 2.3 The Critical Shared vs Private Distinction

This is the single most important thing to understand about threads. Shared between all threads in a process are the code, meaning every thread runs from the same instructions, the global variables, meaning every thread sees the same globals, the heap memory allocated with `malloc`, meaning every thread shares dynamically allocated memory, and the open files, meaning every thread shares the same file descriptors. Private to each thread are its stack, its own registers, meaning its own program counter, stack pointer, and frame pointer, and its own local variables, since each function call's locals belong only to that call on that thread's stack.

This sharing is what makes threads powerful: they can communicate through shared memory without needing any special mechanism at all. It is also exactly what makes threads dangerous: they can just as easily interfere with each other's data by accident, which is the subject the rest of this post keeps circling back to.

## 3. pthread_create: Creating a Thread

### 3.1 The Mental Model First

Before looking at any code, it helps to understand what `pthread_create` actually does in plain words. It says: start a new thread, and that thread should begin executing this specific function, with this specific argument. That is the entire idea. Everything else in this section is just the mechanics of expressing that idea in C.

### 3.2 The Function Signature

```c
int pthread_create(
    pthread_t        *thread,
    pthread_attr_t   *attr,
    void *           (*start_routine)(void *),
    void *           arg
);
```

Four arguments, built up one at a time below.

### 3.3 Argument 1: pthread_t *thread

`pthread_t` is a type that holds a thread's identity, much like a process ID holds a process's identity.

```c
pthread_t p1;    // declare a variable to hold the thread's identity
```

Right now `p1` is empty; it holds nothing meaningful yet. When you call `pthread_create`, you pass `&p1`, the address of `p1`, and the function fills `p1` in with the new thread's identity:

```c
pthread_t p1;
pthread_create(&p1, ...);
// now p1 contains the thread's identity
// you can use p1 later to wait for this thread
```

The reason for passing the address is a basic rule of C: if you want a function to write into your variable, you must pass that variable's address. If you passed `p1` itself, the function would only receive a copy, and any changes it made would never reach your original variable. It is the same idea as the difference between saying "here is my phone number, write the result here," which only gives the function a copy to scribble on, and "here is where my phone number is stored, write the result there," which lets the function update the original.

### 3.4 Argument 2: pthread_attr_t *attr

This argument lets you customize the thread, for example how big its stack should be or what scheduling priority it should have. For almost every program you will ever write, pass `NULL` here, which means "use the default settings," and those defaults are perfectly fine for normal use.

```c
pthread_create(&p1, NULL, ...);
//                   ^^^^
//                   NULL = use defaults, no custom settings needed
```

Only experienced systems programmers working on specialized applications typically need to set custom attributes here.

### 3.5 Argument 3: The Function to Run

This is the most confusing argument, so it is worth building up slowly. In C, functions have addresses in memory just like variables do, and a function pointer is a variable that holds the address of a function. You are already familiar with passing a value to a function, as in `do_something(x)`. A function pointer instead passes a function itself as an argument, as in `do_something(myfunction)`, which lets `do_something` call `myfunction` later. That is exactly what `pthread_create` needs: it has to know which function the new thread should run.

The thread function must follow this exact required signature:

```c
void *myfunction(void *arg) {
    // thread runs this code
    return NULL;
}
```

Breaking this down piece by piece: the return type is `void *`, meaning "pointer to anything," so the thread can return any type of data by casting it to `void *`, and if there is nothing to return, it simply returns `NULL`. The name `myfunction` can be anything you choose. The parameter is `void *arg`, again "pointer to anything," so the thread can receive any type of data as long as it is packaged as a `void *`.

The reason `void *` shows up everywhere is that the thread system is designed to be generic. It has to work for any function with any argument and any return type, and in C, `void *` is the way to express something generic, a container that can point to any type. When you pass your own data in, you cast it to `void *`, and when you receive it inside the thread, you cast it back to its original type.

### 3.6 Argument 4: void *arg, the Argument to Pass

This is the actual data you want to send to the thread. Whatever you pass here becomes the `arg` parameter inside your thread function.

```c
// passing a simple string
pthread_create(&p1, NULL, myfunction, "hello");

// inside myfunction:
void *myfunction(void *arg) {
    char *message = (char *) arg;   // cast void* back to char*
    printf("%s\n", message);        // prints "hello"
    return NULL;
}
```

### 3.7 Complete Simple Example, Step by Step

```c
#include <stdio.h>
#include <pthread.h>

void *mythread(void *arg) {
    printf("%s\n", (char *) arg);
    return NULL;
}

int main() {
    pthread_t p1, p2;
    pthread_create(&p1, NULL, mythread, "A");
    pthread_create(&p2, NULL, mythread, "B");
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    return 0;
}
```

Reading through every line: `pthread_t p1, p2;` declares two thread handle variables, like declaring two pids to hold two thread identities. `pthread_create(&p1, NULL, mythread, "A")` passes `&p1` as where to store this thread's identity, `NULL` for default settings, `mythread` as the function this thread will run, and `"A"` as the string passed into `mythread` as its argument. After this line, a new thread exists and is running `mythread("A")` independently of everything else. The next line, `pthread_create(&p2, NULL, mythread, "B")`, does the same thing again, starting a second thread running `mythread("B")`.

At this point three things are executing at once: the main thread, which has just called `pthread_create` twice and continues to the next line, thread 1, running `mythread` and printing `"A"`, and thread 2, running `mythread` and printing `"B"`. The line `pthread_join(p1, NULL)` makes the main thread stop and wait until thread 1 finishes, a mechanism explained in full in the next section.

Inside `mythread` itself, `arg` arrives as `void *`, a generic pointer. Since it is known to actually be a `char *` here, it gets cast with `(char *) arg`, and from that point on it can be used as an ordinary string: printed, and then `NULL` is returned because there is no result to send back.

### 3.8 Passing Multiple Values Using a Struct

Only one argument can be passed to a thread function. But multiple values can be packaged inside a struct, and a pointer to that struct passed instead.

```c
#include <stdio.h>
#include <pthread.h>

// Step 1: define a struct to hold your arguments
typedef struct {
    int a;
    int b;
} myarg_t;

// Step 2: the thread function receives void*, casts it back to myarg_t*
void *mythread(void *arg) {
    myarg_t *m = (myarg_t *) arg;   // cast: void* → myarg_t*
    printf("%d %d\n", m->a, m->b);  // access struct fields
    return NULL;
}

int main() {
    pthread_t p;
    myarg_t args;       // create the struct
    args.a = 10;        // fill in the values
    args.b = 20;
    // pass address of struct as the argument
    pthread_create(&p, NULL, mythread, &args);
    pthread_join(p, NULL);
    return 0;
}
```

The flow of data through this program:

```
main creates: myarg_t args = {10, 20}
                                │
                                │ &args (address of struct)
                                ▼
pthread_create passes it as void* arg
                                │
                                │ arrives as void* in mythread
                                ▼
mythread casts: myarg_t *m = (myarg_t *) arg
                                │
                                │ now m points to the same struct
                                ▼
m->a = 10, m->b = 20         ← accessible
```

## 4. pthread_join: Waiting for a Thread

### 4.1 The Problem Without Join

If the main thread exits, the entire process exits, and every other thread dies immediately, even if it was in the middle of something.

```c
int main() {
    pthread_t p;
    pthread_create(&p, NULL, mythread, "A");
    return 0;   // main exits immediately
                // the thread might not have printed anything yet
                // it is killed before it can finish
}
```

This is the same as starting a task and leaving before it is done.

### 4.2 What pthread_join Does

`pthread_join` makes the calling thread wait until the specified thread finishes.

```c
int pthread_join(pthread_t thread, void **value_ptr);
```

It takes two arguments: `thread`, which thread to wait for, the same `pthread_t` variable filled in by `pthread_create`, and `value_ptr`, where to store the return value from that thread, or `NULL` if the return value does not matter.

```c
pthread_join(p1, NULL);
// main stops here
// main waits
// main waits
// Thread 1 finishes
// main continues to next line
```

### 4.3 Getting a Return Value from a Thread

Threads can return data back to whoever joins them, and that return value travels through `void *`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

// struct for arguments going IN to the thread
typedef struct {
    int a;
    int b;
} myarg_t;

// struct for results coming OUT of the thread
typedef struct {
    int x;
    int y;
} myret_t;

void *mythread(void *arg) {
    myarg_t *m = (myarg_t *) arg;
    printf("received: %d %d\n", m->a, m->b);
    // allocate the result ON THE HEAP (very important, explained below)
    myret_t *result = malloc(sizeof(myret_t));
    result->x = m->a + m->b;    // compute sum
    result->y = m->a * m->b;    // compute product
    return (void *) result;     // return heap pointer as void*
    // result is a pointer to a struct (myret_t *), but pthread functions
    // must return void *, so it is cast without changing the address itself
}

int main() {
    pthread_t p;
    myarg_t args;
    args.a = 10;
    args.b = 20;
    pthread_create(&p, NULL, mythread, &args);

    // result's type is myret_t * (a pointer to myret_t)
    myret_t *result;
    // to write into a variable in C, you must pass that variable's address,
    // and pthread_join needs to write into result, so it needs &result
    pthread_join(p, (void **) &result);
    // pthread_join is generic: it works for every thread, no matter what
    // that thread returns, so it has no idea what myret_t is and cannot
    // accept myret_t **. It only knows how to accept void **.
    printf("sum=%d product=%d\n", result->x, result->y);
    free(result);   // free heap memory when done
    return 0;
}
```

To understand why `pthread_join` needs a pointer to a pointer here, it helps to start from a simpler case. If a function wants to change a plain `int` variable, you pass it that variable's address: for `int x;`, you pass `&x`, which is a pointer to int. The same idea applies one level deeper here. `result` is declared as `myret_t *result`, a pointer to a `myret_t`. If `pthread_join` wants to change what `result` itself points to, you have to pass the address of `result`, which is `&result`, and since `result` is already a pointer, the address of a pointer is a pointer to a pointer, `myret_t **`. That is exactly why the call above writes `(void **) &result`: it is the address of a pointer variable, generalized to `void **` so that `pthread_join` can work no matter what type of value a given thread happens to return.

Tracing the full flow of data makes this concrete. Sending data to the thread: `main` creates `args = {a=10, b=20}`, `pthread_create` passes `&args` as `void *`, the thread receives it as `void *arg`, casts it with `myarg_t *m = (myarg_t *) arg`, and reads `m->a = 10, m->b = 20`. Getting data back out of the thread: the thread allocates `myret_t *result = malloc(...)`, fills in `result->x = 30, result->y = 200`, and returns `(void *) result`, a heap address cast to `void *`. `pthread_join` receives this `void *` and stores it into `result` back in `main`, which can then read `result->x` and `result->y`, and finally frees that memory with `free(result)`.

### 4.4 The Most Important Rule: Never Return Stack Pointers

This is where most beginners make a catastrophic mistake, so it is worth understanding exactly why. Here is the wrong way to write this:

```c
void *mythread(void *arg) {
    myret_t result;        // THIS IS ON THE STACK
    result.x = 1;
    result.y = 2;
    return (void *) &result;  // returning ADDRESS of a stack variable
}
```

Recall how the stack works from earlier posts in this series. While `mythread` is running, its stack frame looks like this:

```
mythread's stack frame:
┌─────────────────┐
│ result.x = 1     │  ← lives here while mythread is running
│ result.y = 2     │
└─────────────────┘
```

But once `mythread` returns, its entire stack frame is destroyed:

```
After mythread returns:
┌─────────────────┐
│ ??? garbage ???  │  ← this memory is gone, reused for something else
│ ??? garbage ???  │
└─────────────────┘
```

The pointer `&result` now points at destroyed, reused memory. When `main` tries to read from it:

```c
myret_t *m;
pthread_join(p, (void **) &m);
printf("%d %d\n", m->x, m->y);   // reading garbage, or a crash
```

The values that come back are either garbage or the program crashes outright. This is a very common bug and a hard one to catch, because it can sometimes appear to work correctly, simply because that stack memory had not yet been overwritten by anything else.

The right way is to always use the heap instead:

```c
void *mythread(void *arg) {
    // malloc returns a pointer, so it must be assigned to a pointer
    // variable, not a plain struct variable
    myret_t *result = malloc(sizeof(myret_t));  // HEAP allocation
    result->x = 1;
    result->y = 2;
    return (void *) result;   // returning a heap address is SAFE
    // cast to (void *) because the thread function's return type is void *
}
```

Heap memory persists until you explicitly call `free()`. It does not disappear when the function that allocated it returns, so the pointer remains valid after `mythread` exits. Stack memory exists only while a function is actively executing, while heap memory exists until you call `free()`. For any data that needs to outlive the function that created it, always use the heap.

### 4.5 When Not to Use pthread_join

Not every thread needs to be joined. Two common patterns illustrate this. In the parallel computation pattern, threads are created to compute different parts of a problem, the program waits for all of them to finish, and then combines their results, which does call for `pthread_join`. In the long-running worker pattern, a web server might create a new thread for each incoming request while the main thread keeps accepting new connections; worker threads live as long as they need to, and nobody explicitly waits for them.

## 5. Locks: Protecting Shared Data

### 5.1 Why Locks Are Needed

The previous post showed that `counter = counter + 1` is actually three separate CPU instructions: reading `counter` from memory into a register, adding 1 to that register, and writing the register back to memory. If two threads do this simultaneously, they can interfere with each other. Thread 1 might read `counter = 50`, then get interrupted. Thread 2 reads `counter = 50` as well, since thread 1 has not written anything back yet, increments it to 51, and writes 51. Thread 1 then resumes with its own old register value of 51 and writes 51 again. The counter ends up at 51 when it should have reached 52.

A lock prevents this by ensuring only one thread can execute the critical section at a time.

### 5.2 The Mental Model

Think of a lock as a key to a room, where the room is the critical section, the code that touches shared data. Thread 1 takes the key, enters the room, does its work, leaves, and returns the key. Thread 2 wants the key, finds it already taken, and waits outside. Once thread 1 returns the key, thread 2 takes it, enters the room, does its work, and leaves in turn. Only one thread is ever in the room at a time, so there is no interference and no race condition.

### 5.3 pthread_mutex_t vs pthread_mutex_lock

`pthread_mutex_t` is a data type, similar to `int` or `float`, but specifically designed to represent a lock. Writing:

```c
pthread_mutex_t lock; // this only creates memory for the lock; it is NOT ready yet
```

declares a variable called `lock` of type `pthread_mutex_t`. This variable is the lock itself: it holds all the internal information about whether it is currently held, which thread holds it, and which threads are waiting for it. You never look inside this variable directly; you only interact with it through the pthread functions built for that purpose.

`pthread_mutex_lock()` is a function that operates on a `pthread_mutex_t` variable:

```c
pthread_mutex_lock(&lock);
```

This call says: I want to acquire the lock stored in the variable `lock`. If nobody holds it, give it to me immediately. If someone holds it, make me wait until they release it. The address `&lock` is passed rather than `lock` itself because the function needs to modify the lock's internal state, marking it as now held by this thread, and modifying a variable inside a function in C requires passing that variable's address.

These two things sound similar but are completely different: `pthread_mutex_t` is the lock itself, a data structure, a thing; `pthread_mutex_lock()` is an action performed on the lock, a function call. It is the same distinction as a door and the act of locking it:

```
pthread_mutex_t         = the actual physical door with a lock on it
pthread_mutex_lock()    = the act of locking that door
pthread_mutex_unlock()  = the act of unlocking that door

pthread_mutex_t lock;         ← CREATE the lock (a variable/object)

pthread_mutex_lock(&lock);    ← ACQUIRE the lock (a function call)
                                 "I am taking ownership of this lock"

pthread_mutex_unlock(&lock);  ← RELEASE the lock (a function call)
                                 "I am done, others can take it now"
```

The door has to exist before you can lock or unlock it, and in exactly the same way, the `pthread_mutex_t` variable has to exist before you can call `pthread_mutex_lock` on it.

### 5.4 Declaring and Initializing a Lock

The simple way to initialize a lock is statically:

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER; // fills lock with proper defaults
```

Reading each part: `pthread_mutex_t` is the data type, a lock variable; `lock` is the chosen name; `PTHREAD_MUTEX_INITIALIZER` is a special constant that sets the lock up with correct default internal values. This single line both declares and initializes the variable, and it is the right choice when the lock is a global variable, declared outside any function, or otherwise known at compile time.

```c
#include <pthread.h>
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;   // global lock
int counter = 0;                                     // global shared data

void *mythread(void *arg) {
    pthread_mutex_lock(&lock);   // take the lock
    counter++;                   // critical section
    pthread_mutex_unlock(&lock); // release the lock
    return NULL;
}
```

The lock is ready to use here with no other setup needed. Sometimes, though, the lock cannot be initialized at the point of declaration, for example when it lives inside a struct, inside dynamically allocated memory, inside an array created later, or whenever many mutexes are needed dynamically at runtime. In those cases, dynamic initialization is used instead:

```c
#include <stdio.h>
#include <pthread.h>

pthread_mutex_t lock;

int main() {
    pthread_mutex_init(&lock, NULL);   // address of lock, default settings

    pthread_mutex_lock(&lock);         // take ownership of the lock
    printf("critical section\n");
    pthread_mutex_unlock(&lock);       // give up ownership of the lock

    pthread_mutex_destroy(&lock);      // clean up when fully done
    return 0;
}
```

Breaking down `pthread_mutex_init(&lock, NULL)`: `pthread_mutex_init` is the initialization function, `&lock` is the address of your lock variable, since the function needs to write into it, and `NULL` means use default attributes. Breaking down `pthread_mutex_destroy(&lock)`: this is the cleanup function, and `&lock` is the address of the lock to clean up. A lock created with `PTHREAD_MUTEX_INITIALIZER` needs no destroy call; a lock created with `pthread_mutex_init` must always be paired with `pthread_mutex_destroy` once it is no longer needed.

When you call `pthread_mutex_init`, the operating system may create hidden internal data for that lock: waiting queues, thread information, internal state, general OS bookkeeping, none of which you can see directly, but which the OS uses to manage the lock behind the scenes. `pthread_mutex_destroy` releases all of that: it returns the room, so to speak, and cleans up those resources. In a small, short-lived program this is usually not noticeable, since the OS reclaims everything when the program exits anyway. But in a large or long-running program, skipping it can leak memory and resources, let too many mutexes accumulate, and waste OS resources over time.

### 5.5 Always Unlock, Always Check Return Codes

You must always unlock a lock that you locked. If a thread locks but never unlocks, thread 1 locks, does its work, and forgets to unlock; thread 2 then tries to lock and waits forever, and the program hangs. This situation is called a deadlock, and every `pthread_mutex_lock` needs a corresponding `pthread_mutex_unlock`.

Every pthread function also returns an integer describing whether it succeeded. Calling `rc = pthread_mutex_init(&lock, NULL)` sets `rc` to the result of initialization, and calling `rc = pthread_mutex_lock(&lock)` is really asking "can I acquire the lock?" If it succeeds, `rc` is 0 and the mutex is now owned; if `rc` is not 0, the lock was not acquired.

```c
int rc = pthread_mutex_lock(&lock);
if (rc != 0) {
    printf("lock failed with error: %d\n", rc);
    exit(1);
}
counter = counter + 1;
rc = pthread_mutex_unlock(&lock);
if (rc != 0) {
    printf("unlock failed with error: %d\n", rc);
    exit(1);
}
```

The dangerous alternative is to skip this check entirely:

```c
pthread_mutex_lock(&lock);
counter = counter + 1;
pthread_mutex_unlock(&lock);
```

If `pthread_mutex_lock` happens to fail here, meaning it returns non-zero, execution proceeds anyway, without actually holding the lock. Multiple threads can then enter the critical section at once, guaranteeing a race condition, with no indication anywhere that it happened, because the failure was never checked.

Writing this check every single time is repetitive, so a wrapper function is often used instead:

```c
void Pthread_mutex_lock(pthread_mutex_t *mutex) {
    int rc = pthread_mutex_lock(mutex);
    assert(rc == 0);
}
```

`assert(rc == 0)` means that if `rc` is 0, execution continues normally, and if `rc` is not 0, the program crashes immediately with an error message. Crashing immediately is far better than continuing with a broken lock and getting silent data corruption that would be nearly impossible to diagnose later.

### 5.6 Complete Example With a Lock

```c
#include <stdio.h>
#include <pthread.h>

int counter = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;  // global lock

void *mythread(void *arg) {
    int i;
    for (i = 0; i < 10000000; i++) {
        pthread_mutex_lock(&lock);    // take the lock
        counter = counter + 1;        // safe: only one thread here
        pthread_mutex_unlock(&lock);  // release the lock
    }
    return NULL;
}

int main() {
    pthread_t p1, p2;
    pthread_create(&p1, NULL, mythread, NULL);
    pthread_create(&p2, NULL, mythread, NULL);
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    printf("counter = %d\n", counter);   // always prints 20000000
    return 0;
}
```

With the lock in place, the two threads simply take turns: thread 1 locks, increments counter from 50 to 51, and unlocks; thread 2 waits, then locks, increments from 51 to 52, and unlocks; thread 1 locks again, increments from 52 to 53, and so on. The final result is always exactly 20,000,000, with no race condition. Without the lock, as shown in the previous post, thread 1 reads 50 and adds to get 51 but has not written it yet, thread 2 reads 50, which is still 50, adds to get 51, and writes 51, and then thread 1 writes its own stale 51, overwriting thread 2's work. The counter should be 52 but is 51, one increment lost, and the final result across ten million iterations can land anywhere from 10,000,000 to 20,000,000, unpredictably.

### 5.7 trylock and timedlock

A normal lock, acquired with `pthread_mutex_lock`, is a blocking operation: if the mutex is busy, the calling thread waits until it becomes free, potentially forever.

```c
pthread_mutex_lock(&lock);
```

| Situation | Behavior |
|---|---|
| Lock free | Take it immediately |
| Lock busy | Wait, possibly forever |

`pthread_mutex_trylock` is a non-blocking alternative. The thread tries to acquire the mutex exactly once, and if it is already locked, the call returns an error immediately instead of waiting. The key idea is simply "try once, no waiting":

```c
int rc = pthread_mutex_trylock(&lock);
if (rc == 0) {
    // got the lock
    counter++;
    pthread_mutex_unlock(&lock);
} else {
    // lock is held by someone else
    // do something else instead of waiting
    do_alternative_work();
}
```

| Situation | Result |
|---|---|
| Lock free | Thread gets the lock |
| Lock busy | Returns immediately with failure |

The important difference from a normal lock is exactly this: a normal lock responds to "busy?" by waiting, while `trylock` responds to "busy?" by returning immediately. If `pthread_mutex_trylock` succeeds, `rc` is 0 and the mutex was acquired; if it fails, `rc` is non-zero, usually meaning another thread already owns the mutex, and the calling thread can go do something else entirely instead of waiting.

`pthread_mutex_timedlock` sits between the two: it is a lock operation that waits, but only up to a specified time limit, and if the mutex does not become available before that timeout, the function fails. The idea is "wait, but not forever":

```c
pthread_mutex_timedlock(&lock, &timeout);
```

The `timeout` argument is a future point in time. For example, `time(NULL) + 5` means five seconds from now, so the call means "wait up to 5 seconds for the mutex."

```c
struct timespec timeout;
timeout.tv_sec = time(NULL) + 5;   // wait at most 5 seconds
timeout.tv_nsec = 0;

int rc = pthread_mutex_timedlock(&lock, &timeout);
if (rc == 0) {
    // got the lock within 5 seconds
    counter++;
    pthread_mutex_unlock(&lock);
} else {
    // 5 seconds passed, lock still held
    printf("gave up waiting for lock\n");
}
```

| Situation | Result |
|---|---|
| Lock becomes free before timeout | Get the lock |
| Timeout expires | Return failure |

This kind of bounded wait is especially useful for avoiding indefinite waiting, detecting deadlocks, and keeping responsive systems, servers, and real-time applications from getting stuck. Both `trylock` and `timedlock` are advanced tools; plain `pthread_mutex_lock` is the right choice in almost every ordinary case.

## 6. Condition Variables: Sleeping and Waking

### 6.1 The Problem Locks Cannot Solve

Locks prevent simultaneous access to shared data, but sometimes a thread needs to wait until something specific happens elsewhere. The classic scenario is a producer and a consumer: a producer thread creates work items and places them in a queue, much like a customer placing an order, while a consumer thread takes items from that queue and processes them, much like a waiter serving that order. The consumer should wait when the queue is empty, and it should wake up as soon as the producer adds something new.

### 6.2 The Naive Approach: Spin Loop

```c
// Consumer keeps checking until the queue has something
while (queue_empty()) {
    // do nothing, just check again
}
process_item();
```

This is called busy-waiting or spinning. It works, in the sense that it eventually notices new work, but it is a terrible way to do it. While the consumer spins, it burns 100 percent of a CPU core doing nothing useful at all, which starves the producer and every other process of that CPU time, drags down overall system performance, and hurts unrelated processes that simply wanted their fair share of the CPU. The right fix is to let the consumer sleep when there is nothing to do, and wake it up only when work actually arrives.

### 6.3 Condition Variables: The Correct Solution

A condition variable is an object that lets threads sleep until some condition becomes true, and lets other threads wake sleeping threads up once that condition changes. Continuing the restaurant analogy, the waiters are consumer threads, the customers placing orders are producer threads, the order board is the shared queue, the kitchen manager is the mutex lock, and the bell is the condition variable itself.

A waiter first locks access to the board:

```c
pthread_mutex_lock(&lock);
```

This prevents another waiter from modifying the board at the same moment, and then checks whether the board is empty. Instead of wasting energy checking over and over, the waiter sleeps instead:

```c
pthread_cond_wait(&cond, &lock);
```

This means "I am sleeping until someone rings the bell." Internally, the waiter releases the kitchen lock, goes to sleep, and is registered as waiting. Releasing the lock here is essential: if a sleeping waiter kept holding it, customers could never place new orders, and producers would be blocked forever, so going to sleep must release the lock.

When a customer places an order, the producer thread locks the same mutex, safely adds the order to the board, and then rings the bell:

```c
pthread_cond_signal(&cond);
```

This means "wake one sleeping waiter," after which the producer unlocks the mutex. The sleeping waiter wakes up, but before `pthread_cond_wait` actually returns, the waiter must re-acquire the lock, since it is about to touch the shared queue again and needs safe access to do so.

### 6.4 Understanding pthread_cond_wait

```c
pthread_cond_wait(&cond, &lock);
```

This call does three things as a single atomic operation: it releases the lock so other threads can proceed, it puts the calling thread to sleep, and once woken up, it re-acquires the lock before returning. The first two steps, releasing the lock and going to sleep, happen atomically together, so nobody can sneak a signal in between them.

That atomicity matters a great deal. Consider what would go wrong if it were not atomic: the consumer checks the condition, finds it not ready, and releases the lock, but before it actually falls asleep, the producer sneaks in, acquires the lock, sets the condition true, signals, and releases the lock again, all while nobody was actually sleeping yet to receive that signal. The consumer then goes to sleep and waits forever, having missed the one signal meant for it. With release-and-sleep happening atomically instead, the producer cannot signal until the consumer is genuinely, fully asleep, so the signal can never be missed this way.

### 6.5 Understanding pthread_cond_signal

```c
pthread_cond_signal(&cond);
```

This wakes up exactly one thread that is currently sleeping on this condition variable. If no thread happens to be sleeping on it at that moment, the signal is simply lost; it is not stored for some future waiter to pick up later.

### 6.6 Why while Instead of if

```c
// WRONG
if (ready == 0) {
    pthread_cond_wait(&cond, &lock);
}

// CORRECT
while (ready == 0) {
    pthread_cond_wait(&cond, &lock);
}
```

There are two separate reasons `while` is mandatory here rather than `if`.

The first is spurious wakeups. The most important thing to understand about `pthread_cond_signal` is that it does not mean "the condition is true now." It only means "something may have changed, wake up and check again." Beginners often imagine a signal as guaranteeing that exactly one thread wakes and the condition is now certainly true, but that is wrong: condition variables are weaker than that. A signal only ever means "maybe the state changed," and the thread itself must verify the condition again, which is exactly what the `while` loop does. Some pthread implementations can even wake a thread up without anyone calling `pthread_cond_signal` at all, a rare but real event called a spurious wakeup. Using `if` would let the thread proceed right after such a wakeup, assuming the condition is true when it might not be; using `while` makes it check again, find the condition still false, and go back to sleep correctly.

The second reason is multiple consumers. If several threads are waiting and only one item becomes ready, all of them might wake up once signaled, even though only one can actually take that item. Many people assume that one signal wakes exactly one thread and nothing more, but the reality is more subtle: scheduler races happen, multiple threads may run, broadcasts may occur, and by the time any particular thread actually runs, the condition may already be false again. Concretely, suppose three consumers are sleeping, the producer adds one item and signals, and all three happen to wake up. The first consumer takes the item. With `while`, the second consumer's loop checks again, finds no item left, and goes back to sleep correctly, and the third does the same. With `if`, all three would have proceeded anyway, even though there was only ever one item to give out. Always use `while`, never `if`, with condition variables.

### 6.7 The Complete Working Example

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

void *consumer(void *arg) {
    pthread_mutex_lock(&lock);
    while (ready == 0) {
        printf("Consumer: nothing ready, sleeping...\n");
        pthread_cond_wait(&cond, &lock);
        printf("Consumer: woke up, checking...\n");
    }
    printf("Consumer: got the item, processing!\n");
    pthread_mutex_unlock(&lock);
    return NULL;
}

void *producer(void *arg) {
    printf("Producer: preparing item...\n");
    // simulate work
    int i;
    for (i = 0; i < 1000000; i++) ;
    pthread_mutex_lock(&lock);
    ready = 1;
    printf("Producer: item ready, signaling consumer\n");
    pthread_cond_signal(&cond);
    pthread_mutex_unlock(&lock);
    return NULL;
}

int main() {
    pthread_t c, p;
    pthread_create(&c, NULL, consumer, NULL);
    pthread_create(&p, NULL, producer, NULL);
    pthread_join(c, NULL);
    pthread_join(p, NULL);
    return 0;
}
```

A possible run of this program prints:

```
Consumer: nothing ready, sleeping...
Producer: preparing item...
Producer: item ready, signaling consumer
Consumer: woke up, checking...
Consumer: got the item, processing!
```

The consumer sleeps efficiently, wasting no CPU at all, and the producer wakes it up exactly when there is actually something to do.

The textbook this series follows explicitly warns against ever going back to a spin loop instead:

```c
// NEVER DO THIS
while (ready == 0) {
    ;   // empty loop, just spinning
}
```

This wastes CPU completely, with the consumer burning 100 percent of a core checking over and over while other threads cannot run effectively. It also creates a race condition on `ready` itself: without a lock protecting it, reading and writing that variable from two threads at the same time is a data race with undefined behavior. Implementations built this way are also simply easy to get wrong; research on real code has found that a very large share of hand-rolled busy-wait implementations like this one contain bugs that are hard to detect. Condition variables exist precisely to avoid all of this, and they should be used instead.

## 7. Compiling Multi-Threaded Programs

### 7.1 The -pthread Flag

```
gcc -o myprogram myprogram.c -Wall -pthread
```

`-Wall` shows all compiler warnings, catching many common mistakes before the program is ever run, and should always be used. `-pthread` does two things at once: it links the pthreads library, where the pthread functions actually live, and it enables thread-safe compilation settings.

Functions like `pthread_create()`, `pthread_join()`, and `pthread_mutex_lock()` live inside a separate library called the POSIX Threads Library, usually `libpthread`. Your program only contains declarations for these functions, coming from `pthread.h`, during the compile phase. During linking, if the pthread library is missing, the linker cannot find the actual code and produces an error such as `undefined reference to pthread_create`.

You may sometimes see `gcc program.c -lpthread` instead. That flag only performs library linking; it does not enable all of the compiler's thread-related settings. This is why `-pthread` is preferred over `-lpthread`: `-pthread` handles both compilation behavior and linking together. Specifically, it tells the compiler that this program uses threads and that it must not make unsafe single-thread assumptions, adjusting its memory model assumptions, its thread-safe libc behavior, its atomicity expectations, and its handling of thread-local storage. This matters because compilers optimize code aggressively. For a single-threaded program, the compiler may assume that no other thread ever changes a given variable, but in a multithreaded program another thread can modify shared memory at any time, so the compiler has to behave differently to avoid optimizing away code that is actually necessary.

### 7.2 A Common Point of Confusion: Why -pthread and Not -stdio

One thing that can be confusing when first learning pthreads is this: functions like `printf()` from `<stdio.h>` work without passing any special flag to `gcc`, so why do multithreaded programs specifically need `-pthread`? More precisely, `stdio.h` is included with `#include <stdio.h>`, yet there is no equivalent `-stdio` flag, while POSIX threads specifically require `-pthread`. Why are thread libraries treated differently from standard libraries, and what does `-pthread` actually do internally that ordinary headers and libraries do not require?

The first key idea is that a header file like `#include <stdio.h>` does not contain the actual implementation of `printf()`. It typically contains only declarations, macros, and type definitions, for example a simplified line like `int printf(const char *format, ...);`, which merely tells the compiler that a function called `printf` exists, not where its actual machine code lives. The real implementation of `printf()` lives inside the C standard library, `libc`, and `gcc` automatically links that library for every program, behaving internally roughly as if it had been given `-lc` even when that flag was never typed. Functions like `printf`, `malloc`, `scanf`, `fopen`, and `exit` are used almost universally, so compiler designers decided to include the standard C library automatically, for convenience.

Threads are different. They were never part of the original core C language or its runtime; they are optional system functionality, provided separately by the POSIX threads library, and not every program needs them. Because of that, the compiler does not automatically include pthread support the way it automatically includes the standard C library, which is exactly why `-pthread` has to be requested explicitly.

## 8. Eight Rules for Writing Correct Thread Code

**Rule 1: Keep locking logic simple.** A scheme like locking A, then B, then C, doing work, and unlocking in reverse order is complex, easy to get wrong, and hard to reason about. Locking A, doing the work, and unlocking A is simple, clear, and obviously correct. Complex locking schemes create bugs that can take weeks to find, so if multiple locks genuinely seem necessary, it is worth thinking carefully about whether all of them are really required.

**Rule 2: Minimize thread interactions.** Every shared variable is a potential race condition, and every condition variable is a potential missed signal, so fewer interactions between threads mean fewer ways for things to go wrong. A design where thread A shares five variables with thread B, three more with thread C, and signals thread D in four different ways is far harder to reason about than a design where each thread has one clear job and one clear communication channel with each other thread it needs to talk to.

**Rule 3: Always initialize locks and condition variables.** Using an uninitialized lock, as in declaring `pthread_mutex_t lock;` and immediately calling `pthread_mutex_lock(&lock)` without initializing it first, is undefined behavior. Using `pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;` before locking it works correctly. Uninitialized locks cause bugs that appear randomly and are nearly impossible to diagnose after the fact.

**Rule 4: Always check return codes.** Ignoring the possibility of failure, as in calling `pthread_mutex_lock(&lock)` and moving on without checking anything, lets silent failures lead directly to race conditions: the exact bug the lock was meant to prevent is now guaranteed to happen, with no indication of why. Checking with `int rc = pthread_mutex_lock(&lock); assert(rc == 0);` catches this immediately instead.

**Rule 5: Never return stack pointers from thread functions.** Returning the address of a local variable, as in `int result = 42; return &result;`, is dangerous, since that stack memory is gone the instant the function returns. Allocating on the heap instead, as in `int *result = malloc(sizeof(int)); *result = 42; return result;`, is safe, since heap memory persists until it is explicitly freed. This is the same rule covered in section 4.4, restated here as part of the complete list.

**Rule 6: Remember that each thread has its own stack.** A local variable declared inside a thread function, such as `int local_x = 10;`, is private to that thread; no other thread can see it, and it is automatically safe from interference by other threads. Data that genuinely needs to be shared between threads has to live on the heap or in a global variable, and it must be protected by a lock.

**Rule 7: Always use condition variables for signaling between threads.** Spinning on a flag with `while (flag == 0) ;` wastes CPU and often hides subtle race conditions. Using a condition variable instead, as in `while (condition == 0) pthread_cond_wait(&cond, &lock);`, is both efficient and correct, for exactly the reasons covered in section 6.

**Rule 8: Read the manual pages.** Commands like `man pthread_create`, `man pthread_mutex_lock`, `man pthread_cond_wait`, and `man -k pthread`, which lists every pthread function available on the system, explain the exact behavior, every error code, and the edge cases that no single tutorial, including this one, can fully cover.

## Conclusion

This post turned the ideas from the previous one into working code. Creating a thread with `pthread_create` means handing over a function pointer and a `void *` argument, and the new thread begins running independently right away. Waiting for a thread with `pthread_join` means blocking until it finishes, and any data it needs to hand back must live on the heap, never on its own stack, since that stack disappears the moment the thread function returns. Locks, built from `pthread_mutex_t` together with `pthread_mutex_lock` and `pthread_mutex_unlock`, protect a critical section by letting only one thread inside at a time, and every lock must always be initialized, always be paired with an unlock, and always have its return code checked. Condition variables, built from `pthread_cond_t` together with `pthread_cond_wait` and `pthread_cond_signal`, solve the separate problem of waiting efficiently for something to happen, always paired with a mutex, and always checked in a `while` loop rather than an `if`, since a signal only ever means "maybe check again," never "the condition is now guaranteed true." Compiling any of this correctly requires the `-pthread` flag, which changes how the compiler reasons about memory, not just which library gets linked in.

This is also the final post in this series. Starting from raw C fundamentals, the series built up through what a process actually is and how the OS controls it, how the CPU executes instructions and manages its stack, how virtual memory turns physical RAM into a private illusion for every process, how I/O connects the CPU to the outside world, how the scheduler decides who runs next even without knowing how long anything will take, and finally how a single process can split itself into multiple threads and coordinate them safely. Every one of these pieces was really answering the same underlying question in a different part of the machine: how do you take something shared, limited, and messy, and make it feel private, plentiful, and simple to the program running on top of it. That is the whole of what an operating system does.
