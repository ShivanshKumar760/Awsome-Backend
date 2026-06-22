# Python Notes — Variables Through Parallel & Concurrent Programming

Covers topics 27–38 from your course: core language basics (variables, lists, dicts, loops, conditionals, error handling, strings, functions, scope, args/kwargs) and finishes with a deep dive into Parallel and Concurrent Programming — which ties directly into the `threading.Lock` you already used in the Flask long-polling API.

---

## 27. Variables and Data Types

A **variable** is a name bound to a value in memory. Python is **dynamically typed** — you don't declare a type; the variable just points to whatever object you assign it.

```python
age = 25            # int
price = 19.99        # float
name = "Alice"        # str
is_active = True      # bool
nothing = None        # NoneType — represents "no value"
```

**Core built-in types:**

| Type | Example | Notes |
|---|---|---|
| `int` | `42`, `-7` | whole numbers, arbitrary precision |
| `float` | `3.14`, `2.0` | decimal numbers |
| `str` | `"hello"` | immutable sequence of characters |
| `bool` | `True`, `False` | subclass of `int` (`True == 1`) |
| `NoneType` | `None` | Python's "null" |

```python
x = 5
type(x)        # <class 'int'>
x = "now a string"
type(x)        # <class 'str'> — same variable, new type, totally legal
```

**Type conversion ("casting"):**

```python
int("42")      # 42
str(42)        # "42"
float("3.5")   # 3.5
bool(0)        # False — 0, "", None, [], {} are all "falsy"
bool(1)        # True — everything else is "truthy"
```

---

## 28. Working with Lists

A **list** is an ordered, mutable (changeable) collection — Python's general-purpose array.

```python
fruits = ["apple", "banana", "cherry"]

fruits[0]          # "apple"  — indexing starts at 0
fruits[-1]         # "cherry" — negative index counts from the end
fruits[1:3]        # ["banana", "cherry"] — slicing [start:stop]
```

**Common operations:**

```python
fruits.append("mango")        # add to the end
fruits.insert(1, "kiwi")      # insert at a specific index
fruits.remove("banana")       # remove by value (first match)
fruits.pop()                  # remove & return the last item
fruits.pop(0)                 # remove & return item at index 0
len(fruits)                   # number of items
"apple" in fruits              # membership check -> True/False
fruits.sort()                  # sort in place
fruits.reverse()               # reverse in place
```

**List comprehensions** — build a new list in one line:

```python
squares = [n**2 for n in range(5)]          # [0, 1, 4, 9, 16]
evens = [n for n in range(10) if n % 2 == 0]  # [0, 2, 4, 6, 8]
```

**Mutability gotcha:**

```python
a = [1, 2, 3]
b = a            # b points to the SAME list, not a copy
b.append(4)
print(a)         # [1, 2, 3, 4]  — a changed too!

c = a.copy()     # an actual independent copy
```

---

## 29. Working with Dictionaries

A **dictionary** (`dict`) stores **key → value** pairs. Unordered conceptually (though Python 3.7+ preserves insertion order), and keys must be unique and hashable (strings, numbers, tuples — not lists).

```python
user = {
    "username": "priya_d",
    "email": "priya@example.com",
    "age": 29,
}

user["username"]          # "priya_d"
user.get("bio")            # None — .get() doesn't raise an error if missing
user.get("bio", "N/A")     # "N/A" — provide a default
user["bio"] = "Photographer 📸"   # add or update a key
del user["age"]             # remove a key
"email" in user              # True/False — checks KEYS by default
```

**Iterating:**

```python
for key in user:
    print(key, user[key])

for key, value in user.items():
    print(key, value)

for value in user.values():
    print(value)
```

**Dict comprehension:**

