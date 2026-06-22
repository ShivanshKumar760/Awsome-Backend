# Building an Instagram-Style API in Flask — From Zero to Long-Polling

This builds directly on your SQL notes (`users`, `posts`, `likes`). We'll go from "what is a route" all the way to a working API with JWT auth and real-time-ish updates via long polling — **twice**: once with raw SQL (`psycopg2`), once with an ORM (SQLAlchemy). Same features, same endpoints, two implementations, so you can compare line-by-line.

---

## Table of Contents

1. Project setup & how Flask routing works (`@api.route`)
2. Polling 101: Short Polling vs Long Polling vs WebSocket
3. The "global lock" concept and why we need it for long polling
4. JWT authentication, explained from scratch
5. Database schema (same as your SQL notes)
6. **Version 1: Raw SQL API** (full code, explained)
7. **Version 2: SQLAlchemy ORM API** (full code, explained)
8. The Long-Polling endpoint itself, explained line by line
9. Testing the API
10. Where to go from here

---

## 1. Project Setup & How Routing Works

### What even is a route?

A Flask app is, at its core, a big dictionary: **"if a request comes in for this URL + this HTTP method, run this function."** `@api.route(...)` is the decorator that registers that mapping.

```python
from flask import Flask

api = Flask(__name__)

@api.route("/ping", methods=["GET"])
def ping():
    return {"message": "pong"}, 200
```

Breaking this down:
- `api = Flask(__name__)` creates the application object. `__name__` tells Flask where the app's root folder is (for finding templates/static files — not important here).
- `@api.route("/ping", methods=["GET"])` registers the function `ping` to handle `GET /ping`.
- The function returns a dict — Flask automatically converts it to JSON, plus an explicit status code `200`.
- Without `methods=[...]`, Flask defaults to `GET` only.

### Dynamic route segments

```python
@api.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    ...
```

`<int:user_id>` means "match a number here and pass it into the function as the integer `user_id`." You can also use `<string:username>` or just `<username>` (string is the default converter).

### Reading the request body

```python
from flask import request

@api.route("/posts", methods=["POST"])
def create_post():
    data = request.get_json()   # parses incoming JSON body into a Python dict
    caption = data.get("caption")
    ...
```

### Project layout we'll use

```
instagram_api/
├── app_raw_sql.py        # Version 1 entrypoint
├── app_orm.py            # Version 2 entrypoint
├── models.py             # SQLAlchemy models (Version 2 only)
├── auth.py               # JWT helper functions (shared logic, shown once)
├── requirements.txt
└── schema.sql            # CREATE TABLE statements (Version 1 only, ORM auto-creates these)
```

```
# requirements.txt
flask
psycopg2-binary
PyJWT
SQLAlchemy
flask-sqlalchemy
python-dotenv
```

---

## 2. Short Polling vs Long Polling vs WebSocket

You explicitly asked to skip WebSocket and use long polling — but understanding all three side by side makes clear *why*.

### Short Polling

The client asks "anything new?" over and over, on a fixed timer, **regardless of whether anything happened**.

```
Client: "New likes on post 101?"  → Server: "No"      (t=0s)
Client: "New likes on post 101?"  → Server: "No"      (t=2s)
Client: "New likes on post 101?"  → Server: "Yes! +3"  (t=4s)
Client: "New likes on post 101?"  → Server: "No"      (t=6s)
```

- **Pros:** dead simple — a plain HTTP request on a `setInterval`.
- **Cons:** wasteful. Most requests return "nothing changed," but you still pay the cost of opening a connection, hitting your server/DB, and closing it, every single time. At scale this is a lot of pointless load. There's also a delay (latency) up to your poll interval before the client learns of new data.

### Long Polling

The client asks "anything new?" but the **server holds the request open** ("hangs") and only responds **once something actually changes** (or a timeout is hit). The moment it responds, the client immediately re-asks.

```
Client: "New likes on post 101?" ──(server holds connection open)──...
                                   ...3 seconds pass, no likes yet...
                                   ...a like arrives!...
                  Server responds: "Yes! +1"          (t=3s)
Client immediately re-asks: "New likes on post 101?" ──(holds again)──...
```

