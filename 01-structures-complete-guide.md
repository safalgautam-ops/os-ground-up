# Structures: Complete Guide

> Module 0 · Post 1 of 13

Before we can talk about what an operating system does, we need a shared vocabulary for describing how data sits in memory. Structures are the foundation of that vocabulary. Nearly every kernel data structure you will meet later in this series, including the Process Control Block itself, is a structure. This post covers them completely, from the problem they solve through to the more specialised forms such as unions and bit-fields.

## 1. The Problem Structures Solve

Suppose you want to represent a person in a program. Without structures, you would write something like this:

```c
char  name[50];
int   age;
float salary;
char  address[100];
```

These four variables describe one person, but nothing in the code connects them. If you have a hundred employees, you need four hundred separate variables. If you want to pass "an employee" to a function, you must pass four separate arguments. The data is conceptually related but structurally scattered.

A structure allows you to bundle related variables together under a single name:

```c
struct employee {
    char  name[50];
    int   age;
    float salary;
    char  address[100];
};
```

Now one name, `employee`, describes the whole thing. That is the entire motivation. Everything else in this post follows from that single idea.

## 2. Declaring a Structure

```c
struct point {
    int x;
    int y;
};
```

Each piece of this declaration matters:

- `struct` is the keyword announcing that a structure is being defined.
- `point` is the tag, which is the name given to this structure type.
- `int x;` and `int y;` are the members, the variables held inside the structure.
- The semicolon after the closing brace is mandatory and is easy to forget.

There is a critical distinction to draw here. This declaration does not create a variable and does not allocate any memory. It defines a template, a blueprint describing what a `point` looks like. It is comparable to describing what a house should contain without actually building one.

To create an actual variable, you write:

```c
struct point pt;
```

Only now is memory allocated. `pt` is a real variable containing two integers, accessible as `pt.x` and `pt.y`.

## 3. How Structure Memory Actually Looks

Given `struct point pt;`, the variable occupies a consecutive block of memory containing both members:

```
Address 1000: [  pt.x  ]   4 bytes for int
Address 1004: [  pt.y  ]   4 bytes for int
```

Taken as a whole, `pt` occupies 8 bytes. The members are stored one after another. This contiguity is precisely why a structure can be passed around as a single unit: it is one block of memory.

One caveat worth stating early, because it will matter later. Compilers are permitted to insert unused padding bytes between members so that each member begins at an address its type prefers. A structure containing an 8-byte pointer followed by a 4-byte integer will typically occupy 16 bytes rather than 12, because 4 bytes of padding are added at the end. You should therefore never assume the size of a structure is simply the sum of its members. Use `sizeof` instead, as discussed in Section 12.

## 4. Accessing Members: The Dot Operator

The dot operator connects a structure variable to one of its members:

```c
struct point pt;

pt.x = 100;          // assign to the x member
pt.y = 200;          // assign to the y member
printf("%d", pt.x);  // read the x member
```

Read `pt.x` as "the x member of pt."

Members behave exactly like ordinary variables of their type, because that is what they are. They simply live inside a structure:

```c
pt.x = pt.x + 1;                       // increment x
int dist = pt.x * pt.x + pt.y * pt.y;  // use in expressions
```

## 5. Initialising a Structure

As with arrays, a structure can be initialised at the point of declaration:

```c
struct point pt = {100, 200};
// pt.x = 100, pt.y = 200
```

The values fill in order, matching the order in which the members were declared. Alternatively, you may assign member by member:

```c
struct point pt;
pt.x = 100;
pt.y = 200;
```

## 6. Nested Structures

A member of a structure may itself be a structure. This is both natural and common:

```c
struct rect {
    struct point pt1;   // top-left corner
    struct point pt2;   // bottom-right corner
};
```

A `rect` contains two `point` structures. To reach the x coordinate of `pt1`:

