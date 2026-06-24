# SQL Concepts: DISTINCT, LIKE, IS, NOT, IN — and Why No Square Brackets

Every concept in this note is something you'll use in real queries constantly. Each one is
explained from first principles — not just the syntax, but _why_ it works that way and what
mistakes people make with it — using a single consistent sample dataset throughout so you can
see how results change as the query changes.

---

## The Sample Tables (used throughout)

```sql
-- users
CREATE TABLE users (
    user_id  SERIAL PRIMARY KEY,
    username VARCHAR(50),
    email    VARCHAR(100),
    city     VARCHAR(50),
    status   VARCHAR(20)
);

INSERT INTO users (username, email, city, status) VALUES
('alice',       'alice@gmail.com',      'Mumbai',    'active'),
('bob',         'bob@yahoo.com',        'Delhi',     'active'),
('carol',       'carol@gmail.com',      'Mumbai',    'inactive'),
('dave',        'dave@outlook.com',     'Chennai',   'active'),
('eve',         NULL,                   'Mumbai',    'inactive'),
('frank',       'frank@gmail.com',      NULL,        'active'),
('grace',       'grace@hotmail.com',    'Delhi',     NULL),
('heidi',       'heidi@gmail.com',      'Bangalore', 'active');

-- orders
CREATE TABLE orders (
    order_id   SERIAL PRIMARY KEY,
    user_id    INTEGER REFERENCES users(user_id),
    amount     DECIMAL(10, 2),
    city       VARCHAR(50)
);

INSERT INTO orders (user_id, amount, city) VALUES
(1, 250.00, 'Mumbai'),
(1, 180.00, 'Mumbai'),
(2, 500.00, 'Delhi'),
(3, 90.00,  'Mumbai'),
(4, 320.00, 'Chennai'),
(4, 150.00, 'Chennai');
```

---

## 1. DISTINCT — "Give Me One of Each"

### What it is

`DISTINCT` tells SQL to **deduplicate the result set** — if multiple rows would produce the same
output value (or combination of values) for the columns you selected, keep only one of them.

### Without DISTINCT — duplicates appear

```sql
SELECT city FROM users;
```

```
city
---------
Mumbai
Delhi
Mumbai
Chennai
Mumbai
NULL
Delhi
Bangalore
```

Mumbai appears three times. Delhi appears twice. `NULL` appears once. That's how many rows in the
table have those values — `SELECT` without `DISTINCT` just returns one row per source row, always.

### With DISTINCT — one of each unique value

```sql
SELECT DISTINCT city FROM users;
```

```
city
---------
Mumbai
Delhi
Chennai
NULL
Bangalore
```

Each unique value appears exactly once. Notice `NULL` still appears — it's treated as a distinct
value here, even though it means "unknown." Five rows back instead of eight.

### DISTINCT across multiple columns — uniqueness of the _combination_

```sql
SELECT DISTINCT city, status FROM users;
```

```
city        status
---------   --------
Mumbai      active
Delhi       active
Mumbai      inactive
Chennai     active
Mumbai      inactive    ← same as row 3, right? No — eve has city=Mumbai, status=inactive too
NULL        active
Delhi       NULL
Bangalore   active
```

Wait — let's re-check. We have:

- alice: Mumbai, active
- carol: Mumbai, inactive
- eve: Mumbai, inactive ← same combination as carol

```sql
SELECT DISTINCT city, status FROM users;
```

```
city        status
---------   --------
Mumbai      active
Delhi       active
Mumbai      inactive     ← carol and eve both have (Mumbai, inactive) → only one row
Chennai     active
NULL        active
Delhi       NULL
Bangalore   active
```

`DISTINCT` looks at the **entire selected row as a unit**. Two rows are duplicates only if every
selected column matches. carol and eve both had `(Mumbai, inactive)` — so only one appears.

### WHERE comes before DISTINCT deduplication

```sql
SELECT DISTINCT city FROM users WHERE status = 'active';
```