- **Pros:** much closer to real-time than short polling, no wasted "no" responses, and crucially — **it's just plain HTTP**. No special protocol, no persistent socket, works through any proxy/firewall that already allows HTTP, easy to implement in Flask with zero extra libraries.
- **Cons:** still one full HTTP request per "event," and the server has to keep that request's worker/thread occupied while waiting (this is exactly where the **global lock** discussion below comes in). Doesn't scale as well as WebSockets for *very* high-frequency, many-thousands-of-clients use cases.

### WebSocket

A **single persistent, two-way connection** is opened once and stays open. Either side can push messages over it at any time, with no "asking" needed.

```
Client ⇄ Server   (one connection, opened once)
Server → Client: "New like!"   (pushed instantly, whenever it happens)
Server → Client: "New post!"   (pushed instantly, whenever it happens)
```

- **Pros:** truest "real-time," lowest latency, most efficient at scale for high-frequency push.
- **Cons:** needs a different protocol (`ws://`), a persistent connection per client (different infrastructure/load-balancer considerations), and a different programming model (event-driven, not request/response). More moving parts to learn and deploy.

### Why long polling is the right choice for *this* project

It gets you 90% of the "feels real-time" experience of WebSockets, using nothing but Flask routes you already know how to write. It's the natural stepping stone between "the client polls dumbly every 2 seconds" and "I need a whole separate WebSocket server."

---

## 3. The Global Lock — Why We Need It

### The problem long polling creates

A long-polling endpoint works by **blocking** (waiting) inside the request handler until new data shows up. If our "new data" detector is a plain Python variable like:

```python
latest_like_event = None
```

...then **multiple requests, possibly from multiple threads, are all reading and writing this same variable at the same time.** Flask's dev server (and most production WSGI servers like `gunicorn` with threaded workers) will happily run your route function on multiple threads concurrently. Without protection, two things can happen:

1. **Race conditions:** Thread A reads `latest_like_event`, thread B updates it, then thread A overwrites it back to a stale value — a client misses an update.
2. **Inconsistent reads:** Thread A is in the middle of updating a dict (`{"post_id": ..., "likes": ...}`) when thread B reads it half-written.

### The fix: a `Lock`

Python's `threading.Lock` guarantees that **only one thread at a time** can execute the code inside `with lock:` — every other thread that tries gets paused until the lock is free.

```python
import threading

lock = threading.Lock()
latest_event = {"type": None, "data": None, "version": 0}

def publish_event(event_type, data):
    global latest_event
    with lock:                      # only one thread can be inside here at once
        latest_event["type"] = event_type
        latest_event["data"] = data
        latest_event["version"] += 1   # bump a counter so clients can detect "is this new?"
```

We call this our **"global lock"** because it's a single lock shared across the whole app, protecting the single shared piece of state (`latest_event`) that *every* long-polling client is watching.

### How the long-polling loop uses it

```python
def wait_for_new_event(since_version, timeout=25):
    start = time.time()
    while time.time() - start < timeout:
        with lock:
            if latest_event["version"] > since_version:
                return latest_event.copy()   # something changed — return immediately
        time.sleep(0.5)   # nothing new yet, sleep briefly before checking again
    return None   # timed out, nothing happened
```

This is "long polling" implemented as a **short internal poll, hidden behind one long-held HTTP request** — the client only sees one slow request, but under the hood the server is checking the shared state every half-second until something changes or 25 seconds pass, whichever comes first. We use the lock every single time we touch `latest_event`, whether reading or writing, so no two threads ever see a half-updated value.

---

## 4. JWT Authentication, From Scratch

### What problem JWT solves

HTTP is stateless — the server doesn't remember you between requests. After a user logs in, we need a way for every *future* request to prove "this is still user #1, already verified." A **JWT (JSON Web Token)** is a signed, tamper-proof string the server hands the client after login; the client sends it back on every subsequent request (usually via the `Authorization` header), and the server can verify it instantly **without hitting the database** — because the token's signature is checked with a secret key only the server knows.

