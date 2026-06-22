# Instagram API — Node.js + Express + `pg` (Raw SQL)

This is a direct port of the Flask/`psycopg2` API to Node.js, using **Express** for routing and **`pg`** for raw SQL queries against Postgres — same schema, same endpoints, same JWT auth, same long-polling mechanism. Read alongside the Python version; every section below calls out the Node equivalent of each Python concept.

---

## 1. Project Setup

```
instagram_api_node/
├── server.js
├── db.js
├── auth.js
├── package.json
└── schema.sql
```

```bash
npm init -y
npm install express pg jsonwebtoken bcrypt dotenv
```

| Python package | Node equivalent | Role |
|---|---|---|
| `flask` | `express` | web framework / routing |
| `psycopg2` | `pg` | raw SQL Postgres driver |
| `PyJWT` | `jsonwebtoken` | issue/verify JWTs |
| `werkzeug.security` | `bcrypt` | password hashing |
| `threading.Lock` | nothing needed — see Section 4 | concurrency control |

---

## 2. Why There's No "Global Lock" in Node

This is the single biggest conceptual difference, so it's worth covering before any code.

**Python/Flask:** `threaded=True` lets multiple OS threads run your route handlers concurrently. Two requests can genuinely be executing Python at overlapping moments, both touching `latest_event` — hence `threading.Lock()` was required to prevent corruption.

**Node.js/Express:** Node runs your JavaScript on a **single thread**, using an **event loop**. Only one piece of your JS code is ever executing at any instant — there's no "two requests both in the middle of mutating the same object." Node achieves *concurrency* (handling many requests "at once") not via parallel threads, but by Promise: when a request does something slow (a `pg` query, a `setTimeout`, a file read), it hands that operation off to the underlying C++/libuv layer and the event loop immediately moves on to serve other requests. When the slow operation finishes, its callback gets queued and run later — but always one JS callback at a time, never two simultaneously.

