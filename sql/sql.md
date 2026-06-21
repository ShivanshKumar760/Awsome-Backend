# SQL Notes (PostgreSQL) — From Tables to Joins

**Running example used throughout:** an Instagram-style app with three tables — `users`, `posts`, and `likes`. This keeps every concept grounded in something real instead of abstract `foo`/`bar` tables.

---

## 0. The Instagram Example Schema (used in every section below)

Before diving into topics, here's the schema we'll keep reusing. Understanding **why** it's built this way is itself a key SQL skill (Primary Keys, Foreign Keys, Composite Keys).

### Key Concepts: PK, FK, and Composite Key

| Term                 | Meaning                                                                                                                           | Instagram analogy                                                                                                                                                                                                                          |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Primary Key (PK)** | A column (or set of columns) that uniquely identifies each row in a table. Cannot be NULL, cannot repeat.                         | Every user has a unique `user_id`. No two users share one.                                                                                                                                                                                 |
| **Foreign Key (FK)** | A column in one table that points to the Primary Key of another table. This is how tables are _related_ to each other.            | Every post has a `user_id` column that points back to the user who created it.                                                                                                                                                             |
| **Composite Key**    | A Primary Key made of **two or more columns combined**, used when no single column is unique on its own but the _combination_ is. | A "like" isn't unique by `user_id` alone (one user likes many posts) or by `post_id` alone (one post is liked by many users) — but the **pair** (`user_id`, `post_id`) together is unique, since a person can only like a given post once. |

### The Tables

```sql
-- USERS table: one row per Instagram user
CREATE TABLE users (
    user_id     SERIAL PRIMARY KEY,             -- PK: uniquely identifies each user
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(100) UNIQUE NOT NULL,
    full_name   VARCHAR(100),
    bio         TEXT,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- POSTS table: one row per post, each post belongs to exactly one user
CREATE TABLE posts (
    post_id     SERIAL PRIMARY KEY,              -- PK: uniquely identifies each post
    user_id     INTEGER NOT NULL REFERENCES users(user_id), -- FK: links post -> owner
    caption     TEXT,
    image_url   VARCHAR(255),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- LIKES table: a JUNCTION TABLE connecting users and posts (many-to-many)
CREATE TABLE likes (
    user_id     INTEGER NOT NULL REFERENCES users(user_id), -- FK -> users
    post_id     INTEGER NOT NULL REFERENCES posts(post_id), -- FK -> posts
    liked_at    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)  -- COMPOSITE KEY: a user can like a post only once
);
```

**How the relationships map out:**

- `users` → `posts` is **One-to-Many**: one user can have _many_ posts, but each post belongs to _only one_ user. This is captured by simply putting a `user_id` foreign key on the "many" side (`posts`).
- `users` ↔ `posts` via `likes` is **Many-to-Many**: one user can like _many_ posts, and one post can be liked by _many_ users. A many-to-many relationship can't be represented with a single FK column on either side — it needs a **junction (bridge) table** in the middle, here `likes`, which holds two FKs and uses them together as a composite PK.

We'll refer back to this schema constantly below.

### Sample Data (used to compute every join result table below)

**`users`**

| user_id | username | email             | bio                   |
| ------- | -------- | ----------------- | --------------------- |
| 1       | elonmusk | elon@example.com  | Building rockets 🚀   |
| 2       | priya_d  | priya@example.com | Photographer 📸       |
| 3       | rahul99  | rahul@example.com | NULL _(no posts yet)_ |

**`posts`**

| post_id | user_id (FK) | caption              |
| ------- | ------------ | -------------------- |
| 101     | 1            | Beautiful sunset 🌅  |
| 102     | 1            | Rocket launch day 🚀 |
| 103     | 2            | Golden hour shoot    |

**`likes`** _(composite PK = user_id + post_id)_

| user_id (FK) | post_id (FK) |
| ------------ | ------------ |
| 2            | 101          |
| 3            | 101          |
| 1            | 103          |
| 2            | 103          |

> Notice: `rahul99` (user 3) has **no posts**, but **did** like post 101. Post 102 has **no likes** at all. These gaps are intentional — they're what make `LEFT JOIN` behavior visible below instead of every example looking identical to an `INNER JOIN`.

---

## 1. Creating Tables — `CREATE TABLE`

`CREATE TABLE` defines a new table: its columns, data types, and constraints (rules that protect data integrity).

```sql
CREATE TABLE posts (
    post_id     SERIAL PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(user_id),
    caption     TEXT,
    image_url   VARCHAR(255),
    created_at  TIMESTAMP DEFAULT NOW()
);
```

**Key pieces:**