```c
struct rect screen;
screen.pt1.x = 0;      // x of the first point
screen.pt1.y = 0;      // y of the first point
screen.pt2.x = 1920;   // x of the second point
screen.pt2.y = 1080;   // y of the second point
```

Read `screen.pt1.x` as "the x member of the pt1 member of screen." The dot operator chains from left to right, and each individual step is simple even when three levels are involved.

## 7. Structures and Functions

Functions in C work by copying values. When you pass an `int` to a function, the function receives a copy, and modifying it does not affect the original. Exactly the same rule applies to structures.

There are three distinct ways of working with structures in functions. Each has its own use case. The examples below assume the two declarations already introduced:

```c
struct point {      // struct point is a type, like int or float
    int x;
    int y;
};

struct rect {
    struct point pt1;
    struct point pt2;
};
```

### 7.1 Returning a Structure from a Function

The first example is `makepoint`, a function that takes two integers and builds a structure from them:

```c
struct point makepoint(int x, int y)
{
    struct point temp;   // a local variable of type struct point
    temp.x = x;          // fill the structure
    temp.y = y;
    return temp;         // send the structure back
}
```

Every part of this matters. Writing `struct point` before the function name declares that the function returns a structure of type `point`. Inside the body, `temp` is a local variable of that type. Once its members are filled, `return temp` sends a copy of the structure back to the caller.

One subtlety is worth noticing. The parameter `x` and the member `temp.x` share a name. This is not a conflict, because they exist in different scopes. `x` is the parameter and `temp.x` is a member of the structure. Reusing the name arguably makes the relationship clearer.

With `makepoint` available, structures can be built inline:

```c
struct rect  screen;
struct point middle;

screen.pt1 = makepoint(0, 0);
screen.pt2 = makepoint(1920, 1080);
middle = makepoint(
    (screen.pt1.x + screen.pt2.x) / 2,
    (screen.pt1.y + screen.pt2.y) / 2
);
```

This is considerably cleaner than setting `.x` and `.y` by hand at every site. The returned structure is assigned directly to `screen.pt1`, `screen.pt2`, and `middle`.

### 7.2 Passing a Structure by Value

```c
struct point addpoint(struct point p1, struct point p2)
{
    p1.x += p2.x;
    p1.y += p2.y;
    return p1;
}
```

Both parameters are passed by value, so the function receives its own private copies:

```c
struct point a = {3, 4};
struct point b = {1, 2};

struct point c = addpoint(a, b);
// c.x = 4, c.y = 6
// a and b are unchanged
```

Note that although `p1` is modified inside the function, the caller's `a` is untouched, because `p1` was only ever a copy.

### 7.3 Passing a Pointer to a Structure

Passing a structure by value copies every byte of it. For a small `point` holding two integers, that cost is negligible. For a large structure with many members, it is wasteful. The alternative is to pass a pointer, which sends a single address regardless of how large the structure happens to be.

```
copying a large structure = expensive
copying an address        = small and constant
```

A pointer also allows the function to modify the caller's original structure rather than a copy.

```c
struct point *pp;
struct point origin;

pp = &origin;   // pp holds the address of origin
```

`pp` is now a pointer to a structure, while `origin` is an actual `struct point` variable. There are two ways to reach a member through this pointer:

```c
(*pp).x    // dereference pp to obtain the structure, then access .x
pp->x      // shorthand meaning exactly the same thing
```

The parentheses in `(*pp).x` are mandatory. Without them, `*pp.x` would be parsed as `*(pp.x)`, which attempts to dereference `x` as though it were a pointer. Since `x` is an `int`, that is an error. The dot operator binds more tightly than `*`, so the dereference must be forced with parentheses.

Because this pattern is so common, C provides `->` as a dedicated operator meaning "dereference this pointer, then access this member." The arrow form is preferred and reads more cleanly.

## 8. Two Worked Examples

### 8.1 `ptinrect`: Testing Whether a Point Lies Inside a Rectangle

