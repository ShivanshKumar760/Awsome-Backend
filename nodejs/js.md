# JavaScript & TypeScript Essentials — Before You Build an API

Everything here is a building block that shows up directly in API route handlers — destructuring
`req.body`, `async`/`await` around database calls, typing an Express `Request`, narrowing a
Prisma error. Read this before (or alongside) an Express/Postgres tutorial; every example below is
deliberately written in that flavor rather than as abstract syntax.

```
Part A — JavaScript essentials (syntax you'll type constantly)
Part B — The async model: callbacks, Promises, async/await (the deep dive)
Part C — TypeScript essentials (the type layer on top of Part A)
Part D — Self-check before you start building
```

---

## Part A — JavaScript Essentials

### A.1 Variables & scope
```js
let count = 0;        // reassignable, block-scoped
const username = "x"; // not reassignable, block-scoped — default choice
var legacy = "x";     // function-scoped, hoisted oddly — avoid in new code
```
Use `const` by default; reach for `let` only when a value genuinely needs to change (a counter, an
accumulator). You'll essentially never need `var`.

### A.2 Functions — three shapes you'll see constantly
```js
// declaration — hoisted, can be called above its definition
function add(a, b) { return a + b; }

// expression — not hoisted
const subtract = function (a, b) { return a - b; };

// arrow function — shorter, and does NOT bind its own `this`
const multiply = (a, b) => a * b;
const handler = (req, res) => { res.json({ ok: true }); };
```
The `this`-binding difference matters in one common spot: inside a regular `function`, `this` is
determined by *how it's called*; an arrow function instead captures `this` from its surrounding
scope. This is why class methods used as callbacks (e.g., `setTimeout(this.method)`) often need to
be arrow functions or `.bind(this)` — outside of class-based code, you can default to arrow
functions everywhere and rarely think about it.

**Default and rest parameters** (rest shows up in real route code: `req.params` style param lists,
variadic logging helpers):
```js
function greet(name = "guest") { return `Hi, ${name}`; }
function logAll(label, ...values) { console.log(label, values); } // values is an array
```

### A.3 Destructuring — you will write this in nearly every route handler
```js
// object destructuring — this exact line appears in a signup route
const { username, email, password } = req.body;

// with renaming and a default
const { limit = 10, offset = 0 } = req.query;

// array destructuring
const [first, second] = [10, 20];

// nested
const { user: { id, profile: { bio } } } = response;
```

### A.4 Spread — copying and merging
```js
const base = { role: "user" };
const withId = { ...base, id: 5 };       // merge into a new object
const combined = [...arr1, ...arr2];     // concatenate arrays
const safeCopy = { ...originalUser };    // shallow copy, doesn't mutate the original
```

### A.5 Template literals
```js
const url = `https://api.example.com/users/${userId}/posts`;
const msg = `${user.username} created a post at ${new Date().toISOString()}`;
```

### A.6 Array methods — these replace most manual loops in API code
```js
const ids = posts.map(p => p.postId);                      // transform each item
const recent = posts.filter(p => p.createdAt > cutoff);     // keep matching items
const total = likes.reduce((sum, l) => sum + l.count, 0);   // fold into one value
const found = users.find(u => u.username === "alice");       // first match, or undefined
const exists = users.some(u => u.email === input);            // true/false, any match
const allValid = items.every(i => i.price > 0);               // true/false, all match
```
`map`/`filter`/`reduce` return **new** arrays — they don't mutate the original, which matters once
you're passing data through several functions and don't want surprise side effects.

### A.7 Optional chaining & nullish coalescing — guards you'll use on every request
```js
const bio = user?.profile?.bio;            // undefined instead of a TypeError if profile is null
const limit = parseInt(req.query.limit ?? "10", 10);  // "10" only if limit is null/undefined
const count = req.query.count || "10";     // careful: || also replaces "", 0, false — usually ?? is what you want
```
`??` only falls back on `null`/`undefined`; `||` falls back on *any* falsy value. For query-param
defaults specifically, `??` is almost always the one you actually want — `||` would incorrectly
replace a legitimately-provided `0` or `""`.

### A.8 Modules — CommonJS vs ESM
```js
// CommonJS (Node's traditional default, what plain .js files use)
const express = require("express");
module.exports = router;

