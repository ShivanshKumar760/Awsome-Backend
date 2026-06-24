# How Python Actually Deals With Variables

### (and why it's fundamentally different from C, C++, and Java)

Most confusion about Python — mutability bugs, unexpected aliasing, "why did that change over
there when I changed it here" — traces back to one misunderstood fact about how Python variables
work. This note builds the correct mental model from the ground up, contrasting it against C/C++
and Java at every step, then explains how Python's garbage collector works and why it's tied
directly to the same model.

```
Part A — How variables work in C/C++ (the "box" model)
Part B — How variables work in Java (halfway between)
Part C — How variables work in Python (the "label on an object" model)
Part D — Everything is an object — what that actually means
Part E — Mutability: the single most important consequence of the Python model
Part F — Python's Garbage Collector
Part G — Common bugs this mental model prevents
```

---

## Part A — How Variables Work in C / C++ (The "Box" Model)

In C and C++, a variable **is** a named memory location. When you declare:

```c
int age = 30;
```

The compiler reserves exactly 4 bytes of memory (on most platforms), names that location `age`,
and writes the value `30` into it. The variable IS the box. The name IS the address.

```
Memory:
┌──────────────┐
│      30      │  ← address e.g. 0x7ffee4b2c
└──────────────┘
      age
```

When you do:

```c
int age = 30;
int copy = age;
copy = 99;
```

`copy` is a **completely separate box**, a completely separate region of memory. Changing `copy`
has zero effect on `age`. They share a value briefly (at the moment of copy), then diverge — they
are independent, always.

With **pointers**, C explicitly lets you hold an _address_ (reference to a box), not the value
itself:

```c
int age = 30;
int *ptr = &age;    // ptr holds the ADDRESS of age's box
*ptr = 99;           // dereference: go to that address and write 99
// age is now 99 — ptr and age refer to the same memory location
printf("%d", age);   // 99
```

This pointer/value split is explicit, manual, and must be managed by the programmer.
`malloc`/`free`, dangling pointers, buffer overflows — all come from the fact that you're
directly manipulating memory addresses and the language makes no attempt to protect you.

**C++ adds references**, which are like automatic, non-nullable pointers:

```cpp
int age = 30;
int &ref = age;   // ref is NOT a new box — it's another name for the SAME box
ref = 99;
std::cout << age; // 99 — same underlying memory, different name
```

Key point: the C/C++ model is **value-centric**. A variable IS a typed memory location. The name
is baked into the compiled binary as an address offset. At runtime, `age` literally means "the
4 bytes at offset X from the stack frame base."

---

## Part B — How Variables Work in Java (Halfway Between)

Java draws a sharp distinction between **primitives** and **reference types**, and the model
is different for each.

### Primitives — same as C's box model

```java
int age = 30;
int copy = age;
copy = 99;
System.out.println(age); // still 30 — copy is a separate box, same as C
```

For `int`, `double`, `boolean`, `char`, `long`, `float`, `byte`, `short` — Java uses the C-style
box model: the value lives directly inside the variable, copying copies the value, changes are
independent.

### Reference types — the variable holds an address (reference), not the object

```java
User alice = new User("alice");
User alias = alice;         // does NOT copy the User object — copies the REFERENCE (address)
alias.setUsername("bob");
System.out.println(alice.getUsername()); // "bob" — both variables point at the same object
```

```
Stack                     Heap
┌──────────────┐          ┌──────────────────────┐
│  alice  ──────────────▶ │  User{username="bob"} │
├──────────────┤          └──────────────────────┘
│  alias  ──────────────▶ (same object, same address)
└──────────────┘
```

Java variables for reference types are **boxes that hold addresses** (references), not boxes that
hold the objects themselves. The objects live on the **heap**; the variable on the **stack** holds
only a pointer to the heap location.

But — and this is the key Java-vs-Python distinction — in Java you still have the explicit
**primitive/reference split**. You choose whether to use `int` (value in the box) or `Integer`
(reference to an object on the heap). The type system enforces it. You always know which you're
dealing with.

Java also has **explicit typed declarations**:

```java
String name = "alice"; // the type (String) is fixed here — name can never hold an Integer
```