```c
int ptinrect(struct point p, struct rect r)
{
    return p.x >= r.pt1.x && p.x < r.pt2.x
        && p.y >= r.pt1.y && p.y < r.pt2.y;
}
```

This takes a point and a rectangle, both by value, and returns 1 if the point lies inside the rectangle and 0 otherwise. The convention adopted here is that the left and bottom edges are included while the right and top edges are not. This avoids double-counting shared edges when rectangles are adjacent.

To see why that convention matters, consider two rectangles that touch:

```
Rectangle A: (0,0) to (10,10)
Rectangle B: (10,0) to (20,10)
```

They share the vertical edge at x = 10. Now consider the point (10, 5). If the comparison were written as `p.x <= r.pt2.x`, then rectangle A would contain the point and rectangle B would contain it as well. The point would belong to both rectangles simultaneously, which is precisely the ambiguity the convention is designed to eliminate.

Notice also the nested access in `r.pt1.x`. Here `r` is a `rect`, its member `pt1` is a `struct point`, and that point's member is `x`. Three levels of access, each step trivial.

### 8.2 `canonrect`: Ensuring a Rectangle Is Well Formed

`ptinrect` assumes that `pt1` holds the smaller coordinates and `pt2` the larger. What happens if a caller supplies a rectangle whose corners are in the wrong order? `canonrect` corrects this by rearranging the corners so that `pt1` always holds the minimum coordinates and `pt2` the maximum, returning a corrected copy.

```c
#define min(a, b) ((a) < (b) ? (a) : (b))
#define max(a, b) ((a) > (b) ? (a) : (b))

struct rect canonrect(struct rect r)
{
    struct rect temp;
    temp.pt1.x = min(r.pt1.x, r.pt2.x);
    temp.pt1.y = min(r.pt1.y, r.pt2.y);
    temp.pt2.x = max(r.pt1.x, r.pt2.x);
    temp.pt2.y = max(r.pt1.y, r.pt2.y);
    return temp;
}
```

The `min` and `max` macros use the ternary operator. The expression `((a) < (b) ? (a) : (b))` reads as "if a is less than b, yield a, otherwise yield b." The parentheses around `a` and `b` are important in macros, because macro arguments are substituted as raw text and an unparenthesised argument can be regrouped by operator precedence in unexpected ways.

To restate the problem concretely, the rectangle assumes:

```
pt1 = bottom-left  (smaller x, smaller y)
pt2 = top-right    (larger x, larger y)
```

A caller might nevertheless construct a rectangle with `pt1 = (10, 10)` and `pt2 = (0, 0)`, which inverts the assumption. Passing that directly to `ptinrect` produces incorrect results. Running it through `canonrect` first guarantees the assumption holds.

## 9. The `->` Operator: Reading Every Combination

Consider the following declaration:

```c
struct {
    int   len;
    char *str;
} *p;
```

Here `p` is a pointer to a structure containing an `int` named `len` and a `char *` named `str`. The expressions below are frequently confused, so each is worth reading carefully.

**`++p->len`**
Because `->` has higher precedence than `++`, this parses as `++(p->len)`. It increments `len`, the integer member. The pointer `p` itself does not move.

**`(++p)->len`**
The parentheses force `++p` to be evaluated first, so `p` advances to point at the next structure. The expression then accesses the `len` member of that new structure.

**`(p++)->len`**
`p++` is the postfix form, which yields the old value of `p` before incrementing. The `len` accessed therefore belongs to the original structure, and only afterwards does `p` move forward. Compared with `(++p)->len`, `p` ends up pointing at the same place in both cases, but the member accessed is different: the old structure here, the new one there.

**`*p->str`**
This parses as `*(p->str)`. First `p->str` yields the `char *` stored in the structure, then `*` dereferences it to give the character that pointer refers to. For example:

```c
struct Example {
    char *str;
};

struct Example e;
e.str = "hello";

struct Example *p = &e;

// p->str     yields a pointer to the first character
// *(p->str)  yields 'h', the first character itself
```