// ES Modules (what TypeScript route files use via import/export)
import express from "express";
export default router;
```
TypeScript source always uses `import`/`export` syntax; whether the **compiled output** ends up as
CommonJS or native ESM `require`/`import` is controlled by `"module"` in `tsconfig.json`
(`"commonjs"` is the common default for Node APIs, as used in the API tutorials).

### A.9 Error handling basics
```js
try {
  JSON.parse(invalidJson);
} catch (err) {
  console.error("Parse failed:", err.message);
}

// throwing your own errors
function requireFields(body, fields) {
  for (const f of fields) {
    if (!body[f]) throw new Error(`Missing field: ${f}`);
  }
}

// custom error classes — useful once you want to distinguish error types
class NotFoundError extends Error {
  constructor(message) {
    super(message);
    this.name = "NotFoundError";
    this.statusCode = 404;
  }
}
```

### A.10 JSON
```js
const body = JSON.stringify({ username: "alice" }); // object -> string, for sending over HTTP
const obj = JSON.parse('{"username":"alice"}');       // string -> object
```
`express.json()` middleware does exactly this `JSON.parse` step for you on every incoming request
with a JSON body, which is why `req.body` is already a plain object by the time your route handler
runs.

---

## Part B — The Async Model: Callbacks, Promises, `async`/`await`

This is the part that matters most for API work — every database query, every `bcrypt.hash` call,
every outbound HTTP request is asynchronous. Misunderstanding this section is the single biggest
source of bugs in early API code (silently-skipped error handling, requests that hang forever,
data fetched in the wrong order).

### B.1 Why JS needs this at all

JavaScript runs on **one thread** — one call stack, executing one thing at a time. If a database
query took 50ms and JS just *waited* (blocked) for it, your entire server would be frozen for
those 50ms — no other request could be handled, even from a different user. Instead, slow
operations (DB queries, file reads, network calls) are handed off to be done **outside** the
single JS thread (by the OS / libuv in Node's case), and JS keeps running other code. When that
operation finishes, its callback gets queued up and the **event loop** picks it up once the call
stack is empty.

```
Call stack: runs your synchronous code, one frame at a time
     ↓ (when stack is empty)
Microtask queue: resolved Promises, async/await continuations — drained FIRST, fully, every time
     ↓ (when microtask queue is empty)
Macrotask/callback queue: setTimeout, I/O callbacks, etc. — one task, then back to microtasks
```
Quick proof of the ordering, worth running once to internalize:
```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
// logs: 1, 4, 3, 2 — sync code first, then ALL microtasks, then the timer
```

### B.2 Callbacks (what came before Promises — recognize the pattern, then move past it)
```js
fs.readFile("file.txt", (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});
```
The problem this caused at scale — **"callback hell"** — is nesting that grows with every
sequential async step:
```js
getUser(id, (err, user) => {
  if (err) return handleErr(err);
  getPosts(user.id, (err, posts) => {
    if (err) return handleErr(err);
    getComments(posts[0].id, (err, comments) => {
      if (err) return handleErr(err);
      // ...four levels deep and still going
    });
  });
});
```
Promises exist specifically to fix this shape.

### B.3 Promises

A **Promise** is an object representing a value that isn't ready yet. It's always in exactly one
of three states:
- **pending** — not resolved or rejected yet
- **fulfilled** — completed successfully, has a value
- **rejected** — failed, has an error/reason

**Creating one** (you'll rarely do this by hand once you're using libraries that already return
Promises — like `pg`, Prisma, `bcrypt` — but you should know what's underneath):
```js
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) return reject(new Error("ms must be positive"));
    setTimeout(() => resolve(`waited ${ms}ms`), ms);
  });
}
```

**Consuming one** with `.then()`/`.catch()`/`.finally()`:
```js
pool.query("SELECT * FROM users WHERE user_id = $1", [id])
  .then(result => console.log(result.rows[0]))
  .catch(err => console.error("Query failed:", err))
  .finally(() => console.log("Done, success or not"));
```
`.then()` callbacks **chain** — each one can return a value (or another Promise) that becomes the
input to the next `.then()`:
```js
pool.query("SELECT * FROM users WHERE user_id = $1", [id])
  .then(result => result.rows[0])
  .then(user => bcrypt.compare(inputPassword, user.password_hash))
  .then(isMatch => console.log("Password matches:", isMatch))
  .catch(err => console.error(err)); // a single .catch() at the end catches errors from ANY step above
```

**Running independent Promises in parallel** — this is where real performance gains come from:
```js
// Promise.all — waits for all, rejects immediately if ANY one rejects
const [user, postCount] = await Promise.all([
  prisma.user.findUnique({ where: { userId } }),
  prisma.post.count({ where: { userId } }),
]);