The variable `name` has a fixed type at compile time. You can't rebind it to a different type of
object at runtime. The type is a property of the variable, not of the value.

---

## Part C — How Variables Work in Python (The "Label on an Object" Model)

Python has **no box model at all**. A Python variable is not a memory location. It is not a typed
slot. It is not a container. It is **a name in a namespace that points to an object**.

```python
age = 30
```

What actually happens, step by step:

1. Python creates an **integer object** with the value `30` somewhere in memory. The object knows
   what type it is (`int`). The object has a reference count (how many names point at it). The
   object has an identity (its memory address).
2. Python creates an entry in the current **namespace** (a dictionary, literally) that maps the
   string `"age"` to that object.
3. That's it. There is no "int-sized box named age." There's a name, and there's an object, and
   there's an arrow from one to the other.

```
Namespace (a dict)           Heap
┌────────────────┐           ┌───────────────────────────┐
│  "age"  ────────────────▶  │  int object               │
└────────────────┘           │  value: 30                │
                             │  type: int                │
                             │  refcount: 1              │
                             │  id: 140234567891024      │
                             └───────────────────────────┘
```

### What `=` actually does in Python

`=` in Python does **not** write a value into a box. It **rebinds a name** — makes that name
point at a different object:

```python
age = 30    # "age" -> int object 30
age = 99    # "age" now -> int object 99. The int object 30 is unchanged.
age = "hello"  # "age" now -> str object "hello". No type violation. age had no type.
age = [1, 2, 3]  # "age" now -> list object. Still fine.
```

The name `age` has no type. **Objects have types. Names do not.**

```python
x = 30
y = x       # y -> the SAME object as x. No copy. No new object.

print(id(x), id(y))  # id() returns an object's memory address
# same number — they're pointing at the same object
```

```
Namespace
┌────────────────┐           ┌───────────────────────────┐
│  "x"   ────────────────▶  │  int object               │
│  "y"   ────────────────▶  │  value: 30                │
└────────────────┘           │  refcount: 2              │
                             └───────────────────────────┘
```

Two names. One object. Reference count is 2.

### `id()` and `is` — inspecting the model directly

```python
a = [1, 2, 3]
b = a

print(a is b)       # True  — SAME object (same address)
print(a == b)       # True  — same VALUE (equality check)

b = [1, 2, 3]       # rebind b to a BRAND NEW list object
print(a is b)       # False — different objects now
print(a == b)       # True  — still the same value though
```

`is` checks identity (same object in memory — same `id()`).
`==` checks equality (values are equal — calls `__eq__`).

This `is`/`==` distinction is something C and Java don't force you to think about at the
syntactic level. Python surfaces it because two names can point at the same object, and sometimes
you genuinely need to know whether they do.

---

## Part D — Everything Is an Object: What That Actually Means

In C, `30` is a literal value — raw bytes, no metadata.
In Java, `30` is a primitive int — no methods, no type info at runtime.
In Python, `30` is a **full object** — an instance of `int` with an identity, a type, a reference
count, and methods.

```python
x = 30
print(type(x))         # <class 'int'>
print(x.bit_length())  # 5  — it has methods! It's a real object.
print(isinstance(x, int))   # True
print(isinstance(x, object))  # True — int inherits from object, like EVERYTHING in Python

# EVERYTHING:
print(type(30))         # <class 'int'>
print(type(3.14))       # <class 'float'>
print(type("hello"))    # <class 'str'>
print(type([1,2,3]))    # <class 'list'>
print(type(None))       # <class 'NoneType'>
print(type(True))       # <class 'bool'>

def greet(): pass
print(type(greet))      # <class 'function'>  — functions are objects too
print(type(int))        # <class 'type'>       — even CLASSES are objects (instances of `type`)
print(type(type))       # <class 'type'>       — type is an instance of itself. yes really.
```

### What "object" means structurally

Every Python object has exactly three things, always:

- **Identity** — its unique memory address, returned by `id(obj)`. Immutable for the object's
  lifetime.
- **Type** — what class it's an instance of, returned by `type(obj)`. Determines what operations
  are valid and what the object can do. Immutable for the object's lifetime.
- **Value** — the data the object holds. This is the one that **may or may not be changeable**,
  which brings us to the most important concept in this note.

### The type hierarchy

