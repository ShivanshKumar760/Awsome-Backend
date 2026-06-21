# Build Notes: "Replit for Node.js" — Full System in Express + TypeScript (Auth, Projects, Shared PVC, Neon Postgres, K8s Executor Pods)

This is the Node.js/Express/TypeScript port of the Flask version — same architecture, same security model (JWT auth, shared `subPath`-isolated PVC, Neon Postgres, smoke-test pods vs. persistent `Deployment`+`Service`), different stack.

---

## 1. Architecture Overview

```
                        ┌─────────────────────────────┐
   Browser/Client  ───► │  Main API (Express+TS, k8s)  │
                        │   - /auth/signup, /auth/login│
                        │   - /projects (CRUD)         │
                        │   - /projects/<id>/run         │  (smoke test, self-terminates)
                        │   - /projects/<id>/deploy      │  (persistent, stays up)
                        │   - /projects/<id>/stop        │  (tears persistent one down)
                        └───────────┬─────────┬─────────┘
                                    │         │
                         (metadata) │         │ (creates/deletes Deployments,
                                    ▼         │  Services, and Pods via k8s API)
                        ┌──────────────────┐   │
                        │  Neon Postgres   │   │
                        │  users, projects │   │
                        └──────────────────┘   ▼
                                    ┌──────────────────────────────┐
                                    │  Per-project Deployment +     │
                                    │  Service (created on "deploy")│
                                    │  npm install                  │
                                    │  npx ts-node api.ts (stays up,│
                                    │   self-heals on crash)        │
                                    └────────────┬───────────────────┘
                                                 │
                              both mount ──┐     │
                                           ▼     ▼
                              ┌──────────────────────────────┐
                              │   Shared PVC (ReadWriteMany)  │
                              │  /projects/<project_id>/      │
                              │    ├── api.ts                 │
                              │    └── package.json            │
                              └──────────────────────────────┘
```

**Key design decision (unchanged from the Flask version):** the Main API mounts the PVC at its **root**; each executor pod mounts the **same PVC**, scoped with `subPath: projects/<project_id>` — so a tenant can only ever see its own project folder.

---

## 2. Tech Stack

| Layer                     | Choice                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Web framework             | Express                                                                                                                                  |
| Language                  | TypeScript                                                                                                                               |
| Auth                      | `jsonwebtoken` (JWT) + `bcryptjs` (password hashing — pure JS, no native build step needed in slim containers)                           |
| ORM                       | Prisma (for the platform's own `users`/`projects` tables)                                                                                |
| Database                  | Postgres via **Neon**                                                                                                                    |
| Container orchestration   | `@kubernetes/client-node`                                                                                                                |
| Generated project's stack | Express + TypeScript + `pg` (kept simple/raw SQL, run via `ts-node` — no separate compile step needed for the user's own scaffolded app) |
| Process runner (main API) | Compiled with `tsc`, run with plain `node` in production                                                                                 |

---

## 3. Database Schema (Neon Postgres)

Identical to the Flask version — Postgres doesn't care what language is querying it:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at    TIMESTAMP DEFAULT NOW()
);

CREATE TABLE projects (
    id           VARCHAR(80) PRIMARY KEY,
    user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    project_name VARCHAR(100) NOT NULL,
    folder_path  VARCHAR(255) NOT NULL,
    created_at   TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_projects_user_id ON projects(user_id);
```

We'll let **Prisma** own this schema going forward (Section 9) — `prisma migrate dev` generates and applies the equivalent SQL for you, so you don't have to hand-run the above against Neon.

---

## 4. Project Structure (Main API codebase)

```
main-api/
├── src/
│   ├── app.ts
│   ├── config.ts
│   ├── db.ts
│   ├── utils.ts
│   ├── middleware/
│   │   └── auth.ts
│   ├── auth/
│   │   └── routes.ts
│   └── projects/
│       ├── routes.ts
│       ├── templates.ts
│       ├── executor.ts
│       └── deployer.ts
├── prisma/
│   └── schema.prisma
├── k8s/
│   ├── pvc.yaml
│   ├── deployment.yaml
│   ├── rbac.yaml
│   └── networkpolicy.yaml
├── Dockerfile
├── package.json
├── tsconfig.json
└── .env.example
```

---

## 5. `.env.example`

```
NEON_DATABASE_URL=postgresql://user:password@ep-xxxx-pooler.us-east-2.aws.neon.tech/dbname?sslmode=require
JWT_SECRET_KEY=replace-with-a-long-random-secret
PVC_MOUNT_PATH=/data
PVC_CLAIM_NAME=platform-projects-pvc
K8S_EXECUTOR_NAMESPACE=executor
PORT=5000
```

---

## 6. `package.json` (Main API)

```json
{
  "name": "replit-for-express-main-api",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "ts-node src/app.ts",
    "build": "tsc",
    "start": "node dist/app.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "express": "^4.19.2",
    "cors": "^2.8.5",
    "jsonwebtoken": "^9.0.2",
    "bcryptjs": "^2.4.3",
    "@prisma/client": "^5.18.0",
    "@kubernetes/client-node": "^0.21.0",
    "dotenv": "^16.4.5"
  },
  "devDependencies": {
    "typescript": "^5.5.4",
    "ts-node": "^10.9.2",
    "prisma": "^5.18.0",
    "@types/express": "^4.17.21",
    "@types/node": "^20.14.0",
    "@types/cors": "^2.8.17",
    "@types/jsonwebtoken": "^9.0.6",
    "@types/bcryptjs": "^2.4.6"
  }
}
```

---

## 7. `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*.ts"]
}
```

---

## 8. `src/config.ts`

```typescript
import dotenv from "dotenv";
dotenv.config();