**`*p->str++`**
This parses as `*(p->str++)`. It reads the character that `str` currently points to, then increments `str`, the pointer stored inside the structure. The behaviour is identical to the familiar `*s++` idiom for walking a string, except that `s` is a structure member here.

**`(*p->str)++`**
`p->str` yields the pointer, `*` dereferences it to obtain the character, and `++` then increments that character value. The pointer `str` does not move; the character it points at is what changes.

**`*p++->str`**
This parses as `*((p++)->str)`. The postfix `p++` uses the current value of `p` and then advances `p` to the next structure. So `(p++)->str` accesses the `str` member of the structure `p` originally pointed at, and the leading `*` dereferences that pointer to give the character it refers to.

The rule underlying all of these is straightforward: `->` and `.` bind before anything else. Whenever you need a different order of evaluation, use parentheses to force it.

## 10. Arrays of Structures

Consider a concrete scenario. You want to write a program that reads C source code and counts how many times each C keyword appears. How should that data be stored, given that you need to track both what the keyword is and how many times it has appeared?

### 10.1 First Attempt: Two Parallel Arrays

The most obvious approach uses one array for the names and another for the counts:

```c
char *keyword[5];    // stores keyword strings
int   keycount[5];   // stores their counts
```

```c
keyword[0] = "break";    keycount[0] = 0;
keyword[1] = "case";     keycount[1] = 0;
keyword[2] = "for";      keycount[2] = 0;
keyword[3] = "if";       keycount[3] = 0;
keyword[4] = "while";    keycount[4] = 0;
```

This works, but it carries a serious weakness. The correspondence between `keyword[2]` and `keycount[2]` exists only in the programmer's head. Nothing in the code enforces it. Sorting one array leaves the other out of sync. Inserting a keyword at the wrong index mismatches every count after it. Another programmer reading the code has to infer the relationship independently.

The arrays are parallel, meaning they are intended to move together, but the language has no way of knowing that.

### 10.2 Second Attempt: One Structure per Keyword

A structure binds related data together so that it cannot be separated:

```c
struct key {
    char *word;    // the keyword string
    int   count;   // how many times it has been seen
};
```

`word` and `count` are now locked together. They occupy one contiguous region of memory, are accessed as a single entity, and cannot drift apart.

A single `struct key` variable is laid out roughly as follows:

```
┌─────────────────────┬──────────────┐
│  word (8 bytes)     │  count (4B)  │
│  pointer to string  │  integer     │
└─────────────────────┴──────────────┘
```

On a typical 64-bit system the structure occupies 16 bytes rather than 12, because the compiler adds 4 bytes of trailing padding so that arrays of this structure remain correctly aligned. This is the padding behaviour mentioned in Section 3, and it is the reason `sizeof` should always be preferred over manual arithmetic.

Members are accessed with the dot operator as usual:

```c
k.word  = "while";
k.count = 0;
printf("%s appeared %d times\n", k.word, k.count);
```

### 10.3 An Array of Structures

Many keywords need to be stored, not just one, so the natural step is an array in which every element is a `struct key`:

```c
struct key keytab[5];
```

This creates five complete structures laid out consecutively in memory:

```
keytab[0]:  [word ptr | count]
keytab[1]:  [word ptr | count]
keytab[2]:  [word ptr | count]
keytab[3]:  [word ptr | count]
keytab[4]:  [word ptr | count]
```

Accessing a member now takes two steps: select the element with `[]`, then select the member with `.`:

```c
keytab[2].word    // the word of the third entry
keytab[2].count   // the count of the third entry
```

Read this as "the word member of keytab[2]."

### 10.4 Initialising the Array

Rather than assigning each field after declaration, the entire table can be initialised at declaration time:

```c
struct key keytab[] = {
    {"auto",     0},
    {"break",    0},
    {"case",     0},
    {"char",     0},
    {"const",    0},
    {"continue", 0},
    {"default",  0},
    {"unsigned", 0},
    {"void",     0},
    {"volatile", 0},
    {"while",    0}
};
```