```python
squares = {n: n**2 for n in range(5)}   # {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

**Nested structures (very common in real APIs):**

```python
post = {
    "post_id": 101,
    "caption": "Beautiful sunset",
    "author": {"user_id": 1, "username": "elonmusk"},
    "likes": ["priya_d", "rahul99"],
}
post["author"]["username"]   # "elonmusk"
```

This is exactly the shape of the JSON you `jsonify()` in a Flask route — a Python dict and a JSON object are nearly the same concept.

---

## 30. Using `for` Loops

A `for` loop iterates over any **iterable** — lists, dicts, strings, ranges, files, etc.

```python
for fruit in ["apple", "banana", "cherry"]:
    print(fruit)

for i in range(5):           # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 10, 2):    # start, stop, step -> 2, 4, 6, 8
    print(i)

for index, fruit in enumerate(["apple", "banana"]):
    print(index, fruit)      # 0 apple / 1 banana

for name, score in zip(["a", "b"], [90, 80]):
    print(name, score)       # pairs elements from two lists together
```

`break` exits the loop entirely; `continue` skips to the next iteration:

```python
for n in range(10):
    if n == 5:
        break        # stop the loop completely
    if n % 2 == 0:
        continue     # skip even numbers, keep going
    print(n)         # prints 1, 3
```

---

## 31. While Loops Explained

A `while` loop runs **as long as a condition stays true** — used when you don't know the number of iterations ahead of time (unlike `for`, which is usually used for a known sequence).

```python
count = 0
while count < 5:
    print(count)
    count += 1     # without this, the loop never ends — infinite loop!
```

**Real example — exactly the pattern behind long polling:**

```python
import time

start = time.time()
while time.time() - start < 25:    # keep checking for up to 25 seconds
    if something_changed():
        break
    time.sleep(0.5)
```

`while True:` with a `break` inside is a common pattern for "loop forever until some condition fires":

```python
while True:
    command = input("Enter a command: ")
    if command == "quit":
        break
    print(f"Running {command}")
```

---

## 32. If Statements

Conditional branching — run different code depending on a boolean condition.

```python
age = 20

if age < 13:
    category = "child"
elif age < 20:
    category = "teen"
else:
    category = "adult"
```

**Comparison & logical operators:**

```python
==, !=, <, >, <=, >=        # comparisons
and, or, not                # logical combination

if age >= 18 and has_id:
    print("Allowed in")

if not is_banned:
    print("Welcome")
```

**Truthy/falsy shortcuts** (very common in real code):

```python
bio = ""
if bio:                 # False, because empty string is falsy
    print(bio)
else:
    print("No bio set")

# equivalent to: if bio != "" and bio is not None:
```

**Ternary (inline if/else):**

```python
status = "active" if is_logged_in else "guest"
```

---

## 33. `try` and `except`

Used for **exception handling** — gracefully responding to errors instead of crashing the whole program.

```python
try:
    value = int(user_input)
except ValueError:
    print("That wasn't a valid number")
```

**Multiple exception types, plus `else` and `finally`:**

```python
try:
    result = 10 / divisor
except ZeroDivisionError:
    print("Can't divide by zero")
except TypeError:
    print("Invalid type for division")
else:
    print("Division succeeded:", result)   # only runs if NO exception happened
finally:
    print("This always runs, error or not")  # cleanup code (closing files, connections)
```

**Catching the exception object itself:**

```python
try:
    risky_operation()
except Exception as e:
    print(f"Something went wrong: {e}")
```

This is exactly what we did with `psycopg2.errors.UniqueViolation` and `IntegrityError` in the Flask API — catching a *specific* exception type lets you respond with a clean `409 Conflict` instead of letting the whole request crash with a `500`.

```python
try:
    cur.execute("INSERT INTO likes (user_id, post_id) VALUES (%s, %s)", (uid, pid))
except psycopg2.errors.UniqueViolation:
    return jsonify({"error": "Already liked"}), 409
```

---

## 34. String Concatenation

Combining strings together.

```python
first = "Hello"
second = "World"

# 1. Plus operator
greeting = first + " " + second           # "Hello World"

