# Docker — Full Notes: Commands, Dockerfiles, Multi-Stage Builds, Networking, Volumes & Compose

Builds on the three APIs from the GCP deployment notes (Spring Boot, Node/Express+TS, Flask) — every Dockerfile below containerizes one of those exact apps.

---

## Table of Contents

1. Docker Core Concepts (image vs container vs registry)
2. Essential Docker Commands
3. Dockerfile — Instruction by Instruction
4. Single-Stage Dockerfile — Flask API
5. Multi-Stage Dockerfile — Node/Express + TypeScript API
6. Multi-Stage Dockerfile — Spring Boot API
7. Why Multi-Stage Builds Matter (size comparison)
8. Docker Networking
9. Docker Volumes
10. Docker Compose — Running All Three + Postgres Together

---

## 1. Docker Core Concepts

| Term | Meaning |
|---|---|
| **Image** | A read-only, layered snapshot — your app + its OS dependencies + runtime, baked together. Doesn't run by itself. |
| **Container** | A *running instance* of an image — the same relationship as a class to an object. You can run many containers from one image. |
| **Dockerfile** | The recipe — a text file of instructions describing how to build an image. |
| **Registry** | Where images are stored/shared (Docker Hub, GCR/Artifact Registry, GHCR, ECR). `docker push`/`docker pull` move images to/from a registry. |
| **Layer** | Each instruction in a Dockerfile (`RUN`, `COPY`, etc.) creates a new filesystem layer, cached individually — this is the basis of Docker's build caching. |

**The mental model:** `Dockerfile` → (`docker build`) → **Image** → (`docker run`) → **Container**.

---

## 2. Essential Docker Commands

### Images

```bash
docker build -t my-api:latest .          # build an image from a Dockerfile in the current dir
docker build -t my-api:v1 -f Dockerfile.prod .   # build from a specific, non-default Dockerfile name
docker images                              # list local images
docker rmi my-api:latest                   # remove an image
docker image prune                         # remove all dangling (untagged) images
docker tag my-api:latest myuser/my-api:v1    # add another name/tag to an existing image
docker pull postgres:16                    # download an image from a registry
docker push myuser/my-api:v1               # upload an image to a registry (after docker login)
docker history my-api:latest               # see each layer and its size
```

### Containers

```bash
docker run my-api                          # run a container (foreground, attached)
docker run -d my-api                       # detached — runs in background, returns immediately
docker run -d -p 8080:8080 my-api          # map host:container ports
docker run -d --name my-running-api my-api   # give the container a friendly name
docker run -d -e DB_HOST=postgres -e DB_PORT=5432 my-api   # pass environment variables
docker run -it ubuntu bash                  # interactive + allocate a TTY — for shells/debugging
docker run --rm my-api                       # auto-remove the container once it stops (good for one-offs)

docker ps                                   # list RUNNING containers
docker ps -a                                # list ALL containers, including stopped ones
docker stop my-running-api                   # gracefully stop (SIGTERM, then SIGKILL after a timeout)
docker start my-running-api                  # start a stopped container again
docker restart my-running-api
docker rm my-running-api                      # remove a stopped container
docker rm -f my-running-api                   # force-remove, even if running

docker logs my-running-api                    # view stdout/stderr
docker logs -f my-running-api                 # follow logs live (like `tail -f`)
docker exec -it my-running-api bash           # open a shell INSIDE a running container
docker inspect my-running-api                 # full JSON metadata — IP, mounts, env vars, etc.
docker stats                                  # live CPU/memory usage per running container
```

### Cleanup

```bash
docker system df                            # see disk space used by images/containers/volumes
docker system prune                          # remove all stopped containers, unused networks, dangling images
docker system prune -a --volumes              # aggressive cleanup — also removes unused images and volumes
```

---

## 3. Dockerfile — Instruction by Instruction