// Promise.allSettled — waits for all, NEVER short-circuits; gives you each result's status
const results = await Promise.allSettled([sendEmail(a), sendEmail(b), sendEmail(c)]);
// each result: { status: "fulfilled", value } or { status: "rejected", reason }

// Promise.race — settles as soon as the FIRST one settles (fulfilled or rejected)
const result = await Promise.race([fetchData(), timeout(5000)]); // simple request-timeout pattern

// Promise.any — settles as soon as the FIRST one fulfills; ignores rejections unless ALL reject
```
Use `Promise.all` when every operation must succeed and you want the fastest possible failure;
use `allSettled` when you want partial results even if some fail (e.g., sending notifications to
five users — one failing shouldn't stop the other four).

### B.4 `async`/`await` — syntax sugar over Promises

`async`/`await` doesn't replace Promises — it's a different, more readable way to *write* code
that's still Promises underneath. An `async function` **always returns a Promise**, even if you
`return` a plain value inside it:
```js
async function getUser(id) {
  const result = await pool.query("SELECT * FROM users WHERE user_id = $1", [id]);
  return result.rows[0]; // this is wrapped in a fulfilled Promise automatically
}

getUser(1).then(user => console.log(user)); // still a Promise from the outside
```
`await` pauses execution **of that async function only** (not the whole program) until the
Promise on its right settles — fulfilled gives you the value directly, rejected throws, which is
what makes `try`/`catch` work for async code:
```js
async function login(username, password) {
  try {
    const result = await pool.query("SELECT * FROM users WHERE username = $1", [username]);
    const user = result.rows[0];
    if (!user || !(await bcrypt.compare(password, user.password_hash))) {
      throw new Error("Invalid credentials");
    }
    return createToken(user.user_id);
  } catch (err) {
    console.error("Login failed:", err.message);
    throw err; // re-throw so the caller (e.g., the route handler) can also respond appropriately
  }
}
```

**The sequential-await trap** — the single most common async performance bug:
```js
// ❌ BAD: each await blocks the next from starting — 3 independent queries run one after another
const user = await prisma.user.findUnique({ where: { userId } });
const postCount = await prisma.post.count({ where: { userId } });
const likeCount = await prisma.like.count({ where: { userId } });
// total time ≈ sum of all three queries

// ✅ GOOD: these don't depend on each other, so start them together
const [user, postCount, likeCount] = await Promise.all([
  prisma.user.findUnique({ where: { userId } }),
  prisma.post.count({ where: { userId } }),
  prisma.like.count({ where: { userId } }),
]);
// total time ≈ the SLOWEST of the three, not the sum
```
Sequential `await`s are correct (and required) when each step genuinely **depends on** the
previous result (e.g., "look up the user, *then* use `user.userId` to fetch their posts"). They're
a bug when the steps are independent and you're paying for serial latency you don't need to.

**The same trap inside a loop** — `await` in a `for` loop runs one iteration fully before starting
the next:
```js
// ❌ runs sequentially — N round trips, one at a time
for (const id of userIds) {
  await sendWelcomeEmail(id);
}

// ✅ fires all requests concurrently, waits for all to finish
await Promise.all(userIds.map(id => sendWelcomeEmail(id)));
```

**Forgetting `await` entirely** — a quiet, common bug:
```js
function signup(req, res) {
  const passwordHash = bcrypt.hash(req.body.password, 10); // ❌ missing await!
  // passwordHash is now a pending Promise object, NOT a string —
  // it gets inserted into the DB as "[object Promise]"
}
```
TypeScript catches some of these (a `Promise<string>` won't satisfy a parameter typed `string`),
which is one more reason Part C's typing matters for async code specifically — plain JS won't warn
you at all here.

### B.5 The Express-specific gotcha: unhandled rejections in route handlers

This is the single most important async lesson for API building specifically. In **Express 4**
(still extremely common), if an `async` route handler throws or its awaited Promise rejects,
**Express does not catch it automatically** — the request simply hangs or the process can crash,
with no error response sent and no `next(err)` called.
```js
// ❌ if pool.query rejects (bad SQL, DB down, etc.), this error vanishes into the void in Express 4
app.get("/posts", async (req, res) => {
  const result = await pool.query("SELECT * FROM posts"); // throws -> nothing catches it
  res.json(result.rows);
});
```
Two fixes, both legitimate:
```js
// Option 1 — try/catch in every async handler (what the earlier API tutorials do)
app.get("/posts", async (req, res) => {
  try {
    const result = await pool.query("SELECT * FROM posts");
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Internal server error" });
  }
});