```
city
---------
Mumbai
Delhi
Chennai
NULL
Bangalore
```

SQL filters first (`WHERE`), then deduplicates (`DISTINCT`). `DISTINCT` never sees the rows that
`WHERE` already removed.

### COUNT(DISTINCT ...) — count unique values, not total rows

```sql
SELECT COUNT(DISTINCT city) FROM users;    -- how many distinct cities (NULL not counted)
-- result: 4

SELECT COUNT(city) FROM users;             -- how many non-NULL city values total
-- result: 7
```

`COUNT(DISTINCT column)` is a very common pattern — "how many unique X values are there" — and it
counts `NULL` as nothing, same as plain `COUNT(column)`.

---

## 2. LIKE — Pattern Matching on Text

`LIKE` is for when you don't know the exact string you're looking for, but you know something
_about_ its shape. It uses two special wildcard characters:

| Wildcard | Meaning                           |
| -------- | --------------------------------- |
| `%`      | **Zero or more** of any character |
| `_`      | Exactly **one** of any character  |

This section focuses on `%` since it's what you asked about — but `_` is mentioned where useful.

### The three positions of % — and what each means

---

#### Position 1: `%` at the end — "starts with"

```sql
SELECT username, email FROM users WHERE email LIKE 'alice%';
```

The pattern `'alice%'` means: the string must **start with** the literal `alice`, then anything
(including nothing) can follow.

```
username    email
--------    ---------------
alice       alice@gmail.com
```

A more practical "starts with" example — find all Gmail users:

```sql
SELECT username, email FROM users WHERE email LIKE 'alice@gmail%';
-- finds alice@gmail.com, alice@gmail.co.uk, alice@gmail.anything — all start with alice@gmail
```

The `%` at the end is a bucket for "everything that comes after my prefix, whatever it is."

---

#### Position 2: `%` at the front — "ends with"

```sql
SELECT username, email FROM users WHERE email LIKE '%@gmail.com';
```

The pattern `'%@gmail.com'` means: anything can come before (zero or more characters), then the
string must **end with** the literal `@gmail.com`.

```
username    email
--------    ---------------
alice       alice@gmail.com
carol       carol@gmail.com
frank       frank@gmail.com
heidi       heidi@gmail.com
```

Another "ends with" example — find emails from any `.org` domain:

```sql
SELECT username, email FROM users WHERE email LIKE '%.org';
```

The `%` at the front is a bucket for "everything that comes before my suffix, whatever it is."

---

#### Position 3: `%` on both sides — "contains anywhere"

```sql
SELECT username, email FROM users WHERE email LIKE '%gmail%';
```

The pattern `'%gmail%'` means: anything can come before, the string must contain `gmail`
somewhere, and anything can come after.

```
username    email
--------    ---------------
alice       alice@gmail.com
carol       carol@gmail.com
frank       frank@gmail.com
heidi       heidi@gmail.com
```

Both-sides `%` is the most flexible but also **the slowest** of the three — the database
can't use a regular index efficiently because it doesn't know what the string starts with, so it
must scan and check every row. A leading-`%` pattern (`LIKE '%something'`) has the same
performance problem for the same reason. A trailing-`%` pattern (`LIKE 'something%`) can use a
standard B-tree index because the start of the string is fixed.

---

### Visual summary of all three positions

```
Pattern           Means                        Matches           Doesn't match
-----------       --------------------         --------          --------------
'alice%'          starts with "alice"          "alice@g.com"     "hi_alice"
                                               "alice"           "ali"

'%@gmail.com'     ends with "@gmail.com"       "x@gmail.com"     "x@yahoo.com"
                                               "a.b@gmail.com"   "@gmail.co"

'%gmail%'         contains "gmail" anywhere    "alice@gmail.com" "alice@yahoo.com"
                                               "gmailtest@g.com" "gmai"
```

---

### `%` matching zero characters — this surprises people

```sql
SELECT username FROM users WHERE username LIKE 'alice%';
```