| Instruction | Purpose |
|---|---|
| `FROM <image>` | Base image to build on top of. Every Dockerfile starts with this. |
| `WORKDIR <path>` | Sets the working directory for all following instructions — also creates it if missing. |
| `COPY <src> <dest>` | Copies files from your build context (local machine) into the image. |
| `ADD <src> <dest>` | Like `COPY`, but also auto-extracts tarballs and can fetch URLs — generally **prefer `COPY`** unless you specifically need those extras; `ADD`'s "magic" behavior is a common source of confusion. |
| `RUN <command>` | Executes a command **at build time**, producing a new layer (e.g. installing packages). |
| `ENV <key>=<value>` | Sets an environment variable available both at build time and in the running container. |
| `EXPOSE <port>` | **Documentation only** — declares which port the app listens on. Does NOT actually publish the port; that's `-p` on `docker run`. |
| `CMD [...]` | The **default command** to run when the container starts. Can be overridden at `docker run` time. |
| `ENTRYPOINT [...]` | Similar to `CMD`, but harder to override — typically used when the image is meant to always run as a specific executable, with `CMD` supplying default *arguments* to it. |
| `ARG <key>` | Build-time-only variable (not present in the final running container) — e.g. for choosing a version during `docker build --build-arg`. |
| `USER <name>` | Switches which user subsequent instructions (and the final container) run as — security best practice: avoid running as `root`. |
| `VOLUME <path>` | Declares a path that should be treated as a mount point for persistent/shared data (Section 9). |

**`CMD` vs `ENTRYPOINT`, concretely:**

```dockerfile
ENTRYPOINT ["python3"]
CMD ["app.py"]
```

