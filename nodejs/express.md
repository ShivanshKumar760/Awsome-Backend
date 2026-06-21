# Building an Express + TypeScript + PostgreSQL API with JWT Auth

### (Raw SQL first, then the same API rebuilt with Prisma)

This is a hands-on build, not just a reference. We'll build **the same small API twice** —
`POST /signup`, `POST /login`, `POST /posts`, `GET /posts`, `DELETE /posts/:postId` — once
talking to Postgres with raw SQL (`pg`), then again with Prisma ORM, so you can directly compare
what each approach makes easy vs. tedious. The auth/JWT layer is **identical in both** — that's
deliberate, and the point gets called out explicitly in Part 2.

```
Part 0 — Shared concepts: project setup, env vars, JWT mental model
Part 1 — Raw SQL implementation (pg.Pool, hand-written queries)
Part 2 — Prisma implementation (schema.prisma, migrations, type-safe client)
Part 3 — Comparison + when to reach for which
```

---

## Part 0 — Shared Setup & the JWT Mental Model

### 0.1 What "auth middleware with JWT" actually means

1. `/signup` and `/login` issue a **JWT** — a signed token encoding `{ userId }` — instead of
   creating a server-side session. The server stores nothing about "who's logged in"; the token
   itself is the proof.
2. The client stores that token and sends it on every subsequent request in the
   `Authorization: Bearer <token>` header.
3. A piece of **middleware** runs before any protected route, reads that header, **verifies** the
   token's signature (proving it wasn't tampered with and was issued by this server), and attaches
   `req.userId` so the route handler knows who's calling.
4. No verification = no `req.userId` = `401`, before the route handler even runs.

A JWT has three dot-separated parts: `header.payload.signature` — e.g.
`eyJhbGc...` . `eyJ1c2VySWQ...` . `SflKxw...`. The payload is **base64-encoded, not encrypted** —
anyone can decode and read it (try pasting one into jwt.io), so **never put passwords or secrets
in the payload**. The signature is what makes it trustworthy: it's an HMAC of the header+payload
using `JWT_SECRET`, which only your server knows. Change one character of the payload and the
signature no longer matches — that's the entire security model.

### 0.2 Project scaffolding (run for _each_ part — `raw-sql-api/` and `prisma-api/`)

```bash
mkdir raw-sql-api && cd raw-sql-api
npm init -y
npm install express bcrypt jsonwebtoken dotenv pg
npm install -D typescript ts-node-dev @types/node @types/express @types/bcrypt @types/jsonwebtoken @types/pg
npx tsc --init
```

(For the Prisma version in Part 2, swap `pg` + `@types/pg` for `@prisma/client` + `prisma` — full
command given there.)