`'alice%'` matches `'alice'` itself, not just `'alice_something'`. The `%` allows **zero** or
more characters — so "alice followed by nothing" is a valid match. `LIKE 'alice%'` is _not_
the same as "starts with alice AND has more characters after."

```sql
-- these both match 'alice':
WHERE username LIKE 'alice%'    -- matches 'alice', 'aliceX', 'alice123'
WHERE username LIKE '%'         -- matches everything (any string, including empty)
```

---

### `_` — exactly one character (mentioned for completeness)

```sql
SELECT username FROM users WHERE username LIKE '_ob';
-- matches: 'bob', 'rob', 'job' — any single character, then 'ob'

SELECT username FROM users WHERE username LIKE '____';
-- matches any username that is exactly 4 characters long (four underscores = four characters)
-- matches: 'dave', 'heidi'? No — 'heidi' is 5 chars. Only 'dave' here.
```

---

### Case sensitivity in LIKE

Standard SQL `LIKE` **is case-sensitive**. `'alice%'` does not match `'Alice'` or `'ALICE'`.

```sql
-- Postgres: use ILIKE for case-insensitive matching
SELECT username FROM users WHERE email ILIKE '%GMAIL%';
-- matches gmail, Gmail, GMAIL, gMaIl, etc.

-- MySQL: LIKE is case-insensitive by default for most collations — check your collation setting
-- SQL Server: depends on the column's collation (typically case-insensitive by default)
```

---

### LIKE does NOT match NULL

```sql
SELECT username, city FROM users WHERE city LIKE 'M%';
```

frank's `city` is `NULL`. `NULL LIKE 'M%'` does not return `true`. It doesn't return `false`
either. It returns `NULL` — and `WHERE` only keeps rows where the condition evaluates to `true`.
frank is not included. This is the same `NULL` behavior as the next section.

---

## 3. IS — The Only Safe Way to Check for NULL

### Why `= NULL` is wrong

This is one of the most common SQL mistakes, especially coming from Python/JS/Java where
`x == null` is perfectly normal:

```sql
-- ❌ This looks right but returns NO rows — always
SELECT username FROM users WHERE city = NULL;

-- ❌ This also returns NO rows — always
SELECT username FROM users WHERE city != NULL;
```

`NULL` in SQL means **unknown**. Is unknown equal to unknown? SQL says: _unknown_ — it can't
know. The result of `NULL = NULL` is `NULL`, not `TRUE`. The result of `NULL != NULL` is also
`NULL`, not `TRUE` or `FALSE`. `WHERE` keeps only rows where the condition is `TRUE` — a `NULL`
result is treated the same as `FALSE`, so every row with a `NULL` is silently dropped.

### `IS NULL` — find rows with missing values

```sql
SELECT username, city FROM users WHERE city IS NULL;
```

```
username    city
--------    ----
frank       NULL
```

`IS NULL` is specifically designed to test for `NULL` — it evaluates to `TRUE` when the value is
`NULL`, and it's the only thing that does. There's no "trick" here; it's a distinct comparison
operator with different semantics than `=`.

### `IS NOT NULL` — find rows where the value is present

```sql
SELECT username, email FROM users WHERE email IS NOT NULL;
```

```
username    email
--------    -----------------
alice       alice@gmail.com
bob         bob@yahoo.com
carol       carol@gmail.com
dave        dave@outlook.com
frank       frank@gmail.com
grace       grace@hotmail.com
heidi       heidi@gmail.com
```

eve is excluded — her `email` is `NULL`. Seven rows instead of eight.

### IS with boolean-like values (Postgres)

```sql
-- Postgres supports IS TRUE / IS FALSE / IS UNKNOWN for boolean columns
SELECT * FROM flags WHERE is_verified IS TRUE;
SELECT * FROM flags WHERE is_verified IS NOT FALSE;
```