- `SERIAL` — PostgreSQL auto-incrementing integer (1, 2, 3, …). Great for surrogate PKs.
- `PRIMARY KEY` — enforces uniqueness + NOT NULL automatically.
- `REFERENCES users(user_id)` — declares a **Foreign Key constraint**: Postgres will reject any `post_id`'s `user_id` that doesn't already exist in `users`. This is what keeps your data consistent — you can't have a post "belonging" to a user who doesn't exist.
- `NOT NULL` — column must always have a value.
- `UNIQUE` — no two rows can share the same value in that column.
- `DEFAULT NOW()` — auto-fills the current timestamp if you don't supply one.

---

## 2. Inserting Data — `INSERT INTO`

Adds new rows to a table.

```sql
INSERT INTO users (username, email, full_name)
VALUES ('elonmusk', 'elon@example.com', 'Elon Musk');

-- Insert a post from that user (user_id = 1, assuming first row)
INSERT INTO posts (user_id, caption, image_url)
VALUES (1, 'Beautiful sunset 🌅', 'https://cdn.example.com/img1.jpg');

-- Insert multiple rows at once
INSERT INTO users (username, email, full_name) VALUES
    ('priya_d', 'priya@example.com', 'Priya Desai'),
    ('rahul99', 'rahul@example.com', 'Rahul Sharma');
```