**`tsconfig.json`** — same for both parts:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
```

**`package.json` scripts** — same for both parts:

```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  }
}
```

**`.env`** — same shape for both parts (Prisma will also read `DATABASE_URL` from here):

```
DATABASE_URL=postgres://postgres:postgres@localhost:5432/blog_api
JWT_SECRET=replace-this-with-a-long-random-string
PORT=5000
```

Generate a real secret instead of typing one: `openssl rand -hex 32`.

### 0.3 Create the database (once, before either part)

```bash
createdb blog_api
# or: psql -U postgres -c "CREATE DATABASE blog_api;"
```

---

## Part 1 — Raw SQL Implementation

### 1.1 Folder structure

```
raw-sql-api/
├── src/
│   ├── server.ts
│   ├── db.ts
│   ├── auth.ts                # JWT sign/verify + middleware
│   ├── types/
│   │   └── express.d.ts       # augments Request with userId
│   └── routes/
│       ├── auth.routes.ts
│       └── posts.routes.ts
├── schema.sql
├── .env
├── tsconfig.json
└── package.json
```

### 1.2 Schema — write and apply the SQL by hand

```sql
-- schema.sql
CREATE TABLE users (
  user_id       SERIAL PRIMARY KEY,
  username      VARCHAR(50) UNIQUE NOT NULL,
  email         VARCHAR(255) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE posts (
  post_id    SERIAL PRIMARY KEY,
  user_id    INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
  caption    TEXT,
  image_url  TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

```bash
psql -d blog_api -f schema.sql
```

This is the core tradeoff of the raw-SQL approach up front: **you** own the schema file and
**you** run it. There's no migration history tracked by tooling — if you change a column later,
you write and run another `.sql` file (or a tool like `node-pg-migrate`) yourself.

### 1.3 `src/db.ts` — the connection pool

```ts
import { Pool } from "pg";
import dotenv from "dotenv";
dotenv.config();

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
```

A `Pool`, not a single `Client`: each query borrows a connection from a pool of reusable ones
instead of opening/closing a TCP connection to Postgres per request — this matters as soon as you
have concurrent requests.

### 1.4 `src/types/express.d.ts` — teaching TypeScript about `req.userId`

```ts
import "express";

declare module "express" {
  interface Request {
    userId?: number;
  }
}
```

Without this, `req.userId = payload.userId` in the middleware below is a TypeScript error —
`Request` has no such property by default. This file extends the type, it emits no JS.

### 1.5 `src/auth.ts` — JWT creation + the auth middleware

```ts
import jwt from "jsonwebtoken";
import { Request, Response, NextFunction } from "express";

const JWT_SECRET = process.env.JWT_SECRET;
if (!JWT_SECRET) {
  throw new Error("JWT_SECRET is not set — check your .env file");
}

interface JwtPayload {
  userId: number;
}

export function createToken(userId: number): string {
  return jwt.sign({ userId } satisfies JwtPayload, JWT_SECRET, {
    expiresIn: "7d",
  });
}

export function tokenRequired(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization; // "Bearer <token>"

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res
      .status(401)
      .json({ error: "Missing or malformed Authorization header" });
  }

  const token = authHeader.slice("Bearer ".length);

  try {
    const payload = jwt.verify(token, JWT_SECRET) as JwtPayload;
    req.userId = payload.userId;
    next(); // hand off to the actual route handler
  } catch {
    // covers both "bad signature" and "expired" — jwt.verify throws for both
    return res.status(401).json({ error: "Invalid or expired token" });
  }
}
```

Walk through what `tokenRequired` does, in order: pull the header → reject if it's missing or
malformed → strip `"Bearer "` → `jwt.verify` (throws if the signature doesn't match `JWT_SECRET`
or the token's `exp` has passed) → on success, attach `userId` to the request and call `next()` →
on failure, respond `401` and **never call `next()`**, so the route never runs.

> Note the `process.env.JWT_SECRET` guard at the top. `process.env` values are typed
> `string | undefined` in TS — `jwt.sign` won't accept `undefined` as a secret, so this check
> isn't defensive boilerplate, it's required for the file to type-check at all.

### 1.6 `src/routes/auth.routes.ts` — signup & login, raw SQL

```ts
import { Router } from "express";
import bcrypt from "bcrypt";
import { pool } from "../db";
import { createToken } from "../auth";

const router = Router();

router.post("/signup", async (req, res) => {
  const { username, email, password } = req.body;
  if (!username || !email || !password) {
    return res
      .status(400)
      .json({ error: "username, email, password are required" });
  }

  const passwordHash = await bcrypt.hash(password, 10); // 10 = bcrypt cost factor

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
  } catch (err: any) {
    if (err.code === "23505") {
      // Postgres unique_violation — username or email already exists
      return res.status(409).json({ error: "username or email already taken" });
    }
    console.error(err);
    return res.status(500).json({ error: "Internal server error" });
  }
});

router.post("/login", async (req, res) => {
  const { username, password } = req.body;

  const result = await pool.query("SELECT * FROM users WHERE username = $1", [
    username,
  ]);
  const user = result.rows[0];

  // Compare against bcrypt.compare even when no user is found, by failing the same way either
  // way — don't let response shape/timing reveal whether a username exists.
  if (!user || !(await bcrypt.compare(password, user.password_hash))) {
    return res.status(401).json({ error: "Invalid username or password" });
  }

  const token = createToken(user.user_id);
  return res.status(200).json({ token });
});

export default router;
```

Note we never return `password_hash` in any response — `RETURNING` on signup explicitly lists
safe columns, and login only ever sends back a token.

### 1.7 `src/routes/posts.routes.ts` — CRUD, raw SQL, JWT-protected where needed

```ts
import { Router } from "express";
import { pool } from "../db";
import { tokenRequired } from "../auth";

const router = Router();

// Protected: needs a valid JWT, tokenRequired sets req.userId before this runs
router.post("/posts", tokenRequired, async (req, res) => {
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
  return res.status(201).json(result.rows[0]);
});

// Public: no tokenRequired, anyone can browse the feed
router.get("/posts", async (req, res) => {
  const limit = parseInt((req.query.limit as string) ?? "10", 10);
  const offset = parseInt((req.query.offset as string) ?? "0", 10);

  const result = await pool.query(
    `SELECT posts.post_id, posts.caption, posts.image_url, posts.created_at, users.username
     FROM posts
     JOIN users ON posts.user_id = users.user_id
     ORDER BY posts.created_at DESC
     LIMIT $1 OFFSET $2`,
    [limit, offset]
  );
  return res.status(200).json(result.rows);
});

// Protected + ownership check: the WHERE clause itself enforces "only your own post"
router.delete("/posts/:postId", tokenRequired, async (req, res) => {
  const postId = parseInt(req.params.postId, 10);

  const result = await pool.query(
    "DELETE FROM posts WHERE post_id = $1 AND user_id = $2 RETURNING post_id",
    [postId, req.userId]
  );

  if (!result.rowCount) {
    return res.status(404).json({ error: "Post not found or not yours" });
  }
  return res.status(204).send();
});

export default router;
```

The delete route is worth lingering on: `WHERE post_id = $1 AND user_id = $2` does the ownership
check **inside the SQL itself**, in one round trip — no separate "fetch post, check
`post.userId === req.userId` in JS, then delete" sequence, which would be slower and race-prone.

### 1.8 `src/server.ts` — wiring it together

```ts
import express from "express";
import dotenv from "dotenv";
dotenv.config();

import authRoutes from "./routes/auth.routes";
import postsRoutes from "./routes/posts.routes";

const app = express();
app.use(express.json());

app.use(authRoutes);
app.use(postsRoutes);

app.get("/health", (_req, res) => res.json({ status: "ok" }));

const PORT = process.env.PORT ?? 5000;
app.listen(PORT, () =>
  console.log(`Raw-SQL API running on http://localhost:${PORT}`)
);
```

### 1.9 Run it and test the full auth flow

```bash
npm run dev
```

```bash
# signup -> get a token back
curl -s -X POST localhost:5000/signup -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com","password":"hunter2"}'

# login -> get a token
TOKEN=$(curl -s -X POST localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"hunter2"}' | jq -r .token)

# protected route, with the token
curl -s -X POST localhost:5000/posts \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"caption":"hello","image_url":"https://example.com/a.jpg"}'

# same route, no token -> 401
curl -i -X POST localhost:5000/posts -H "Content-Type: application/json" \
  -d '{"caption":"hello","image_url":"https://example.com/a.jpg"}'
```

---

## Part 2 — Same API, Rebuilt with Prisma

### 2.1 What's about to stay identical vs. change

**Stays exactly the same:** `src/auth.ts` (JWT signing/verification), `src/types/express.d.ts`,
the route _shape_ (`/signup`, `/login`, `/posts`), the `tokenRequired` middleware, and how it's
wired into `server.ts`. **This is the main lesson of doing it twice** — JWT auth is a concern that
sits _above_ your data layer; swapping how you talk to Postgres doesn't touch it at all.

**Changes:** `db.ts` becomes a Prisma client instead of a `pg.Pool`; `schema.sql` becomes
`schema.prisma`; every hand-written SQL string becomes a Prisma Client method call.

### 2.2 Folder structure

```
prisma-api/
├── prisma/
│   └── schema.prisma
├── src/
│   ├── server.ts
│   ├── db.ts                  # Prisma client singleton (was pg Pool)
│   ├── auth.ts                # identical to Part 1
│   ├── types/
│   │   └── express.d.ts       # identical to Part 1
│   └── routes/
│       ├── auth.routes.ts
│       └── posts.routes.ts
├── .env
├── tsconfig.json
└── package.json
```

### 2.3 Install and initialize Prisma

```bash
mkdir prisma-api && cd prisma-api
npm init -y
npm install express bcrypt jsonwebtoken dotenv @prisma/client
npm install -D typescript ts-node-dev @types/node @types/express @types/bcrypt @types/jsonwebtoken prisma
npx tsc --init
npx prisma init
```

`prisma init` creates `prisma/schema.prisma` and a starter `.env` with a `DATABASE_URL`
placeholder — overwrite it with your real connection string (same value as Part 1's `.env`).

### 2.4 `prisma/schema.prisma` — the schema, Prisma's way

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  userId       Int      @id @default(autoincrement()) @map("user_id")
  username     String   @unique
  email        String   @unique
  passwordHash String   @map("password_hash")
  createdAt    DateTime @default(now()) @map("created_at")
  posts        Post[]

  @@map("users")
}

model Post {
  postId    Int      @id @default(autoincrement()) @map("post_id")
  userId    Int      @map("user_id")
  caption   String?
  imageUrl  String   @map("image_url")
  createdAt DateTime @default(now()) @map("created_at")
  user      User     @relation(fields: [userId], references: [userId], onDelete: Cascade)

  @@map("posts")
}
```

Two things to notice immediately versus `schema.sql`:

- Prisma models use **camelCase** field names (`passwordHash`, `imageUrl`) by convention, while
  `@map("password_hash")` tells it the actual Postgres column is snake_case. You write idiomatic
  TS field names; Prisma handles the translation to the real column at query time.
- The `posts Post[]` / `user User` lines declare the **relation** — Prisma now understands
  `User` ↔ `Post` as a relationship, not just two unrelated tables, which is what lets you write
  `include: { user: true }` later instead of a manual `JOIN`.

### 2.5 Run the migration (this replaces `psql -f schema.sql`)

```bash
npx prisma migrate dev --name init
```

This does three things in one command: generates a timestamped SQL migration file under
`prisma/migrations/`, applies it to your `blog_api` database, and regenerates the type-safe
Prisma Client based on the schema. Compare this to Part 1, where applying `schema.sql` and
re-running it for every future change was entirely manual — `migrate dev` gives you a tracked,
re-playable migration history for free.

### 2.6 `src/db.ts` — Prisma client singleton

```ts
import { PrismaClient } from "@prisma/client";

// In dev, ts-node-dev hot-reloads modules, which can spin up a new PrismaClient (and a new DB
// connection pool) on every file save if you're not careful. Stashing it on `global` avoids that.
const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

### 2.7 `src/auth.ts` — unchanged from Part 1

Copy it over verbatim. This is the point being made explicit: nothing about JWT signing or
verifying cares whether `req.userId` eventually gets used in a `pool.query` call or a
`prisma.post.create` call.

### 2.8 `src/routes/auth.routes.ts` — same routes, Prisma Client instead of SQL strings

```ts
import { Router } from "express";
import bcrypt from "bcrypt";
import { Prisma } from "@prisma/client";
import { prisma } from "../db";
import { createToken } from "../auth";

const router = Router();

router.post("/signup", async (req, res) => {
  const { username, email, password } = req.body;
  if (!username || !email || !password) {
    return res
      .status(400)
      .json({ error: "username, email, password are required" });
  }

  const passwordHash = await bcrypt.hash(password, 10);

  try {
    const user = await prisma.user.create({
      data: { username, email, passwordHash },
      select: { userId: true, username: true, email: true }, // same "don't leak the hash" idea
    });
    const token = createToken(user.userId);
    return res.status(201).json({ user, token });
  } catch (err) {
    if (
      err instanceof Prisma.PrismaClientKnownRequestError &&
      err.code === "P2002"
    ) {
      // Prisma's equivalent of Postgres's 23505 unique_violation
      return res.status(409).json({ error: "username or email already taken" });
    }
    console.error(err);
    return res.status(500).json({ error: "Internal server error" });
  }
});

router.post("/login", async (req, res) => {
  const { username, password } = req.body;

  const user = await prisma.user.findUnique({ where: { username } });

  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ error: "Invalid username or password" });
  }

  const token = createToken(user.userId);
  return res.status(200).json({ token });
});