Standard SQL also defines `IS` for `TRUE`/`FALSE`/`UNKNOWN` — but in practice with most
databases you'll mostly use `IS NULL` and `IS NOT NULL`, which work identically everywhere.

### NULL in comparisons — the rule to remember

Any comparison with `NULL` using `=`, `!=`, `<`, `>`, `LIKE` etc. produces `NULL` (unknown),
never `TRUE` — which means the row is never kept by `WHERE`. The only way to test for `NULL`
is `IS NULL` or `IS NOT NULL`. No exceptions.

```
NULL = NULL        → NULL  (not TRUE)
NULL != NULL       → NULL  (not TRUE)
NULL = 'alice'     → NULL  (not TRUE)
NULL LIKE '%'      → NULL  (not TRUE)
NULL IS NULL       → TRUE  ✅
NULL IS NOT NULL   → FALSE
```

---

## 4. NOT — Inverting Any Condition

`NOT` flips a condition from `TRUE` to `FALSE` and from `FALSE` to `TRUE`. It sits in front of
whatever condition it inverts.

```sql
SELECT username, status FROM users WHERE NOT status = 'active';
-- equivalent: WHERE status != 'active'
```

```
username    status
--------    --------
carol       inactive
eve         inactive
grace       NULL      ← wait, is NULL included?
```

No — `NULL != 'active'` produces `NULL` (unknown), and `NOT NULL` is also `NULL`. `WHERE` drops
it. grace does NOT appear.

```sql
-- to get non-active AND include NULLs:
SELECT username, status FROM users WHERE status != 'active' OR status IS NULL;
```

```
username    status
--------    --------
carol       inactive
eve         inactive
grace       NULL
```

### NOT with LIKE

```sql
SELECT username, email FROM users WHERE email NOT LIKE '%gmail.com';
```

```
username    email
--------    -----------------
bob         bob@yahoo.com
dave        dave@outlook.com
grace       grace@hotmail.com
```

Note: eve (whose email is `NULL`) does NOT appear — `NULL NOT LIKE '%gmail.com'` is `NULL`, same
NULL-propagation rule as always.

### NOT with IN

```sql
SELECT username, city FROM users WHERE city NOT IN ('Mumbai', 'Delhi');
```

```
username    city
--------    ---------
dave        Chennai
heidi       Bangalore
```

frank (city is `NULL`) and grace (in Delhi) are not included. `NULL NOT IN (...)` is `NULL` —
disappears. This is covered in more detail in the next section.

### NOT with IS

```sql
WHERE status IS NOT NULL    -- this is "IS NOT NULL", not "NOT IS NULL"
WHERE NOT status IS NULL    -- also valid in standard SQL — means the same thing, just unusual
```

The `IS NOT NULL` form is what everyone writes — it reads naturally. `NOT IS NULL` is valid SQL
but no one writes it that way.

---

## 5. IN — Matching Against a List of Values

### What it replaces

Without `IN`, checking if a value matches one of several possibilities requires chained `OR`:

```sql
SELECT username, city FROM users WHERE city = 'Mumbai' OR city = 'Delhi' OR city = 'Chennai';
```

This works but grows ugly fast. `IN` is the clean version of the same logic:

```sql
SELECT username, city FROM users WHERE city IN ('Mumbai', 'Delhi', 'Chennai');
```

```
username    city
--------    --------
alice       Mumbai
bob         Delhi
carol       Mumbai
dave        Chennai
eve         Mumbai
```

frank (city is `NULL`) is not included — same rule: `NULL IN (...)` is `NULL`, not `TRUE`.

### The list is a comma-separated set of values in `()`

```sql
WHERE city IN ('Mumbai', 'Delhi', 'Chennai')
WHERE user_id IN (1, 2, 4)
WHERE status IN ('active', 'pending', 'trial')
```

The list is enclosed in `()` (parentheses) — not `[]`, not `{}`. This is covered in full detail
in Part 6 below.

### NOT IN — exclude a list of values