```
object
  ├── int
  ├── float
  ├── bool       (True and False are 1 and 0 — bool is a subclass of int)
  ├── str
  ├── bytes
  ├── NoneType   (None is a singleton — there is exactly one NoneType object, ever)
  ├── list
  ├── tuple
  ├── dict
  ├── set
  ├── function
  └── type       (the metaclass — classes themselves are instances of type)
```

```python
print(True + True + True)   # 3 — bool IS an int, so arithmetic works
print(True == 1)             # True
print(True is 1)             # False — different objects (True is a bool singleton, 1 is an int)
```

### The integer cache — a concrete implementation detail

CPython (the standard Python interpreter) **caches small integer objects** from -5 to 256,
creating them once at startup and reusing the same objects forever. This makes integer-heavy code
faster (no allocation cost) and produces a counter-intuitive-looking result:

```python
a = 100
b = 100
print(a is b)   # True  — same cached object

a = 1000
b = 1000
print(a is b)   # False — outside the cache; two separate objects
                #         (though this can vary by context and Python version)
```

This is an implementation optimization, not a language guarantee — and it's a perfect example of
why `==` is what you should use for value comparison, not `is`. `is` is for checking object
identity specifically, not value equality.

---

## Part E — Mutability: The Single Most Important Consequence

Because Python variables are just names pointing at objects, and multiple names can point at the
same object, whether an object's **value can change** becomes a critically important property.

### Immutable types — the value cannot change after creation

`int`, `float`, `bool`, `str`, `tuple`, `bytes`, `frozenset`

```python
x = "hello"
y = x           # y and x point at the same string object

x = x + " world"  # this creates a BRAND NEW string object — does NOT mutate "hello"
                   # x is now rebound to the new object; y still points at "hello"

print(y)   # "hello" — unchanged
```

When you "modify" an immutable object, Python actually creates a new object and rebinds your name
to it. The original object is untouched — any other names pointing at it are unaffected.

```python
s = "hello"
print(id(s))
s += " world"
print(id(s))    # different id — completely new object
```

### Mutable types — the value CAN change in-place

`list`, `dict`, `set`, and user-defined class instances by default

```python
a = [1, 2, 3]
b = a           # b and a point at THE SAME list object

b.append(4)     # mutates the object IN-PLACE — does NOT rebind b to a new object

print(a)        # [1, 2, 3, 4] — a sees the change because a and b share the same object
print(a is b)   # True — still the same object
```

```
Before append:
  "a" ──▶ [1, 2, 3]  ◀── "b"

After b.append(4):
  "a" ──▶ [1, 2, 3, 4]  ◀── "b"    (same object, mutated)
```

This is the source of most "my variable changed unexpectedly" bugs in Python.

### Copying — shallow vs. deep

```python
import copy

original = [[1, 2], [3, 4]]

# slice copy / list() / copy.copy() — all SHALLOW copies
shallow = original[:]
# shallow is a NEW list object... but its ELEMENTS are the same objects as original's elements

shallow[0].append(99)  # mutates the inner list object — which original[0] ALSO points at
print(original)         # [[1, 2, 99], [3, 4]] — oops

# deep copy — recursively copies every object in the structure
deep = copy.deepcopy(original)
deep[0].append(999)
print(original)  # [[1, 2, 99], [3, 4]] — untouched this time
```

### The function argument gotcha: "pass by object reference"

Python is neither "pass by value" nor "pass by reference" in the C/Java sense. It's more
precisely called **pass by object reference** (or "pass by assignment") — the function receives
a **copy of the reference**, pointing at the same object.

```python
def try_rebind(x):
    x = 99          # rebinds the LOCAL name x — doesn't affect the caller's name

def try_mutate(lst):
    lst.append(99)  # mutates the OBJECT that both lst and the caller's name point at

n = 10
try_rebind(n)
print(n)  # 10 — rebinding inside the function had no effect outside

items = [1, 2, 3]
try_mutate(items)
print(items)  # [1, 2, 3, 99] — the mutation happened on the shared object
```

The rule: **rebinding a parameter inside a function never affects the caller. Mutating a mutable
object the parameter points at always does.** Once this model is clear, this behavior is not
surprising — it follows directly from the name/object model.

---