export const config = {
  jwtSecret: process.env.JWT_SECRET_KEY as string,
  jwtExpiresIn: "12h",
  pvcMountPath: process.env.PVC_MOUNT_PATH || "/data",
  pvcClaimName: process.env.PVC_CLAIM_NAME || "platform-projects-pvc",
  k8sExecutorNamespace: process.env.K8S_EXECUTOR_NAMESPACE || "executor",
  port: process.env.PORT ? parseInt(process.env.PORT) : 5000,
};

if (!config.jwtSecret) {
  throw new Error("JWT_SECRET_KEY is not set");
}
```

---

## 9. `prisma/schema.prisma` + `src/db.ts`

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("NEON_DATABASE_URL")
}

model User {
  id           String    @id @default(uuid())
  email        String    @unique
  passwordHash String    @map("password_hash")
  createdAt    DateTime  @default(now()) @map("created_at")
  projects     Project[]

  @@map("users")
}

model Project {
  id          String   @id
  userId      String   @map("user_id")
  projectName String   @map("project_name")
  folderPath  String   @map("folder_path")
  createdAt   DateTime @default(now()) @map("created_at")
  owner       User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("projects")
}
```

```typescript
// src/db.ts
import { PrismaClient } from "@prisma/client";

export const prisma = new PrismaClient();
```