Because `posts.user_id` is a Foreign Key, trying to insert a post with `user_id = 999` (a user that doesn't exist) will throw an error — Postgres protects the relationship for you.

---

## 3. Updating Data — `UPDATE`

Modifies existing rows. **Always use `WHERE`**, or you'll update every row in the table.

```sql
-- A user updates their bio
UPDATE users
SET bio = 'Building rockets 🚀 and memes'
WHERE user_id = 1;

-- Update multiple columns at once
UPDATE users
SET full_name = 'Elon R. Musk', bio = 'CEO'
WHERE username = 'elonmusk';
```

⚠️ Running `UPDATE users SET bio = 'hi';` with **no WHERE clause** sets every single user's bio to `'hi'`. This is a very common and costly mistake.

---

## 4. Filtering Data — `WHERE`

`WHERE` filters which rows are returned/affected, using conditions.

```sql
-- All posts by a specific user
SELECT * FROM posts WHERE user_id = 1;

-- Posts after a certain date
SELECT * FROM posts WHERE created_at > '2026-01-01';

-- Combine conditions
SELECT * FROM posts
WHERE user_id = 1 AND created_at > '2026-01-01';

-- Match against a list
SELECT * FROM users WHERE user_id IN (1, 2, 3);

-- Range
SELECT * FROM posts WHERE post_id BETWEEN 10 AND 20;

-- NULL checks (never use = for NULL)
SELECT * FROM users WHERE bio IS NULL;
```

Operators: `=`, `!=` / `<>`, `<`, `>`, `<=`, `>=`, `AND`, `OR`, `NOT`, `IN`, `BETWEEN`, `IS NULL` / `IS NOT NULL`.

---

## 5. Deleting Data — `DELETE`

Removes rows. Like `UPDATE`, **always pair with `WHERE`**.

```sql
-- Delete one specific like (a user "unlikes" a post)
DELETE FROM likes
WHERE user_id = 2 AND post_id = 5;

-- Delete all posts by a deleted user
DELETE FROM posts WHERE user_id = 1;
```

`DELETE FROM posts;` with no `WHERE` wipes the entire table. To delete everything _fast_ (and reset auto-increment), Postgres also offers `TRUNCATE TABLE posts;`.

---

## 6. Using `ILIKE` — Case-Insensitive Pattern Matching

`LIKE` matches text patterns but is case-sensitive. `ILIKE` is PostgreSQL's **case-insensitive** version — very handy for search bars (e.g., searching usernames).

```sql
-- Find users whose username contains "priya", regardless of case
SELECT * FROM users WHERE username ILIKE '%priya%';

-- Starts with "el"
SELECT * FROM users WHERE username ILIKE 'el%';

-- Ends with "99"
SELECT * FROM users WHERE username ILIKE '%99';
```

Wildcards: `%` = any number of characters, `_` = exactly one character.

---

## 7. `GROUP BY` Clause

`GROUP BY` collapses multiple rows into one row **per group**, usually paired with aggregate functions like `COUNT()`, `SUM()`, `AVG()`.

```sql
-- How many posts has each user made?
SELECT user_id, COUNT(*) AS total_posts
FROM posts
GROUP BY user_id;

-- How many likes does each post have? (the classic Instagram "like count")
SELECT post_id, COUNT(*) AS like_count
FROM likes
GROUP BY post_id;

-- Filter groups using HAVING (WHERE filters rows BEFORE grouping; HAVING filters AFTER)
SELECT post_id, COUNT(*) AS like_count
FROM likes
GROUP BY post_id
HAVING COUNT(*) > 100;   -- only posts with more than 100 likes
```

**Rule of thumb:** every column in `SELECT` that isn't inside an aggregate function must appear in `GROUP BY`.

---

## 8. `ORDER BY` Clause

Sorts the result set.

```sql
-- Newest posts first
SELECT * FROM posts ORDER BY created_at DESC;

-- Oldest first (ASC is the default, so it's optional)
SELECT * FROM posts ORDER BY created_at ASC;

-- Sort by multiple columns: most-liked users, then alphabetically
SELECT user_id, COUNT(*) AS total_posts
FROM posts
GROUP BY user_id
ORDER BY total_posts DESC, user_id ASC;
```

---

## 9. `LIMIT` Results

Caps how many rows come back — essential for pagination (e.g., loading an Instagram feed in chunks).

```sql
-- Top 10 most recent posts (a typical "feed" query)
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10;

-- Pagination: skip the first 10, then take the next 10 (page 2)
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10 OFFSET 10;
```

---

## 10. Using `COALESCE`

`COALESCE(a, b, c, ...)` returns the **first non-NULL value** in the list. Great for supplying fallback/default values.

```sql
-- Show "No bio yet" instead of NULL when a user hasn't written one
SELECT username, COALESCE(bio, 'No bio yet') AS bio
FROM users;

-- Default a missing caption to an empty string
SELECT post_id, COALESCE(caption, '') AS caption
FROM posts;

-- Useful in math too: treat NULL like counts as 0
SELECT post_id, COALESCE(SUM(like_count), 0) AS total_likes
FROM post_stats;
```

---

## 11. Using `CASE` Statements

`CASE` is SQL's if/else-style logic — lets you create computed labels/categories inline.

```sql
-- Classify posts by how many likes they have
SELECT
    post_id,
    COUNT(likes.user_id) AS like_count,
    CASE
        WHEN COUNT(likes.user_id) = 0 THEN 'No likes'
        WHEN COUNT(likes.user_id) < 10 THEN 'Low engagement'
        WHEN COUNT(likes.user_id) < 100 THEN 'Decent engagement'
        ELSE 'Viral 🔥'
    END AS engagement_level
FROM posts
LEFT JOIN likes ON posts.post_id = likes.post_id
GROUP BY posts.post_id;
```

`CASE` always needs `WHEN ... THEN ...` pairs and should end with `END`; `ELSE` is the fallback if no condition matches.

---

## 12. Subqueries

A **subquery** is a query nested inside another query — used when you need the result of one query to feed into another.

```sql
-- Find users who have never posted (subquery in WHERE)
SELECT username
FROM users
WHERE user_id NOT IN (
    SELECT DISTINCT user_id FROM posts
);

-- Find the post with the most likes (subquery to compute a max)
SELECT post_id, caption
FROM posts
WHERE post_id = (
    SELECT post_id
    FROM likes
    GROUP BY post_id
    ORDER BY COUNT(*) DESC
    LIMIT 1
);

-- Subquery in the FROM clause (treated like a temporary table)
SELECT user_id, total_likes
FROM (
    SELECT user_id, COUNT(*) AS total_likes
    FROM likes
    GROUP BY user_id
) AS user_like_counts
WHERE total_likes > 50;
```

---

## 13. Common Table Expressions (CTEs) — `WITH`

A CTE is a **named, temporary result set** defined with `WITH`, used right before the main query. It's like giving a subquery a readable name — makes complex queries far easier to follow than nesting subqueries nested in subqueries.

```sql
WITH post_like_counts AS (
    SELECT post_id, COUNT(*) AS like_count
    FROM likes
    GROUP BY post_id
)
SELECT posts.post_id, posts.caption, post_like_counts.like_count
FROM posts
JOIN post_like_counts ON posts.post_id = post_like_counts.post_id
ORDER BY post_like_counts.like_count DESC
LIMIT 5;   -- top 5 most-liked posts
```

You can chain multiple CTEs:

```sql
WITH user_post_counts AS (
    SELECT user_id, COUNT(*) AS post_count
    FROM posts
    GROUP BY user_id
),
active_users AS (
    SELECT user_id FROM user_post_counts WHERE post_count > 5
)
SELECT users.username
FROM users
JOIN active_users ON users.user_id = active_users.user_id;
```

**CTE vs Subquery:** functionally similar, but CTEs are more readable, can be reused multiple times in the same query, and (in Postgres) can even be recursive (e.g., for traversing a comment-reply tree).

---

## 14. `ROLLBACK` and SQL Transactions

A **transaction** groups multiple statements so they either **all succeed together** or **all fail together** — protecting data integrity for multi-step operations.

```sql
BEGIN;  -- start a transaction

UPDATE users SET bio = 'New bio' WHERE user_id = 1;
DELETE FROM posts WHERE user_id = 1 AND post_id = 99;

-- if everything looks right:
COMMIT;   -- permanently saves both changes

-- if something looks wrong instead:
ROLLBACK; -- undoes everything since BEGIN, as if nothing happened
```

**Real-world example:** imagine a "delete account" feature. Deleting a user should also delete their posts and likes. If the post-deletion succeeds but the like-deletion fails halfway through (e.g., server crash), you'd be left with orphaned, broken data. Wrapping all the deletes in one transaction guarantees it's all-or-nothing:

```sql
BEGIN;

DELETE FROM likes WHERE user_id = 5;
DELETE FROM likes WHERE post_id IN (SELECT post_id FROM posts WHERE user_id = 5);
DELETE FROM posts WHERE user_id = 5;
DELETE FROM users WHERE user_id = 5;

COMMIT;
```

If any statement above errors out, you `ROLLBACK` instead of `COMMIT`, and the database is left exactly as it was before `BEGIN` — no partial damage.

`SAVEPOINT` lets you roll back to a specific point within a larger transaction instead of undoing everything:

```sql
BEGIN;
UPDATE users SET bio = 'Test' WHERE user_id = 1;
SAVEPOINT before_risky_change;
DELETE FROM posts WHERE user_id = 1;
ROLLBACK TO before_risky_change;  -- undoes only the DELETE, keeps the bio UPDATE
COMMIT;
```

---

## 15. Types of SQL Joins — INNER vs LEFT vs RIGHT vs FULL vs CROSS vs SELF

A `JOIN` combines rows from two (or more) tables using a related column. **The keyword you choose (`INNER`, `LEFT`, `RIGHT`, `FULL`, `CROSS`) only controls one thing: what happens to rows that _don't_ find a match on the other side.** Everything else about a join works the same way. This section explains each type on its own, then compares them head-to-head on the exact same data so the differences are visible, not just described.

### Visual overview

```
INNER JOIN              LEFT JOIN               RIGHT JOIN              FULL JOIN
 (only overlap)        (all of left)            (all of right)         (everything)

 ┌─────┐ ┌─────┐        ┌─────┐ ┌─────┐          ┌─────┐ ┌─────┐         ┌─────┐ ┌─────┐
 │     │█│     │        │█████│█│     │          │     │█│█████│         │█████│█│█████│
 │     │█│     │        │█████│█│     │          │     │█│█████│         │█████│█│█████│
 └─────┘ └─────┘        └─────┘ └─────┘          └─────┘ └─────┘         └─────┘ └─────┘
  Left   Right            Left   Right             Left   Right            Left   Right
```

`CROSS JOIN` doesn't fit the Venn-diagram picture at all — it ignores matching entirely and pairs **every row with every row** (a multiplication table, not an overlap). `SELF JOIN` isn't a different keyword — it's any of the above joins, applied to a table **and itself**.

### 1. `INNER JOIN` (a.k.a. just `JOIN`)

Returns only the rows where the join condition matches **on both sides**. If a row on either side has no match, it's dropped entirely. This is the default/most common join.

```sql
SELECT posts.caption, users.username
FROM posts
INNER JOIN users ON posts.user_id = users.user_id;
```

_(Writing `JOIN` with no keyword in front means `INNER JOIN` — they're identical.)_

### 2. `LEFT JOIN` (a.k.a. `LEFT OUTER JOIN`)

Returns **every row from the left (first-named) table**, plus matching data from the right table where it exists. If there's no match on the right, those columns come back `NULL` — but the left row is never dropped.

```sql
SELECT users.username, posts.caption
FROM users
LEFT JOIN posts ON users.user_id = posts.user_id;
```

### 3. `RIGHT JOIN` (a.k.a. `RIGHT OUTER JOIN`)

The mirror image of `LEFT JOIN`: returns **every row from the right (second-named) table**, plus matching data from the left table where it exists. Unmatched left-side columns come back `NULL`.

```sql
SELECT users.username, posts.caption
FROM users
RIGHT JOIN posts ON users.user_id = posts.user_id;
```

_(`A RIGHT JOIN B` always gives the same result as `B LEFT JOIN A` — `RIGHT JOIN` exists mainly for convenience so you don't have to reorder your `FROM` clause.)_

### 4. `FULL JOIN` (a.k.a. `FULL OUTER JOIN`)

Returns **everything** — every row from both tables. Matched rows are combined as usual; unmatched rows from _either_ side still appear, padded with `NULL` on whichever side is missing. It's effectively `LEFT JOIN` + `RIGHT JOIN` combined, with duplicates removed.

```sql
SELECT users.username, posts.caption
FROM users
FULL JOIN posts ON users.user_id = posts.user_id;
```

### 5. `CROSS JOIN`

No `ON` condition at all — every row of the left table is paired with **every** row of the right table (a Cartesian product). If the left table has 3 rows and the right has 3 rows, you get 3 × 3 = 9 result rows.

```sql
SELECT users.username, posts.caption
FROM users
CROSS JOIN posts;
```

### 6. `SELF JOIN`

Not a separate keyword — it's any join type (usually `INNER` or `LEFT`) where a table is joined **to itself**, used when rows in a table relate to _other rows in that same table_ (e.g., a comment replying to another comment, an employee reporting to a manager who is also an employee). Because both "copies" of the table have identical column names, you **must** alias them.

```sql
SELECT a.comment_text, b.comment_text
FROM comments AS a
JOIN comments AS b ON a.parent_comment_id = b.comment_id;
```

---

### Side-by-side comparison, same data, just changing the keyword

Using our `users` and `posts` sample data — keeping the `SELECT`/`FROM`/`ON` identical and only swapping the join keyword:

```sql
SELECT users.username, posts.post_id, posts.caption
FROM users
<<<JOIN TYPE HERE>>> posts ON users.user_id = posts.user_id;
```

| Join type      | Result                                                                                                                                                                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **INNER JOIN** | `elonmusk/101`, `elonmusk/102`, `priya_d/103` → **3 rows**. `rahul99` is dropped (no posts).                                                                                                                                                                                    |
| **LEFT JOIN**  | same 3 rows **+** `rahul99/NULL/NULL` → **4 rows**. Every user is kept.                                                                                                                                                                                                         |
| **RIGHT JOIN** | identical to INNER JOIN here → **3 rows**. _(Why? Because `posts.user_id` has a `NOT NULL` FK constraint — every post is guaranteed to have a real user, so there's no "unmatched post" to reveal RIGHT JOIN's real behavior. See below for a case where it actually differs.)_ |
| **FULL JOIN**  | identical to LEFT JOIN here → **4 rows**, for the same reason — no orphan posts exist to add anything extra on the right.                                                                                                                                                       |

**Key takeaway:** in a clean FK-enforced one-to-many relationship like `users`→`posts`, `RIGHT JOIN` and `FULL JOIN` often look identical to `INNER`/`LEFT JOIN` — because the foreign key guarantees the "many" side always has a valid parent. To actually _see_ `RIGHT JOIN` and `FULL JOIN` behave differently, you need a relationship where **both** sides can have unmatched rows — which is exactly what a non-FK lookup, like a moderation list, gives us:

### Seeing RIGHT JOIN and FULL JOIN actually differ — a `banned_usernames` example

```sql
-- NOT linked by a foreign key — just a plain text watchlist a moderation team maintains
CREATE TABLE banned_usernames (
    banned_username VARCHAR(50) PRIMARY KEY,
    reason          TEXT
);

INSERT INTO banned_usernames (banned_username, reason) VALUES
    ('rahul99',      'reported for spam'),   -- this username DOES currently belong to a real user
    ('spamking123',  'bot account'),         -- this one never signed up / already deleted — no matching user
    ('baduser1',     'harassment');          -- same — no matching user
```

Because this table is just plain text (no FK to `users`), it can contain entries with **no matching user**, and `users` can have rows with **no matching banned entry** — real overlap on both sides.

```sql
SELECT users.username, banned_usernames.banned_username, banned_usernames.reason
FROM users
<<<JOIN TYPE HERE>>> banned_usernames ON users.username = banned_usernames.banned_username;
```

| Join type      | Result                                                                                  | Row count                                                                               |
| -------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **INNER JOIN** | only `rahul99` (matches both sides)                                                     | 1 row                                                                                   |
| **LEFT JOIN**  | `elonmusk/NULL`, `priya_d/NULL`, `rahul99/rahul99`                                      | 3 rows (all users kept)                                                                 |
| **RIGHT JOIN** | `rahul99/rahul99`, `NULL/spamking123`, `NULL/baduser1`                                  | 3 rows (all banned entries kept — **now** you can see unmatched right-side rows appear) |
| **FULL JOIN**  | `elonmusk/NULL`, `priya_d/NULL`, `rahul99/rahul99`, `NULL/spamking123`, `NULL/baduser1` | 5 rows (absolutely everything, matched or not, from both tables)                        |

This is the clearest possible demonstration: **LEFT** protects the left table's rows, **RIGHT** protects the right table's rows, **FULL** protects both, **INNER** protects neither.

### `CROSS JOIN` example

```sql
SELECT users.username, posts.caption
FROM users
CROSS JOIN posts;
```

With 3 users and 3 posts, this returns **9 rows** — every user paired with every post, ownership completely ignored:

| username | caption              |
| -------- | -------------------- |
| elonmusk | Beautiful sunset 🌅  |
| elonmusk | Rocket launch day 🚀 |
| elonmusk | Golden hour shoot    |
| priya_d  | Beautiful sunset 🌅  |
| priya_d  | Rocket launch day 🚀 |
| priya_d  | Golden hour shoot    |
| rahul99  | Beautiful sunset 🌅  |
| rahul99  | Rocket launch day 🚀 |
| rahul99  | Golden hour shoot    |

**When is this actually useful?** Generating every combination of two independent lists — e.g., every product paired with every size/color option, or every day-of-week paired with every time-slot to build an empty schedule grid. **The common gotcha:** if you write a comma-style join (`FROM users, posts`) and _forget_ the `WHERE` condition linking them, Postgres silently gives you a `CROSS JOIN` — a frequent source of accidental duplicate-row bugs.

### `SELF JOIN` example — threaded comment replies

Self joins shine whenever a table references _itself_. Instagram comment replies are a perfect case: a reply is just a comment whose `parent_comment_id` points to another row in the very same `comments` table.

```sql
CREATE TABLE comments (
    comment_id        SERIAL PRIMARY KEY,
    post_id            INTEGER NOT NULL REFERENCES posts(post_id),
    user_id            INTEGER NOT NULL REFERENCES users(user_id),
    parent_comment_id  INTEGER REFERENCES comments(comment_id),  -- FK pointing back to THIS table
    comment_text       TEXT NOT NULL
);
```

**Sample data:**

| comment_id | post_id | user_id | parent_comment_id | comment_text    |
| ---------- | ------- | ------- | ----------------- | --------------- |
| 1          | 101     | 2       | NULL              | "Love this!"    |
| 2          | 101     | 3       | 1                 | "Totally agree" |
| 3          | 101     | 1       | 1                 | "Thanks!"       |

```sql
SELECT
    replies.comment_text  AS reply,
    original.comment_text AS replying_to
FROM comments AS replies
JOIN comments AS original ON replies.parent_comment_id = original.comment_id;
```

**Result:**

| reply `(replies)` | replying_to `(original)` |
| ----------------- | ------------------------ |
| Totally agree     | Love this!               |
| Thanks!           | Love this!               |

`comment_id = 1` has `parent_comment_id = NULL`, meaning it's a top-level comment, not a reply — so it never appears in the `replies` role here, only as someone else's `original`. Self joins **always** need table aliases (`replies`, `original`) because both sides come from the same physical table, and Postgres needs a way to tell which "copy" each column reference belongs to.

### Cheat sheet — how they all differ

| Join type    | Keeps unmatched LEFT rows? | Keeps unmatched RIGHT rows? | Needs an `ON` condition?              | Typical real-world use                                                                                                |
| ------------ | -------------------------- | --------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `INNER JOIN` | ❌ No                      | ❌ No                       | ✅ Yes                                | "Give me only complete, matched pairs"                                                                                |
| `LEFT JOIN`  | ✅ Yes                     | ❌ No                       | ✅ Yes                                | "Give me all of A, with B's info if it exists" (e.g., all users, with their posts if any)                             |
| `RIGHT JOIN` | ❌ No                      | ✅ Yes                      | ✅ Yes                                | "Give me all of B, with A's info if it exists" (rarely used — usually rewritten as a `LEFT JOIN` with tables swapped) |
| `FULL JOIN`  | ✅ Yes                     | ✅ Yes                      | ✅ Yes                                | "Give me everything from both, matched or not" (e.g., reconciliation reports)                                         |
| `CROSS JOIN` | N/A                        | N/A                         | ❌ No (intentionally)                 | "Give me every possible combination"                                                                                  |
| `SELF JOIN`  | depends on join type used  | depends on join type used   | ✅ Yes, comparing the table to itself | "Relate rows within the same table" (replies, org charts, hierarchies)                                                |

---

## 16. One-to-Many Joins

**Relationship recap:** one `user` → many `posts`. Each post has exactly one owner (`posts.user_id` is the FK pointing to `users.user_id`, the PK).

```
   users (1)                    posts (many)
┌───────────────┐          ┌──────────────────────┐
│ user_id  PK ●──┼──────────┼──● user_id   FK      │
│ username      │          │   post_id   PK        │
│ bio           │          │   caption             │
└───────────────┘          └──────────────────────┘
        1                              ∞
   "one user"                   "many posts"
```

### Query A — `INNER JOIN`

```sql
SELECT posts.post_id, posts.caption, users.username
FROM posts
JOIN users ON posts.user_id = users.user_id;
```

**Columns used, by table:**

| Column in result                                   | Comes from table                            | Role                                  |
| -------------------------------------------------- | ------------------------------------------- | ------------------------------------- |
| `post_id`                                          | `posts`                                     | PK of `posts`                         |
| `caption`                                          | `posts`                                     | regular column                        |
| `username`                                         | `users`                                     | regular column (looked up via the FK) |
| _(join condition)_ `posts.user_id = users.user_id` | `posts.user_id` (FK) ↔ `users.user_id` (PK) | this is the link                      |

**Result (computed from the sample data above):**

| post_id `(posts)` | caption `(posts)`    | username `(users)` |
| ----------------- | -------------------- | ------------------ |
| 101               | Beautiful sunset 🌅  | elonmusk           |
| 102               | Rocket launch day 🚀 | elonmusk           |
| 103               | Golden hour shoot    | priya_d            |

Notice `rahul99` (user 3, no posts) **doesn't appear at all** — `INNER JOIN` only keeps rows where both sides match.

### Query B — `LEFT JOIN`

```sql
SELECT users.username, posts.post_id, posts.caption
FROM users
LEFT JOIN posts ON users.user_id = posts.user_id;
```

**Columns used, by table:**

| Column in result | Comes from table                                             |
| ---------------- | ------------------------------------------------------------ |
| `username`       | `users` (the "left"/base table — every row is kept)          |
| `post_id`        | `posts` (the "right" table — filled with `NULL` if no match) |
| `caption`        | `posts`                                                      |

**Result:**

| username `(users)` | post_id `(posts)` | caption `(posts)`    |
| ------------------ | ----------------- | -------------------- |
| elonmusk           | 101               | Beautiful sunset 🌅  |
| elonmusk           | 102               | Rocket launch day 🚀 |
| priya_d            | 103               | Golden hour shoot    |
| rahul99            | NULL              | NULL                 |

Now `rahul99` **does** show up — `LEFT JOIN` keeps every row from `users` (the left table) no matter what, padding with `NULL` where `posts` has nothing to offer.

### Query C — `LEFT JOIN` + `GROUP BY` (post count per user)

```sql
SELECT users.username, COUNT(posts.post_id) AS total_posts
FROM users
LEFT JOIN posts ON users.user_id = posts.user_id
GROUP BY users.username;
```

| username `(users)` | total_posts `(COUNT of posts.post_id)` |
| ------------------ | -------------------------------------- |
| elonmusk           | 2                                      |
| priya_d            | 1                                      |
| rahul99            | 0                                      |

**Why this is "One-to-Many":** in Query A's result, `username` can repeat across _multiple rows_ (`elonmusk` appears twice — once per post), while each `post_id` appears exactly once. The "many" side is `posts`, and that's also the table that physically holds the foreign key.

---

## 17. Many-to-Many Joins

**Relationship recap:** one `user` can like many `posts`, and one `post` can be liked by many `users`. This needs the **junction table** `likes` sitting in between, joined twice — once to `users`, once to `posts`.

```
   users (many)        likes (junction)        posts (many)
┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ user_id  PK ●──┼───┼─● user_id  FK     │   │                  │
│ username      │   │  post_id  FK    ●─┼───┼─● post_id   PK   │
│               │   │  (composite PK    │   │  caption         │
│               │   │   = user_id+      │   │                  │
│               │   │   post_id)        │   │                  │
└───────────────┘   └──────────────────┘   └──────────────────┘
        ∞                                            ∞
  "many users"                                "many posts"
                  ↑ likes bridges them ↑
```

### Query A — Who liked a specific post?

```sql
SELECT users.username
FROM likes
JOIN users ON likes.user_id = users.user_id
WHERE likes.post_id = 101;
```

**Columns used, by table:**

| Column in result                                   | Comes from table                            | Role                           |
| -------------------------------------------------- | ------------------------------------------- | ------------------------------ |
| `username`                                         | `users`                                     | regular column                 |
| _(join condition)_ `likes.user_id = users.user_id` | `likes.user_id` (FK) ↔ `users.user_id` (PK) | link #1                        |
| _(filter)_ `likes.post_id = 101`                   | `likes`                                     | picks which post we care about |

**Result (post 101 = "Beautiful sunset 🌅"):**

| username `(users)` |
| ------------------ |
| priya_d            |
| rahul99            |

### Query B — Full many-to-many traversal (every like, who + what)

```sql
SELECT users.username, posts.caption
FROM likes
JOIN users ON likes.user_id = users.user_id
JOIN posts ON likes.post_id = posts.post_id;
```

**Columns used, by table:**

| Column in result | Comes from table | Linked via                                  |
| ---------------- | ---------------- | ------------------------------------------- |
| `username`       | `users`          | `likes.user_id` (FK) ↔ `users.user_id` (PK) |
| `caption`        | `posts`          | `likes.post_id` (FK) ↔ `posts.post_id` (PK) |

**Result:**

| username `(users)` | caption `(posts)`   |
| ------------------ | ------------------- |
| priya_d            | Beautiful sunset 🌅 |
| rahul99            | Beautiful sunset 🌅 |
| elonmusk           | Golden hour shoot   |
| priya_d            | Golden hour shoot   |

This is the **many-to-many "fan-out"** in action: `priya_d` appears **twice** (she liked 2 different posts) and "Beautiful sunset 🌅" appears **twice** (liked by 2 different users). Neither side is unique in the output — that's the signature of a many-to-many join, as opposed to one-to-many where only one side repeats.

### Query C — Each user's total likes given out

```sql
SELECT users.username, COUNT(likes.post_id) AS posts_liked
FROM users
LEFT JOIN likes ON users.user_id = likes.user_id
GROUP BY users.username;
```

| username `(users)` | posts_liked `(COUNT of likes.post_id)` |
| ------------------ | -------------------------------------- |
| elonmusk           | 1                                      |
| priya_d            | 2                                      |
| rahul99            | 1                                      |

**Why `likes` needs a composite PK:** without `PRIMARY KEY (user_id, post_id)`, nothing would stop the same user from "liking" the same post 5 times, inserting 5 duplicate rows. The composite key enforces the real-world rule: _one user, one post, one like_ — while still letting that same user appear in many other rows (liking other posts) and that same post appear in many other rows (liked by other users).

**Mental model for Many-to-Many in general:** whenever you catch yourself thinking _"a thing can have many of the other thing, AND vice versa,"_ — that's your signal to reach for a junction table with two FKs (and usually a composite PK on those two FKs), exactly like `likes` between `users` and `posts`. The same pattern works for Instagram **followers** too: a `follows` table with `follower_id` and `followee_id`, both FKs to `users`, composite PK on the pair — because a user can follow many users, and be followed by many users.

---

## Quick Reference Summary

| Topic             | Keyword(s)                                             | One-line purpose                                                   |
| ----------------- | ------------------------------------------------------ | ------------------------------------------------------------------ |
| Creating Tables   | `CREATE TABLE`                                         | Define structure, columns, PK/FK constraints                       |
| Inserting Data    | `INSERT INTO`                                          | Add new rows                                                       |
| Updating Data     | `UPDATE ... SET`                                       | Modify existing rows                                               |
| Filtering Data    | `WHERE`                                                | Select specific rows by condition                                  |
| Deleting Data     | `DELETE FROM`                                          | Remove rows                                                        |
| ILIKE             | `ILIKE`                                                | Case-insensitive pattern search                                    |
| GROUP BY          | `GROUP BY`                                             | Aggregate rows into per-group summaries                            |
| ORDER BY          | `ORDER BY`                                             | Sort results                                                       |
| LIMIT             | `LIMIT` / `OFFSET`                                     | Cap row count, paginate                                            |
| COALESCE          | `COALESCE()`                                           | Fallback value when NULL                                           |
| CASE              | `CASE WHEN ... END`                                    | Inline if/else logic                                               |
| Subqueries        | nested `SELECT`                                        | Query using the result of another query                            |
| CTEs              | `WITH ... AS`                                          | Named, readable temporary result sets                              |
| Transactions      | `BEGIN/COMMIT/ROLLBACK`                                | All-or-nothing multi-statement safety                              |
| Join Types        | `INNER` / `LEFT` / `RIGHT` / `FULL` / `CROSS` / `SELF` | Controls which unmatched rows survive a join                       |
| One-to-Many Join  | `JOIN`                                                 | One row's children, via a single FK                                |
| Many-to-Many Join | `JOIN` (x2) via junction table                         | Both sides have many of each other, via composite-key bridge table |
