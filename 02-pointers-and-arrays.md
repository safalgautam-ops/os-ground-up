---
title: "Pointers and Arrays in C"
sidebar_position: 2
---

> Module 0 · Post 2 of 13

Pointers are among the most powerful features of C, and also among the most confusing for beginners. The difficulty is that they deal with memory directly, which is something we rarely think about when writing ordinary code.

This post builds the concept step by step, beginning with simple variables and moving gradually toward pointers and arrays. Rather than memorising rules, the aim is to understand how things actually work inside memory, so that the rules follow naturally. By the end, the deep connection between arrays and pointers in C should be clear, and that connection is what the rest of this series depends on.

## 1. What Is a Pointer?

Before discussing pointers, consider what happens with an ordinary variable:

```c
int x = 10;
```

When you write this, the machine finds a free location in memory, stores the value 10 there, and records that the name `x` refers to that location.

Every location in memory has an address. It is helpful to think of memory as a long street where each house has a unique number. Suppose `x` lives at address 5000. The value stored at address 5000 is 10.

```
Address:  5000
Value:    10
Name:     x
```

A pointer is simply a variable that stores an address rather than an ordinary value.

```c
int x = 10;
int *p;      // p is a pointer; it will hold an address
p = &x;      // & means "give me the address of x"
             // p now holds 5000
```

The situation is now:

- `x` holds 10
- `p` holds 5000, the address of `x`
- `*p` means "go to address 5000 and read the value stored there," which gives 10

### 1.1 The Two Jobs of `*`

The asterisk performs two different jobs, and conflating them is the single most common source of early confusion.

**In a declaration, it announces a pointer:**

```c
int *p;    // declares p as a pointer to int
```

**In an expression, it dereferences:**

```c
*p = 20;   // go to wherever p points and store 20 there
           // this changes x to 20
```

It is worth stating the corollary explicitly, because it is frequently misremembered. The pointer `p` stores the address. The expression `*p` never stores an address; it means "go to the address held in `p` and access the value found there."

## 2. Why Pointers Exist

C passes everything to functions by value, which is to say it passes a copy.

```c
void tryToDouble(int n) {
    n = n * 2;   // modifies only the copy, not the original
}

int main(void) {
    int x = 5;
    tryToDouble(x);
    printf("%d", x);   // still prints 5
}
```

This is the fundamental limitation. Without pointers, a function cannot modify a variable belonging to its caller.

The solution is to pass the address instead of the value:

```c
void actuallyDouble(int *n) {   // accept an address
    *n = *n * 2;                // go to that address and double what is there
}

int main(void) {
    int x = 5;
    actuallyDouble(&x);         // pass the address of x
    printf("%d", x);            // prints 10
}
```

This is precisely why the familiar `swap` function must be written with pointers:

```c
// Incorrect: swaps copies only
void swap(int x, int y) {
    int temp = x;
    x = y;
    y = temp;
    // x and y are local copies; the caller's variables are untouched
}

// Correct: swaps the actual values
void swap(int *px, int *py) {
    int temp = *px;   // save the value at address px
    *px = *py;        // put the value at py into address px
    *py = temp;       // put the saved value into address py
}

int a = 3, b = 7;
swap(&a, &b);         // pass addresses
// a is now 7, b is now 3
```

The general rule is that you pass an address when the function expects a pointer, and particularly when you intend the function to modify the original data.

## 3. Reading Pointer Declarations

```c
int    *ip;   // ip is a pointer to int
double *dp;   // dp is a pointer to double
char   *cp;   // cp is a pointer to char
```

The reliable way to read these is to begin at the variable name, read `*` as "pointer to," and then read the type. So `int *ip` reads as "ip is a pointer to int."

## 4. Pointers and Arrays: The Deep Connection

This is where C becomes both powerful and initially disorienting. Consider an array declaration:

```c
int a[5] = {10, 20, 30, 40, 50};
```

The elements are laid out consecutively in memory:

```
Address: 1000  1004  1008  1012  1016
Value:     10    20    30    40    50
Index:    [0]   [1]   [2]   [3]   [4]
```