```sql
SELECT username, city FROM users WHERE city NOT IN ('Mumbai', 'Delhi');
```

```
username    city
--------    ---------
dave        Chennai
heidi       Bangalore
```

### ⚠️ The NOT IN + NULL trap — the most dangerous SQL gotcha

This is the single most important thing to understand about `IN` and `NOT IN`:

```sql
SELECT username FROM users WHERE city NOT IN ('Mumbai', 'Delhi', NULL);
```

**Returns zero rows.**

Why? SQL expands `NOT IN (...)` to a series of `AND NOT` comparisons:

```
city NOT IN ('Mumbai', 'Delhi', NULL)
is equivalent to:
city != 'Mumbai' AND city != 'Delhi' AND city != NULL
```

And `city != NULL` is always `NULL` (unknown). `anything AND NULL` is `NULL`. So every single row
fails the condition — even rows where `city` is `'Chennai'`. Zero rows returned.

In practice this happens when you write:

```sql
WHERE city NOT IN (SELECT city FROM some_table)
-- and some_table has a NULL city value
```

You get zero results and no error — it just silently returns nothing.

**The fix:** always filter `NULL` out of a subquery used with `NOT IN`:

```sql
WHERE city NOT IN (SELECT city FROM some_table WHERE city IS NOT NULL)
```

Or use `NOT EXISTS` instead of `NOT IN` with subqueries — `NOT EXISTS` handles `NULL` correctly
by design. This is one of the main reasons experienced SQL writers prefer `NOT EXISTS` over
`NOT IN` for subqueries.

### IN with a subquery

`IN` is commonly used to check against the result of another query:

```sql
-- users who have placed at least one order
SELECT username FROM users
WHERE user_id IN (SELECT user_id FROM orders);
```

```
username
--------
alice
bob
carol
dave
```

```sql
-- users who have NOT placed any order (safe — orders.user_id has no NULLs here)
SELECT username FROM users
WHERE user_id NOT IN (SELECT user_id FROM orders);
```

```
username
--------
eve
frank
grace
heidi
```

The subquery returns a result set; `IN` checks each row against every value in that set. If the
subquery returns `NULL` values and you're using `NOT IN`, see the trap above.

---

## 6. Why SQL Uses `()` for Lists, Not `[]`

This is a question that comes up naturally when you already know Python (where lists are `[1, 2,
3]`), JavaScript (same), or Java (`List.of(1, 2, 3)`). SQL uses `()` for `IN` lists, and this
feels inconsistent or arbitrary at first. There's a concrete reason.

### `()` in SQL already means many things — and they're all the same idea

In SQL, parentheses `()` always denote a **grouping or a set of values for evaluation**. That one
meaning applies consistently across:

```sql
-- 1. Arithmetic grouping (same as every language)
SELECT (price * quantity) + tax FROM orders;

-- 2. Subquery — a complete query treated as a value or set
SELECT username FROM users WHERE user_id IN (SELECT user_id FROM orders);

-- 3. A list of values for IN
WHERE city IN ('Mumbai', 'Delhi', 'Chennai')

-- 4. A row of values for INSERT
INSERT INTO users (username, email) VALUES ('alice', 'alice@gmail.com');

-- 5. Column list in CREATE TABLE
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50)
);

-- 6. Function arguments
SELECT UPPER(username), LENGTH(email) FROM users;
```

All six use `()`. The pattern is: parentheses enclose **a set of things that are evaluated or
treated as a group**. A list of values for `IN` fits naturally into this pattern.

### `[]` already has a different meaning in SQL

Square brackets are used in **SQL Server and MS Access** specifically as a **quoting mechanism
for identifiers** — to handle reserved words or names with spaces used as column/table names:

```sql
-- SQL Server / MS Access syntax:
SELECT [user id], [from] FROM [order table];

-- In standard SQL and Postgres, you use double quotes for the same purpose:
SELECT "user id", "from" FROM "order table";
```