# 2. f-strings (modern, preferred — supports inline expressions)
greeting = f"{first} {second}"             # "Hello World"
greeting = f"{first.upper()}, {2 + 2}!"     # "HELLO, 4!"

# 3. .format()
greeting = "{} {}".format(first, second)

# 4. join() — best for combining many strings (e.g. a list)
words = ["Hello", "World", "!"]
greeting = " ".join(words)                 # "Hello World !"
```

**Gotcha:** `+` between a string and a non-string raises `TypeError` — you must convert first:

```python
age = 25
"Age: " + age          # ❌ TypeError
"Age: " + str(age)     # ✅ "Age: 25"
f"Age: {age}"          # ✅ easiest — f-strings auto-convert
```

---

## 35. Functions

A **function** is a reusable, named block of code.

```python
def greet(name):
    return f"Hello, {name}!"

greet("Priya")     # "Hello, Priya!"
```

**Default arguments, multiple returns, type hints:**

```python
def create_post(caption, image_url, is_public=True):   # default value
    return {"caption": caption, "image_url": image_url, "is_public": is_public}

create_post("Sunset", "https://...")                # is_public defaults to True
create_post("Sunset", "https://...", is_public=False) # override the default

def divide(a, b):
    if b == 0:
        return None, "Cannot divide by zero"   # return multiple values as a tuple
    return a / b, None

result, error = divide(10, 2)
```

```python
def add(a: int, b: int) -> int:    # type hints — purely documentation, not enforced at runtime
    return a + b
```

**Functions are first-class objects** — you can pass them around like any other value:

```python
def apply_twice(func, value):
    return func(func(value))

apply_twice(lambda x: x * 2, 5)   # 20  — lambda = small anonymous inline function
```

This is exactly what powers decorators like `@token_required` — a decorator is a function that takes a function and returns a new (wrapped) function.

---

## 36. Global Variables

By default, a variable assigned **inside** a function is **local** to that function — it disappears when the function ends and doesn't affect anything outside.

```python
counter = 0

def increment():
    counter = counter + 1   # ❌ UnboundLocalError! Python sees the assignment
                              #    and treats `counter` as local for the whole function
```

To actually modify a variable defined **outside** the function, use `global`:

```python
counter = 0

def increment():
    global counter
    counter += 1

increment()
increment()
print(counter)   # 2
```

**This is exactly the pattern from the Flask long-polling code:**

```python
latest_event = {"version": 0, "type": None, "data": None}

def publish_event(event_type, data):
    global latest_event           # not even strictly required here since we're
    with lock:                    # mutating the dict in place, not reassigning it —
        latest_event["version"] += 1   # but it signals intent clearly
        latest_event["type"] = event_type
        latest_event["data"] = data
```

**Important subtlety:** `global` is only required when **reassigning** the name itself (`counter = counter + 1`). If you're just **mutating** an existing mutable object (`my_list.append(x)`, `my_dict["key"] = x`), you don't technically need `global` — but it's still good practice to declare it for readability, especially in multi-threaded code where you want it obvious which variables are shared state.

**Why global variables need care in a web server:** as covered in the Flask notes, multiple requests can run concurrently on different threads — all reading/writing the *same* global variable. That's precisely why `latest_event` is wrapped in a `threading.Lock()` every time it's touched.

---

## 37. `*args` and `**kwargs`

These let a function accept a **variable number of arguments** — useful when you don't know in advance how many values will be passed.

### `*args` — variable positional arguments

Collects extra positional arguments into a **tuple**.

```python
def total(*args):
    return sum(args)

total(1, 2)         # 3
total(1, 2, 3, 4)    # 10
total()              # 0
```

### `**kwargs` — variable keyword arguments

Collects extra keyword arguments into a **dict**.

```python
def create_user(**kwargs):
    print(kwargs)

create_user(username="priya_d", email="priya@example.com")
# {'username': 'priya_d', 'email': 'priya@example.com'}
```

### Combining everything

Order matters: regular args → `*args` → keyword-only defaults → `**kwargs`.

```python
def log_event(event_type, *args, source="api", **kwargs):
    print(f"[{source}] {event_type}", args, kwargs)