// Option 2 — a wrapper that forwards rejections to Express's error-handling middleware
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.get("/posts", asyncHandler(async (req, res) => {
  const result = await pool.query("SELECT * FROM posts"); // a rejection now reaches next(err)
  res.json(result.rows);
}));

// paired with a centralized error handler
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: "Internal server error" });
});
```
(**Express 5**, once it's what you're running, fixes this at the framework level — async handler
rejections are forwarded to error-handling middleware automatically, no wrapper needed. Worth
checking which major version a given project is on before assuming either behavior.)

### B.6 Top-level `await`
In native ES modules (`"type": "module"` in `package.json`, or `.mjs` files), `await` is legal
directly at the top of a file, outside any function — handy for one-off scripts (e.g., a seed
script) but not something you'd use inside a route handler, which is always inside an `async`
function already.

---

## Part C — TypeScript Essentials

TypeScript is JavaScript plus a type layer that's checked at compile time and then **erased
entirely** — none of it exists at runtime. Every JS concept in Part A and Part B still applies
exactly as written; TS adds annotations on top.

### C.1 Basic types
```ts
let username: string = "alice";
let age: number = 30;
let isActive: boolean = true;
let tags: string[] = ["js", "ts"];
let coords: [number, number] = [12.9, 77.6]; // tuple — fixed length, fixed types per position
```

### C.2 `any` vs `unknown` vs `never` vs `void`
```ts
let a: any;       // opts OUT of type checking entirely — avoid; defeats the point of TS
let u: unknown;    // like any, but you MUST narrow it before using it — the safe alternative
let n: never;      // a function that never returns (always throws, or infinite loop)
function log(msg: string): void { console.log(msg); } // returns nothing meaningful

function process(input: unknown) {
  if (typeof input === "string") {
    input.toUpperCase(); // fine — TS knows it's a string here, narrowed
  }
}
```
Default to `unknown` over `any` whenever a value's type genuinely isn't known yet (e.g., a
`catch (err)` block — `err` is typed `unknown` in strict TS, not `any`, on purpose).

### C.3 Interfaces vs. type aliases
```ts
interface User {
  userId: number;
  username: string;
  bio?: string;          // optional property
  readonly createdAt: Date; // can be read but not reassigned after creation
}

type UserId = number;                      // aliasing a primitive
type Result = { ok: true } | { ok: false; error: string }; // union — type aliases support this, interfaces don't directly
```
Rule of thumb: use `interface` for object shapes you expect to extend or that represent
"things" (a `User`, a request body shape); use `type` for unions, intersections, or aliasing
non-object types. Both are structurally compatible with each other in practice.

### C.4 Function typing
```ts
function add(a: number, b: number): number {
  return a + b;
}

// optional and default params, same idea as JS, now typed
function greet(name: string = "guest"): string {
  return `Hi, ${name}`;
}

// arrow function type
const multiply: (a: number, b: number) => number = (a, b) => a * b;

// typing an async function — note the return type is Promise<T>, not T
async function getUser(id: number): Promise<User | null> {
  const result = await pool.query("SELECT * FROM users WHERE user_id = $1", [id]);
  return result.rows[0] ?? null;
}
```

### C.5 Union types, literal types, and narrowing
```ts
function formatId(id: string | number): string {
  if (typeof id === "string") return id.trim();   // narrowed to string
  return id.toFixed(0);                             // narrowed to number
}

type Role = "admin" | "editor" | "viewer"; // literal union — often better than an enum (see C.6)

function isPrismaUniqueError(err: unknown): err is Prisma.PrismaClientKnownRequestError {
  return err instanceof Prisma.PrismaClientKnownRequestError && err.code === "P2002";
}
// a "type guard" function — `err is X` tells TS that after this returns true, err IS that type
```
This `instanceof` narrowing pattern is exactly what the Prisma error-handling code in API routes
relies on (`if (err instanceof Prisma.PrismaClientKnownRequestError && err.code === "P2002")`) —
TS only lets you access `.code` after the `instanceof` check has narrowed `err` away from
`unknown`.

### C.6 Enums vs. literal unions
```ts
enum Role { Admin, Editor, Viewer }       // works, but compiles to extra runtime JS