If `[]` were also used for lists (like Python uses them), there would be an immediate ambiguity:
is `[status]` the column named `status`, or a list containing the value `status`? SQL Server
avoided this by keeping `[]` strictly for identifier quoting — not for lists.

### `{}` is used for different purposes in SQL dialects

`{}` (curly braces) appear in some SQL contexts for **template parameters and ODBC escape
sequences**, not for data structures:

```sql
-- ODBC escape sequence for date literals (some drivers)
WHERE created_at > {d '2024-01-01'}

-- Positional parameter in some ORMs/drivers
WHERE user_id = {0}
```

### The deeper reason: SQL predates Python and modern languages

SQL was designed in the early 1970s (IBM's SEQUEL, which became SQL). Python's list syntax `[]`
didn't exist yet — Python itself was created in 1991. SQL's use of `()` for grouping was already
established and internally consistent. When SQL needed a "match against multiple values" operator
(`IN`), using `()` was the natural and consistent choice given the language's existing conventions.

The languages you're comparing SQL to (Python, JavaScript, Java) made different choices for their
own internal consistency — `[]` for ordered mutable lists in Python/JS, `List.of()` in Java.
SQL made `()` work for everything because SQL only has one kind of "collection of values" in
this context: an unordered set of literals to match against.

### Summary: which symbol does what, in each language

| Symbol | Python                         | JavaScript              | Java                    | SQL                                                                  |
| ------ | ------------------------------ | ----------------------- | ----------------------- | -------------------------------------------------------------------- |
| `()`   | function call, tuple, grouping | function call, grouping | function call, grouping | function call, grouping, IN list, subquery, column list, VALUES list |
| `[]`   | list literal, indexing         | array literal, indexing | array access            | identifier quoting (SQL Server only)                                 |
| `{}`   | dict/set literal, f-string     | object literal          | block, generic bound    | ODBC escapes, template parameters                                    |

SQL has no concept of a "list literal" as a value you store in a variable — SQL is a query
language, not a general-purpose programming language, so there's no need for one. `IN (...)` is
not "a variable holding a list" — it's a syntactic clause that SQL evaluates inline as part of
the query. Parentheses do that job everywhere in SQL, and `IN` is no different.

---

## Quick Reference

```sql
-- DISTINCT: one row per unique value (or combination)
SELECT DISTINCT city FROM users;
SELECT DISTINCT city, status FROM users;
SELECT COUNT(DISTINCT city) FROM users;

-- LIKE: pattern matching
WHERE email LIKE 'alice%'        -- starts with "alice"
WHERE email LIKE '%@gmail.com'   -- ends with "@gmail.com"
WHERE email LIKE '%gmail%'       -- contains "gmail" anywhere
WHERE email NOT LIKE '%gmail%'   -- does NOT contain "gmail" (NULLs are excluded silently)
WHERE username LIKE '_ob'        -- exactly one char, then "ob" (e.g. "bob", "rob")
WHERE username ILIKE '%alice%'   -- case-insensitive (Postgres only)

-- IS: the ONLY correct way to check for NULL
WHERE city IS NULL               -- city has no value
WHERE city IS NOT NULL           -- city has a value

-- NOT: invert any condition
WHERE NOT status = 'active'      -- same as != 'active', but NULLs still excluded
WHERE status IS NOT NULL         -- the standard "not null" form
WHERE city NOT LIKE '%Mumbai%'   -- does not contain Mumbai (NULLs excluded)

-- IN: match against a list
WHERE city IN ('Mumbai', 'Delhi', 'Chennai')
WHERE user_id IN (1, 2, 4)
WHERE user_id IN (SELECT user_id FROM orders)

-- NOT IN: exclude a list — ALWAYS filter NULLs from subqueries
WHERE city NOT IN ('Mumbai', 'Delhi')
WHERE user_id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL)

-- () not [] — SQL uses parentheses for all value grouping, IN lists, subqueries, VALUES
-- [] is only used in SQL Server/Access for quoting identifier names, not for lists
```
