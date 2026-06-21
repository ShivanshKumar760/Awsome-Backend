# Replit Clone — Microservices Architecture

A self-hosted Replit-like platform built with Node.js microservices, Docker, RabbitMQ, and WebSockets. Each project gets its own isolated Docker container with a live terminal accessible over WebSocket.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Services](#services)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Setup & Running](#setup--running)
- [API Reference](#api-reference)
- [WebSocket Terminal](#websocket-terminal)
- [Docker Communication](#docker-communication)
- [Volume Mount Chain](#volume-mount-chain)
- [Common Errors & Fixes](#common-errors--fixes)
- [Environment Variables](#environment-variables)

---

## Architecture Overview

```
Client (curl / wscat / Postman)
        │
        ├── REST  ──► auth-service     :3001  ──► PostgreSQL :5432
        ├── REST  ──► project-service  :3002  ──► PostgreSQL :5432
        ├── REST  ──► execution-service :3003 ──► RabbitMQ   :5672
        └── WS    ──► execution-service :4000 ──► container.exec() → /bin/sh
                                                        │
                              RabbitMQ "execution" queue│
                                                        ▼
                                               sandbox-worker
                                                        │
                                               docker run --volumes-from
                                                        │
                                                        ▼
                                           repl-<projectId>  (mini-replit-node)
                                           /workspace = /workspaces/<projectId>
```

---

## Services

### auth-service (`:3001`)

Handles user registration and login. Issues signed JWT tokens used by all other services.

- `POST /register` — create account
- `POST /login` — returns JWT

### project-service (`:3002`)

Creates projects and scaffolds initial files into the shared `/workspaces` volume.

- `POST /create` — creates a project directory with `index.js`, `package.json`, and `flake.nix`

### execution-service (`:3003` + `:4000`)

Two responsibilities:

1. **REST** — receives `POST /run/:id`, pushes `{ projectId }` to RabbitMQ `execution` queue
2. **WebSocket** — accepts connections on `:4000`, verifies JWT, then calls `container.exec()` to open an interactive shell (`/bin/sh`) inside the project's running container

### sandbox-worker (internal)

Consumes messages from the RabbitMQ `execution` queue. For each message it:

1. Checks the project path exists in `/workspaces/<projectId>`
2. Removes any old container with the same name
3. Runs `docker run --volumes-from sandbox-worker mini-replit-node`

---

## Tech Stack

| Layer            | Technology             |
| ---------------- | ---------------------- |
| Runtime          | Node.js 18 (ESM)       |
| HTTP             | Express.js             |
| Message Queue    | RabbitMQ 3 (amqplib)   |
| WebSocket        | ws                     |
| Database         | PostgreSQL 15 (pg)     |
| Containerization | Docker + Dockerode     |
| Sandbox Image    | nixos/nix + Node.js 18 |
| Orchestration    | Docker Compose v3.9    |
| Auth             | JWT (jsonwebtoken)     |

---

## Project Structure

```
replit/
├── docker-compose.yml
├── .env
├── shared/
│   ├── db.js              # pg Pool
│   ├── authMiddleware.js  # JWT middleware
│   └── jwt.js             # sign / verify
├── auth-service/
│   ├── dockerfile
│   ├── package.json
│   └── index.js
├── project-service/
│   ├── dockerfile
│   ├── package.json
│   └── index.js
├── execution-service/
│   ├── dockerfile
│   ├── package.json
│   └── index.js
├── sandbox-worker/
│   ├── dockerfile
│   ├── package.json
│   └── index.js
├── docker/
│   └── dockerfile         # mini-replit-node image
└── workspaces/            # one folder per project
    └── <projectId>/
        ├── index.js
        ├── package.json
        └── flake.nix
```

---

## Setup & Running

### Prerequisites

- Docker Desktop
- Node.js 18+
- Build the sandbox image first:

```bash
cd docker
docker build -t mini-replit-node .
```

### Start all services

```bash
docker-compose up --build
```

### First run note

All services wait for their dependencies via healthchecks:

- `auth-service` and `project-service` wait for **postgres** to be healthy
- `execution-service` and `sandbox-worker` wait for **rabbitmq** to be healthy

---

## API Reference

### Register

```bash
curl -X POST http://localhost:3001/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"password123"}'
```

### Login

```bash
curl -X POST http://localhost:3001/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"password123"}'
# returns { token: "eyJ..." }
```

### Create Project

```bash
curl -X POST http://localhost:3002/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"name":"my-app","dependencies":{"express":"^4.18.0"}}'
# returns { projectId: "uuid" }
```

### Run Project

```bash
curl -X POST http://localhost:3003/run/<PROJECT_ID> \
  -H "Authorization: Bearer <TOKEN>"
# returns { message: "Event queued" }
```

### Get Container ID

```bash
docker ps | grep repl-<PROJECT_ID>
# copy the container ID from output
```

---

## WebSocket Terminal

Connect using **wscat**:

```bash
npm install -g wscat

wscat -c "ws://localhost:4000/?token=<TOKEN>&id=<CONTAINER_ID>"
```

Once connected you have a full interactive shell:

```
Connected (press CTRL+C to quit)
> ls
index.js  node_modules  package.json  flake.nix
> node index.js
Server running on port 3000
> cat package.json
{ "name": "my-app", ... }
```

### How it works

1. Client connects to WebSocket on `:4000` with `?token=<JWT>&id=<containerId>`
2. `execution-service` verifies the JWT
3. Calls `container.exec({ Tty: true, Cmd: ["/bin/sh"] })`
4. Bidirectional stream: keystrokes → container stdin, container stdout → WebSocket

---

## Docker Communication

### Docker-in-Docker (DinD) via Socket

Both `sandbox-worker` and `execution-service` mount the Docker socket:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

This allows them to communicate with the **host Docker daemon** directly — spawning and exec-ing into containers without needing a nested Docker daemon.

### Why `--volumes-from sandbox-worker` instead of `-v /host/path`

This is the most important design decision in the whole project. Here's the full reasoning:

#### The Problem with `-v /path:/workspace`

`sandbox-worker` spawns containers by calling the **host Docker daemon** via `/var/run/docker.sock`. This means when it runs:

```bash
docker run -v /workspaces/<id>:/workspace mini-replit-node
```

The path `/workspaces/<id>` is a path **inside the sandbox-worker container** — but Docker always resolves volume paths on the **host machine**, not inside whichever container issued the command. The host has no directory called `/workspaces/<id>` so the mount silently fails or produces an empty directory.

#### On Windows It Gets Even Worse

The actual host path looks like:

```
D:/coding/development/Web Development/100x/100xBackendProject/08.replitV2/replit/workspaces/<id>
```

Two additional problems:

1. **Spaces in the path** (`Web Development`, `100x`) break shell command parsing — the path splits into multiple arguments
2. **Windows drive letters** (`D:/`) are meaningless inside Linux containers

We tried passing the host path via an environment variable but hit both issues immediately.

#### The Fix: `--volumes-from sandbox-worker`

Instead of specifying a path at all, we tell Docker:

> _"Give this new container the exact same volume mounts that `sandbox-worker` already has."_

```bash
docker run --volumes-from sandbox-worker mini-replit-node
```

Docker Compose already resolved `./workspaces → /workspaces` **correctly on the host** when it started `sandbox-worker`. By using `--volumes-from`, the spawned container inherits that binding directly — we never need to know, pass, or escape the host path. This works identically on Windows, Mac, and Linux with zero code changes.

#### Why `container_name: sandbox-worker` Is Required

`--volumes-from` needs a **stable, predictable container name** to reference. Without explicitly setting it, Docker Compose auto-generates a name like `replit-sandbox-worker-1` which:

- Changes if you rename the project folder
- Could increment (`-2`, `-3`) on restarts
- Makes `--volumes-from` fail with "no such container"

Setting `container_name: sandbox-worker` in `docker-compose.yml` guarantees the name is always `sandbox-worker` regardless of context.

```yaml
sandbox-worker:
  container_name: sandbox-worker # ← fixed name, required for --volumes-from
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - ./workspaces:/workspaces
```

#### Summary

| Approach                                      | Problem                                                                 |
| --------------------------------------------- | ----------------------------------------------------------------------- |
| `-v /workspaces/<id>:/workspace`              | Path is inside container, not on host — Docker can't find it            |
| `-v D:/coding/.../workspaces/<id>:/workspace` | Spaces break shell, Windows paths don't work in Linux                   |
| `--volumes-from sandbox-worker`               | ✅ Inherits already-resolved host mount, cross-platform, no path needed |

### Why `container.exec()` instead of `container.attach()`

|               | `attach()`                      | `exec()`              |
| ------------- | ------------------------------- | --------------------- |
| Connects to   | Running process (node index.js) | New process (/bin/sh) |
| Interactive   | No                              | Yes (with Tty: true)  |
| Commands work | No                              | Yes                   |
| Use case      | Log streaming                   | Terminal access       |

`attach()` connects to the existing PID 1 process which is `node index.js` — it doesn't understand `ls` or `pwd`. `exec()` spawns a fresh shell process inside the container.

### Why `docker.modem.demuxStream()` (attach approach)

Docker's attach stream is **multiplexed** — stdout and stderr are interleaved with 8-byte headers per chunk. Without demuxing, `stream.write()` fails and output is garbled. `demuxStream` splits the stream into separate stdout/stderr channels.

---

## Volume Mount Chain

```
Host filesystem: ./workspaces/
                      │
        ┌─────────────┴──────────────┐
        │                            │
project-service              sandbox-worker
/workspaces (rw)             /workspaces (ro)
writes scaffold files        reads project files
        │                            │
        │                  docker run --volumes-from sandbox-worker
        │                            │
        └──────────────┬─────────────┘
                       │
              repl-<projectId>
              /workspaces/<id> accessible as /workspace
              runs: npm install && node index.js
```

---

## Common Errors & Fixes

### `COPY ../shared ./shared` — not found

Docker's build context cannot access parent directories. Fix: set `context: .` (project root) in `docker-compose.yml` and update COPY paths:

```dockerfile
# Before (broken)
COPY package.json ./
COPY ../shared ./shared

# After (fixed)
COPY auth-service/package.json ./
COPY shared ./shared
```

### `ERR_MODULE_NOT_FOUND: /shared/db.js`

Shared files are copied to `/app/shared`, not `/shared`. Fix imports:

```js
// Before
import db from "/shared/db.js";

// After
import db from "./shared/db.js";
```

### `SASL: client password must be a string`

The `.env` file is not being read (dotenv injecting 0 vars). Ensure `.env` exists at the project root and has:

```env
DB_USER=repl
DB_PASSWORD=repl
DB_HOST=postgres
DB_NAME=replit
```

### `ECONNREFUSED` on RabbitMQ

Services start before RabbitMQ is ready. Fix with healthcheck:

```yaml
rabbitmq:
  healthcheck:
    test: ["CMD", "rabbitmq-diagnostics", "ping"]
    interval: 10s
    retries: 5

sandbox-worker:
  depends_on:
    rabbitmq:
      condition: service_healthy
```

### `includes invalid characters for a local volume name`

Caused by passing a Windows path with spaces to `-v`. Fix: use `--volumes-from sandbox-worker` instead.

### `stream.write is not a function`

Using `container.attach()` returns a multiplexed stream. Either use `docker.modem.demuxStream()` or switch to `container.exec()` with `Tty: true`.

### `ERR_UNESCAPED_CHARACTERS`

Container ID or URL contains whitespace. Fix:

```js
const containerId = params.get("id")?.trim();
```

### Token expired — WebSocket closes immediately

JWT tokens expire after 1 hour. Re-login to get a fresh token before connecting.

---

## Environment Variables

Create a `.env` file in the project root:

```env
# Database
DB_USER=repl
DB_PASSWORD=repl
DB_HOST=postgres
DB_PORT=5432
DB_NAME=replit

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672

# JWT
JWT_SECRET=your_secret_key

# Ports
AUTH_PORT=3001
PROJECT_PORT=3002
EXECUTION_PORT=3003
WEBSOCKET_PORT=4000
```

---

## Notes

- The `mini-replit-node` image must be built manually before running `docker-compose up`
- Each project gets one container: `repl-<projectId>`. Re-running removes and recreates it
- The `sandbox-worker` has `container_name: sandbox-worker` (fixed name) so `--volumes-from` always works
- `execution-service` needs `/var/run/docker.sock` mounted to use Dockerode
- Tokens are valid for 1 hour — reconnect with a fresh token if the WebSocket refuses