Each part of this syntax is worth naming explicitly. `struct key keytab[]` declares an array of `struct key`, and the empty brackets instruct the compiler to count the elements itself. The outer braces enclose the initialisers for the entire array. Each inner pair of braces, such as `{"auto", 0}`, initialises one structure: `"auto"` fills the `word` member and `0` fills `count`, in the order the members were declared.

The keywords appear in alphabetical order for a specific reason. The lookup routine uses binary search, and binary search requires sorted data. If the table were unordered, the search would return incorrect results.

## 11. Computing `NKEYS` with `sizeof`

The program needs to know how many keywords the table contains. One could count them by hand and write `#define NKEYS 11`, but that breaks the moment a keyword is added or removed and the constant is not updated alongside it.

C provides `sizeof`, an operator returning the size of something in bytes, evaluated at compile time with no runtime cost:

```c
sizeof keytab          // total bytes occupied by the entire array
sizeof(struct key)     // bytes occupied by one element
```

A `struct` is a fixed layout of bytes in memory, so `sizeof(struct key)` yields the total number of bytes required to store one complete `struct key` object, padding included. Dividing the size of the whole array by the size of one element therefore gives the number of elements:

```c
#define NKEYS (sizeof keytab / sizeof(struct key))
```

This recalculates automatically. Add a keyword and `sizeof keytab` grows, so `NKEYS` grows with it. Remove one and the same applies. The line never needs to be touched again.

A marginally better formulation is:

```c
#define NKEYS (sizeof keytab / sizeof(keytab[0]))
```

This version is preferable because it does not name the type. If `struct key` is ever renamed or replaced, this line continues to work without modification.

The outer parentheses are not decorative. `#define` performs text substitution, so writing `sizeof keytab / sizeof(struct key)` without them and then using `NKEYS` inside a larger expression such as `2 * NKEYS` would expand to `2 * sizeof keytab / sizeof(struct key)`. By operator precedence that evaluates as `(2 * sizeof keytab) / sizeof(struct key)`, which is not the intended result. The outer parentheses prevent this class of error.

## 12. Binary Search

Binary search is what makes the keyword lookup fast. Before reading the code, it helps to follow the algorithm on a concrete example. Suppose we are looking for `"for"` in this sorted table:

```
Index:    0        1       2       3        4        5
Word:   "auto"  "break" "case"  "char"  "const"  "for"
```

A linear scan checks indices 0, 1, 2, 3, 4, and 5, finding the match at index 5 after six comparisons. Binary search instead examines the middle element first:

```
Step 1: low=0, high=5, mid=2 → "case"
        "for" > "case", so it must lie in the right half
        set low = mid + 1 = 3

Step 2: low=3, high=5, mid=4 → "const"
        "for" > "const", right half again
        set low = mid + 1 = 5

Step 3: low=5, high=5, mid=5 → "for"
        "for" == "for", found at index 5
```

Three comparisons instead of six. With 32 keywords, binary search requires at most 5 comparisons; with 1000 keywords, at most 10. The improvement comes from halving the remaining search space at every step. The requirement, as noted above, is that the data must be sorted, which is why the keyword table is alphabetical.

### 12.1 The Code, Line by Line

```c
int binsearch(char *word, struct key tab[], int n)
{
    int cond;
    int low, high, mid;

    low  = 0;
    high = n - 1;
    while (low <= high) {
        mid = (low + high) / 2;
        if ((cond = strcmp(word, tab[mid].word)) < 0)
            high = mid - 1;
        else if (cond > 0)
            low = mid + 1;
        else
            return mid;
    }
    return -1;
}
```

The parameters are as follows. `char *word` is the word being searched for. `struct key tab[]` is the table to search; note that in a parameter list, `struct key tab[]` means exactly the same thing as `struct key *tab`, since it is a pointer to the first element. `int n` is the number of entries in the table.