log_event("new_like", 101, 1, source="poll", note="from like endpoint")
# [poll] new_like (101, 1) {'note': 'from like endpoint'}
```

### Unpacking — the reverse direction

`*` and `**` can also **expand** an existing list/dict into a function call:

```python
def greet(first, last):
    return f"Hello, {first} {last}"

name_parts = ["Priya", "Desai"]
greet(*name_parts)              # unpacks list into 2 positional args -> "Hello, Priya Desai"

user_data = {"first": "Priya", "last": "Desai"}
greet(**user_data)              # unpacks dict into keyword args -> "Hello, Priya Desai"
```

**Where this shows up in real code — decorators:** `*args, **kwargs` is exactly how a decorator can wrap *any* function, regardless of what arguments it takes:

```python
def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):     # accepts whatever the wrapped route needs
        # ... auth check ...
        return f(*args, **kwargs)        # forwards everything through untouched
    return decorated
```

---

## 38. Parallel and Concurrent Programming

This is the conceptual foundation behind why our Flask long-polling code needed `threading.Lock()`.

### Concurrency vs Parallelism — they are *not* the same thing

| | Concurrency | Parallelism |
|---|---|---|
| **Definition** | Multiple tasks **make progress** during overlapping time periods — but not necessarily at the *exact same instant* | Multiple tasks run **literally at the same time**, on separate CPU cores |
| **Analogy** | One chef juggling 3 dishes — stirring pot A, then chopping for B, then checking oven for C, switching rapidly | Three chefs, each fully focused on their own dish, simultaneously |
| **Requires multiple cores?** | No — can happen on a single core via fast switching | Yes — genuinely needs multiple CPU cores |
| **Goal** | Stay responsive while waiting on slow things (network, disk, user input) | Finish CPU-heavy work faster by splitting it up |

Concurrency is about **structure** ("can my program deal with many things being in-progress at once?"); parallelism is about **execution** ("are things physically happening simultaneously?"). You can have concurrency without parallelism (single core, switching fast) and you can have parallelism without much concurrency (one big task split into independent chunks).

### Why Python complicates this: the GIL

CPython (the standard Python interpreter) has a **Global Interpreter Lock (GIL)** — only **one thread** can execute Python bytecode at any given instant, even on a multi-core machine. This means:

- **`threading` does NOT give you true parallelism for CPU-bound Python code.** Two threads doing heavy math don't actually run simultaneously — they take turns.
- **`threading` IS great for I/O-bound work** — waiting on a network request, a database query, a file read, `time.sleep()`. While one thread is *waiting* (not executing Python bytecode), the GIL is released and another thread can run. This is exactly why `threaded=True` works for our Flask long-polling endpoint: each long-poll request spends almost all its time inside `time.sleep(0.5)`, releasing the GIL constantly so other requests get served too.
- **`multiprocessing` DOES give you true parallelism** — it spins up entirely separate OS processes, each with its own Python interpreter and its own GIL, genuinely running on different CPU cores. Good for CPU-bound work (image processing, heavy computation), but each process has its own memory — sharing data between them is more expensive (you need `multiprocessing.Queue`, shared memory, etc.) than sharing a variable between threads.

### `threading` example — I/O-bound, benefits from concurrency

```python
import threading
import time

def fetch_data(name, delay):
    print(f"{name}: starting")
    time.sleep(delay)         # simulates a slow network/DB call — GIL is released here
    print(f"{name}: done")

t1 = threading.Thread(target=fetch_data, args=("Request A", 2))
t2 = threading.Thread(target=fetch_data, args=("Request B", 2))

t1.start()
t2.start()
t1.join()    # wait for t1 to finish
t2.join()    # wait for t2 to finish
# Total time: ~2 seconds, NOT 4 — both waits happened concurrently
```

### `multiprocessing` example — CPU-bound, benefits from real parallelism

```python
import multiprocessing
import time