type Role = "admin" | "editor" | "viewer"; // usually preferred — zero runtime cost, same safety
```
Modern TS guidance leans toward literal unions over `enum` for simple cases like this — they type-
check identically but don't generate any actual JS object at runtime.

### C.7 Generics — writing code that works across types, without losing type safety
```ts
function firstItem<T>(arr: T[]): T | undefined {
  return arr[0];
}
firstItem<number>([1, 2, 3]);     // T = number
firstItem(["a", "b"]);             // T inferred as string, no need to specify it

// You've already been using generics: Promise<T> and Array<T> are generic types
async function findById<T>(table: string, id: number): Promise<T | null> { /* ... */ }
```
The mental model: a generic is a *type parameter* — a placeholder filled in per call-site, the
same way a normal function parameter is a *value* placeholder filled in per call.

### C.8 Classes
```ts
class ApiError extends Error {
  constructor(
    public statusCode: number,   // `public` here both declares AND assigns the field — shorthand
    message: string,
    private readonly internalCode?: string // not accessible outside this class
  ) {
    super(message);
  }
}

throw new ApiError(404, "User not found");
```
`public` / `private` / `protected` control visibility outside the class; `readonly` prevents
reassignment after the constructor runs. You'll see this pattern most in custom error classes and
in service/repository-style classes if a codebase grows past simple route handlers.

### C.9 Utility types — built-in type transformations, genuinely useful for API DTOs
```ts
interface User {
  userId: number;
  username: string;
  email: string;
  bio?: string;
}

type NewUser = Omit<User, "userId">;        // everything except userId — for a signup payload
type UserPreview = Pick<User, "userId" | "username">; // just these two fields — for a feed response
type UserUpdate = Partial<User>;             // all fields optional — for a PATCH /users/me body
type UserMap = Record<number, User>;         // an object keyed by userId, valued as User
```
These show up constantly once you start typing request bodies and response shapes instead of
leaving them as implicit `any`.

### C.10 Module augmentation — what's actually happening with `declare module "express"`
```ts
import "express";

declare module "express" {
  interface Request {
    userId?: number;
  }
}
```
This isn't defining a new type from scratch — it's **merging** new fields onto Express's existing
`Request` interface (TypeScript interfaces support "declaration merging": multiple `interface`
declarations with the same name combine into one). This is exactly the mechanism used to add
`req.userId` after the JWT middleware runs, as seen in the earlier API tutorials — it emits zero
JS, it only exists for the type checker.

### C.11 Type-only imports
```ts
import type { Request, Response } from "express"; // erased entirely at compile time
import { Router } from "express";                    // a real runtime import
```
`import type` is a hint (and in stricter configs, an enforced rule) that something is used purely
for types, never as a runtime value — keeps compiled output smaller and avoids accidentally
importing something whose only purpose was type information.

### C.12 `tsconfig.json` — the handful of options that actually matter day to day
```json
{
  "compilerOptions": {
    "target": "ES2022",       // which JS features the output can assume exist
    "module": "commonjs",      // require/module.exports output, standard for Node APIs
    "outDir": "dist",          // compiled .js goes here
    "rootDir": "src",          // your .ts source lives here
    "strict": true,             // turns on the full set of strict checks — keep this on
    "esModuleInterop": true     // lets `import express from "express"` work cleanly with CJS packages
  }
}
```
`strict: true` is the one to never turn off — it's what makes `unknown` the default for `catch`
blocks, requires you to handle `undefined`/`null` explicitly, and is responsible for most of the
"TypeScript caught a real bug" moments you'll have.

---

## Part D — Self-Check Before You Start Building

You're ready to build an Express/Postgres API once you can comfortably do all of these without
looking them up:

- [ ] Destructure values out of `req.body` / `req.query` / `req.params`, with defaults via `??`
- [ ] Explain, in one sentence, the difference between `.then()/.catch()` and `async`/`await` (and
      that they're the same underlying mechanism)
- [ ] Identify when two `await` calls should be sequential vs. wrapped in `Promise.all`
- [ ] Explain why a rejected Promise inside an `async` Express 4 route handler can hang a request
      if you forget `try`/`catch` (or an `asyncHandler` wrapper)
- [ ] Write a `try`/`catch` around an `await`ed call and re-throw or respond on failure
- [ ] Read an `interface` and know which fields are optional (`?`) vs required
- [ ] Narrow an `unknown`/union-typed value with `typeof`, `instanceof`, or a custom type guard
      before using type-specific properties on it
- [ ] Explain what `declare module "express" { interface Request { ... } }` is doing and why it
      doesn't produce any runtime JavaScript

If any of those feel shaky, re-read that section above with a real editor open and actually type
the examples — async/await especially is something that clicks from writing it, not from reading
about it.