This example assumes each `int` occupies 4 bytes.

The key fact is that the name `a` is the address of the first element. To be precise, in most expressions `a` becomes `&a[0]`. This conversion is called array-to-pointer decay. Note carefully that `a` is not itself a pointer; it merely converts to one in most contexts.

```c
a == &a[0]      // identical
int *pa = a;    // pa points to a[0], same as: int *pa = &a[0];
```

`pa` now holds address 1000.

### 4.1 Pointer Arithmetic

When you evaluate `pa + 1`, C does not add 1 to the address. It adds `1 * sizeof(int)`, which is 4. So `pa + 1` yields address 1004, which is `a[1]`.

This scaling happens automatically according to the pointed-to type. A `double *` would advance by 8 bytes, a `char *` by 1. The compiler handles the arithmetic for you.

```c
*pa        // value at a[0], which is 10
*(pa + 1)  // value at a[1], which is 20
*(pa + 2)  // value at a[2], which is 30
*(pa + i)  // value at a[i]
```

Stated as a principle: adding a number to a pointer moves it forward in memory, but not by that number of bytes. It moves by that number multiplied by the size of the type it points to.

### 4.2 The Central Equivalence

```c
a[i]   ==  *(a + i)   // identical
&a[i]  ==  a + i      // identical
```

C literally converts `a[i]` into `*(a + i)` internally. Square bracket notation is nothing more than a more readable spelling of pointer arithmetic.

The consequence is that array notation works on pointers as well:

```c
pa[0]  ==  *pa        ==  a[0]
pa[1]  ==  *(pa + 1)  ==  a[1]
pa[2]  ==  *(pa + 2)  ==  a[2]
```

## 5. One Critical Difference: Array Name Versus Pointer

An array name resembles a pointer but is not a variable. It is a fixed address.

```c
int a[5];
int *pa = a;

pa = pa + 1;   // legal: pa is a variable and can be changed
pa++;          // legal

a = pa;        // illegal: a is not a variable and cannot be assigned to
a++;           // illegal
```

It helps to think of `a` as a constant pointer whose address is locked in place, whereas `pa` is free to move.

## 6. Arrays as Function Parameters

The governing rule is that arrays decay into pointers when passed to functions. When you write:

```c
int a[5] = {10, 20, 30, 40, 50};
someFunction(a);
```

what actually happens is equivalent to:

```c
someFunction(&a[0]);
```

Not all five elements are passed, and no copy is made. A single address crosses into the function, namely the address of `a[0]`. The function receives a pointer.

This is why the following two declarations are identical as far as the compiler is concerned:

```c
void myFunc(int arr[], int size);   // "arr is an array of int"
void myFunc(int *arr, int size);    // "arr is a pointer to int"
```

The first form claims that `arr` is an array, but consider whether it could actually be one inside the function. The array itself lives in the caller's memory. A parameter is a single variable and cannot hold five integers in one parameter slot. The function received only an address, a single number. There is no array present to declare.

The C standard resolves this contradiction with a specific rule: in a function parameter list only, `int arr[]` is automatically rewritten as `int *arr` before anything else happens. The empty brackets carry no meaning as a parameter, so the compiler discards them and substitutes `*`. This is what is meant when people say there is no difference in the compiled code. The compiler never sees `int arr[]` as a parameter at all, because the conversion happens first.

This rule applies only in function parameters. Everywhere else, `[]` and `*` remain entirely different things.

### 6.1 `sizeof` Exposes What Actually Happened

The clearest demonstration that the parameter is a pointer rather than an array comes from `sizeof`:

```c
int a[5];
printf("%zu", sizeof(a));       // prints 20 (5 ints of 4 bytes each)
                                // sizeof sees a genuine array

void f(int arr[]) {
    printf("%zu", sizeof(arr)); // prints 8 (the size of a pointer)
                                // sizeof sees a pointer, not an array
}
```