def heavy_computation(n):
    total = 0
    for i in range(n):
        total += i ** 2
    return total

if __name__ == "__main__":
    with multiprocessing.Pool(processes=4) as pool:
        results = pool.map(heavy_computation, [10_000_000] * 4)
    # all 4 calls genuinely run on separate cores simultaneously
```

### Race conditions — the core danger of shared state

A **race condition** happens when multiple threads read and write the same shared data without coordination, and the final result depends on unpredictable timing.

```python
counter = 0

def increment():
    global counter
    for _ in range(100_000):
        counter += 1     # NOT atomic! This is actually 3 steps:
                          # 1. read counter   2. add 1   3. write counter back

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # you'd EXPECT 500,000 — but it'll likely be LESS,
                  # because two threads can both "read" the same old value
                  # before either "writes" their updated result back
```

### The fix: `Lock` (exactly what we used in the Flask API)

```python
counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100_000):
        with lock:              # only one thread executes this block at a time
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # always exactly 500,000 now
```

`with lock:` acquires the lock on entry and **guarantees** it's released on exit — even if an exception happens inside the block. This is the same `threading.Lock()` pattern from `publish_event()` and `wait_for_new_event()` in the long-polling API: every read or write of the shared `latest_event` dict happens inside `with lock:`, so no two requests can corrupt it by overlapping.

### Other concurrency tools worth knowing

| Tool | Best for |
|---|---|
| `threading.Lock` | Protecting a single shared resource (what we used) |
| `threading.Event` | One thread signaling "something happened" to others waiting on it — an alternative to polling-with-sleep for long polling |
| `queue.Queue` | Thread-safe producer/consumer pattern — naturally already locks internally |
| `multiprocessing.Pool` | Splitting CPU-heavy work across cores |
| `asyncio` | Single-threaded concurrency via `async`/`await` — cooperative multitasking, great for huge numbers of concurrent I/O-bound tasks (e.g. thousands of long-poll connections) without one OS thread per connection |

### Tying it back to the Flask long-polling API

| Concept here | Where it appeared in the Flask notes |
|---|---|
| Threads handling requests concurrently | `api.run(threaded=True)` |
| Shared global state | `latest_event = {...}` |
| Race condition risk | Two requests reading/writing `latest_event` at once |
| The fix | `lock = threading.Lock()` + `with lock:` around every access |
| I/O-bound waiting releasing the GIL | `time.sleep(0.5)` inside `wait_for_new_event()`, letting other requests get served while one is "waiting" |

This is exactly why the "global lock" section in the Flask notes matters — it's not an arbitrary detail, it's the direct, practical application of everything in this concurrency topic.

---

## Quick Reference Summary

| Topic | Key idea |
|---|---|
| Variables & Data Types | Names bound to dynamically-typed values; `int`, `float`, `str`, `bool`, `None` |
| Lists | Ordered, mutable; index/slice/append/comprehension |
| Dictionaries | Key→value pairs; `.get()`, `.items()`, nested structures mirror JSON |
| `for` loops | Iterate over any iterable; `range()`, `enumerate()`, `zip()` |
| `while` loops | Loop while a condition holds; basis of polling loops |
| `if` statements | Branching logic; truthy/falsy; ternary expressions |
| `try`/`except` | Graceful error handling; `else`/`finally`; catching specific exception types |
| String concatenation | `+`, f-strings (preferred), `.format()`, `.join()` |
| Functions | Reusable code blocks; default args, multiple returns, first-class objects |
| Global variables | `global` keyword to modify outer-scope variables; shared state risk in threads |
| `*args`/`**kwargs` | Variable-length positional/keyword arguments; unpacking with `*`/`**` |
| Parallel & Concurrent Programming | Concurrency (overlapping progress) vs parallelism (simultaneous execution); GIL; `threading` for I/O, `multiprocessing` for CPU; race conditions; `Lock` as the fix |