`low = 0; high = n - 1;` establishes the search range. `low` is the index of the leftmost element still under consideration and `high` the index of the rightmost, beginning with the full table.

`while (low <= high)` continues while at least one element remains in the range. Once `low` passes `high`, the range is empty and the word is not present.

`mid = (low + high) / 2` finds the middle index. Integer division rounds down automatically, so with `low = 3` and `high = 5`, `mid` becomes 4.

`strcmp(word, tab[mid].word)` compares the search word against the middle element. The result is assigned to `cond` inside the condition itself, which avoids calling `strcmp` a second time in the `else if` branch.

`if (cond < 0) high = mid - 1;` handles the case where the search word precedes the middle element alphabetically, so the target must lie in the left half. `else if (cond > 0) low = mid + 1;` handles the mirror case for the right half. `else return mid;` is reached when `cond` is zero, meaning the strings match and the index can be returned.

`return -1` is reached only when the loop exits without a match, indicating the word is absent from the table.

### 12.2 What `strcmp` Returns

```
negative   if a comes before b alphabetically   ("apple" vs "banana")
zero       if a equals b                        ("for"   vs "for")
positive   if a comes after b                   ("zoo"   vs "apple")
```

Binary search uses this return value to decide which direction to narrow:

```
cond < 0   word is alphabetically earlier than tab[mid].word
           so word must lie in the LEFT half  → high = mid - 1

cond > 0   word is alphabetically later than tab[mid].word
           so word must lie in the RIGHT half → low = mid + 1

cond == 0  word matches tab[mid].word exactly → return mid
```

## 13. Self-Referential Structures

This is the gateway to almost every interesting data structure, including linked lists, trees, and graphs. The idea itself is simple: a structure that contains a pointer to another structure of the same type. A structure cannot contain an instance of itself, since that would require infinite space, but it can perfectly well contain a pointer to one, because a pointer has a fixed and known size.

## 14. `typedef`: Giving Types New Names

`typedef` creates a new name for an existing type. It does not create a new type; it creates an alternative way of spelling one that already exists.

```c
typedef int Length;
```

After this line, `Length` and `int` are interchangeable throughout the code:

```c
Length x;         // same as: int x
Length arr[10];   // same as: int arr[10]
Length *p;        // same as: int *p
```

### 14.1 Reading the Syntax

The governing rule is that in a `typedef`, the name being defined occupies the position where the variable name would appear in an ordinary declaration:

```c
int x;               // ordinary: x is a variable of type int
typedef int Length;  // typedef: Length is a name for the type int
```

This becomes more interesting with pointers:

```c
char *p;                // ordinary: p is a pointer to char
typedef char *String;   // typedef: String means "pointer to char"

String s;               // same as: char *s
String lines[100];      // same as: char *lines[100]
```

The distinction to hold onto is that in an ordinary declaration the `*` belongs to the variable, whereas in a `typedef` the `*` becomes part of the type name.

### 14.2 Reason One: Portability

Different machines represent data differently. An `int` may be 16, 32, or 64 bits depending on the platform. Suppose your program requires exactly 32-bit integers. Rather than writing `int` throughout, you write:

```c
typedef int Int32;
```

and then declare variables as `Int32 a, b, c;`. If you later port the program to a machine where `int` is the wrong width and `long` is 32 bits, only one line changes:

```c
typedef long Int32;
```

Every declaration of `Int32 a, b, c;` continues to work unmodified.

### 14.3 Reason Two: Documentation

`typedef` can also make code more readable by giving a type a meaningful name. Consider:

```c
struct tnode *p;
```

This tells the reader that `p` is a pointer to a `struct tnode`, but it says nothing about the role `p` plays. Compare:

```c
typedef struct tnode *Treeptr;

Treeptr p;
```

`Treeptr` is simply a shorter name for `struct tnode *`, so `Treeptr p;` means precisely `struct tnode *p;`. The advantage is that the name communicates intent: this pointer is meant to refer to a tree node. Across a whole file, this:

