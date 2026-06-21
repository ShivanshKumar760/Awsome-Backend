# 🚀 Mini Replit Clone – Microservices Architecture (Clean Rendered Version)

---

# 📁 Final Project Structure

```
mini-replit-micro/
│
├── docker-compose.yml
├── .env
│
├── shared/
│   ├── db.js
│   ├── jwt.js
│   └── authMiddleware.js
│
├── auth-service/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
│
├── project-service/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
│
├── execution-service/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
│
├── sandbox-worker/
│   ├── Dockerfile
│   ├── package.json
│   └── worker.js
│
├── docker-runtime/
│   ├── Dockerfile
│   └── flake.nix
│
└── workspaces/
```

---

# 🌍 Root Configuration

## 📄 `.env`

```
JWT_SECRET=SUPER_SECRET
DB_USER=repl
DB_PASS=repl
DB_NAME=repl
DB_HOST=postgres
RABBITMQ_URL=amqp://rabbitmq
```

---

## 📄 `docker-compose.yml`

```
version: "3.9"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: repl
      POSTGRES_PASSWORD: repl
      POSTGRES_DB: repl
    ports:
      - "5432:5432"

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  auth-service:
    build: ./auth-service
    ports:
      - "3001:3001"
    env_file: .env
    depends_on:
      - postgres

  project-service:
    build: ./project-service
    ports:
      - "3002:3002"
    env_file: .env
    volumes:
      - ./workspaces:/workspaces
    depends_on:
      - postgres

  execution-service:
    build: ./execution-service
    ports:
      - "3003:3003"
      - "4000:4000"
    env_file: .env
    depends_on:
      - rabbitmq

  sandbox-worker:
    build: ./sandbox-worker
    env_file: .env
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./workspaces:/workspaces
    depends_on:
      - rabbitmq
```

---

# 📦 Shared Modules

## 📄 `shared/db.js`

```
import pg from "pg";

export const pool = new pg.Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
});
```

---

## 📄 `shared/jwt.js`

```
import jwt from "jsonwebtoken";

export const signToken = (payload) =>
  jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: "1d" });

export const verifyToken = (token) =>
  jwt.verify(token, process.env.JWT_SECRET);
```

---

## 📄 `shared/authMiddleware.js`

```
import { verifyToken } from "./jwt.js";

export function authMiddleware(req, res, next) {
  const header = req.headers.authorization;
  if (!header) return res.status(401).json({ error: "No token" });

  const token = header.split(" ")[1];

  try {
    const decoded = verifyToken(token);
    req.user = decoded;
    next();
  } catch {
    return res.status(401).json({ error: "Invalid token" });
  }
}
```

---

# 🔐 Auth Service

## 📄 `auth-service/server.js`

```
import express from "express";
import bcrypt from "bcrypt";
import { pool } from "../shared/db.js";
import { signToken } from "../shared/jwt.js";

const app = express();
app.use(express.json());

await pool.query(`
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE TABLE IF NOT EXISTS users(
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  email TEXT UNIQUE,
  password TEXT
);
`);

app.post("/register", async (req, res) => {
  const { email, password } = req.body;
  const hash = await bcrypt.hash(password, 10);

  const result = await pool.query(
    "INSERT INTO users(email,password) VALUES($1,$2) RETURNING id",
    [email, hash]
  );

  res.json({ userId: result.rows[0].id });
});

app.post("/login", async (req, res) => {
  const { email, password } = req.body;

  const user = await pool.query(
    "SELECT * FROM users WHERE email=$1",
    [email]
  );

  if (!user.rows.length) return res.status(401).send("Invalid");

  const match = await bcrypt.compare(password, user.rows[0].password);
  if (!match) return res.status(401).send("Invalid");

  const token = signToken({ id: user.rows[0].id });
  res.json({ token });
});

app.listen(3001, () => console.log("Auth running"));
```

---

# 📁 Project Service

## 📄 `project-service/server.js`

```
import express from "express";
import fs from "fs";
import path from "path";
import { v4 as uuid } from "uuid";
import { pool } from "../shared/db.js";
import { authMiddleware } from "../shared/authMiddleware.js";

const app = express();
app.use(express.json());

await pool.query(`
CREATE TABLE IF NOT EXISTS projects(
  id UUID PRIMARY KEY,
  user_id UUID,
  name TEXT,
  status TEXT
);
`);

const WORKSPACE = "/workspaces";

if (!fs.existsSync(WORKSPACE)) {
  fs.mkdirSync(WORKSPACE, { recursive: true });
}

app.post("/create", authMiddleware, async (req, res) => {
  const { name, dependencies = {} } = req.body;
  const id = uuid();
  const projectPath = path.join(WORKSPACE, id);

  fs.mkdirSync(projectPath, { recursive: true });

  fs.writeFileSync(
    path.join(projectPath, "package.json"),
    JSON.stringify({
      name,
      version: "1.0.0",
      type: "module",
      dependencies
    }, null, 2)
  );

  fs.writeFileSync(
    path.join(projectPath, "index.js"),
    `console.log("Project ${name} running");`
  );

  await pool.query(
    "INSERT INTO projects(id,user_id,name,status) VALUES($1,$2,$3,$4)",
    [id, req.user.id, name, "created"]
  );

  res.json({ projectId: id });
});

app.listen(3002, () => console.log("Project service running"));
```