export default router;
```

### 2.9 `src/routes/posts.routes.ts` — same routes, Prisma Client instead of SQL strings

```ts
import { Router } from "express";
import { prisma } from "../db";
import { tokenRequired } from "../auth";

const router = Router();

router.post("/posts", tokenRequired, async (req, res) => {
  const { caption, image_url } = req.body;
  if (!image_url) {
    return res.status(400).json({ error: "image_url is required" });
  }

  const post = await prisma.post.create({
    data: { caption, imageUrl: image_url, userId: req.userId! },
  });
  return res.status(201).json(post);
});

router.get("/posts", async (req, res) => {
  const limit = parseInt((req.query.limit as string) ?? "10", 10);
  const offset = parseInt((req.query.offset as string) ?? "0", 10);

  const posts = await prisma.post.findMany({
    take: limit,
    skip: offset,
    orderBy: { createdAt: "desc" },
    include: { user: { select: { username: true } } }, // the "JOIN", expressed declaratively
  });
  return res.status(200).json(posts);
});

router.delete("/posts/:postId", tokenRequired, async (req, res) => {
  const postId = parseInt(req.params.postId, 10);

  // deleteMany (not delete) because we want "delete WHERE postId AND userId match" in one call —
  // same ownership-check-inside-the-query idea as Part 1's SQL, not a separate fetch-then-delete.
  const deleted = await prisma.post.deleteMany({
    where: { postId, userId: req.userId },
  });

  if (deleted.count === 0) {
    return res.status(404).json({ error: "Post not found or not yours" });
  }
  return res.status(204).send();
});