```c
struct tnode *left;
struct tnode *right;
struct tnode *root;
```

becomes:

```c
Treeptr left;
Treeptr right;
Treeptr root;
```

which is cleaner and signals that these variables belong to the same structure.

### 14.4 Why `typedef` Is Preferable to `#define`

```c
typedef char *String;   // typedef
#define String char *   // define
```

The two look similar, but the difference emerges as soon as multiple variables are declared on one line:

```c
typedef char *String;
String p, q;   // both p and q are char *   (correct)
```

```c
#define String char *
String p, q;   // expands to: char *p, q
               // p is char *, but q is only char   (incorrect)
```

`#define` performs blind textual substitution before compilation, whereas `typedef` is understood by the compiler as a genuine type alias. For anything beyond the most trivial case, `typedef` is safer and more correct.

## 15. Unions: One Variable, Several Types

A union resembles a structure syntactically but behaves quite differently:

```c
union u_tag {
    int   ival;
    float fval;
    char *sval;
} u;
```

In a structure, every member occupies its own separate memory. The total size is the sum of the members plus any padding. In a union, all members share the same memory. The total size is that of the largest member.

```c
// Structure: three separate slots
struct { int i; float f; char *s; } s;
// sizeof(s) is approximately 16 bytes (4 + 4 + 8, plus alignment)

// Union: one shared slot
union { int i; float f; char *s; } u;
// sizeof(u) is 8 bytes (the largest member, char *, is 8 bytes)
```

The contrast is easier to see laid out side by side:

```
struct: separate slots              union: one shared slot
┌───────┬───────┬────────┐          ┌────────────────────────────┐
│ ival  │ fval  │  sval  │          │  the same 8 bytes, shared  │
│ 4 B   │ 4 B   │  8 B   │          │  ival OR fval OR sval,     │
└───────┴───────┴────────┘          │  never all three at once   │
total 16 bytes,                     └────────────────────────────┘
each member independent             total 8 bytes
                                    (size of the largest member)
```

Conceptually, a union is a box capable of holding different types, but only one of them at any given moment. The declaration above means that `u` can store an `int`, or a `float`, or a string, but never more than one simultaneously.

This leads directly to the critical rule governing unions. You may store only one type at a time, and reading a member other than the one you last wrote gives undefined results. The bytes are not converted between types; they are simply reinterpreted, so reading `fval` after storing into `ival` yields whatever those particular bits happen to mean when treated as a floating-point number. That is almost never anything useful.

Crucially, the union itself does not record which member is currently valid. Keeping track of that is the programmer's responsibility. The conventional solution is to pair the union with a separate tag: an integer recording which type is currently stored.

```c
union u_tag {
    int   ival;
    float fval;
    char *sval;
} u;

int utype;   // records which member is active: INT, FLOAT, or STRING

if (utype == INT)
    printf("%d", u.ival);
else if (utype == FLOAT)
    printf("%f", u.fval);
```

Every write to the union must update `utype` alongside it, and every read must consult `utype` first. A union paired with a tag in this way is usually called a tagged union, and wrapping both inside an enclosing structure keeps them from being separated.

### 15.1 When Unions Are Genuinely Useful

Suppose you need to store a value that might be an integer such as 10, or a float such as 3.14, or a string such as `"hello"`. A first attempt might use a structure:

```c
struct {
    int   ival;
    float fval;
    char *sval;
} x;
```

The difficulty is that this reserves space for all three members simultaneously, even though only one will ever be in use. That is wasted memory, and the waste multiplies if many such values are stored.

A union solves this directly:

```c
union {
    int   ival;
    float fval;
    char *sval;
} x;
```

Only one slot is created, and that slot is reused for whichever type is currently held.

## 16. Bit-fields: Packing Several Values into One Integer

Sometimes several values each require only a handful of bits. Rather than spending an entire `int` on each, they can be packed together.