Inside the function, `sizeof(arr)` reports the size of a pointer, which is 8 bytes on a typical 64-bit machine, not the size of the array. The array is gone. Only its address arrived. This is also why array parameters must always be accompanied by a separate size argument: the function has no way of recovering the length on its own.

Since `arr` is a pointer inside the function, both notations work interchangeably:

```c
void printAll(int *arr, int size) {
    for (int i = 0; i < size; i++) {
        printf("%d\n", arr[i]);     // the compiler converts this to *(arr + i)
        printf("%d\n", *(arr + i)); // the same thing, written explicitly
    }
}
```

### 6.2 The Subarray Technique

Because only an address is passed, you can pass a slice of an array without copying anything:

```c
int a[10] = {0,1,2,3,4,5,6,7,8,9};

printAll(a,     10);   // begins at a[0], covers all 10
printAll(a + 3, 7);    // begins at a[3], covers 7 elements
printAll(&a[3], 7);    // exactly the same thing in different notation
```

`a + 3` is simply the address of `a[3]`. The function has no idea it has received the middle of a larger array; it sees a pointer and a count. This is how slices are passed in C without copying.

## 7. `strlen` and Pointer Subtraction

The goal is to count the characters in a string until the terminating `'\0'` is reached. Here is the version built on pointer subtraction:

```c
int strlen(char *s)     /* s is a pointer to char; it holds an address */
{
    char *p = s;        /* p points to the first character */
    while (*p != '\0')
        p++;
    return p - s;
}
```

The insight is that `p` and `s` are both pointers into the same array. `s` remains frozen at the beginning while `p` walks forward until it reaches `'\0'`. Subtracting two pointers into the same array yields the count of elements between them.

```
String "hello\0" in memory:

 'h'    'e'    'l'    'l'    'o'    '\0'
2000   2001   2002   2003   2004   2005
  ▲                                  ▲
  │                                  │
  s                                  p
never moves                    walked to here

           p - s = 2005 - 2000 = 5
```

Note carefully that pointer subtraction gives the number of elements between the two pointers, not the number of bytes. For a `char *` each element occupies 1 byte, so the address difference happens to equal the character count. For an `int *` the compiler would divide the address difference by 4. This scaling is automatic, exactly as it is for pointer addition.

If the distinction between a pointer and the value it refers to still feels unstable, this reformulation often helps:

```c
// This single line:
char *p = s;

// means exactly the same as these two lines:
char *p;   // declare p as a pointer
p = s;     // store the address held in s into p
```

The declaration syntax puts the `*` next to `p`, which makes it look as though something is being assigned to `*p`. It is not. The address is being stored in `p` itself.

## 8. Character Pointers and Strings

Strings in C are arrays of characters terminated by `'\0'`.

```c
char amessage[] = "hello";   // an array: h, e, l, l, o, \0
char *pmessage  = "hello";   // a pointer to a string literal
```

These two lines look almost identical and behave very differently. The difference concerns who owns the memory and where that memory lives.

```
char amessage[] = "hello";

    your stack memory, which you own
    ┌───┬───┬───┬───┬───┬────┐
    │ h │ e │ l │ l │ o │ \0 │      writable
    └───┴───┴───┴───┴───┴────┘
    amessage is the array itself


char *pmessage = "hello";

    pmessage (a variable on the stack)
    ┌──────────┐
    │  9000    │  holds an address
    └────┬─────┘
         │ points into
         ▼
    read-only memory belonging to the program
    ┌───┬───┬───┬───┬───┬────┐
    │ h │ e │ l │ l │ o │ \0 │      NOT writable
    └───┴───┴───┴───┴───┴────┘
```

With `amessage` you own the memory and may change individual characters:

```c
amessage[0] = 'H';   // yields "Hello"; entirely fine
```

With `pmessage` you point at a string literal stored in read-only memory. Modifying it is undefined behaviour, which in practice means a crash or something worse:

```c
pmessage[0] = 'H';    // dangerous: undefined behaviour
pmessage = "world";   // fine: moving the pointer itself is permitted
```