`docker run my-image` → runs `python3 app.py`. `docker run my-image other.py` → runs `python3 other.py` (only `CMD`'s part is overridden; `ENTRYPOINT` always runs).

---

## 4. Single-Stage Dockerfile — Flask API

```dockerfile
# flask-api/Dockerfile
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8082

CMD ["gunicorn", "--bind", "0.0.0.0:8082", "app:app"]
```

```bash
docker build -t flask-api:latest ./flask-api
docker run -d -p 8082:8082 --name flask-api flask-api:latest
curl http://localhost:8082/items
```

This works fine as-is — Python's interpreted, there's no separate "compile" step, so single-stage is reasonable here. (You *can* still multi-stage Python builds to separate dependency-compilation tooling from the runtime image — shown as an exercise in Section 7's "where to go further" — but it's far less impactful than for Node/Java.)

---

## 5. Multi-Stage Dockerfile — Node/Express + TypeScript API

This is where multi-stage builds earn their keep: TypeScript needs to be **compiled** to JavaScript, and that compilation needs `devDependencies` (`typescript`, `@types/*`) that have **no business being in the final running image**.

```dockerfile
# node-api/Dockerfile

# ---------- Stage 1: build ----------
FROM node:20-slim AS builder

WORKDIR /app

# Copy only manifest files first — caches npm install separately from source changes
COPY package*.json ./
RUN npm install

# Now copy source and compile TypeScript -> JavaScript
COPY . .
RUN npm run build   # runs "tsc", produces /app/dist


# ---------- Stage 2: runtime ----------
FROM node:20-slim AS runtime

WORKDIR /app

# Only copy production dependencies — skip devDependencies entirely in the final image
COPY package*.json ./
RUN npm install --omit=dev

# Copy ONLY the compiled output from the "builder" stage — none of its source,
# node_modules (dev), or TypeScript compiler comes along for the ride
COPY --from=builder /app/dist ./dist

EXPOSE 8081

CMD ["node", "dist/server.js"]
```

```bash
docker build -t node-api:latest ./node-api
docker run -d -p 8081:8081 --name node-api node-api:latest
curl http://localhost:8081/items
```

**What `COPY --from=builder` is doing:** Docker keeps each named stage (`AS builder`, `AS runtime`) as a separate, isolated filesystem during the build. `COPY --from=builder /app/dist ./dist` reaches *into the first stage's filesystem* and pulls out just the compiled `dist/` folder — none of `builder`'s TypeScript source files, its `devDependencies`, or the compiler itself end up in the final image. The `builder` stage is discarded entirely once the build finishes; only `runtime` becomes the actual image you `docker run`.

---

## 6. Multi-Stage Dockerfile — Spring Boot API

Java's case is even more dramatic: the **JDK** (needed to *compile* Java) is much larger than the **JRE** (needed only to *run* already-compiled `.class` files) — and Maven itself, plus the entire `~/.m2` dependency cache it downloads, has zero reason to exist in your final image.

```dockerfile
# spring-api/Dockerfile

# ---------- Stage 1: build ----------
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copy only the POM first — lets Docker cache the downloaded dependencies
# separately from your actual source code changes
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Now copy source and build the actual JAR
COPY src ./src
RUN mvn clean package -DskipTests -B
# produces /app/target/api-1.0.0.jar


# ---------- Stage 2: runtime ----------
FROM eclipse-temurin:17-jre-alpine AS runtime

WORKDIR /app

# Copy ONLY the built JAR from the "builder" stage — no Maven, no JDK,
# no downloaded dependency cache, no source code
COPY --from=builder /app/target/api-1.0.0.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

```bash
docker build -t spring-api:latest ./spring-api
docker run -d -p 8080:8080 --name spring-api spring-api:latest
curl http://localhost:8080/items
```

**`mvn dependency:go-offline` before copying `src/`** is the Java equivalent of the Node "copy `package.json` before the rest" trick (and the earlier Flask "`COPY requirements.txt` before `COPY . .`" trick): it lets Docker cache the slow dependency-download layer, only invalidating it when `pom.xml` itself changes — not every time you edit a `.java` file.

---

## 7. Why Multi-Stage Builds Matter — Size Comparison

```bash
docker images
```

| Image | Approx. size (illustrative) | Why |
|---|---|---|
| Single-stage Node (`node:20` + full `node_modules` incl. devDeps + TS source) | ~1.1 GB | Carries the entire TypeScript compiler, type definitions, and raw `.ts` source nobody runs |
| **Multi-stage Node** (above) | ~180 MB | Only compiled `.js` + production `node_modules` |
| Single-stage Java (`maven:3.9-eclipse-temurin-17` as the *runtime* image too) | ~900 MB+ | Carries the full JDK, Maven itself, and the entire downloaded `~/.m2` cache |
| **Multi-stage Java** (above) | ~200 MB | Only the JRE + one JAR file |

**Why smaller images matter beyond disk space:**
- **Faster deploys** — less to push/pull from a registry, faster `docker pull` on every new VM/Pod that starts (directly relevant to the GCP MIGs and Kubernetes Deployments from earlier notes — every new instance/Pod has to pull the image before it can serve traffic).
- **Smaller attack surface** — a JDK + Maven + your full dependency cache sitting in a *production* container is a lot of unnecessary tooling (compilers, package managers) an attacker could potentially abuse if they got a shell.
- **Cache efficiency** — smaller, more focused final layers mean faster incremental pulls across deployments.

**General multi-stage pattern, language-agnostic:**

```dockerfile
FROM <build-tooling-image> AS builder
# ... install deps, compile/build ...

FROM <minimal-runtime-image> AS runtime
COPY --from=builder <built-artifact-only> <dest>
CMD [...]
```

---

## 8. Docker Networking

By default, every container Docker creates gets attached to a network — understanding the network **drivers** is the key to understanding how containers reach each other (and the outside world).

### Network drivers

| Driver | Behavior |
|---|---|
| **`bridge`** (default) | Containers get their own private internal IP on a virtual network local to the Docker host; can reach each other by IP, and by **container name** if on a *custom* (non-default) bridge network. Containers reach the outside world via NAT through the host. |
| **`host`** | Container shares the host machine's network namespace directly — no isolation, no port mapping needed (`-p` is ignored), but you lose all network isolation. Use sparingly (e.g. performance-critical cases). |
| **`none`** | No networking at all — fully isolated container. Rare, used for security-sensitive batch jobs that need zero network access. |
| **`overlay`** | Spans **multiple Docker hosts** — used in Docker Swarm (and conceptually similar to what Kubernetes' pod networking does across Nodes). Not relevant for single-host `docker run`/Compose setups. |

### The default bridge vs a custom bridge network — a critical distinction

```bash
# Using the DEFAULT bridge network (just `docker run` with no --network flag):
docker run -d --name api1 my-api
docker run -d --name api2 my-api
# api1 CANNOT reach api2 by the name "api2" — DNS resolution by container
# name only works on CUSTOM networks, not Docker's default bridge.
```

```bash
# Creating and using a CUSTOM bridge network:
docker network create my-app-network

docker run -d --name api1 --network my-app-network my-api
docker run -d --name api2 --network my-app-network my-api
# NOW api1 can reach api2 simply via the hostname "api2" — Docker's
# embedded DNS resolves container names automatically on custom networks.
```

This is exactly analogous to Kubernetes Services resolving Pods by name via CoreDNS, or how `postgres-service` worked in the earlier k8s notes — Docker's custom bridge networks give you the same "reach things by name, not by IP" convenience, just at a much smaller, single-host scale.

```bash
docker network ls                       # list all networks
docker network inspect my-app-network     # see which containers are attached, their IPs
docker network connect my-app-network some-container   # attach an already-running container to a network
docker network rm my-app-network          # delete (only if no containers are attached)
```

### Port publishing — `-p` flag

```bash
docker run -d -p 8080:80 my-api
#              ^^^^  ^^
#              host  container
```

`-p HOST_PORT:CONTAINER_PORT` maps a port on the Docker **host machine** to a port inside the **container**. Without `-p`, the container's port is reachable from *other containers* on the same Docker network, but **not** from the host machine or outside world at all.

```bash
docker run -d -p 127.0.0.1:8080:80 my-api   # bind ONLY to localhost on the host — not externally reachable
docker run -d -P my-api                      # publish ALL exposed ports to random high host ports
```

---

## 9. Docker Volumes

Exactly the same underlying problem as Kubernetes Volumes (your k8s notes, Section 8): **a container's own filesystem is ephemeral** — `docker rm` (or even just replacing a container with a new image) wipes anything written inside it. Volumes give you data that **outlives the container**.

### Types of mounts

| Type | What it is | Use case |
|---|---|---|
| **Named volume** | Docker-managed storage, lives in Docker's own storage area, independent of any specific host path | The default, recommended choice for persistent data (e.g. a database's files) |
| **Bind mount** | Maps an exact **host machine path** directly into the container | Local development — e.g. mounting your source code live into a container so edits show up without rebuilding |
| **tmpfs mount** | Stored in host memory only, never written to disk, gone when the container stops | Sensitive temporary data you explicitly don't want persisted anywhere |

```bash
# Named volume
docker volume create pgdata
docker run -d --name postgres -v pgdata:/var/lib/postgresql/data postgres:16

docker volume ls
docker volume inspect pgdata
docker volume rm pgdata          # only works if no container is using it

# Bind mount (host path : container path)
docker run -d --name node-dev -v $(pwd)/src:/app/src -p 8081:8081 node-api:latest

# tmpfs mount
docker run -d --tmpfs /app/cache my-api
```

**`-v` shorthand vs `--mount` (more explicit, generally preferred in scripts/Compose):**

```bash
# Equivalent, more verbose/explicit form:
docker run -d --name postgres \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres:16
```

**Why a named volume, specifically, for Postgres data** (mirroring the exact PVC reasoning from your k8s notes): Postgres's data files must survive container recreation (image updates, crashes, `docker-compose down && up`). A bind mount *can* technically do this too, but a named volume is the Docker-native, portable choice — it doesn't depend on a specific host directory existing in a specific place, which matters once you move from your laptop to a real server.

---

## 10. Docker Compose — Running All Three + Postgres Together

Compose lets you define **multiple containers, their networking, and their volumes, in one YAML file** — instead of remembering a long chain of `docker run`/`docker network create`/`docker volume create` commands.

```yaml
# docker-compose.yml
version: "3.9"

services:
  postgres:
    image: postgres:16
    container_name: postgres
    environment:
      POSTGRES_DB: instagram_clone
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: yourpassword
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  flask-api:
    build: ./flask-api          # builds from flask-api/Dockerfile instead of pulling a prebuilt image
    container_name: flask-api
    ports:
      - "8082:8082"
    depends_on:
      postgres:
        condition: service_healthy   # waits for Postgres's healthcheck to pass, not just "container started"
    environment:
      DB_HOST: postgres              # resolves via Compose's automatic service-name DNS
      DB_PORT: 5432

  node-api:
    build: ./node-api
    container_name: node-api
    ports:
      - "8081:8081"
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DB_HOST: postgres
      DB_PORT: "5432"

  spring-api:
    build: ./spring-api
    container_name: spring-api
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/instagram_clone
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: yourpassword

volumes:
  pgdata:    # declares the named volume used above — Compose creates it automatically if missing
```

### Commands

```bash
docker compose up                 # build (if needed) + start everything, attached (logs in terminal)
docker compose up -d               # same, but detached (runs in background)
docker compose up --build          # force a rebuild of any service with a `build:` key, even if cached
docker compose down                # stop and remove all containers + the default network (volumes kept)
docker compose down -v              # ALSO remove named volumes — wipes your Postgres data, use carefully
docker compose ps                   # list this project's containers and their status
docker compose logs -f flask-api     # follow logs for one specific service
docker compose exec flask-api bash    # open a shell inside a running service's container
docker compose restart node-api        # restart just one service
```

### What Compose gives you "for free" that plain `docker run` doesn't

- **An automatic custom network** — every service in the same `docker-compose.yml` can reach every other service **by its service name** (`postgres`, `flask-api`, etc.) with zero manual `docker network create` step. This is exactly the "custom bridge network = DNS by name" behavior from Section 8, set up implicitly.
- **`depends_on` + `condition: service_healthy`** — ensures, e.g., `flask-api` doesn't even attempt to start until Postgres's own `healthcheck` reports healthy, not just "the container process exists" (a container can be "running" long before Postgres is actually ready to accept connections).
- **One command to bring the whole stack up or down**, instead of four separate `docker run` invocations plus manual network/volume setup — directly comparable to how `kubectl apply -f k8s/` brought up your whole Kubernetes stack in one shot in the earlier notes.

---

## Quick Reference Summary

| Concept | One-line purpose |
|---|---|
| Image vs Container | Image = blueprint (read-only); Container = a running instance of it |
| `docker build` / `docker run` | Create an image from a Dockerfile / start a container from an image |
| `FROM`/`COPY`/`RUN`/`CMD` | Core Dockerfile instructions: base image, copy files, run build-time commands, default runtime command |
| `CMD` vs `ENTRYPOINT` | `CMD` easily overridden at `docker run`; `ENTRYPOINT` always runs, `CMD` becomes its default args |
| Multi-stage build | Separate "build tooling" stage from "runtime" stage — final image keeps only what's needed to *run*, not *build*, the app |
| `COPY --from=<stage>` | Pulls specific built artifacts out of an earlier stage, discarding everything else from it |
| Bridge network (custom) | Containers reach each other by name via Docker's built-in DNS |
| `-p host:container` | Publishes a container's port to the host machine |
| Named volume | Docker-managed persistent storage, survives container removal — the right choice for database data |
| Bind mount | Maps a host path directly in — useful for live local development |
| Docker Compose | Defines a whole multi-container stack (services, network, volumes) in one YAML file, run with one command |