Run `npx prisma migrate dev --name init` once against your Neon database to create the tables — Prisma generates the SQL for you from the schema above (equivalent to Section 3's DDL).

---

## 10. `src/middleware/auth.ts` — JWT verification (Express's equivalent of `@jwt_required()`)

```typescript
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { config } from "../config";

export interface AuthRequest extends Request {
  userId?: string;
}

export function authenticateToken(
  req: AuthRequest,
  res: Response,
  next: NextFunction
) {
  const authHeader = req.headers["authorization"];
  const token = authHeader && authHeader.split(" ")[1];

  if (!token) {
    return res.status(401).json({ error: "Missing token" });
  }

  jwt.verify(token, config.jwtSecret, (err, decoded) => {
    if (err || !decoded) {
      return res.status(401).json({ error: "Invalid or expired token" });
    }
    req.userId = (decoded as { userId: string }).userId;
    next();
  });
}
```

---

## 11. `src/utils.ts` — slug helper (keeps names filesystem- and Kubernetes-safe)

```typescript
import { randomBytes } from "crypto";

export function slugify(text: string, maxLen = 40): string {
  let s = text.toLowerCase().trim();
  s = s.replace(/[^a-z0-9-]/g, "-");
  s = s.replace(/-+/g, "-").replace(/^-|-$/g, "");
  return s.slice(0, maxLen) || "project";
}

export function makeProjectId(userId: string, projectName: string): string {
  const userPart = slugify(userId, 8);
  const namePart = slugify(projectName, 30);
  const suffix = randomBytes(4).toString("hex"); // 8 hex chars
  return `${userPart}-${namePart}-${suffix}`;
}
```

---

## 12. `src/auth/routes.ts`

```typescript
import { Router } from "express";
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
import { prisma } from "../db";
import { config } from "../config";

export const authRouter = Router();

authRouter.post("/signup", async (req, res) => {
  const email = (req.body.email || "").trim().toLowerCase();
  const password = req.body.password || "";

  if (!email || !password) {
    return res.status(400).json({ error: "email and password are required" });
  }
  if (password.length < 8) {
    return res
      .status(400)
      .json({ error: "password must be at least 8 characters" });
  }

  const existing = await prisma.user.findUnique({ where: { email } });
  if (existing) {
    return res
      .status(409)
      .json({ error: "An account with that email already exists" });
  }

  const passwordHash = await bcrypt.hash(password, 10);
  const user = await prisma.user.create({ data: { email, passwordHash } });

  const token = jwt.sign({ userId: user.id }, config.jwtSecret, {
    expiresIn: config.jwtExpiresIn,
  });
  res
    .status(201)
    .json({ access_token: token, user: { id: user.id, email: user.email } });
});

authRouter.post("/login", async (req, res) => {
  const email = (req.body.email || "").trim().toLowerCase();
  const password = req.body.password || "";

  const user = await prisma.user.findUnique({ where: { email } });
  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ error: "Invalid email or password" });
  }

  const token = jwt.sign({ userId: user.id }, config.jwtSecret, {
    expiresIn: config.jwtExpiresIn,
  });
  res
    .status(200)
    .json({ access_token: token, user: { id: user.id, email: user.email } });
});
```

---

## 13. `src/projects/templates.ts` — generates the starter files for every new project

```typescript
export function generatePackageJson(): string {
  return JSON.stringify(
    {
      name: "my-express-api",
      version: "1.0.0",
      main: "api.ts",
      scripts: {
        start: "ts-node api.ts",
      },
      dependencies: {
        express: "^4.19.2",
        cors: "^2.8.5",
        pg: "^8.12.0",
        dotenv: "^16.4.5",
      },
      devDependencies: {
        typescript: "^5.5.4",
        "ts-node": "^10.9.2",
        "@types/express": "^4.17.21",
        "@types/node": "^20.14.0",
        "@types/pg": "^8.11.6",
        "@types/cors": "^2.8.17",
      },
    },
    null,
    2
  );
}

export function generateApiTs(): string {
  return `import express from "express";
import cors from "cors";
import { Pool } from "pg";

const app = express();
app.use(express.json());
app.use(cors());

const pool = new Pool({
  connectionString: process.env.DATABASE_URL, // injected by the platform at run time
  ssl: { rejectUnauthorized: false },
});

async function initDb() {
  await pool.query(\`
    CREATE TABLE IF NOT EXISTS items (
      id SERIAL PRIMARY KEY,
      name TEXT NOT NULL,
      description TEXT,
      created_at TIMESTAMP DEFAULT NOW()
    );
  \`);
}

app.get("/items", async (req, res) => {
  const result = await pool.query("SELECT * FROM items ORDER BY id;");
  res.json(result.rows);
});

app.post("/items", async (req, res) => {
  const { name, description } = req.body;
  if (!name) return res.status(400).json({ error: "name is required" });
  const result = await pool.query(
    "INSERT INTO items (name, description) VALUES ($1, $2) RETURNING *;",
    [name, description ?? null]
  );
  res.status(201).json(result.rows[0]);
});

app.get("/items/:id", async (req, res) => {
  const result = await pool.query("SELECT * FROM items WHERE id = $1;", [req.params.id]);
  if (result.rows.length === 0) return res.status(404).json({ error: "Not found" });
  res.json(result.rows[0]);
});

app.put("/items/:id", async (req, res) => {
  const { name, description } = req.body;
  const result = await pool.query(
    "UPDATE items SET name = $1, description = $2 WHERE id = $3 RETURNING *;",
    [name, description, req.params.id]
  );
  if (result.rows.length === 0) return res.status(404).json({ error: "Not found" });
  res.json(result.rows[0]);
});

app.delete("/items/:id", async (req, res) => {
  const result = await pool.query("DELETE FROM items WHERE id = $1;", [req.params.id]);
  if (result.rowCount === 0) return res.status(404).json({ error: "Not found" });
  res.status(204).send();
});

const PORT = process.env.PORT ? parseInt(process.env.PORT) : 8080;
initDb().then(() => {
  app.listen(PORT, "0.0.0.0", () => console.log(\`Listening on port \${PORT}\`));
});
`;
}
```

This is intentionally a working, generic CRUD scaffold (`/items`) — same as the Flask version's template, just in Express/TS. The user edits it into whatever their actual API needs to be.

---

## 14. `src/projects/routes.ts` — create / list / get / edit / delete

```typescript
import { Router } from "express";
import fs from "fs/promises";
import path from "path";
import { prisma } from "../db";
import { config } from "../config";
import { makeProjectId } from "../utils";
import { generatePackageJson, generateApiTs } from "./templates";
import { AuthRequest, authenticateToken } from "../middleware/auth";

export const projectsRouter = Router();
projectsRouter.use(authenticateToken);

const EDITABLE_FILES = ["api.ts", "package.json"];

async function getOwnedProjectOrRespond(
  projectId: string,
  userId: string,
  res: any
) {
  const project = await prisma.project.findUnique({ where: { id: projectId } });
  if (!project) {
    res.status(404).json({ error: "Project not found" });
    return null;
  }
  if (project.userId !== userId) {
    res.status(403).json({ error: "Forbidden" });
    return null;
  }
  return project;
}

projectsRouter.post("/", async (req: AuthRequest, res) => {
  const userId = req.userId!;
  const projectName = (req.body.project_name || "").trim();
  if (!projectName) {
    return res.status(400).json({ error: "project_name is required" });
  }

  const projectId = makeProjectId(userId, projectName);
  const folderPath = `projects/${projectId}`;
  const fullPath = path.join(config.pvcMountPath, folderPath);

  await fs.mkdir(fullPath, { recursive: true });

  const packageJsonContent = generatePackageJson();
  const apiTsContent = generateApiTs();

  await fs.writeFile(path.join(fullPath, "package.json"), packageJsonContent);
  await fs.writeFile(path.join(fullPath, "api.ts"), apiTsContent);

  const project = await prisma.project.create({
    data: { id: projectId, userId, projectName, folderPath },
  });

  res.status(201).json({
    ...project,
    files: {
      "package.json": packageJsonContent,
      "api.ts": apiTsContent,
    },
  });
});

projectsRouter.get("/", async (req: AuthRequest, res) => {
  const projects = await prisma.project.findMany({
    where: { userId: req.userId! },
    orderBy: { createdAt: "desc" },
  });
  res.json(projects);
});

projectsRouter.get("/:projectId", async (req: AuthRequest, res) => {
  const project = await getOwnedProjectOrRespond(
    req.params.projectId,
    req.userId!,
    res
  );
  if (!project) return;
  res.json(project);
});

projectsRouter.get(
  "/:projectId/files/:filename",
  async (req: AuthRequest, res) => {
    const project = await getOwnedProjectOrRespond(
      req.params.projectId,
      req.userId!,
      res
    );
    if (!project) return;
    if (!EDITABLE_FILES.includes(req.params.filename)) {
      return res.status(400).json({ error: "Unsupported file" });
    }

    const fullPath = path.join(
      config.pvcMountPath,
      project.folderPath,
      req.params.filename
    );
    try {
      const content = await fs.readFile(fullPath, "utf-8");
      res.json({ filename: req.params.filename, content });
    } catch {
      res.status(404).json({ error: "File not found" });
    }
  }
);

projectsRouter.put(
  "/:projectId/files/:filename",
  async (req: AuthRequest, res) => {
    const project = await getOwnedProjectOrRespond(
      req.params.projectId,
      req.userId!,
      res
    );
    if (!project) return;
    if (!EDITABLE_FILES.includes(req.params.filename)) {
      return res.status(400).json({ error: "Unsupported file" });
    }

    const { content } = req.body;
    if (content === undefined) {
      return res.status(400).json({ error: "content is required" });
    }

    const fullPath = path.join(
      config.pvcMountPath,
      project.folderPath,
      req.params.filename
    );
    await fs.writeFile(fullPath, content);
    res.json({ message: "Saved" });
  }
);

projectsRouter.delete("/:projectId", async (req: AuthRequest, res) => {
  const project = await getOwnedProjectOrRespond(
    req.params.projectId,
    req.userId!,
    res
  );
  if (!project) return;

  const fullPath = path.join(config.pvcMountPath, project.folderPath);
  await fs.rm(fullPath, { recursive: true, force: true });
  await prisma.project.delete({ where: { id: project.id } });
  res.status(204).send();
});
```

---

## 15. `src/projects/executor.ts` — quick smoke-test run (optional, NOT for hosting)

Same self-terminating-by-design pattern as the Flask version: `timeout 20` force-kills the process, the pod has `restartPolicy: Never`, and the loop waits for `Succeeded`/`Failed`. Use **Section 16** if you want the backend to actually stay running.

```typescript
import { Router } from "express";
import * as k8s from "@kubernetes/client-node";
import { prisma } from "../db";
import { config } from "../config";
import { AuthRequest, authenticateToken } from "../middleware/auth";

export const executorRouter = Router();
executorRouter.use(authenticateToken);

const kc = new k8s.KubeConfig();
try {
  kc.loadFromCluster();
} catch {
  kc.loadFromDefault();
}
const coreV1Api = kc.makeApiClient(k8s.CoreV1Api);

executorRouter.post("/:projectId/run", async (req: AuthRequest, res) => {
  const userId = req.userId!;
  const project = await prisma.project.findUnique({
    where: { id: req.params.projectId },
  });
  if (!project) return res.status(404).json({ error: "Project not found" });
  if (project.userId !== userId)
    return res.status(403).json({ error: "Forbidden" });

  const namespace = config.k8sExecutorNamespace;
  const podName = `run-${project.id}-${Date.now()}`
    .slice(0, 63)
    .replace(/-+$/, "");

  const podManifest: k8s.V1Pod = {
    apiVersion: "v1",
    kind: "Pod",
    metadata: {
      name: podName,
      labels: { app: "executor", project: project.id },
    },
    spec: {
      restartPolicy: "Never",
      automountServiceAccountToken: false,
      volumes: [
        {
          name: "project-storage",
          persistentVolumeClaim: { claimName: config.pvcClaimName },
        },
      ],
      containers: [
        {
          name: "node-runner",
          image: "node:20-slim",
          workingDir: "/app",
          // npm install dominates this 20s budget on a cold run — see Section 21 for a note on this.
          command: [
            "sh",
            "-c",
            "npm install --no-audit --no-fund && timeout 20 npx ts-node api.ts",
          ],
          env: [
            {
              name: "DATABASE_URL",
              valueFrom: {
                secretKeyRef: { name: "neon-credentials", key: "database_url" },
              },
            },
          ],
          volumeMounts: [
            {
              name: "project-storage",
              mountPath: "/app",
              subPath: project.folderPath,
            },
          ],
          resources: {
            limits: { memory: "256Mi", cpu: "500m" },
            requests: { memory: "128Mi", cpu: "100m" },
          },
          securityContext: {
            allowPrivilegeEscalation: false,
            readOnlyRootFilesystem: false, // npm install needs to write node_modules
            runAsNonRoot: true,
            runAsUser: 1000,
          },
        },
      ],
    },
  };

  try {
    await coreV1Api.createNamespacedPod(namespace, podManifest);

    const timeoutMs = 30000; // a little above the in-container `timeout 20` for scheduling + install time
    const start = Date.now();
    let completed = false;

    while (Date.now() - start < timeoutMs) {
      const { body } = await coreV1Api.readNamespacedPodStatus(
        podName,
        namespace
      );
      const phase = body.status?.phase;
      if (phase === "Succeeded" || phase === "Failed") {
        completed = true;
        break;
      }
      await new Promise((r) => setTimeout(r, 500));
    }

    if (!completed) {
      return res
        .status(400)
        .json({
          error: "Execution timeout",
          output: "Run took too long (max 30s).",
        });
    }

    const { body: logs } = await coreV1Api.readNamespacedPodLog(
      podName,
      namespace
    );
    const { body: statusBody } = await coreV1Api.readNamespacedPodStatus(
      podName,
      namespace
    );
    const exitCode =
      statusBody.status?.containerStatuses?.[0]?.state?.terminated?.exitCode ??
      null;

    res.json({ exit_code: exitCode, output: logs });
  } catch (err: any) {
    res.status(500).json({ error: err.body?.message || err.message });
  } finally {
    try {
      await coreV1Api.deleteNamespacedPod(podName, namespace);
    } catch {
      /* ignore */
    }
  }
});
```

> ⚠️ **`@kubernetes/client-node` version gotcha:** the shape of these calls (whether responses come back as `{ body }` or directly, how many positional arguments methods like `listNamespacedPod` take) has changed across major versions of this package. The code above matches the widely-used `0.18.x`–`0.20.x` API. Before wiring this up, check the TypeScript types for the exact version you've installed (`node_modules/@kubernetes/client-node`) — your editor's autocomplete will show you the real signature.

---

## 16. `src/projects/deployer.ts` — deploy a project as a persistent service (the real "Run" button)

```typescript
import { Router } from "express";
import * as k8s from "@kubernetes/client-node";
import { prisma } from "../db";
import { config } from "../config";
import { AuthRequest, authenticateToken } from "../middleware/auth";

export const deployerRouter = Router();
deployerRouter.use(authenticateToken);

const kc = new k8s.KubeConfig();
try {
  kc.loadFromCluster();
} catch {
  kc.loadFromDefault();
}
const coreV1Api = kc.makeApiClient(k8s.CoreV1Api);
const appsV1Api = kc.makeApiClient(k8s.AppsV1Api);

const APP_PORT = 8080; // must match the port api.ts listens on (Section 13's template)

function names(projectId: string) {
  const base = projectId.slice(0, 50);
  return { deploymentName: `deploy-${base}`, serviceName: `svc-${base}` };
}

function buildDeploymentManifest(
  project: { id: string; folderPath: string },
  deploymentName: string
): k8s.V1Deployment {
  return {
    apiVersion: "apps/v1",
    kind: "Deployment",
    metadata: {
      name: deploymentName,
      labels: { app: "executor", project: project.id },
    },
    spec: {
      replicas: 1,
      selector: { matchLabels: { project: project.id } },
      template: {
        metadata: {
          labels: { app: "executor", project: project.id },
          // changing this on every deploy call forces a rollout, so saved file edits get picked up
          annotations: { "platform/deployed-at": String(Date.now()) },
        },
        spec: {
          automountServiceAccountToken: false,
          volumes: [
            {
              name: "project-storage",
              persistentVolumeClaim: { claimName: config.pvcClaimName },
            },
          ],
          containers: [
            {
              name: "node-runner",
              image: "node:20-slim",
              workingDir: "/app",
              // NOTE: no `timeout` wrapper — restartPolicy is implicitly "Always" for Deployment-managed
              // pods, so if api.ts crashes, Kubernetes restarts the container automatically.
              command: [
                "sh",
                "-c",
                "npm install --no-audit --no-fund && npx ts-node api.ts",
              ],
              ports: [{ containerPort: APP_PORT }],
              env: [
                {
                  name: "DATABASE_URL",
                  valueFrom: {
                    secretKeyRef: {
                      name: "neon-credentials",
                      key: "database_url",
                    },
                  },
                },
              ],
              volumeMounts: [
                {
                  name: "project-storage",
                  mountPath: "/app",
                  subPath: project.folderPath,
                },
              ],
              resources: {
                limits: { memory: "256Mi", cpu: "500m" },
                requests: { memory: "128Mi", cpu: "100m" },
              },
              securityContext: {
                allowPrivilegeEscalation: false,
                readOnlyRootFilesystem: false,
                runAsNonRoot: true,
                runAsUser: 1000,
              },
              readinessProbe: {
                tcpSocket: { port: APP_PORT },
                initialDelaySeconds: 5,
                periodSeconds: 5,
              },
            },
          ],
        },
      },
    },
  };
}

deployerRouter.post("/:projectId/deploy", async (req: AuthRequest, res) => {
  const userId = req.userId!;
  const project = await prisma.project.findUnique({
    where: { id: req.params.projectId },
  });
  if (!project) return res.status(404).json({ error: "Project not found" });
  if (project.userId !== userId)
    return res.status(403).json({ error: "Forbidden" });

  const namespace = config.k8sExecutorNamespace;
  const { deploymentName, serviceName } = names(project.id);
  const manifest = buildDeploymentManifest(project, deploymentName);

  let action = "deployed";
  try {
    await appsV1Api.readNamespacedDeployment(deploymentName, namespace);
    const patchOptions = {
      headers: {
        "Content-type": k8s.PatchUtils.PATCH_FORMAT_STRATEGIC_MERGE_PATCH,
      },
    };
    await appsV1Api.patchNamespacedDeployment(
      deploymentName,
      namespace,
      manifest,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      patchOptions
    );
    action = "redeployed";
  } catch (err: any) {
    const statusCode = err.statusCode || err.response?.statusCode;
    if (statusCode !== 404) {
      return res.status(500).json({ error: err.body?.message || err.message });
    }
    await appsV1Api.createNamespacedDeployment(namespace, manifest);
  }

  try {
    await coreV1Api.readNamespacedService(serviceName, namespace);
  } catch (err: any) {
    const statusCode = err.statusCode || err.response?.statusCode;
    if (statusCode !== 404) {
      return res.status(500).json({ error: err.body?.message || err.message });
    }
    const serviceManifest: k8s.V1Service = {
      apiVersion: "v1",
      kind: "Service",
      metadata: { name: serviceName, labels: { project: project.id } },
      spec: {
        selector: { project: project.id },
        ports: [{ port: 80, targetPort: APP_PORT as any }],
        type: "ClusterIP",
      },
    };
    await coreV1Api.createNamespacedService(namespace, serviceManifest);
  }

  res.status(202).json({
    message: action,
    internal_url: `http://${serviceName}.${namespace}.svc.cluster.local`,
  });
});

deployerRouter.get("/:projectId/status", async (req: AuthRequest, res) => {
  const userId = req.userId!;
  const project = await prisma.project.findUnique({
    where: { id: req.params.projectId },
  });
  if (!project) return res.status(404).json({ error: "Project not found" });
  if (project.userId !== userId)
    return res.status(403).json({ error: "Forbidden" });

  const namespace = config.k8sExecutorNamespace;
  const { deploymentName } = names(project.id);

  try {
    const { body } = await appsV1Api.readNamespacedDeployment(
      deploymentName,
      namespace
    );
    const ready = body.status?.readyReplicas || 0;
    const desired = body.spec?.replicas || 0;
    const status = desired > 0 && ready >= desired ? "running" : "starting";
    res.json({ status, ready_replicas: ready, desired_replicas: desired });
  } catch (err: any) {
    const statusCode = err.statusCode || err.response?.statusCode;
    if (statusCode === 404) {
      return res.json({ status: "stopped" });
    }
    res.status(500).json({ error: err.body?.message || err.message });
  }
});

deployerRouter.get("/:projectId/logs", async (req: AuthRequest, res) => {
  const userId = req.userId!;
  const project = await prisma.project.findUnique({
    where: { id: req.params.projectId },
  });
  if (!project) return res.status(404).json({ error: "Project not found" });
  if (project.userId !== userId)
    return res.status(403).json({ error: "Forbidden" });

  const namespace = config.k8sExecutorNamespace;
  const { body: podList } = await coreV1Api.listNamespacedPod(
    namespace,
    undefined,
    undefined,
    undefined,
    undefined,
    `project=${project.id}`
  );
  if (podList.items.length === 0) {
    return res.status(404).json({ error: "No running pod for this project" });
  }

  const podName = podList.items[0].metadata!.name!;
  try {
    const { body: logs } = await coreV1Api.readNamespacedPodLog(
      podName,
      namespace,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      200
    );
    res.json({ pod: podName, logs });
  } catch (err: any) {
    res.status(500).json({ error: err.body?.message || err.message });
  }
});

deployerRouter.post("/:projectId/stop", async (req: AuthRequest, res) => {
  const userId = req.userId!;
  const project = await prisma.project.findUnique({
    where: { id: req.params.projectId },
  });
  if (!project) return res.status(404).json({ error: "Project not found" });
  if (project.userId !== userId)
    return res.status(403).json({ error: "Forbidden" });

  const namespace = config.k8sExecutorNamespace;
  const { deploymentName, serviceName } = names(project.id);

  const deletions: Array<[() => Promise<unknown>, string]> = [
    [
      () => appsV1Api.deleteNamespacedDeployment(deploymentName, namespace),
      deploymentName,
    ],
    [
      () => coreV1Api.deleteNamespacedService(serviceName, namespace),
      serviceName,
    ],
  ];

  for (const [fn] of deletions) {
    try {
      await fn();
    } catch (err: any) {
      const statusCode = err.statusCode || err.response?.statusCode;
      if (statusCode !== 404) {
        return res
          .status(500)
          .json({ error: err.body?.message || err.message });
      }
    }
  }

  res.json({ message: "stopped" });
});
```

**Why a `Deployment` and not a bare long-running `Pod`?** Same reasoning as the Flask version — self-healing for free. If the container crashes, Kubernetes recreates it; a standalone `Pod` you create directly just stays dead.

---

## 17. `src/app.ts` — ties it all together

```typescript
import express from "express";
import cors from "cors";
import { config } from "./config";
import { authRouter } from "./auth/routes";
import { projectsRouter } from "./projects/routes";
import { executorRouter } from "./projects/executor";
import { deployerRouter } from "./projects/deployer";

const app = express();
app.use(express.json());
app.use(cors());

app.use("/auth", authRouter);
app.use("/projects", projectsRouter);
app.use("/projects", executorRouter);
app.use("/projects", deployerRouter);

app.listen(config.port, "0.0.0.0", () => {
  console.log(`Main API listening on port ${config.port}`);
});
```

---

## 18. `Dockerfile` (Main API)

```dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json tsconfig.json ./
COPY prisma ./prisma
RUN npm ci
COPY src ./src
RUN npx prisma generate && npm run build

FROM node:20-slim
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/prisma ./prisma
EXPOSE 5000
CMD ["node", "dist/app.js"]
```

A multi-stage build keeps the final image lean — `tsc` and the Prisma generator only run in the `build` stage, and the runtime image just gets the compiled `dist/` output plus `node_modules`.

---

## 19. Kubernetes Manifests

### `k8s/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: platform-projects-pvc
  namespace: executor
spec:
  accessModes:
    - ReadWriteMany # REQUIRED — same reasoning as the Flask version: many pods, possibly different nodes.
  resources:
    requests:
      storage: 20Gi
  storageClassName: efs-sc # swap for whatever RWX-capable storage class your cluster has
```

### `k8s/deployment.yaml` (Main API)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main-api
  namespace: platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: main-api
  template:
    metadata:
      labels:
        app: main-api
    spec:
      serviceAccountName: main-api-sa
      containers:
        - name: main-api
          image: your-registry/main-api:latest # built from the Dockerfile in Section 18
          ports:
            - containerPort: 5000
          envFrom:
            - secretRef:
                name: main-api-secrets # NEON_DATABASE_URL, JWT_SECRET_KEY
          volumeMounts:
            - name: project-storage
              mountPath: /data
      volumes:
        - name: project-storage
          persistentVolumeClaim:
            claimName: platform-projects-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: main-api
  namespace: platform
spec:
  selector:
    app: main-api
  ports:
    - port: 80
      targetPort: 5000
```

### `k8s/rbac.yaml`

Identical to the Flask version — these permissions are about Kubernetes resources, not the language calling them:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: main-api-sa
  namespace: platform
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: executor
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "delete"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "get", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["create", "get", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: main-api-pod-manager-binding
  namespace: executor
subjects:
  - kind: ServiceAccount
    name: main-api-sa
    namespace: platform
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

### `k8s/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: executor-default-deny
  namespace: executor
spec:
  podSelector:
    matchLabels:
      app: executor
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

> Same caveat as the Flask version: plain `NetworkPolicy` can't match by hostname — for tighter "only Neon, nothing else" egress, use an FQDN-aware CNI (e.g., Cilium) or a proxy.

---

## 20. Sequences: "Run" (smoke test) vs "Deploy"/"Stop" (persistent)

**Smoke test (`/run`, Section 15):**

```
User clicks "Quick test"
   │
   ▼
POST /projects/<id>/run  (JWT required)
   │
   ▼
Main API verifies project ownership
   │
   ▼
Main API calls k8s API → creates Pod (restartPolicy: Never)
   - volumeMount: same PVC, subPath = projects/<id>
   - command: npm install && timeout 20 npx ts-node api.ts
   │
   ▼
Main API polls pod status (max 30s)
   │
   ▼
Pod finishes (Succeeded/Failed) → Main API reads logs + exit code
   │
   ▼
Main API deletes the pod (cleanup, in `finally`)
   │
   ▼
Response: { "exit_code": ..., "output": "..." }
```

**Persistent run (`/deploy` + `/stop`, Section 16):**

```
User clicks "Run"
   │
   ▼
POST /projects/<id>/deploy  (JWT required)
   │
   ▼
Main API verifies project ownership
   │
   ▼
Deployment exists?
   ├─ No  → create Deployment (1 replica, restartPolicy: Always) + Service
   └─ Yes → patch Deployment (new annotation timestamp triggers a rollout,
            so saved code edits get picked up)
   │
   ▼
Response: { "message": "deployed", "internal_url": "http://svc-<id>...svc.cluster.local" }
   │
   ▼
... pod keeps running, self-heals on crash via the Deployment controller ...
   │
   ▼
User polls GET /projects/<id>/status  → { "status": "running" }
User polls GET /projects/<id>/logs    → tails the live pod's logs
   │
   ▼
User clicks "Stop" (or closes the project)
   │
   ▼
POST /projects/<id>/stop
   │
   ▼
Main API deletes the Deployment + Service → pod terminates, compute is freed
```

---

## 21. Security & Practical Notes

- **`subPath` isolation is still the whole ballgame.** Never mount the PVC root into an executor pod — always scope it to `projects/<project_id>`.
- **`automountServiceAccountToken: false`** on every executor pod.
- **RBAC scoped to the `executor` namespace only**, with the minimum verbs needed on `pods`, `pods/log`, `services`, and `deployments`.
- **No user input goes into a shell string.** The command is fixed (`npm install ... && ts-node api.ts`); the user's code lives in `api.ts` on disk, never interpolated into the command itself.
- **Secrets belong in Kubernetes Secrets** (`NEON_DATABASE_URL`, `JWT_SECRET_KEY`, `neon-credentials`), never baked into the image.
- **`npm install` is slower than `pip install` for an equivalent dependency set, and it eats directly into your 20-second smoke-test budget.** Consider pre-baking common dependencies (`express`, `pg`, `cors`, `ts-node`) into the `node:20-slim`-based runner image itself as a base layer, falling back to `npm install` only for anything extra the user adds — this can cut cold-start time dramatically. Once a project has a `package-lock.json`, switch the install command to `npm ci` (faster, reproducible) instead of `npm install`.
- **Rate-limit project creation and run frequency per user** (e.g., with `express-rate-limit`) before going past prototyping.
- **Cap concurrently running deployments per user**, and consider an idle-timeout reaper that calls `/stop` on inactive projects — a `Deployment` left running until manually stopped is a real, ongoing cost.
- **Pin your `@kubernetes/client-node` version and check its types before relying on the exact call signatures above** — this package's API shape (response wrapping, positional vs. options-object parameters) has changed across major versions; the code here matches the `0.18.x`–`0.20.x` line.

---

## 22. Natural Next Steps

- **Public exposure:** the `Service` from Section 16 is `ClusterIP`-only. Add an `Ingress`/gateway routing `https://yourplatform.com/p/<project_id>/*` (or a per-project subdomain) to give users a real, shareable URL.
- **Per-project database isolation:** all projects currently share one Neon database. Neon's branching API could give each project its own isolated branch.
- **Build/run quotas:** track CPU-seconds, run-count, or total running-time per user to enforce fair-use limits.
- **Precompiled execution for production projects:** once a user's project is stable, you could `tsc`-compile it once and run the compiled JS directly (`node api.js`) instead of `ts-node` on every restart — faster startup, same code.