The quick mental check is this. If you declared it with `[]`, you own the memory and may write to it. If you declared it with `*` pointing at a literal in double quotes, you are borrowing read-only memory. Read it, but do not modify it.

## 9. `strcpy`: Three Versions, One Growing Insight

The progression from the first version to the third is a lesson in how experienced C programmers come to think, and it rewards careful reading.

### 9.1 Version One: Index-Based

```c
void strcpy(char *s, char *t)
{
    int i = 0;
    while ((s[i] = t[i]) != '\0')
        i++;
}
```

The subtlety is the assignment `s[i] = t[i]` sitting inside the loop condition. An assignment in C evaluates to the value that was assigned. If `t[i]` holds `'h'`, the assignment stores `'h'` into `s[i]` and also yields `'h'`. The condition then tests whether that yielded value is `'\0'`. When `'\0'` is finally copied, the condition evaluates to 0, which is false, and the loop ends.

### 9.2 Version Two: Pointer-Based and Explicit

```c
void strcpy(char *s, char *t)
{
    while ((*s = *t) != '\0') {
        s++;
        t++;
    }
}
```

The logic is identical. Instead of an index `i`, the pointers themselves advance. `*s = *t` copies the character at `t` into the location at `s`, and both pointers then step forward.

### 9.3 Version Three: The Idiomatic Form

```c
void strcpy(char *s, char *t)
{
    while (*s++ = *t++)
        ;
}
```

This appears cryptic at first. Recall that `while (condition)` continues when the condition is non-zero and stops when it is zero. Ordinary characters such as `'A'` and `'B'` have non-zero ASCII values, so the loop continues. The terminator `'\0'` has the value 0, so the loop stops. The line is therefore equivalent to:

```c
while ((*s++ = *t++) != '\0')
```

Tracing the execution makes it concrete. Initially `s` points at the destination and `t` at the source. Evaluating the condition performs three actions in one expression: it copies the character at `t` into the location at `s`, advances `s` to the next position, and advances `t` to the next position. The value copied is also the value the assignment yields, so it serves as the loop condition. If that value is not `'\0'`, the loop continues.

```
initial:   s → [ _ ][ _ ][ _ ][ _ ][ _ ][ _ ]
           t → [ h ][ e ][ l ][ l ][ o ][ \0 ]

step 1:    copy 'h', both advance, 'h' is non-zero → continue
           s → [ h ][ _ ][ _ ][ _ ][ _ ][ _ ]
                      ^
step 2:    copy 'e', both advance, continue
...
step 6:    copy '\0', both advance, value is 0 → loop exits
           s → [ h ][ e ][ l ][ l ][ o ][ \0 ]
```

The terminator is copied before the loop exits, which is essential. The destination ends up properly terminated rather than merely holding the visible characters.

## 10. Arrays of Pointers

First, read the declaration without alarm:

```c
char *names[5];   // an array of 5 pointers, each pointing to char
```

Read it as "`names` is an array of 5 elements, each element being a `char *`." Each slot holds a pointer, and each pointer refers to a string.

The reason this arrangement beats a two-dimensional array for storing strings is memory efficiency:

```
2D array: char m[12][10]        Pointer array: char *m[12]

┌──────────┬───┐                 m[0] ──→ "January\0"    (8 bytes)
│ January  │▒▒▒│  padded         m[1] ──→ "February\0"   (9 bytes)
├──────────┼───┤                 m[2] ──→ "March\0"      (6 bytes)
│ February │▒▒▒│  padded         ...
├──────────┼───┤
│ March    │▒▒▒│  padded         each string occupies
├──────────┼───┤                 exactly what it needs
│ ...      │▒▒▒│
└──────────┴───┘
every row padded to 10,          no padding, and no fixed
and no string may exceed it      upper limit on length
```

With a two-dimensional array, every row is padded out to the maximum length, which wastes bytes and imposes a hard ceiling on how long any string may be. With an array of pointers, each string occupies exactly as many bytes as it requires. There is no waste and no artificial limit.

