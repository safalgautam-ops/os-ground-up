---
title: "Null Pointer, Dangling Pointer, Void Pointer, and Wild Pointer"
sidebar_position: 3
---

> Module 0 · Post 3 of 13

The previous post established what a pointer is and how it relates to arrays. This one addresses a narrower but equally important question: the specific ways a pointer can fail to point at anything useful. Four situations come up repeatedly, two of which are always bugs, one of which is a deliberate safety mechanism, and one of which is a genuine language feature. Being able to name them precisely is what makes the difference between reading a crash report and guessing at it.

## 1. Quick Recap

A pointer is a variable that holds a memory address.

```c
int x = 42;
int *p = &x;    // p holds the address of x
```

```
Address 1000: [ 42 ]     ← x lives here
Address 2000: [ 1000 ]   ← p lives here, holding the address of x
```

When `p` holds address 1000, it points to `x`. Writing `*p` means "go to address 1000 and retrieve what is stored there," which yields 42.

## 2. Null Pointer: Pointing at Nothing, Deliberately

### 2.1 What It Is

A null pointer is one that has been explicitly set to hold the value zero, meaning it deliberately points at nothing. It is a way of stating that this pointer has no valid target at present.

```c
int *p = NULL;    // p is a null pointer
```

`NULL` is defined in `<stdio.h>` and several other headers as 0 cast to a pointer type. On every machine, address 0 is reserved by the operating system and no real data is ever placed there, which is what makes 0 a universally safe choice for "points at nothing."

### 2.2 Why It Exists

Without null pointers, a function would have no standard way of signalling failure. Consider `fopen`, which returns a `FILE *`. If the file cannot be opened, what should it return? It cannot return a valid pointer, because there is no file. It returns `NULL`:

```c
FILE *fp = fopen("missing.txt", "r");
if (fp == NULL) {
    printf("file not found\n");
}
```

### 2.3 How It Is Used

The first common use is initialising a pointer you are not yet ready to use:

```c
int *p = NULL;             // a safe starting state

// later
p = malloc(sizeof(int));   // p now holds a real address
```

The second is checking whether a function succeeded:

```c
char *result = malloc(1000);
if (result == NULL) {
    // malloc failed, meaning the system is out of memory
}
```

### 2.4 The Critical Rule: Never Dereference NULL

```c
int *p = NULL;
*p = 10;            // crash: you instructed the OS to write to address 0
printf("%d", *p);   // crash: you instructed the OS to read from address 0
```

The operating system protects address 0, so touching it produces a segmentation fault. This is in fact desirable behaviour. It means null pointer bugs crash immediately and loudly rather than silently corrupting data elsewhere.

### 2.5 Holding NULL Is Perfectly Safe

The distinction worth internalising is that a pointer variable containing `NULL` is not itself a problem. The crash occurs only on dereference, that is, when you write `*p` or `p->member`.

```c
int *p = NULL;
if (p == NULL)       // safe: merely comparing the value held in p
    printf("null");
printf("%p", (void *)p);   // safe: merely printing the value
```

## 3. Dangling Pointer: Pointing at Memory That Is Gone

### 3.1 What It Is

A dangling pointer once pointed at valid memory, but that memory has since been freed or destroyed. The pointer nevertheless retains the old address. It still holds a number that was meaningful, but the memory at that address is no longer yours.

Unlike a null pointer, which you set intentionally, a dangling pointer is always a bug. It arises by accident.

### 3.2 Three Ways It Happens

**Returning the address of a local variable:**

```c
int *badFunction(void) {
    int local = 42;     // local lives on the stack
    return &local;      // returning the address of local
}                       // local is destroyed here; the frame is gone

int main(void) {
    int *p = badFunction();
    printf("%d", *p);   // p points at destroyed memory
}
```

When `badFunction` returns, its entire stack frame is released. The memory at that address now belongs to nobody, or to whichever function is called next. `p` still holds the old address, but what resides there is garbage, or whatever the next function happened to place there.

**Using memory after freeing it:**

```c
int *p = malloc(sizeof(int));
*p = 42;

free(p);              // memory returned to the allocator
printf("%d", *p);     // danger: p still holds the old address
*p = 99;              // danger: writing to freed memory may corrupt
                      // the allocator's own bookkeeping
```

After `free(p)`, the memory is handed back and `malloc` may reuse it for a future allocation. Writing to it now corrupts something else in your program, silently and without any error message.

**Holding a pointer into a block that is later freed:**

```c
int *arr = malloc(5 * sizeof(int));
int *p = &arr[2];     // p points into the array

free(arr);            // the whole array is freed
*p = 10;              // danger: the memory for arr[2] is gone
```