### Anatomy of a JWT

A JWT looks like `xxxxx.yyyyy.zzzzz` — three base64-encoded parts separated by dots:

| Part | Contains | Example |
|---|---|---|
| **Header** | algorithm used | `{"alg": "HS256", "typ": "JWT"}` |
| **Payload** | your actual data ("claims") | `{"user_id": 1, "exp": 1750000000}` |
| **Signature** | `HMAC-SHA256(header + payload, SECRET_KEY)` | unreadable hash |

Anyone can *read* the payload (it's just base64, not encrypted) — but only someone with `SECRET_KEY` can produce a *valid* signature for it. So if a client tampers with the payload (e.g. changes `user_id` from 1 to 2), the signature no longer matches, and the server's verification step rejects the token.

### Issuing a token (login)

```python
import jwt
import datetime

SECRET_KEY = "change-this-to-a-real-secret-in-production"

def create_token(user_id):
    payload = {
        "user_id": user_id,
        "exp": datetime.datetime.utcnow() + datetime.timedelta(days=7),  # expiry
        "iat": datetime.datetime.utcnow(),                                # issued-at
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")
```

### Verifying a token on every protected request

```python
from functools import wraps
from flask import request, jsonify

def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        if not auth_header.startswith("Bearer "):
            return jsonify({"error": "Missing or malformed Authorization header"}), 401

        token = auth_header.split(" ")[1]
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401

        request.user_id = payload["user_id"]   # stash it for the route to use
        return f(*args, **kwargs)
    return decorated
```

This is a **decorator** — wrap any route with `@token_required` and it automatically blocks unauthenticated requests before your route's own code ever runs:

```python
@api.route("/posts", methods=["POST"])
@token_required
def create_post():
    user_id = request.user_id   # set by token_required above
    ...
```

### The client's job

1. `POST /login` with username/password → gets back `{"token": "xxxxx.yyyyy.zzzzz"}`.
2. Stores that token (e.g. in memory, or `localStorage` on web — outside our scope here).
3. Sends it on every future request: header `Authorization: Bearer xxxxx.yyyyy.zzzzz`.

---

## 5. Database Schema

Identical to your SQL notes — we're not changing the data model at all:

```sql
CREATE TABLE users (
    user_id      SERIAL PRIMARY KEY,
    username     VARCHAR(50) UNIQUE NOT NULL,
    email        VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    bio          TEXT,
    created_at   TIMESTAMP DEFAULT NOW()
);

CREATE TABLE posts (
    post_id      SERIAL PRIMARY KEY,
    user_id      INTEGER NOT NULL REFERENCES users(user_id),
    caption      TEXT,
    image_url    VARCHAR(255) NOT NULL,
    created_at   TIMESTAMP DEFAULT NOW()
);

CREATE TABLE likes (
    user_id      INTEGER NOT NULL REFERENCES users(user_id),
    post_id      INTEGER NOT NULL REFERENCES posts(post_id),
    liked_at     TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);
```

(We added `password_hash` to `users` since we now need to authenticate people — never store plain-text passwords.)

---

## 6. Version 1 — Raw SQL API (`psycopg2`)

In this version, *you* write every SQL statement, and `psycopg2` just sends it to Postgres and gives you back rows as tuples.

```python
# app_raw_sql.py
import threading
import time
import jwt
import datetime
import psycopg2
import psycopg2.extras
from functools import wraps
from flask import Flask, request, jsonify
from werkzeug.security import generate_password_hash, check_password_hash

api = Flask(__name__)
SECRET_KEY = "change-this-to-a-real-secret-in-production"

DB_CONFIG = dict(
    dbname="instagram_clone",
    user="postgres",
    password="yourpassword",
    host="localhost",
    port=5432,
)

def get_db():
    # a fresh connection per request — simplest possible approach for learning.
    # In production you'd use a connection pool (e.g. psycopg2.pool, or pgbouncer).
    conn = psycopg2.connect(**DB_CONFIG)
    conn.autocommit = True
    return conn


# ---------- JWT helpers ----------

def create_token(user_id):
    payload = {
        "user_id": user_id,
        "exp": datetime.datetime.utcnow() + datetime.timedelta(days=7),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")


def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        if not auth_header.startswith("Bearer "):
            return jsonify({"error": "Missing or malformed Authorization header"}), 401
        token = auth_header.split(" ")[1]
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
        request.user_id = payload["user_id"]
        return f(*args, **kwargs)
    return decorated


# ---------- Shared state for long polling ----------

lock = threading.Lock()
latest_event = {"version": 0, "type": None, "data": None}


def publish_event(event_type, data):
    with lock:
        latest_event["version"] += 1
        latest_event["type"] = event_type
        latest_event["data"] = data


def wait_for_new_event(since_version, timeout=25):
    start = time.time()
    while time.time() - start < timeout:
        with lock:
            if latest_event["version"] > since_version:
                return dict(latest_event)
        time.sleep(0.5)
    return None


# ---------- Auth routes ----------

@api.route("/signup", methods=["POST"])
def signup():
    data = request.get_json()
    username = data.get("username")
    email = data.get("email")
    password = data.get("password")
    if not all([username, email, password]):
        return jsonify({"error": "username, email, password are required"}), 400

    password_hash = generate_password_hash(password)

    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    try:
        cur.execute(
            """
            INSERT INTO users (username, email, password_hash)
            VALUES (%s, %s, %s)
            RETURNING user_id, username, email
            """,
            (username, email, password_hash),
        )
        user = cur.fetchone()
    except psycopg2.errors.UniqueViolation:
        return jsonify({"error": "username or email already taken"}), 409
    finally:
        cur.close()
        conn.close()

    token = create_token(user["user_id"])
    return jsonify({"user": user, "token": token}), 201


@api.route("/login", methods=["POST"])
def login():
    data = request.get_json()
    username = data.get("username")
    password = data.get("password")

    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute("SELECT * FROM users WHERE username = %s", (username,))
    user = cur.fetchone()
    cur.close()
    conn.close()

    if not user or not check_password_hash(user["password_hash"], password):
        return jsonify({"error": "Invalid username or password"}), 401

    token = create_token(user["user_id"])
    return jsonify({"token": token}), 200


# ---------- Profile routes ----------

@api.route("/users/<int:user_id>", methods=["GET"])
def get_profile(user_id):
    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute(
        "SELECT user_id, username, email, bio, created_at FROM users WHERE user_id = %s",
        (user_id,),
    )
    user = cur.fetchone()
    cur.close()
    conn.close()
    if not user:
        return jsonify({"error": "User not found"}), 404
    return jsonify(user), 200


@api.route("/users/me", methods=["PUT"])
@token_required
def update_profile():
    data = request.get_json()
    bio = data.get("bio")

    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute(
        """
        UPDATE users SET bio = %s WHERE user_id = %s
        RETURNING user_id, username, bio
        """,
        (bio, request.user_id),
    )
    user = cur.fetchone()
    cur.close()
    conn.close()
    return jsonify(user), 200


# ---------- Post routes ----------

@api.route("/posts", methods=["POST"])
@token_required
def create_post():
    data = request.get_json()
    caption = data.get("caption")
    image_url = data.get("image_url")
    if not image_url:
        return jsonify({"error": "image_url is required"}), 400

    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute(
        """
        INSERT INTO posts (user_id, caption, image_url)
        VALUES (%s, %s, %s)
        RETURNING post_id, user_id, caption, image_url, created_at
        """,
        (request.user_id, caption, image_url),
    )
    post = cur.fetchone()
    cur.close()
    conn.close()

    publish_event("new_post", post_to_jsonable(post))
    return jsonify(post), 201


def post_to_jsonable(post):
    # psycopg2 returns datetime objects; JSON needs them as strings
    post = dict(post)
    post["created_at"] = str(post["created_at"])
    return post


@api.route("/feed", methods=["GET"])
def get_feed():
    limit = int(request.args.get("limit", 10))
    offset = int(request.args.get("offset", 0))

    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute(
        """
        SELECT posts.post_id, posts.caption, posts.image_url, posts.created_at,
               users.username,
               COUNT(likes.user_id) AS like_count
        FROM posts
        JOIN users ON posts.user_id = users.user_id
        LEFT JOIN likes ON posts.post_id = likes.post_id
        GROUP BY posts.post_id, posts.caption, posts.image_url, posts.created_at, users.username
        ORDER BY posts.created_at DESC
        LIMIT %s OFFSET %s
        """,
        (limit, offset),
    )
    posts = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify([post_to_jsonable(p) for p in posts]), 200


# ---------- Like routes ----------

@api.route("/posts/<int:post_id>/like", methods=["POST"])
@token_required
def like_post(post_id):
    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    try:
        cur.execute(
            "INSERT INTO likes (user_id, post_id) VALUES (%s, %s)",
            (request.user_id, post_id),
        )
    except psycopg2.errors.UniqueViolation:
        cur.close()
        conn.close()
        return jsonify({"error": "Already liked"}), 409

    cur.execute("SELECT COUNT(*) FROM likes WHERE post_id = %s", (post_id,))
    like_count = cur.fetchone()["count"]
    cur.close()
    conn.close()

    publish_event("new_like", {"post_id": post_id, "like_count": like_count})
    return jsonify({"post_id": post_id, "like_count": like_count}), 201


@api.route("/posts/<int:post_id>/like", methods=["DELETE"])
@token_required
def unlike_post(post_id):
    conn = get_db()
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute(
        "DELETE FROM likes WHERE user_id = %s AND post_id = %s",
        (request.user_id, post_id),
    )
    cur.execute("SELECT COUNT(*) FROM likes WHERE post_id = %s", (post_id,))
    like_count = cur.fetchone()["count"]
    cur.close()
    conn.close()

    publish_event("unlike", {"post_id": post_id, "like_count": like_count})
    return jsonify({"post_id": post_id, "like_count": like_count}), 200


# ---------- Long polling endpoint ----------

@api.route("/events/poll", methods=["GET"])
def poll_events():
    since_version = int(request.args.get("since", 0))
    event = wait_for_new_event(since_version)
    if event is None:
        # Timed out — nothing happened. Client should immediately call /events/poll again.
        return jsonify({"timeout": True, "version": since_version}), 204
    return jsonify(event), 200


if __name__ == "__main__":
    api.run(debug=True, threaded=True)   # threaded=True is REQUIRED for long polling to work!
```

### Notes on Version 1

- `cursor_factory=psycopg2.extras.RealDictCursor` makes rows come back as dicts (`{"user_id": 1, ...}`) instead of plain tuples, so we can `jsonify()` them directly.
- `RETURNING ...` on `INSERT`/`UPDATE` is Postgres-specific syntax that hands back the row you just wrote, in the same round-trip — no need for a second `SELECT`.
- We catch `psycopg2.errors.UniqueViolation` to turn "duplicate like" (violating our composite PK) into a clean `409 Conflict` instead of a raw 500 error.
- `threaded=True` in `api.run(...)` is **not optional** — without it, Flask's dev server handles one request at a time, and a long-polling request would freeze your entire server for every other user until it times out.

---

## 7. Version 2 — SQLAlchemy ORM API

Same exact endpoints, same behavior — but instead of writing SQL strings, we define Python classes (**models**) that represent tables, and call methods/attributes on them. SQLAlchemy translates that into SQL behind the scenes.

```python
# models.py
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = "users"
    user_id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    email = db.Column(db.String(100), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    bio = db.Column(db.Text, nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    posts = db.relationship("Post", backref="author", lazy=True)
    likes = db.relationship("Like", backref="user", lazy=True)

    def to_dict(self):
        return {
            "user_id": self.user_id,
            "username": self.username,
            "email": self.email,
            "bio": self.bio,
        }


class Post(db.Model):
    __tablename__ = "posts"
    post_id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey("users.user_id"), nullable=False)
    caption = db.Column(db.Text, nullable=True)
    image_url = db.Column(db.String(255), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    likes = db.relationship("Like", backref="post", lazy=True)

    def to_dict(self):
        return {
            "post_id": self.post_id,
            "user_id": self.user_id,
            "username": self.author.username,
            "caption": self.caption,
            "image_url": self.image_url,
            "created_at": str(self.created_at),
            "like_count": len(self.likes),
        }


class Like(db.Model):
    __tablename__ = "likes"
    user_id = db.Column(db.Integer, db.ForeignKey("users.user_id"), primary_key=True)
    post_id = db.Column(db.Integer, db.ForeignKey("posts.post_id"), primary_key=True)
    liked_at = db.Column(db.DateTime, default=datetime.utcnow)
```

Notice: `db.relationship(...)` is the ORM's way of expressing the FK relationships from your SQL notes — `User.posts` gives you every `Post` row belonging to a user **without writing a JOIN yourself**; SQLAlchemy generates it. `primary_key=True` on *both* `user_id` and `post_id` in `Like` is exactly the **composite key** from your notes, just expressed in Python instead of `PRIMARY KEY (user_id, post_id)`.

```python
# app_orm.py
import threading
import time
import jwt
import datetime
from functools import wraps
from flask import Flask, request, jsonify
from werkzeug.security import generate_password_hash, check_password_hash
from sqlalchemy.exc import IntegrityError
from models import db, User, Post, Like

api = Flask(__name__)
SECRET_KEY = "change-this-to-a-real-secret-in-production"
api.config["SQLALCHEMY_DATABASE_URI"] = "postgresql://postgres:yourpassword@localhost:5432/instagram_clone"
api.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
db.init_app(api)

with api.app_context():
    db.create_all()   # creates all tables from models.py if they don't already exist


# ---------- JWT helpers (identical to Version 1) ----------

def create_token(user_id):
    payload = {
        "user_id": user_id,
        "exp": datetime.datetime.utcnow() + datetime.timedelta(days=7),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")


def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        if not auth_header.startswith("Bearer "):
            return jsonify({"error": "Missing or malformed Authorization header"}), 401
        token = auth_header.split(" ")[1]
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
        request.user_id = payload["user_id"]
        return f(*args, **kwargs)
    return decorated


# ---------- Shared state for long polling (identical concept to Version 1) ----------

lock = threading.Lock()
latest_event = {"version": 0, "type": None, "data": None}


def publish_event(event_type, data):
    with lock:
        latest_event["version"] += 1
        latest_event["type"] = event_type
        latest_event["data"] = data


def wait_for_new_event(since_version, timeout=25):
    start = time.time()
    while time.time() - start < timeout:
        with lock:
            if latest_event["version"] > since_version:
                return dict(latest_event)
        time.sleep(0.5)
    return None


# ---------- Auth routes ----------

@api.route("/signup", methods=["POST"])
def signup():
    data = request.get_json()
    username, email, password = data.get("username"), data.get("email"), data.get("password")
    if not all([username, email, password]):
        return jsonify({"error": "username, email, password are required"}), 400

    user = User(username=username, email=email, password_hash=generate_password_hash(password))
    db.session.add(user)
    try:
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "username or email already taken"}), 409

    token = create_token(user.user_id)
    return jsonify({"user": user.to_dict(), "token": token}), 201


@api.route("/login", methods=["POST"])
def login():
    data = request.get_json()
    user = User.query.filter_by(username=data.get("username")).first()
    if not user or not check_password_hash(user.password_hash, data.get("password")):
        return jsonify({"error": "Invalid username or password"}), 401
    return jsonify({"token": create_token(user.user_id)}), 200


# ---------- Profile routes ----------

@api.route("/users/<int:user_id>", methods=["GET"])
def get_profile(user_id):
    user = User.query.get(user_id)
    if not user:
        return jsonify({"error": "User not found"}), 404
    return jsonify(user.to_dict()), 200


@api.route("/users/me", methods=["PUT"])
@token_required
def update_profile():
    user = User.query.get(request.user_id)
    user.bio = request.get_json().get("bio")
    db.session.commit()
    return jsonify(user.to_dict()), 200


# ---------- Post routes ----------

@api.route("/posts", methods=["POST"])
@token_required
def create_post():
    data = request.get_json()
    image_url = data.get("image_url")
    if not image_url:
        return jsonify({"error": "image_url is required"}), 400

    post = Post(user_id=request.user_id, caption=data.get("caption"), image_url=image_url)
    db.session.add(post)
    db.session.commit()

    publish_event("new_post", post.to_dict())
    return jsonify(post.to_dict()), 201


@api.route("/feed", methods=["GET"])
def get_feed():
    limit = int(request.args.get("limit", 10))
    offset = int(request.args.get("offset", 0))
    posts = Post.query.order_by(Post.created_at.desc()).limit(limit).offset(offset).all()
    return jsonify([p.to_dict() for p in posts]), 200


# ---------- Like routes ----------

@api.route("/posts/<int:post_id>/like", methods=["POST"])
@token_required
def like_post(post_id):
    existing = Like.query.get((request.user_id, post_id))
    if existing:
        return jsonify({"error": "Already liked"}), 409

    db.session.add(Like(user_id=request.user_id, post_id=post_id))
    db.session.commit()

    like_count = Like.query.filter_by(post_id=post_id).count()
    publish_event("new_like", {"post_id": post_id, "like_count": like_count})
    return jsonify({"post_id": post_id, "like_count": like_count}), 201


@api.route("/posts/<int:post_id>/like", methods=["DELETE"])
@token_required
def unlike_post(post_id):
    like = Like.query.get((request.user_id, post_id))
    if like:
        db.session.delete(like)
        db.session.commit()

    like_count = Like.query.filter_by(post_id=post_id).count()
    publish_event("unlike", {"post_id": post_id, "like_count": like_count})
    return jsonify({"post_id": post_id, "like_count": like_count}), 200


# ---------- Long polling endpoint (identical to Version 1) ----------

@api.route("/events/poll", methods=["GET"])
def poll_events():
    since_version = int(request.args.get("since", 0))
    event = wait_for_new_event(since_version)
    if event is None:
        return jsonify({"timeout": True, "version": since_version}), 204
    return jsonify(event), 200


if __name__ == "__main__":
    api.run(debug=True, threaded=True)
```

### Raw SQL vs ORM — what actually changed

| Concern | Raw SQL (Version 1) | ORM (Version 2) |
|---|---|---|
| Table definitions | `schema.sql`, run manually | Python classes in `models.py`, `db.create_all()` |
| Inserting a row | Hand-written `INSERT ... RETURNING` | `db.session.add(obj)` + `db.session.commit()` |
| Reading a row | Hand-written `SELECT ... WHERE` | `Model.query.filter_by(...)` / `Model.query.get(...)` |
| Joins | Written explicitly (`JOIN posts ON ...`) | Often implicit via `relationship()` (`post.author.username`) |
| Duplicate-like protection | Catch `psycopg2.errors.UniqueViolation` | Catch `IntegrityError`, or check via `Like.query.get(...)` first |
| Composite PK | `PRIMARY KEY (user_id, post_id)` in SQL | Two columns both marked `primary_key=True` in the model |
| What you actually control | Every query, fully | SQLAlchemy generates the SQL — you describe *what*, it decides *how* |

**The long-polling logic, the JWT logic, and the global lock are 100% identical in both versions** — those concerns live entirely outside the database layer, which is exactly why we could swap SQL for an ORM without touching them at all.

---

## 8. The Long-Polling Endpoint, Line by Line

```python
@api.route("/events/poll", methods=["GET"])
def poll_events():
    since_version = int(request.args.get("since", 0))
    event = wait_for_new_event(since_version)
    if event is None:
        return jsonify({"timeout": True, "version": since_version}), 204
    return jsonify(event), 200
```

- `request.args.get("since", 0)` — the client tells us "the last event version I already know about." First call ever, it sends `since=0`.
- `wait_for_new_event(since_version)` — this is where the magic happens: the function **does not return immediately**. It loops, checking the shared `latest_event` (protected by `lock`) every 0.5 seconds, for up to 25 seconds, looking for a version number higher than what the client already has.
- If something new happened during that window → return it immediately, with its new version number.
- If nothing happened in 25 seconds → return `204` (no content) with a `timeout` flag. The client treats this exactly like "no news," and should **immediately call `/events/poll` again** — this re-opening is what makes it "long polling" rather than "one giant request that eventually just gives up."

### The matching client-side loop (JavaScript, for context — not part of the Flask app)

```javascript
let sinceVersion = 0;

async function pollLoop() {
    while (true) {
        const res = await fetch(`/events/poll?since=${sinceVersion}`);
        if (res.status === 200) {
            const event = await res.json();
            sinceVersion = event.version;
            handleEvent(event);   // e.g. update the UI: bump a like counter, show a new post
        }
        // whether it was a real event or a timeout, immediately loop and ask again
    }
}

function handleEvent(event) {
    if (event.type === "new_like") {
        document.getElementById(`like-count-${event.data.post_id}`).innerText = event.data.like_count;
    } else if (event.type === "new_post") {
        prependPostToFeed(event.data);
    }
}

pollLoop();
```

This client never sleeps on a timer (unlike short polling) — it just immediately re-asks the moment it gets *any* response, real or timeout. The server is doing all the "waiting" work.

### Why this specific design choice — a single `latest_event`, not a queue

For learning purposes we keep **one shared "latest event," with a version counter**, rather than a full event queue per client. This is intentionally simple:

- ✅ Simple to understand and implement with just a `Lock` and a dict.
- ✅ Works great for "everyone watching the global feed sees the same updates."
- ⚠️ **Limitation:** if two events happen in the same instant before a client polls, the client only sees the *latest* one — it doesn't see a full history of everything it missed. For a real production app, you'd replace `latest_event` with a list (a small ring buffer) of the last N events, and have `wait_for_new_event` return *all* events newer than `since_version`, not just the newest one. The locking pattern stays exactly the same — only the data structure being protected changes.

---

## 9. Testing the API (example requests)

```bash
# Sign up
curl -X POST http://localhost:5000/signup \
  -H "Content-Type: application/json" \
  -d '{"username": "priya_d", "email": "priya@example.com", "password": "secret123"}'

# Log in (grab the token from the response)
curl -X POST http://localhost:5000/login \
  -H "Content-Type: application/json" \
  -d '{"username": "priya_d", "password": "secret123"}'

# Create a post (note the Bearer token)
curl -X POST http://localhost:5000/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN HERE>" \
  -d '{"caption": "Golden hour shoot", "image_url": "https://cdn.example.com/img3.jpg"}'

# Like a post
curl -X POST http://localhost:5000/posts/1/like \
  -H "Authorization: Bearer <TOKEN HERE>"

# Long-poll for updates (this call will "hang" until something happens, or ~25s pass)
curl "http://localhost:5000/events/poll?since=0"
```

Open two terminals: in one, run the `curl .../events/poll?since=0` command and watch it hang. In the other, like a post. The first terminal should immediately return with the like event — that's long polling working.

---

## 10. Where To Go From Here

- **Connection pooling:** replace `psycopg2.connect(...)` per-request with `psycopg2.pool.SimpleConnectionPool`, or for the ORM version, SQLAlchemy already pools connections for you by default.
- **Pagination on the feed:** already there via `limit`/`offset` — try adding cursor-based pagination (using `created_at` + `post_id` as a cursor) as a SQL-skills exercise straight from your notes (subqueries/CTEs).
- **Multiple events in flight:** swap `latest_event` for a small ring-buffer list, as mentioned in Section 8.
- **Refresh tokens:** JWTs here just expire after 7 days and force a re-login; a production app usually pairs a short-lived **access token** with a long-lived **refresh token** stored more securely.
- **Followers table:** your SQL notes already describe this — a `follows` table with `follower_id`/`followee_id`, both FKs to `users`, composite PK — try adding `/users/<id>/follow` as a follow-up exercise using everything covered here.