Access follows the pattern established earlier. `months[i]` yields the `char *` pointer, `months[i][j]` yields character `j` of string `i`, and `*months[i]` yields the first character.

```c
char *months[] = {
    "January", "February", "March", "April",
    "May", "June", "July", "August",
    "September", "October", "November", "December"
};

months[0]      // "January", a char pointer
*months[0]     // 'J', the first character
months[0][2]   // 'n', the third character of January
```

## 11. One-Dimensional Arrays in Memory

```c
int a[4] = {10, 20, 30, 40};
```

Memory is a long strip of numbered bytes, and the array occupies a consecutive chunk of it:

```
Byte address:  1000 1001 1002 1003 | 1004 1005 1006 1007 | 1008 ...
               [     int 10      ] | [     int 20      ] | [ int 30 ...
```

Each `int` occupies 4 bytes, so:

- `a[0]` holds 10 and lives at address 1000
- `a[1]` holds 20 and lives at address 1004
- `a[2]` holds 30 and lives at address 1008
- `a[3]` holds 40 and lives at address 1012

Finding element `i` requires a simple formula:

```
address of a[i] = base + (i * sizeof(int))
                = 1000 + (i * 4)
```

The compiler needs `sizeof(int)`, which it always knows from the type, and it needs `i`. Nothing else. With one dimension there is no difficulty.

## 12. Multi-Dimensional Arrays

The most important point to establish first is that a two-dimensional array in C is stored as one flat block of memory, row after row, left to right.

```c
int a[3][4];
```

This is not a grid in memory. The grid picture is useful for reasoning but wrong about the actual layout. What exists is 12 consecutive integers:

```
a[0][0] a[0][1] a[0][2] a[0][3] a[1][0] a[1][1] a[1][2] a[1][3] a[2][0] a[2][1] a[2][2] a[2][3]
└──────────── row 0 ───────────┘└──────────── row 1 ───────────┘└──────────── row 2 ───────────┘
```

### 12.1 How C Locates `a[i][j]`

C stores the array in row-major order, meaning rows are laid out one after another. With `sizeof(int)` equal to 4 and four columns per row, each row occupies 16 bytes:

```
Row 0 → 4 elements → 16 bytes
Row 1 → 4 elements → 16 bytes
Row 2 → 4 elements → 16 bytes
```

To reach row `i`, you skip `i` complete rows:

```
address of row i = base + (i * row_size)
where row_size = number_of_columns * sizeof(int) = 4 * 4 = 16
```

If the base address is 1000:

```
Row 0 → 1000 + (0 * 16) = 1000
Row 1 → 1000 + (1 * 16) = 1016
Row 2 → 1000 + (2 * 16) = 1032
```

Having reached row `i`, you still need to move `j` elements into it, which is `j * 4` bytes. Combining both movements:

```
address = base + (i * 16) + (j * 4)
```

Factoring out `sizeof(int)`:

```
= base + (i * 4 * 4) + (j * 4)
= base + (i * 4 + j) * 4
```

Verifying with `a[1][2]`:

```
= 1000 + (1 * 4 + 2) * 4
= 1000 + 6 * 4
= 1000 + 24
= 1024
```

Checking this against the layout above confirms that `a[1][2]` sits at address 1024.

The critical quantity in that formula is the number of columns, because it is what tells C how wide each row is. Without it the formula cannot be evaluated:

```
address of a[i][j] = base + (i * ? + j) * sizeof(int)
```

If the column count is unknown, the `?` is unknown and the address cannot be computed. This is not a design choice or an arbitrary restriction. It is arithmetic. You cannot skip `i` rows without knowing how wide a row is.

### 12.2 What Is Actually Passed to a Function

You already know from one-dimensional arrays that passing an array means passing the address of the first element. The same holds for two-dimensional arrays, but the phrase "first element" now needs care.

```c
int a[3][4];
```

What is the first element here? It is not `a[0][0]`, which is a single `int`. The first element is `a[0]`, which is an entire row of four integers.