This case is easy to overlook, because `p` was never passed to `free` itself. Freeing the block invalidates every pointer into it, not just the one you happened to pass.

### 3.3 Why Dangling Pointers Are the Worst Kind of Bug

Null pointer bugs crash immediately, because the operating system stops you at address 0. Dangling pointer bugs behave far less predictably:

```
Null pointer:      always crashes immediately, so it is easy to locate

Dangling pointer:  might appear to work, if the old data is coincidentally intact
                   might crash later, once the memory is reused
                   might corrupt data silently, having overwritten something else
```

The failure may surface much later and in entirely unrelated code, which makes the original cause extremely difficult to trace.

### 3.4 How to Avoid Them

**Set pointers to NULL immediately after freeing:**

```c
free(p);
p = NULL;    // an accidental later use now crashes immediately,
             // at the correct location rather than silently elsewhere
```

This converts an undetectable bug into a detectable one, which is the whole point.

**Never return the address of a local variable:**

```c
// Wrong
int *wrong(void) {
    int x = 5;
    return &x;   // x is destroyed when the function returns
}

// Right: return the value rather than the address
int right(void) {
    int x = 5;
    return x;    // the value is copied out
}

// Also right: if a pointer must be returned, allocate on the heap
int *alsoRight(void) {
    int *p = malloc(sizeof(int));
    *p = 5;
    return p;    // heap memory persists after the function returns
}
```

**Null out pointers to freed blocks as well:**

```c
free(arr);
arr = NULL;
```

## 4. Void Pointer: A Valid Address Without a Type

### 4.1 What It Is

A void pointer, written `void *`, holds a valid memory address but carries no type information. It states that there is something at this address without specifying what type that something is.

```c
void *p;    // capable of holding the address of anything
```

Every other pointer type is specific:

```c
int    *ip;   // points to an int
char   *cp;   // points to a char
double *dp;   // points to a double
```

`void *` is the generic form and can point at any of these.

### 4.2 Why It Exists

Some functions genuinely cannot know what type they are dealing with. `malloc` is the clearest example:

```c
void *malloc(size_t n);
```

`malloc` allocates `n` bytes. It has no idea whether you intend to store integers, doubles, structures, or anything else. It supplies raw memory, so it returns `void *`, in effect saying "here is your memory, you decide what it holds."

Without `void *`, a separate allocator would be required for every type, which is plainly unworkable:

```c
int    *intmalloc(size_t n);
double *doublemalloc(size_t n);
// and so on, indefinitely
```

### 4.3 Converting To and From `void *`

In C, `void *` may be assigned to any pointer type and any pointer type may be assigned to `void *`, automatically and without a cast. C++ differs here and requires an explicit cast.

```c
int x = 42;
int *ip = &x;

void *vp  = ip;    // int * to void *: automatic in C
int  *ip2 = vp;    // void * to int *: automatic in C

int *ip3 = (int *) malloc(sizeof(int));   // the cast is optional in C
```

### 4.4 The Critical Restriction: `void *` Cannot Be Dereferenced

Dereferencing a pointer means accessing the value stored at the address it holds. A `void *` cannot be dereferenced, because the compiler does not know what type resides there, and therefore knows neither how many bytes to read nor how to interpret them.

```c
void *vp = &x;
*vp = 10;           // illegal: compiler error
printf("%d", *vp);  // illegal: compiler error
```

You must first convert it back to a specific type:

```c
void *vp = &x;
int *ip = (int *)vp;   // inform the compiler that it is an int
*ip = 10;              // now legal: the compiler knows to write 4 bytes
```

## 5. Wild Pointer: Never Initialised at All

### 5.1 What It Is

A wild pointer has never been initialised. It contains whatever bytes happened to occupy that memory location when the program started, and it therefore points at some completely unknown address.

```c
int *p;      // wild: p contains garbage
             // it might be 0, or 4829472, or anything at all
*p = 10;     // writing to a completely random memory location
```

That is the entire definition. A pointer that was never given a value.

### 5.2 How It Happens

Through one simple omission: declaring a pointer and forgetting to initialise it.

```c
int         *p;   // wild
char        *s;   // wild
struct node *n;   // wild
```

Local variables in C are not automatically zeroed. Whatever bytes previously occupied that stack memory become the value of your variable, and for a pointer those arbitrary bytes are interpreted as a memory address.

```c
void someFunction(void) {
    int *p;              // p might contain 0x7fff3a82, or any garbage
    printf("%d", *p);    // reads from that random address
}
```

### 5.3 Why They Are Especially Dangerous

The core problem is that they sometimes appear to work.

```c
int *p;        // wild pointer
*p = 42;       // write 42 to a random location
```