### 16.1 Without Bit-fields: Manual Bitmasks

```c
#define KEYWORD  01   // binary: 001
#define EXTERNAL 02   // binary: 010
#define STATIC   04   // binary: 100

int flags;
flags |= EXTERNAL;             // turn the EXTERNAL bit on
flags &= ~STATIC;              // turn the STATIC bit off
if (flags & KEYWORD) { ... }   // test the KEYWORD bit
```

Suppose we want to store three true or false values: is this a keyword, is it external, is it static. Turning flags on, turning them off, and testing them all require bitwise operators. The result is verbose, hard to read, and easy to get wrong.

### 16.2 With Bit-fields: The Compiler Handles the Bits

```c
struct {
    unsigned int is_keyword : 1;   // 1 bit
    unsigned int is_extern  : 1;   // 1 bit
    unsigned int is_static  : 1;   // 1 bit
} flags;

flags.is_extern = 1;    // set
flags.is_static = 0;    // clear
if (flags.is_keyword)   // test
```

The `: 1` following a member name declares that the member is one bit wide. Wider fields are permitted as well: `: 4` would give a four-bit field capable of holding values from 0 to 15. Bit-fields therefore let you store data using an exact number of bits while writing ordinary member access syntax.

### 16.3 Limitations You Must Know

```c
&flags.is_extern   // illegal
```

The address of a bit-field cannot be taken with `&`, because bit-fields do not have individual addresses. They share a byte with neighbouring fields.

More significantly, bit-fields are implementation-dependent in almost every respect: whether they are packed left to right or right to left, whether a field may span a word boundary, and their exact resulting sizes are all left to the compiler. This makes them useful for internal data structures where you control both ends, but risky for external formats such as network packets or file formats, where the bit layout must be exact. For those cases, manual bitmasks are safer and more portable.

In short, bit-fields offer a compact and readable way to store small values, but they are not reliable across different machines.

### 16.4 On Bit Ordering

Bit-fields do not exist independently. They must live inside a normal C type such as `unsigned int`. A 32-bit `unsigned int` is laid out as:

```
bit positions:
31 ......... 3 2 1 0

bit 0  = rightmost (least significant bit)
bit 31 = leftmost
```

Now consider packing three one-bit fields into it:

```c
struct {
    unsigned int a : 1;
    unsigned int b : 1;
    unsigned int c : 1;
} flags;
```

Three values, `a`, `b`, and `c`, each requiring one bit. Each declaration says, in effect, "take an `unsigned int`, but use only part of it." The compiler takes one `unsigned int` and packs all three inside it.

Where inside those 32 bits they are placed is the part that varies. Some compilers pack left to right:

```
bit: 31 30 29 ...
      a  b  c
```

Others pack right to left:

```
bit: ... 2 1 0
         c b a
```

Different processors handle bit ordering differently, and C is designed to remain workable across all of them, so the standard leaves this detail to the compiler. That flexibility is exactly why bit-fields should not be relied upon when an exact external layout is required.

## Conclusion

Structures give you a way to describe a compound object as a single named entity, and everything that follows in this series depends on that ability. You have seen how a structure is declared and laid out in memory, how its members are reached both directly and through pointers, how arrays of structures support real lookup tables, and how the more specialised forms, unions and bit-fields, trade generality for compactness.

Two ideas in particular are worth carrying forward. First, a structure is a contiguous block of memory whose exact size includes padding you did not write, which is why `sizeof` is always preferable to counting bytes by hand. Second, passing a structure by value copies it, while passing a pointer to it does not. Both of these will reappear the moment we start looking at how the operating system tracks a running program.

That said, this post has used pointers freely without examining them closely. Before going further, we need to be precise about what a pointer actually holds, and about the relationship between pointers and arrays that C treats as almost, but not quite, interchangeable.

**Next:** [Pointers and Arrays in C →](./02-pointers-and-arrays.md)