## Part F — Python's Garbage Collector

Python manages memory automatically. You never call `free()` (C) or `delete` (C++). The garbage
collector's job is to figure out when an object is no longer reachable and reclaim its memory.
Python uses **two complementary mechanisms** to do this.

### F.1 Reference counting — the primary mechanism

Every Python object maintains a **reference count**: the number of names (in any namespace) that
currently point at it. When that count reaches zero, the object's memory is freed immediately —
not eventually, not at some future GC cycle, but **right then**.

```python
import sys

x = [1, 2, 3]
print(sys.getrefcount(x))   # 2 — x itself, plus the temporary reference getrefcount takes

y = x
print(sys.getrefcount(x))   # 3 — x, y, and getrefcount's temp ref

del y                         # rebinds y to nothing, drops one reference
print(sys.getrefcount(x))   # back to 2

# When x goes out of scope (function ends, module ends), refcount drops to 0 → freed immediately
```

What increases the reference count:

- Assigning a name (`x = obj`)
- Passing as a function argument (the parameter is a new name → +1)
- Appending to a container (`list.append(obj)` → the list holds a reference)
- Returning from a function (the caller receives a reference)

What decreases it:

- `del name` — removes the name from the namespace
- Rebinding (`x = something_else` — x no longer points at the old object)
- A container is deleted or an element removed
- A name goes out of scope (function returns, with-block exits)

### F.2 Why reference counting alone is not enough — cycles

Reference counting has one known weakness: **circular references**.

```python
a = {}
b = {}
a["partner"] = b    # a holds a reference to b
b["partner"] = a    # b holds a reference to a

# now even if we do:
del a
del b
# the objects still reference EACH OTHER, so each refcount is still 1, never reaches 0
# they become unreachable from any live name, but they never get freed by refcounting alone
```

```
External namespace (after del a, del b):
  nothing pointing at either object

But internally:
  dict object A  ──▶  dict object B
  dict object B  ──▶  dict object A

  refcount of A: 1 (held by B)
  refcount of B: 1 (held by A)
  → neither ever reaches 0 → memory leak without the cycle collector
```

### F.3 The cyclic garbage collector — the secondary mechanism

Python includes a dedicated **cyclic garbage collector** (in the `gc` module) that runs
periodically and finds groups of objects that are only reachable from each other — i.e., isolated
reference cycles that will never be reachable from any live name again — and frees them all.

It uses a **generational** approach, based on the empirical observation that most objects die
young (a list created inside a function, used, and gone when the function returns):

```
Generation 0  — newly created objects (collected most often, cheapest to scan)
Generation 1  — survived one Gen 0 collection
Generation 2  — survived Gen 0 + Gen 1 (long-lived: module-level caches, singletons, etc.)
```

Objects get promoted from Gen 0 → Gen 1 → Gen 2 by surviving collections. Gen 0 is collected
frequently; Gen 2 is collected rarely. Most cycles involve short-lived objects and are caught in
Gen 0 collections, which are cheap and fast.

```python
import gc

gc.collect()              # manually trigger a full collection
print(gc.get_count())     # (gen0_count, gen1_count, gen2_count) — pending objects per generation
gc.disable()              # turn the cyclic GC off (reference counting still works — this only
                          # affects cycle detection)
gc.enable()               # turn it back on
```

You almost never need to touch `gc` manually. It exists, it runs automatically, and most Python
programs never think about it. The one time you might: extremely performance-critical code that
creates and destroys millions of objects per second — disabling the cyclic GC and managing object
lifetimes carefully can reduce GC pauses, but this is an optimization, not normal usage.

### F.4 The complete picture: what happens to an object over its lifetime

```
1. Object is created (e.g., list literal, function call, class instantiation)
   → allocated on the heap
   → refcount = 1
   → placed in Generation 0 of the cyclic GC

2. More names point at it (assignment, function args, container adds)
   → refcount increments for each

3. Names stop pointing at it (del, rebind, out of scope, container shrinks)
   → refcount decrements for each

4a. If refcount hits 0 at any point
    → memory freed IMMEDIATELY
    → __del__ finalizer called (if defined)
    → no cyclic GC involvement needed

4b. If refcount never hits 0 (cycle exists or object is long-lived)
    → cyclic GC eventually runs
    → if object is unreachable from any live name → freed
    → if still reachable → promoted to next generation
```