Three outcomes are possible. The address may fall in protected memory, in which case the operating system terminates the program. This is actually the best outcome, being both fast and obvious. Alternatively the address may land inside your program's own data, silently overwriting some other variable; the program continues running but produces incorrect results, and the visible symptom appears somewhere entirely unrelated. Or the address may land in unused memory, so the program works this run and crashes on the next one when the memory layout differs. That last case is the hardest of all to diagnose, because the fault appears intermittently.

### 5.4 The Garbage Value Changes Between Runs

With a null pointer, `p` is always 0, identically every time. With a wild pointer, the garbage value depends on whatever previously occupied that region of the stack:

```c
void someFunction(void) {
    int *p;                      // whatever was here before
    printf("%p\n", (void *)p);   // prints a different value every run
}
```

This is what makes the bug so difficult to reproduce reliably.

The cast in that example is worth explaining. The `%p` format specifier in `printf` expects a `void *`, the generic pointer type, whereas `p` has type `int *`. Writing `(void *)p` performs the required conversion.

### 5.5 Wild Versus Dangling

These two are frequently confused, and the distinction is worth stating plainly:

```
Wild pointer:      NEVER held a valid value. Uninitialised from birth.
Dangling pointer:  HELD a valid address, then lost it when the memory
                   was freed or destroyed.
```

Both are always bugs, and both are hard to detect. The difference is purely one of history.

### 5.6 The Fix: Always Initialise

The entire problem disappears with a single habit: never declare a pointer without giving it a value.

```c
// If you have something to point at:
int x = 42;
int *p = &x;                     // initialised to the address of x

// If you are allocating memory:
int *p = malloc(sizeof(int));    // initialised to a heap address

// If you are not ready to use it yet:
int *p = NULL;                   // safe to hold; check before using
```

The rule is that the moment you write `int *p`, you must also write `= something` on the same line. There are no exceptions worth making.

## 6. All Four Side by Side

The four situations are best understood by asking a single question of each: what is actually stored inside the pointer variable?

```
WILD                              NULL
int *p;                           int *p = NULL;
┌──────────────────┐              ┌──────────────────┐
│ p: 0x7ff3a9??    │              │ p: 0             │
│    GARBAGE       │              │    KNOWN         │
└──────────────────┘              └──────────────────┘
 → a random address                → address 0
 → never initialised               → intentional
 → the worst bug type              → checkable


DANGLING                          VOID
free(p), then use p               void *p = &x;
┌──────────────────┐              ┌──────────────────┐
│ p: 2000          │              │ p: 3000          │
│    STALE         │              │    VALID         │
└────────┬─────────┘              └────────┬─────────┘
         │                                 │
         ▼                                 ▼
   ┌ ─ ─ ─ ─ ─ ─ ┐                 ┌──────────────┐
     addr 2000                     │  addr 3000   │
   │   FREED     │                 │  real data   │
    ─ ─ ─ ─ ─ ─ ─┘                 └──────────────┘
```

Summarised as a table:

| Pointer | Value held | Memory it refers to | Detectable | Intent |
|---|---|---|---|---|
| Wild | Garbage | Unknown | No | Accident, always a bug |
| Null | 0, and known | None | Yes | Intentional, and safe |
| Dangling | A stale address | Freed | No | Accident, always a bug |
| Void | A valid address | Valid | Cast, then use | Intentional, a feature |

The column that matters most is detectability. Null and void pointers can be reasoned about: a null pointer can be tested with a comparison before use, and a void pointer simply requires a cast before dereferencing. Wild and dangling pointers offer nothing to test against. In both cases the pointer holds a number that looks exactly like a legitimate address, and no comparison you could write would reveal otherwise. That is precisely why the discipline of initialising every pointer at declaration, and nulling every pointer after freeing, matters so much. Both habits exist to convert an undetectable failure into a detectable one.

## Conclusion

Four situations, two of which are features and two of which are faults. A null pointer is a deliberate statement that there is no target at present, and it fails loudly and immediately if dereferenced. A void pointer is a valid address stripped of its type, which is what allows a single `malloc` to serve every type in the language. A dangling pointer once had a valid target and has outlived it. A wild pointer never had one at all.

The two bugs share a common shape and a common remedy. Both leave the pointer holding a plausible-looking address that no test can distinguish from a real one, and both are prevented by discipline at the point of declaration and the point of release rather than by checking at the point of use.

That completes the C groundwork. We now have the vocabulary needed to talk about memory precisely: addresses, ownership, lifetimes, and what it means for memory to belong to something. From here we turn to the operating system itself, and to the question of what actually happens between a program sitting inert on disk and that same program running.

**Next:** [Part 1: The Process](../02-processes/01-the-process.md)