**Consequence:** a plain JS object like `let latestEvent = {...}` can be read and written directly, with **no lock**, and it's still safe — there's no point in time where two pieces of code are mid-mutation of it at once. This is the most common "aha" moment going from Python concurrency to Node concurrency: **Node's single-threaded event loop gives you the same practical safety a lock gives you in Python, for free, as long as you don't introduce actual multi-process state (e.g. running Node in `cluster` mode across multiple CPU cores — at that point you're back to needing shared coordination, just like `multiprocessing` in Python).**

So in the code below, you'll see `let latestEvent` mutated directly inside `publishEvent()` — no `Mutex`/`Lock` library required, and that's not a shortcut, it's just how Node's model works.

---

## 3. Database Connection — `pg`

```js
// db.js
const { Pool } = require("pg");

const pool = new Pool({
  user: "postgres",
  host: "localhost",
  database: "instagram_clone",
  password: "yourpassword",
  port: 5432,
});

module.exports = pool;
```

Unlike the Python version (which opened a brand-new `psycopg2.connect()` per request), `pg`'s `Pool` is a **connection pool** out of the box — it keeps a small number of open connections and hands them out to queries as needed, reusing them. This is the production-grade pattern the Python notes mentioned as a "next step"; in Node it's the default, idiomatic way to use `pg` from day one.

```js
// any query, anywhere in the app:
const result = await pool.query("SELECT * FROM users WHERE user_id = $1", [userId]);
result.rows;       // array of row objects — same role as psycopg2's RealDictCursor result
```

Note Postgres placeholders in `pg` are `$1, $2, $3...` (positional, 1-indexed) instead of `psycopg2`'s `%s`.

---

## 4. JWT Auth — `jsonwebtoken` + `bcrypt`

```js
// auth.js
const jwt = require("jsonwebtoken");

const SECRET_KEY = "change-this-to-a-real-secret-in-production";

function createToken(userId) {
  return jwt.sign({ user_id: userId }, SECRET_KEY, { expiresIn: "7d" });
}

function tokenRequired(req, res, next) {
  const authHeader = req.headers["authorization"] || "";
  if (!authHeader.startsWith("Bearer ")) {
    return res.status(401).json({ error: "Missing or malformed Authorization header" });
  }

  const token = authHeader.split(" ")[1];
  try {
    const payload = jwt.verify(token, SECRET_KEY);
    req.userId = payload.user_id;   // stash it for the route handler, like request.user_id in Flask
    next();                          // move on to the actual route handler
  } catch (err) {
    if (err.name === "TokenExpiredError") {
      return res.status(401).json({ error: "Token expired" });
    }
    return res.status(401).json({ error: "Invalid token" });
  }
}

module.exports = { createToken, tokenRequired };
```

**`tokenRequired` is Express's version of `@token_required`.** In Flask, a decorator wraps a function and runs code before/after it. In Express, **middleware** is the equivalent concept: a function `(req, res, next)` that runs before the route handler, and calls `next()` to let the request continue (or sends a response itself and stops it there). You attach it the same way you'd stack a decorator:

```js
app.post("/posts", tokenRequired, createPost);
//                  ^^^^^^^^^^^^^ middleware runs first, exactly like @token_required above a Flask route
```

---

## 5. Routing — Express vs `@api.route`

| Flask | Express |
|---|---|
| `@api.route("/posts", methods=["POST"])` <br> `def create_post(): ...` | `app.post("/posts", createPost)` |
| `@api.route("/users/<int:user_id>")` | `app.get("/users/:userId", getProfile)` — note: Express doesn't auto-cast to int, you `parseInt()` yourself |
| `request.get_json()` | `req.body` (after enabling the `express.json()` middleware) |
| `request.args.get("limit", 10)` | `req.query.limit || 10` |
| `return jsonify(x), 201` | `res.status(201).json(x)` |

```js
const express = require("express");
const app = express();
app.use(express.json());   // equivalent of Flask auto-parsing request.get_json() — REQUIRED for req.body to work
```

---

## 6. Full Server Code

```js
// server.js
require("dotenv").config();
const express = require("express");
const bcrypt = require("bcrypt");
const pool = require("./db");
const { createToken, tokenRequired } = require("./auth");

const app = express();
app.use(express.json());

// ---------- Shared state for long polling ----------
// No lock needed — see Section 2. Node's single-threaded event loop means
// no two pieces of JS code ever run "at the same instant."

let latestEvent = { version: 0, type: null, data: null };

function publishEvent(eventType, data) {
  latestEvent.version += 1;
  latestEvent.type = eventType;
  latestEvent.data = data;
}

function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function waitForNewEvent(sinceVersion, timeoutMs = 25000) {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    if (latestEvent.version > sinceVersion) {
      return { ...latestEvent };   // shallow copy, same role as dict(latest_event) in Python
    }
    await sleep(500);   // yields control back to the event loop so other requests get served
  }
  return null;
}

// ---------- Auth routes ----------

app.post("/signup", async (req, res) => {
  const { username, email, password } = req.body;
  if (!username || !email || !password) {
    return res.status(400).json({ error: "username, email, password are required" });
  }

  const passwordHash = await bcrypt.hash(password, 10);

  try {
    const result = await pool.query(
      `INSERT INTO users (username, email, password_hash)
       VALUES ($1, $2, $3)
       RETURNING user_id, username, email`,
      [username, email, passwordHash]
    );
    const user = result.rows[0];
    const token = createToken(user.user_id);
    return res.status(201).json({ user, token });
  } catch (err) {
    if (err.code === "23505") {   // Postgres unique_violation error code — pg's equivalent of UniqueViolation
      return res.status(409).json({ error: "username or email already taken" });
    }
    console.error(err);
    return res.status(500).json({ error: "Internal server error" });
  }
});

app.post("/login", async (req, res) => {
  const { username, password } = req.body;

  const result = await pool.query("SELECT * FROM users WHERE username = $1", [username]);
  const user = result.rows[0];

  if (!user || !(await bcrypt.compare(password, user.password_hash))) {
    return res.status(401).json({ error: "Invalid username or password" });
  }

  const token = createToken(user.user_id);
  return res.status(200).json({ token });
});

// ---------- Profile routes ----------

app.get("/users/:userId", async (req, res) => {
  const userId = parseInt(req.params.userId, 10);

  const result = await pool.query(
    "SELECT user_id, username, email, bio, created_at FROM users WHERE user_id = $1",
    [userId]
  );
  const user = result.rows[0];
  if (!user) {
    return res.status(404).json({ error: "User not found" });
  }
  return res.status(200).json(user);
});

app.put("/users/me", tokenRequired, async (req, res) => {
  const { bio } = req.body;

  const result = await pool.query(
    `UPDATE users SET bio = $1 WHERE user_id = $2
     RETURNING user_id, username, bio`,
    [bio, req.userId]
  );
  return res.status(200).json(result.rows[0]);
});

// ---------- Post routes ----------

app.post("/posts", tokenRequired, async (req, res) => {
  const { caption, image_url } = req.body;
  if (!image_url) {
    return res.status(400).json({ error: "image_url is required" });
  }

  const result = await pool.query(
    `INSERT INTO posts (user_id, caption, image_url)
     VALUES ($1, $2, $3)
     RETURNING post_id, user_id, caption, image_url, created_at`,
    [req.userId, caption, image_url]
  );
  const post = result.rows[0];

  publishEvent("new_post", post);
  return res.status(201).json(post);
});

app.get("/feed", async (req, res) => {
  const limit = parseInt(req.query.limit || "10", 10);
  const offset = parseInt(req.query.offset || "0", 10);

  const result = await pool.query(
    `SELECT posts.post_id, posts.caption, posts.image_url, posts.created_at,
            users.username,
            COUNT(likes.user_id) AS like_count
     FROM posts
     JOIN users ON posts.user_id = users.user_id
     LEFT JOIN likes ON posts.post_id = likes.post_id
     GROUP BY posts.post_id, posts.caption, posts.image_url, posts.created_at, users.username
     ORDER BY posts.created_at DESC
     LIMIT $1 OFFSET $2`,
    [limit, offset]
  );
  return res.status(200).json(result.rows);
});

// ---------- Like routes ----------

app.post("/posts/:postId/like", tokenRequired, async (req, res) => {
  const postId = parseInt(req.params.postId, 10);

  try {
    await pool.query(
      "INSERT INTO likes (user_id, post_id) VALUES ($1, $2)",
      [req.userId, postId]
    );
  } catch (err) {
    if (err.code === "23505") {
      return res.status(409).json({ error: "Already liked" });
    }
    console.error(err);
    return res.status(500).json({ error: "Internal server error" });
  }

  const countResult = await pool.query("SELECT COUNT(*) FROM likes WHERE post_id = $1", [postId]);
  const likeCount = parseInt(countResult.rows[0].count, 10);

  publishEvent("new_like", { post_id: postId, like_count: likeCount });
  return res.status(201).json({ post_id: postId, like_count: likeCount });
});

app.delete("/posts/:postId/like", tokenRequired, async (req, res) => {
  const postId = parseInt(req.params.postId, 10);

  await pool.query("DELETE FROM likes WHERE user_id = $1 AND post_id = $2", [req.userId, postId]);

  const countResult = await pool.query("SELECT COUNT(*) FROM likes WHERE post_id = $1", [postId]);
  const likeCount = parseInt(countResult.rows[0].count, 10);

  publishEvent("unlike", { post_id: postId, like_count: likeCount });
  return res.status(200).json({ post_id: postId, like_count: likeCount });
});

// ---------- Long polling endpoint ----------

app.get("/events/poll", async (req, res) => {
  const sinceVersion = parseInt(req.query.since || "0", 10);
  const event = await waitForNewEvent(sinceVersion);

  if (event === null) {
    // Timed out — nothing happened. Client should immediately call /events/poll again.
    return res.status(204).json({ timeout: true, version: sinceVersion });
  }
  return res.status(200).json(event);
});

const PORT = 5000;
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

```sql
-- schema.sql — identical to the Python version
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

---

## 7. The Long-Polling Endpoint, Line by Line (Node Version)

```js
async function waitForNewEvent(sinceVersion, timeoutMs = 25000) {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    if (latestEvent.version > sinceVersion) {
      return { ...latestEvent };
    }
    await sleep(500);
  }
  return null;
}
```

- `Date.now()` — milliseconds since epoch; equivalent to Python's `time.time()` (which is in seconds — hence `25` vs `25000` here).
- `while (Date.now() - start < timeoutMs)` — same loop shape as the Python `while time.time() - start < timeout:`.
- `await sleep(500)` — this is the critical line. `sleep()` wraps `setTimeout` in a Promise. `await`-ing it **pauses only this specific async function**, immediately handing control back to Node's event loop so it can process *other* incoming requests during that 500ms — this is Node's equivalent of Python's `time.sleep(0.5)` releasing the GIL. Nothing else is blocked while we wait.
- `return { ...latestEvent }` — the `...` spread operator copies the object's own properties into a new object, the same role as Python's `dict(latest_event)`: we don't want to return a live reference that could change after we hand it back.
- No `lock` anywhere — see Section 2 for why that's safe here.

```js
app.get("/events/poll", async (req, res) => {
  const sinceVersion = parseInt(req.query.since || "0", 10);
  const event = await waitForNewEvent(sinceVersion);

  if (event === null) {
    return res.status(204).json({ timeout: true, version: sinceVersion });
  }
  return res.status(200).json(event);
});
```

This route handler is declared `async` specifically so it can `await waitForNewEvent(...)` — while this `await` is pending, Express is completely free to accept and respond to *other* requests in parallel (conceptually parallel — still one thread, but interleaved via the event loop). That's the whole trick: one "slow" request doesn't block anyone else, with zero manual thread/lock management.

The same matching client-side JS loop from the Python notes works **unchanged** against this server — long polling is a client/server *protocol* convention, not something tied to the server's language:

```javascript
let sinceVersion = 0;

async function pollLoop() {
  while (true) {
    const res = await fetch(`/events/poll?since=${sinceVersion}`);
    if (res.status === 200) {
      const event = await res.json();
      sinceVersion = event.version;
      handleEvent(event);
    }
  }
}
```

---

## 8. Side-by-Side Cheat Sheet: Flask/`psycopg2` vs Express/`pg`

| Concern | Flask + psycopg2 | Express + pg |
|---|---|---|
| Route registration | `@api.route("/posts", methods=["POST"])` | `app.post("/posts", handler)` |
| Path param | `<int:user_id>` (auto-cast) | `:userId` (string — cast with `parseInt`) |
| Request body | `request.get_json()` | `req.body` (needs `express.json()` middleware) |
| Query string | `request.args.get("limit", 10)` | `req.query.limit \|\| 10` |
| Response | `return jsonify(x), 201` | `res.status(201).json(x)` |
| Auth gate | decorator `@token_required` | middleware `tokenRequired` passed as a route arg |
| SQL placeholders | `%s` | `$1, $2, ...` |
| Row results | `RealDictCursor` → dict | `result.rows` → array of objects |
| Duplicate-key error | `psycopg2.errors.UniqueViolation` | `err.code === "23505"` |
| Connection management | manual `get_db()` / `.close()` per request | `Pool` handles it automatically |
| Concurrency model | OS threads (`threaded=True`) + GIL | single-threaded event loop |
| Shared state protection | `threading.Lock()` required | not required — single thread guarantees no overlap |
| Password hashing | `werkzeug.security` | `bcrypt` |
| JWT library | `PyJWT` | `jsonwebtoken` |

---

## 9. Testing (identical `curl` calls — only the port may differ)

```bash
curl -X POST http://localhost:5000/signup \
  -H "Content-Type: application/json" \
  -d '{"username": "priya_d", "email": "priya@example.com", "password": "secret123"}'

curl -X POST http://localhost:5000/login \
  -H "Content-Type: application/json" \
  -d '{"username": "priya_d", "password": "secret123"}'

curl -X POST http://localhost:5000/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN HERE>" \
  -d '{"caption": "Golden hour shoot", "image_url": "https://cdn.example.com/img3.jpg"}'

curl -X POST http://localhost:5000/posts/1/like \
  -H "Authorization: Bearer <TOKEN HERE>"

curl "http://localhost:5000/events/poll?since=0"
```

Same test as before: open two terminals, hang one on `/events/poll?since=0`, like a post from the other, and watch the first terminal return immediately.

---

## 10. Where to Go From Here

- **`async`/`await` everywhere, no callback pyramids:** every route above is an `async (req, res) => {...}` function, which is why `await pool.query(...)` reads almost identically to synchronous code while still being non-blocking under the hood.
- **Error handling middleware:** right now uncaught errors in an `async` route can crash the process; wrap routes in a try/catch or add a global Express error-handling middleware (`app.use((err, req, res, next) => {...})`) for production use.
- **Scaling beyond one core:** Node's single-threaded model means one process uses one CPU core. Node's built-in `cluster` module (or a process manager like PM2) lets you run multiple Node processes that share a port — at that point, like `multiprocessing` in Python, `latestEvent` would need to live somewhere shared across processes (e.g. Redis) instead of a plain in-memory variable, since each process gets its own separate memory.
- **Migrating to an ORM:** the Node ecosystem's equivalent of SQLAlchemy is typically **Prisma** or **Sequelize** — a natural follow-up exercise mirroring the raw-SQL-vs-ORM comparison from the Python version.