### F.5 `__del__` — finalizers (and why to be careful with them)

```python
class Connection:
    def __init__(self):
        print("Connection opened")

    def __del__(self):
        print("Connection closed")  # called when refcount hits 0 or cyclic GC collects it

c = Connection()   # "Connection opened"
del c              # "Connection closed" — immediately, because refcount hit 0
```

> ⚠️ Don't rely on `__del__` for critical cleanup (closing files, releasing locks, closing DB
> connections). The timing of `__del__` is guaranteed for non-cyclic objects (refcount → 0 =
> immediate call) but **not** for objects collected by the cyclic GC — and it may never be called
> at all if the interpreter exits with cycles unresolved. Use context managers (`with` statement /
> `__enter__`/`__exit__`) for deterministic cleanup — that's what they exist for.

### F.6 Memory pools — why Python's allocator looks different from C's

CPython doesn't call `malloc`/`free` per object — it maintains its own **memory pool**, called
**pymalloc**, for small objects (≤ 512 bytes). Memory is requested from the OS in large blocks
("arenas") and subdivided internally. This reduces the number of `malloc` system calls and the
fragmentation that would result from millions of small, frequent C-level allocations.

This is also why CPython's memory usage can appear to grow then stay high — freed objects return
to the Python-managed pool rather than immediately back to the OS. The pool is reused for future
Python objects; the OS gets it back only when an entire arena becomes empty.

---

## Part G — Common Bugs This Mental Model Prevents

### Bug 1: mutable default argument (the most infamous Python footgun)

```python
# ❌ wrong — the list is created ONCE when the def is executed, then SHARED across all calls
def add_item(item, cart=[]):
    cart.append(item)
    return cart

print(add_item("apple"))   # ["apple"]
print(add_item("banana"))  # ["apple", "banana"]  ← what?!

# ✅ right — use None as default, create a new list inside the function
def add_item(item, cart=None):
    if cart is None:
        cart = []
    cart.append(item)
    return cart
```

The default value `[]` is an object created when Python parses the `def` statement — once. Every
call that uses the default is using the **same list object**. The name/object model explains
exactly why: the default is a reference stored on the function object, not a value that gets
copied per call.

### Bug 2: aliasing a list you meant to copy

```python
# ❌ wrong
original = [1, 2, 3]
backup = original       # backup IS original — same object
original.append(4)
print(backup)           # [1, 2, 3, 4] — "backup" is also changed

# ✅ right — make an actual copy
backup = original[:]          # shallow copy
backup = list(original)       # also shallow copy
backup = original.copy()      # also shallow copy
# for nested structures:
import copy
backup = copy.deepcopy(original)
```

### Bug 3: comparing with `is` instead of `==`

```python
# ❌ wrong — works "by accident" for small ints due to the cache, breaks for larger ones
if user_id is 1000:
    ...

# ✅ right
if user_id == 1000:
    ...

# ✅ the ONE correct use of `is` — checking for None (None is a singleton, always same object)
if result is None:
    ...
if result is not None:
    ...
```

### Bug 4: "why didn't my variable change in the function"

```python
def double_it(x):
    x = x * 2   # rebinds the LOCAL name x — the CALLER's name is unchanged

n = 5
double_it(n)
print(n)  # 5 — passing an int (immutable) and rebinding inside a function never affects the caller

# ✅ if you need to "return" a new value, actually return it
def double_it(x):
    return x * 2

n = double_it(n)
print(n)  # 10
```

---

## The Three-Line Summary

1. **Python variables are names (labels), not boxes.** A name points at an object; multiple
   names can point at the same object; a name can be rebound to any object at any time, regardless
   of type.
2. **Objects have types, not variables.** Every object also has an identity (`id()`) and a
   mutability — whether its value can change in-place. Immutable objects are safe to share freely;
   mutable objects are shared by default and mutated in-place unless you explicitly copy them.
3. **Memory is managed by reference counting (immediate) + a cyclic GC (for cycles).** When
   nothing points at an object anymore — its reference count hits zero, or the cyclic GC finds it
   unreachable — the memory is reclaimed. You never manage this manually.