So `a` is not a pointer to `int`. It is a pointer to an array of four integers. That distinction matters enormously, because it is exactly what carries the row width across into the function.

```
int a[3][4];

  a          decays to      pointer to "array of 4 int"
  │                          │
  ▼                          ▼
┌──────────────┐          ┌──────────────┐
│ row 0: 4 int │  ◄────── │ the type itself encodes
├──────────────┤          │ "each step forward
│ row 1: 4 int │          │  moves 4 ints = 16 bytes"
├──────────────┤          └──────────────┘
│ row 2: 4 int │
└──────────────┘
```

When you call `f(a)`, what crosses into the function is a pointer whose type states that it points to a group of four integers. The column count travels along inside the type.

### 12.3 The Three Valid Declarations

```c
void f(int a[3][4]) { ... }   // form 1
void f(int a[][4])  { ... }   // form 2
void f(int (*a)[4]) { ... }   // form 3
```

All three compile to identical machine code. Notice what is common to all of them: the number 4 appears in every case. The first dimension may be omitted because it is never needed for the address arithmetic, but the column count can never be omitted.

The parentheses in the third form are not optional, and dropping them produces something entirely different:

```
int  *a[4]     []  binds before *  (higher precedence)
               →  an array of 4 pointers to int

int (*a)[4]    () forces * to bind first
               →  a pointer to an array of 4 ints
```

The first is four separate pointers, each of which could point anywhere. The second is one pointer to a contiguous block of four integers, which is what a decayed row of a 2D array actually is. Only the second form is correct here.

## 13. Pointers to Functions

Here is the conceptual leap. Functions also live in memory, which means they have addresses. You can store such an address in a pointer and call the function through it later.

```c
int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }

int (*fp)(int, int);   // fp is a pointer to a function
                       // taking two ints and returning int

fp = add;              // store the address of add in fp
fp(3, 4);              // calls add, returning 7

fp = mul;              // fp now points to mul
fp(3, 4);              // calls mul, returning 12
```

Reading function pointer declarations is arguably the hardest syntax in C. The rule is to read outward from the name, respecting parentheses:

```
int (*fp)(int, int);
     ^
     1. start at the name:            fp
    ^^^^
     2. inside the parentheses, *:    fp is a pointer
         ^^^^^^^^^^
     3. followed by a parameter list: ...to a function taking (int, int)
^^^
     4. and the leading type:         ...returning int

     "fp is a pointer to a function taking two ints and returning int"
```

The parentheses around `*fp` are essential. Without them the declaration means something entirely different:

```
int  *fp(int, int);    // fp is a FUNCTION returning a pointer to int
int (*fp)(int, int);   // fp is a POINTER to a function returning int
```

The reason is precedence. Function-call parentheses bind more tightly than `*`, so in the first line `fp` attaches to the parameter list first and becomes a function. Adding parentheses around `*fp` forces the pointer interpretation to be applied first.

## 14. Command-Line Arguments

Every C program's `main` can receive arguments:

```c
int main(int argc, char *argv[]) { ... }
```

`argc` is the count of arguments, and it is always at least 1 because the program name itself counts. `argv` is an array of pointers to char.

The declaration deserves unpacking. `argv[]` indicates an array, and `char *` indicates that each element is a pointer to char. Since a `char *` pointing at a sequence terminated by `'\0'` is what C calls a string, an array of pointers to char is an array of strings.

Running `./myprogram hello world` produces:

```
argc = 3
argv[0] = "./myprogram"
argv[1] = "hello"
argv[2] = "world"
argv[3] = NULL      (guaranteed by the standard)
```

A simple use:

```c
int main(int argc, char *argv[]) {
    for (int i = 1; i < argc; i++)   // start at 1 to skip the program name
        printf("%s\n", argv[i]);
}
```

## 15. Common Beginner Mistakes

### 15.1 Uninitialised Pointers

```c
int *p;
*p = 10;
```

Understanding why this is dangerous requires being precise about what `int *p` does and does not do. It creates the pointer variable `p`, but no address has been written into it, so it contains garbage, meaning some arbitrary address. The statement `*p = 10` then says "go to the address held in `p` and store 10 there." In effect you are instructing the program to go somewhere unknown in memory and modify it.