---

# 🚀 Execution Service

## 📄 `execution-service/server.js`

```
import express from "express";
import amqp from "amqplib";
import { WebSocketServer } from "ws";
import Docker from "dockerode";
import { authMiddleware } from "../shared/authMiddleware.js";
import { verifyToken } from "../shared/jwt.js";

const app = express();
app.use(express.json());

const docker = new Docker({ socketPath: "/var/run/docker.sock" });

const conn = await amqp.connect(process.env.RABBITMQ_URL);
const channel = await conn.createChannel();
await channel.assertQueue("execution");

app.post("/run/:id", authMiddleware, async (req, res) => {
  channel.sendToQueue(
    "execution",
    Buffer.from(JSON.stringify({ projectId: req.params.id }))
  );

  res.json({ message: "Queued" });
});

const wss = new WebSocketServer({ port: 4000 });

wss.on("connection", (ws, req) => {
  const params = new URLSearchParams(req.url.split("?")[1]);
  const token = params.get("token");
  const containerId = params.get("id");

  try {
    verifyToken(token);

    const container = docker.getContainer(containerId);

    container.attach({
      stream: true,
      stdin: true,
      stdout: true,
      stderr: true
    }).then(stream => {
      ws.on("message", msg => stream.write(msg));
      stream.on("data", chunk => ws.send(chunk.toString()));
    });

  } catch {
    ws.close();
  }
});

app.listen(3003, () => console.log("Execution service running"));
```

---

# 🧠 Sandbox Worker

## 📄 `sandbox-worker/worker.js`

```
import amqp from "amqplib";
import { exec } from "child_process";
import fs from "fs";
import path from "path";

const WORKSPACE_ROOT = "/workspaces";

const conn = await amqp.connect(process.env.RABBITMQ_URL);
const channel = await conn.createChannel();
await channel.assertQueue("execution");

channel.consume("execution", async msg => {
  const { projectId } = JSON.parse(msg.content.toString());

  const projectPath = path.join(WORKSPACE_ROOT, projectId);

  if (!fs.existsSync(projectPath)) {
    console.error("Workspace does not exist for", projectId);
    channel.ack(msg);
    return;
  }

  const flakePath = path.join(projectPath, "flake.nix");

  if (!fs.existsSync(flakePath)) {
    const flakeContent = `{
  description = "Node.js Dev Environment";
  inputs = { nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11"; };
  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux";
      pkgs = import nixpkgs { inherit system; };
    in {
      devShells.${"${system}"}.default = pkgs.mkShell {
        buildInputs = [ pkgs.nodejs_18 pkgs.nodePackages.npm ];
      };
    };
}`;

    fs.writeFileSync(flakePath, flakeContent, "utf-8");
  }

  const command = `
  docker run -d \
    --name repl-${projectId} \
    -v ${projectPath}:/workspace \
    -p 0:3000 \
    --memory=256m \
    --cpus=0.5 \
    mini-replit-node \
    nix develop --command sh -c "npm install && node index.js"
  `;

  exec(command);
  channel.ack(msg);
});
```

---

# 🐳 Docker Runtime

## 📄 `docker-runtime/Dockerfile`

```
FROM nixos/nix
WORKDIR /workspace
RUN mkdir -p /etc/nix && \
    echo "experimental-features = nix-command flakes" >> /etc/nix/nix.conf
EXPOSE 3000
CMD ["nix", "develop"]
```

---

# 🚀 How To Run

1️⃣ Build runtime image

```
docker build -t mini-replit-node ./docker-runtime
```

2️⃣ Start system

```
docker-compose up --build
```

---

# 🎯 What You Now Have

✔ Auth microservice  
✔ JWT middleware  
✔ Project workspace generator  
✔ RabbitMQ execution queue  
✔ Docker sandbox worker  
✔ Nix reproducible environment  
✔ WebSocket terminal  
✔ Resource-limited containers

This is a production-style cloud IDE backend similar to Replit or GitHub Codespaces.
