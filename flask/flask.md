# Flask + PostgreSQL REST API Notes

Covers: **Python REST API Development with Flask + SQL** and **Non-Blocking API Calls with Threading and Concurrency in Python**.

We'll keep building on the same Instagram-style schema from the SQL notes (`users`, `posts`, `likes`) — now we expose it over HTTP using Flask.

---

## 0. What We're Building

A small REST API with endpoints to manage users, posts, and likes:

| Method | Endpoint                      | What it does                           |
| ------ | ----------------------------- | -------------------------------------- |
| GET    | `/users`                      | List all users                         |
| GET    | `/users/<id>`                 | Get one user                           |
| POST   | `/users`                      | Create a user                          |
| GET    | `/posts`                      | List all posts (with owner's username) |
| POST   | `/posts`                      | Create a post                          |
| GET    | `/posts/<id>/likes`           | List who liked a post                  |
| POST   | `/posts/<id>/likes`           | Like a post                            |
| DELETE | `/posts/<id>/likes/<user_id>` | Unlike a post                          |

---

## 1. Project Setup

```bash
mkdir instagram-api && cd instagram-api
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install flask psycopg2-binary python-dotenv
```

- **flask** — the web framework
- **psycopg2-binary** — PostgreSQL driver for Python
- **python-dotenv** — loads DB credentials from a `.env` file instead of hardcoding them

**`.env`** (never commit this file):

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=instagram
DB_USER=postgres
DB_PASSWORD=your_password
```

**Project structure:**

```
instagram-api/
├── .env
├── app.py
└── db.py
```

---

## 2. Flask Basics — Your First Route

```python
# app.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({"message": "Instagram API is running"})

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

Run it with `python app.py`, then visit `http://localhost:5000/`.

- `@app.route("/")` — maps a URL path to a Python function (a "view").
- `jsonify(...)` — converts a Python dict into a proper JSON HTTP response (sets the `Content-Type` header for you).
- `debug=True` — auto-reloads on code changes and shows detailed error pages. **Never use this in production.**

---

## 3. Connecting Flask to PostgreSQL

We use `psycopg2` to talk to Postgres. Two important upgrades over the bare-bones version:

1. **`RealDictCursor`** — returns rows as dictionaries (`{'user_id': 1, 'username': 'elonmusk'}`) instead of plain tuples, so `jsonify()` can serialize them directly.
2. **A connection pool** — instead of opening a brand-new database connection on every request (slow, and Postgres has a connection limit), we keep a small pool of reusable connections open.

```python
# db.py
import os
import psycopg2
from psycopg2 import pool
from psycopg2.extras import RealDictCursor
from dotenv import load_dotenv

load_dotenv()

connection_pool = psycopg2.pool.SimpleConnectionPool(
    1, 10,   # min 1, max 10 connections
    host=os.getenv("DB_HOST"),
    port=os.getenv("DB_PORT"),
    dbname=os.getenv("DB_NAME"),
    user=os.getenv("DB_USER"),
    password=os.getenv("DB_PASSWORD"),
)

def get_connection():
    return connection_pool.getconn()

def release_connection(conn):
    connection_pool.putconn(conn)
```

**Why a pool, and not just one global connection?** A single shared connection breaks under concurrent requests — two requests querying it at once can corrupt each other's results. A single _new_ connection per request works but is slow (TCP handshake + auth every time) and can exhaust Postgres's `max_connections` under load. A pool gives you the best of both: connections are reused, but each request still gets exclusive use of one for its duration.

---

## 4. Building REST API Endpoints (CRUD)

### GET — list all users

```python
# app.py
from flask import Flask, jsonify, request
from psycopg2.extras import RealDictCursor
from db import get_connection, release_connection

app = Flask(__name__)

@app.route("/users", methods=["GET"])
def get_users():
    conn = get_connection()
    try:
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute("SELECT user_id, username, full_name, bio FROM users ORDER BY user_id;")
        users = cur.fetchall()
        cur.close()
        return jsonify(users), 200
    finally:
        release_connection(conn)
```

### GET — a single user by ID (path parameter)

```python
@app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    conn = get_connection()
    try:
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute("SELECT user_id, username, full_name, bio FROM users WHERE user_id = %s;", (user_id,))
        user = cur.fetchone()
        cur.close()
        if user is None:
            return jsonify({"error": "User not found"}), 404
        return jsonify(user), 200
    finally:
        release_connection(conn)
```

`<int:user_id>` tells Flask to extract that part of the URL, convert it to an `int`, and pass it as a function argument. `%s` with a tuple `(user_id,)` is **parameterized SQL** — psycopg2 escapes the value safely, preventing SQL injection. **Never** build queries with f-strings or `.format()` using user input.

### POST — create a new user

```python
@app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json()

    if not data or "username" not in data or "email" not in data:
        return jsonify({"error": "username and email are required"}), 400

    conn = get_connection()
    try:
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute(
            """
            INSERT INTO users (username, email, full_name, bio)
            VALUES (%s, %s, %s, %s)
            RETURNING user_id, username, email, full_name, bio;
            """,
            (data["username"], data["email"], data.get("full_name"), data.get("bio")),
        )
        new_user = cur.fetchone()
        conn.commit()
        cur.close()
        return jsonify(new_user), 201
    except Exception as e:
        conn.rollback()
        return jsonify({"error": str(e)}), 500
    finally:
        release_connection(conn)
```

- `request.get_json()` — parses the incoming JSON body of the POST request.
- `RETURNING ...` — a Postgres feature that hands back the row you just inserted (including the auto-generated `user_id`) in the same query, no second `SELECT` needed.
- `conn.commit()` — without this, the `INSERT` stays uncommitted and is invisible to other connections (see the Transactions section of the SQL notes).
- `conn.rollback()` — on any error, undo whatever partial work happened on this connection so the next request that reuses it from the pool starts clean.
- **201 Created** is the correct status code for a successful `POST` that creates a resource (not `200`).

### GET — posts, joined with their owner's username

```python
@app.route("/posts", methods=["GET"])
def get_posts():
    conn = get_connection()
    try:
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute(
            """
            SELECT posts.post_id, posts.caption, posts.created_at, users.username
            FROM posts
            JOIN users ON posts.user_id = users.user_id
            ORDER BY posts.created_at DESC;
            """
        )
        posts = cur.fetchall()
        cur.close()
        return jsonify(posts), 200
    finally:
        release_connection(conn)
```

This is the same `JOIN` from the SQL notes — the API layer doesn't change the SQL logic, it just wraps the result in JSON.

### POST — like a post

```python
@app.route("/posts/<int:post_id>/likes", methods=["POST"])
def like_post(post_id):
    data = request.get_json()
    if not data or "user_id" not in data:
        return jsonify({"error": "user_id is required"}), 400

    conn = get_connection()
    try:
        cur = conn.cursor(cursor_factory=RealDictCursor)
        cur.execute(
            "INSERT INTO likes (user_id, post_id) VALUES (%s, %s) ON CONFLICT DO NOTHING RETURNING user_id, post_id;",
            (data["user_id"], post_id),
        )
        result = cur.fetchone()
        conn.commit()
        cur.close()
        if result is None:
            return jsonify({"message": "Already liked"}), 200
        return jsonify(result), 201
    except Exception as e:
        conn.rollback()
        return jsonify({"error": str(e)}), 500
    finally:
        release_connection(conn)
```

`ON CONFLICT DO NOTHING` leans directly on the **composite primary key** `(user_id, post_id)` from the SQL notes — if that exact pair already exists, Postgres silently skips the insert instead of throwing a duplicate-key error, which is exactly the "like it once" behavior we want.

### DELETE — unlike a post

```python
@app.route("/posts/<int:post_id>/likes/<int:user_id>", methods=["DELETE"])
def unlike_post(post_id, user_id):
    conn = get_connection()
    try:
        cur = conn.cursor()
        cur.execute("DELETE FROM likes WHERE post_id = %s AND user_id = %s;", (post_id, user_id))
        deleted = cur.rowcount
        conn.commit()
        cur.close()
        if deleted == 0:
            return jsonify({"error": "Like not found"}), 404
        return "", 204
    except Exception as e:
        conn.rollback()
        return jsonify({"error": str(e)}), 500
    finally:
        release_connection(conn)
```

`cur.rowcount` tells you how many rows the last query affected — useful for knowing whether a `DELETE`/`UPDATE` actually matched anything. **204 No Content** is the standard response for a successful delete with nothing to return.

---

## 5. HTTP Status Codes Cheat Sheet

| Code | Meaning               | Use it when                             |
| ---- | --------------------- | --------------------------------------- |
| 200  | OK                    | Successful GET/PUT/general success      |
| 201  | Created               | Successful POST that created a resource |
| 204  | No Content            | Successful DELETE, nothing to return    |
| 400  | Bad Request           | Missing/invalid input from the client   |
| 404  | Not Found             | The requested resource doesn't exist    |
| 409  | Conflict              | A unique constraint would be violated   |
| 500  | Internal Server Error | Something broke on the server side      |

---

## 6. Non-Blocking API Calls with Threading and Concurrency

### The problem

By default, the Flask development server handles **one request at a time**. If one request takes 3 seconds (e.g., calling a slow external API, sending an email, processing an image), every other user is stuck waiting behind it.

```python
# app.run() by default is single-threaded — bad for slow operations
app.run(debug=True)
```

Fix #1 — let Flask's dev server handle multiple requests concurrently using threads:

```python
app.run(debug=True, threaded=True)
```

This alone helps, but it only solves _concurrent requests_. It doesn't solve the deeper problem: a single request that needs to do something slow (like notifying followers) shouldn't make the _user_ wait for that slow part if the result isn't needed in the response.

### Fix #2 — Python's `threading` module for fire-and-forget work

**Real-world example:** when someone likes a post, you might want to send a push notification to the post's owner. The HTTP client doesn't need to wait for that notification to actually be delivered — it just needs to know the like was saved.

```python
import threading

def send_notification(username, post_id):
    # imagine this calls a slow push-notification service
    import time
    time.sleep(3)
    print(f"Notified {username} about a new like on post {post_id}")

@app.route("/posts/<int:post_id>/likes", methods=["POST"])
def like_post(post_id):
    data = request.get_json()
    # ... (insert the like into the DB, same as before) ...

    # Fire the notification in a background thread — don't block the response
    owner_username = "elonmusk"  # (you'd look this up from the DB)
    threading.Thread(target=send_notification, args=(owner_username, post_id)).start()

    return jsonify({"message": "Liked"}), 201
```

The client gets their `201 Created` response immediately — the 3-second notification work happens in the background, in a separate thread, without holding up the request.

### Fix #3 — `ThreadPoolExecutor` for _controlled_ concurrency

Spawning a brand-new `threading.Thread` for every request works, but it's unbounded — under heavy traffic you could spin up thousands of threads and exhaust system resources. A **thread pool** caps how many run at once and queues the rest.

```python
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=5)  # at most 5 background tasks at once

@app.route("/posts/<int:post_id>/likes", methods=["POST"])
def like_post(post_id):
    # ... insert the like ...
    executor.submit(send_notification, "elonmusk", post_id)
    return jsonify({"message": "Liked"}), 201
```

`executor.submit(...)` queues the function to run on the next available worker thread instead of creating a brand-new thread every time — much safer under load.

### A critical gotcha: database connections aren't automatically thread-safe

If your background thread also needs to talk to Postgres, **don't reuse the same connection object that's handling the main request** — psycopg2 connections aren't safe to use from multiple threads simultaneously. Always pull a fresh connection from the pool inside the threaded function, and release it when done:

```python
def send_notification_and_log(user_id, post_id):
    conn = get_connection()
    try:
        cur = conn.cursor()
        cur.execute("INSERT INTO notification_log (user_id, post_id) VALUES (%s, %s);", (user_id, post_id))
        conn.commit()
        cur.close()
    finally:
        release_connection(conn)   # always return it to the pool
```

This is exactly why we built a **connection pool** back in Section 3 — each thread can safely check out its own connection.

### Fix #4 — `async`/`await` (Flask 2.0+, the modern alternative)

Flask views can also be declared `async def`, which suits I/O-bound work like calling external HTTP APIs (note: this is a different concurrency model than threading — it requires `await`-able libraries, e.g. `httpx` instead of the blocking `requests`):

```python
import httpx

@app.route("/posts/<int:post_id>/notify-external", methods=["POST"])
async def notify_external_service(post_id):
    async with httpx.AsyncClient() as client:
        response = await client.post("https://example.com/webhook", json={"post_id": post_id})
    return jsonify({"status": response.status_code}), 200
```

**`threading`/`ThreadPoolExecutor` vs `async`/`await` — when to use which:**

| Approach                               | Best for                                                       | Notes                                                                                     |
| -------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `threading.Thread`                     | Simple fire-and-forget background tasks                        | Easiest to add to existing sync code; uncapped if used carelessly                         |
| `ThreadPoolExecutor`                   | Background tasks under predictable load                        | Same as above, but caps concurrency                                                       |
| `async`/`await`                        | High volume of I/O-bound requests (calling many external APIs) | Needs async-compatible libraries throughout; more efficient at scale but a bigger rewrite |
| Background job queue (e.g. Celery, RQ) | Heavy or long-running tasks (video processing, bulk emails)    | Out of scope here, but the natural next step once `threading` isn't enough                |

---

## 7. Running in Production

The Flask dev server (`app.run()`) is **not** meant for production — it's single-process and not designed for real traffic. Use a proper WSGI server like **gunicorn**:

```bash
pip install gunicorn

# 4 worker processes, each capable of handling requests independently
gunicorn -w 4 -b 0.0.0.0:5000 app:app
```

- `-w 4` — runs 4 worker _processes_ (separate from the threading discussed above — this is process-level parallelism, useful because Python's Global Interpreter Lock limits true CPU parallelism within a single process).
- Always set `debug=False` (or just don't pass `debug=True`) in production — debug mode can leak stack traces and even allow remote code execution if exposed publicly.
- Keep using the connection pool from Section 3 — under multiple gunicorn workers, each worker process gets its **own** pool (pools aren't shared across processes), so size `max_workers` and your pool's max connections together so you don't exceed Postgres's `max_connections`.

---

## Quick Reference Summary

| Topic                  | Key tool/code                        | Purpose                                      |
| ---------------------- | ------------------------------------ | -------------------------------------------- |
| Flask basics           | `@app.route()`, `jsonify()`          | Define endpoints, return JSON                |
| DB connection          | `psycopg2.pool.SimpleConnectionPool` | Reusable, safe DB connections                |
| Dict-style rows        | `RealDictCursor`                     | Query results as JSON-friendly dicts         |
| Safe queries           | `%s` placeholders                    | Prevent SQL injection                        |
| Create + return row    | `RETURNING ...`                      | Get the inserted row back in one query       |
| Avoid duplicate likes  | `ON CONFLICT DO NOTHING`             | Leans on the composite PK from the SQL notes |
| Background work        | `threading.Thread`                   | Fire-and-forget, don't block the response    |
| Controlled concurrency | `ThreadPoolExecutor`                 | Cap background thread count                  |
| Modern async I/O       | `async def` + `await`                | High-volume I/O-bound endpoints              |
| Production server      | `gunicorn -w 4`                      | Multi-process, production-grade serving      |