Three outcomes are possible:

```
Case 1: the address belongs to other data in your program
        → you silently corrupt another variable
        → the program misbehaves with no error message

Case 2: the address belongs to the OS or protected memory
        → immediate crash (segmentation fault)

Case 3: the address happens to be 0 (NULL)
        → crash on most systems (null dereference)
```

The first case is the most dangerous, because nothing announces that anything went wrong.

The fix is to give the pointer a real address before using it:

```c
int x;
int *p = &x;   // p holds the address of x, a real and safe location
*p = 10;       // writes 10 into x, perfectly well defined
```

### 15.2 Off-by-One with Arrays

```c
int a[5];
a[5] = 99;
```

Arrays in C are zero-indexed. `int a[5]` provides slots `a[0]` through `a[4]`. The number in the declaration is the count, not the last index, so the valid range is always 0 to count minus 1.

What sits at `a[5]`? Whatever happens to occupy the memory immediately after the array. C places no wall there and performs no bounds checking. It simply computes `base + 5 * sizeof(int)` and writes to that address without objection.

```
valid indices                    not yours
┌─────┬─────┬─────┬─────┬─────┐ ┌─────────────────┐
│ [0] │ [1] │ [2] │ [3] │ [4] │ │      [5]        │
└─────┴─────┴─────┴─────┴─────┘ └─────────────────┘
1000  1004  1008  1012  1016     1020
                                  ▲
                                  │ could be another variable,
                                  │ a saved register, or a return
                                  │ address. C will write here
                                  │ without complaint.
```

That last possibility is worth noting, because overwriting a saved return address is the mechanism behind an entire category of security vulnerabilities that this series will return to much later.

### 15.3 Returning a Pointer to a Local Variable

```c
int *badFunction(void) {
    int local = 42;
    return &local;
}
```

Each time a function is called, C sets aside a region of memory called a stack frame for it. All local variables live inside that frame. When the function returns, the entire frame is reclaimed, meaning the memory is marked free and available for reuse. The variables are not zeroed out, but you no longer own that memory.

The mistake is returning the address of something inside a frame that no longer exists. It amounts to giving somebody directions to a house that has already been demolished.

```
1. badFunction() is called
2. local = 42 is created in badFunction's stack frame
3. &local captures its address, say 5000
4. badFunction returns and its stack frame is destroyed
5. address 5000 now belongs to nobody
6. the returned address 5000 is a dangling pointer
```

```
during the call                  after the return
┌──────────────────┐             ┌──────────────────┐
│ badFunction frame│             │  reclaimed,      │
│   local = 42     │  addr 5000  │  free for reuse  │  addr 5000
└──────────────────┘             └──────────────────┘
         ▲                                ▲
         │                                │
     p = 5000                         p = 5000
   (valid, briefly)                (dangling: the frame is gone,
                                    but p still holds the address)
```

The caller now holds address 5000, but whatever resides there is no longer under its control:

```c
int *p = badFunction();
printf("%d", *p);   // might print 42, might print garbage,
                    // depending on whether 5000 has been reused
*p = 99;            // writing to reclaimed memory: corruption
```

The cruelty of this bug is that it often appears to work immediately after the call, because the memory has not yet been reused. It then fails mysteriously later when some other function call overwrites that region. This makes it one of the hardest classes of bug to track down.

Three safe alternatives exist. The simplest is to return the value rather than the address, which sidesteps the problem entirely:

```c
int goodFunction(void) {
    int local = 42;
    return local;      // return the value, not the address
}
```

The second is to have the caller supply the memory, so that ownership stays with somebody whose frame still exists:

```c
void goodFunction(int *result) {
    *result = 42;      // write into memory the caller owns
}
```

The third allocates on the heap, which is not tied to any stack frame and therefore outlives the call:

```c
int *goodFunction(void) {
    int *p = malloc(sizeof(int));
    *p = 42;
    return p;
}
```