export default router;
```

The `req.userId!` non-null assertion on `POST /posts`: TypeScript can't statically know that
`tokenRequired` ran and set `userId` before this handler executes — that fact only exists at
runtime, via route ordering. The `!` says "trust me, the middleware guarantees this." It's safe
here specifically _because_ the route is wired behind `tokenRequired`.

### 2.10 `src/server.ts` — identical shape to Part 1

```ts
import express from "express";
import dotenv from "dotenv";
dotenv.config();

import authRoutes from "./routes/auth.routes";
import postsRoutes from "./routes/posts.routes";

const app = express();
app.use(express.json());

app.use(authRoutes);
app.use(postsRoutes);

app.get("/health", (_req, res) => res.json({ status: "ok" }));

const PORT = process.env.PORT ?? 5000;
app.listen(PORT, () =>
  console.log(`Prisma API running on http://localhost:${PORT}`)
);
```

Run it the same way: `npm run dev`. The exact same `curl` sequence from Section 1.9 works
unchanged — that's the point of building both against the same route contract.

---

## Part 3 — Raw SQL vs. Prisma: What Actually Differs

|                        | Raw SQL (`pg`)                                                             | Prisma                                                                                        |
| ---------------------- | -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Schema changes**     | Hand-write & run `.sql` files yourself                                     | `prisma migrate dev` generates + tracks + applies migrations                                  |
| **Type safety**        | None automatically — `result.rows[0]` is `any` unless you type it yourself | Full autocomplete + compile-time errors from the generated client                             |
| **Query control**      | Total — you write exactly the SQL that runs                                | High-level API; complex queries sometimes need `prisma.$queryRaw` anyway                      |
| **Joins/relations**    | Manual `JOIN` syntax                                                       | Declarative `include`/`select`, reads closer to the data shape you want back                  |
| **Learning curve**     | Need to actually know SQL                                                  | Need to learn Prisma's query API on top of still needing to know SQL for anything non-trivial |
| **Refactoring safety** | Rename a column → grep every SQL string by hand                            | Rename a field → `tsc` flags every call site that breaks                                      |
| **Performance tuning** | Easy to hand-optimize a specific slow query                                | Possible, but you sometimes fight the abstraction to get the exact query you want             |
| **N+1 risk**           | You see every query you write, so it's visible                             | Easy to accidentally trigger N+1s via relation access if you're not using `include`           |

**Practical guidance:** Prisma is usually the better default for new CRUD-heavy APIs — migration
tracking and type safety pay for themselves quickly, and you saw above the route code ends up
shorter and harder to typo. Reach for raw SQL (either in a `pg`-only project, or via
`prisma.$queryRaw` inside a Prisma project) when you hit a genuinely complex query — heavy
aggregations, window functions, recursive CTEs — where fighting an ORM's query builder costs more
time than just writing the SQL. Most real Prisma codebases end up using both: Prisma Client for
the 90% of routes that are straightforward CRUD, `$queryRaw` for the 10% that aren't.

---

## Appendix — Common Gotchas in Both Versions

- **`JWT_SECRET` must be set before the server starts**, in both parts — the `auth.ts` guard
  throws immediately on import if it's missing, which is intentional: better a crash on boot than
  silently signing tokens with `undefined`.
- **`bcrypt.compare` against a non-existent user**: both login routes call `bcrypt.compare` only
  when `user` exists, short-circuiting via `!user ||`. If you want to fully defend against
  username-enumeration timing attacks, compare against a dummy hash even when the user doesn't
  exist, so both code paths take roughly the same time — not implemented above for simplicity, but
  worth knowing as the next hardening step.
- **Token expiry (`expiresIn: "7d"`)** means `jwt.verify` starts throwing automatically after 7
  days — that's the entire mechanism, there's no separate "expiration check" code anywhere; it's
  baked into the token's `exp` claim and checked inside `jwt.verify` itself.
- **Never commit `.env`** — add it to `.gitignore` in both projects; `JWT_SECRET` and
  `DATABASE_URL` are credentials.