The return type is `int *` in the third case because the function returns the address of an integer rather than an integer itself. Note also that the heap version transfers responsibility to the caller, who must eventually call `free`.

A fourth option, declaring the local as `static`, does technically work, because a `static` variable lives for the duration of the program rather than the call. It is worth knowing but not worth reaching for, since every call would return the address of the same single variable, which is rarely what anyone wants.

The rule to carry away is simple: never return the address of anything declared inside a function.

### 15.4 Confusing `*p++` with `(*p)++`

Operator precedence decides which operator claims its operand first when two appear together. Postfix `++` binds more tightly than unary `*`, so in `*p++` the compiler sees:

```
*p++   →   *(p++)      ++ claims p first, then * applies to the result
```

Postfix `++` has the further property of yielding the old value before incrementing. The sequence is therefore:

```
1. save the current value of p (the old address)
2. increment p so it points at the next element
3. apply * to the OLD value of p, the saved address
4. read whatever is at that old address

the POINTER changed, not the value
```

Parentheses force a different order in `(*p)++`:

```
1. (*p) dereferences p, giving the value at that address (x = 5)
2. ++ increments that value, so x becomes 6
3. p does not move

the VALUE changed, not the pointer
```

Tracing both with concrete values:

```c
int x = 5, y = 0;
int *p = &x;

// --- *p++ ---
y = *p++;
// step 1: the old value of p is the address of x, say 1000
// step 2: p becomes 1004, pointing past x
// step 3: y = *(1000) = 5, read through the old address
// result: y = 5, x = 5 (unchanged), p now points past x

// --- (*p)++ ---
p = &x;      // reset p back to x
(*p)++;
// step 1: *p is the value at the address of x, which is 5
// step 2: that value is incremented, so x becomes 6
// step 3: p still points at x
// result: x = 6, p unchanged
```

The practical rule is straightforward. To change the value a pointer refers to, use `(*p)++`. To advance the pointer while reading the old location, as when walking through an array, use `*p++`. Decide which you want before writing it.

### 15.5 Modifying String Literals

```c
char *s = "hello";
s[0] = 'H';        // crash or undefined behaviour
```

compared with:

```c
char s[] = "hello";
s[0] = 'H';        // perfectly fine
```

These look nearly identical, and the difference is exactly the ownership question from Section 8. In the first case the string literal is stored in a read-only section of the program, and `s` merely holds its address. Attempting to write there is refused by the operating system. In the second case `char s[]` creates a new character array in your stack memory and copies the characters into it, so you hold your own writable copy:

```c
s = { 'h', 'e', 'l', 'l', 'o', '\0' };
```

Writing `s[0] = 'H';` is then permitted, because you are modifying your own copy rather than the program's read-only data.

## Conclusion

Every one of the mistakes above reduces to the same root cause. C trusts you completely and checks nothing at runtime. It will let you write to memory you do not own, read memory that no longer exists, and walk off the end of an array, all without a single error message. The language grants full control over memory and assumes you know precisely what you are doing. These mistakes are the price of that power, and understanding exactly why each one breaks is what allows you to avoid them.

Everything in this post connects to a single thread. C gives you direct control over memory addresses, and that address model extends outward to arrays, to strings, and even to functions. Once you stop seeing pointers as a special feature and start seeing them as the natural way C talks about where things live, the notation begins to read naturally.

The key is to think in terms of memory: where values are stored, how addresses are used, and what happens when data is accessed or modified through a pointer. Once that clicks, much of what follows, including dynamic memory, data structures, and the kernel structures we will meet later in this series, becomes far easier to follow.

What this post has not yet addressed is the specific ways a pointer can be invalid. We have seen an uninitialised pointer and a dangling one in passing, but there are several distinct failure modes, each with its own name, cause, and consequence. Naming them precisely is the last piece of groundwork before we turn to the operating system itself.

**Next:** [Null Pointer, Dangling Pointer, Void Pointer, and Wild Pointer](./03-pointer-types.md)
