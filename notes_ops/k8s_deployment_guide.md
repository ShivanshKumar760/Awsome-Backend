# Dockerizing & Deploying the Instagram Flask API to Kubernetes

This walks through containerizing the Flask + `psycopg2` API and deploying it to Kubernetes with six manifests: `ConfigMap.yml`, `deployment.yml`, `Hpa.yml`, `Pvc.yml`, `Secrets.yml`, `Services.yml`. Postgres runs as a plain **Deployment** (not a StatefulSet) since we only ever need one instance with one persistent disk attached.

---

## Table of Contents

1. The Dockerfile, explained
2. Why a Deployment instead of a StatefulSet for Postgres
3. ConfigMap vs Secret — what goes where
4. PVC — persistent storage for Postgres
5. The two Deployments (Postgres + Flask API)
6. Services — how Pods talk to each other and the outside world
7. HPA — autoscaling the API (and *not* Postgres)
8. Putting it all together — deploy order & commands
9. Verifying everything works

---

## 1. The Dockerfile, Explained

```dockerfile
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

Line by line:

- **`FROM python:3.12-slim`** — `slim` is a smaller Debian-based image with just enough to run Python; far smaller than the full `python:3.12` image, faster to pull in a cluster.
- **`PYTHONUNBUFFERED=1`** — without this, Python buffers stdout, so `print()`/log statements can sit in a buffer and not show up immediately in `kubectl logs`. Always set this in containers.
- **`gcc` / `libpq-dev`** — `psycopg2` (the non-`binary` version) needs to compile a C extension against Postgres's client library at install time. We're using `psycopg2-binary` in `requirements.txt` which ships precompiled wheels, but `libpq-dev`/`gcc` are kept here as a safety net in case the binary wheel isn't available for your exact base image/architecture (e.g. on `arm64`).
- **`COPY requirements.txt .` before `COPY . .`** — this ordering is a deliberate Docker layer-caching trick: Docker caches each instruction as a layer. If you copy *all* your code first, then `pip install`, then any time you change a single line of Python, Docker invalidates the cache and **re-runs the entire pip install** from scratch. Copying just `requirements.txt` first means the slow `pip install` layer only re-runs when your dependencies actually change.
- **`gunicorn` instead of `flask run` / `app.run()`** — Flask's built-in dev server (`debug=True`) is single-threaded, auto-reloads on file changes, and explicitly warns it's "not for production use." `gunicorn` is a real WSGI server: `--workers 4` runs 4 separate OS processes handling requests, giving you actual concurrency and resilience (one worker crashing doesn't take down the others).

```
# requirements.txt
flask
psycopg2-binary
gunicorn
```

**Build and push:**

```bash
docker build -t your-dockerhub-username/instagram-api:latest .
docker push your-dockerhub-username/instagram-api:latest
```

Your cluster pulls images from a registry (Docker Hub, GHCR, ECR, etc.) — it can't see images that only exist on your local machine unless you're using a local-cluster tool like `minikube` with `minikube image load` or `kind load docker-image`.

---

## 2. Why a Deployment (Not a StatefulSet) for Postgres

This is the core design decision you called out, so it's worth justifying clearly.

**What StatefulSet exists for:** managing a *group of replicas* of a stateful app where **each replica needs its own stable identity and its own separate disk** that follows it around — e.g. a 3-node Postgres replication cluster (`postgres-0`, `postgres-1`, `postgres-2`, each with its own PVC: `postgres-0`'s data never gets mixed up with `postgres-1`'s), or a Kafka/Cassandra cluster. StatefulSet guarantees stable network names (`postgres-0.postgres-service`), ordered/predictable startup, and a 1:1 Pod↔PVC mapping per replica.

**What we actually have here:** exactly **one** Postgres instance, with exactly **one** PVC. We are not running replicas, we don't need stable per-replica identity (there's only one identity), and we don't need StatefulSet's ordered scale-up/scale-down guarantees (there's nothing to order — there's one Pod).

A plain `Deployment` with `replicas: 1` gives us everything we actually need — restart on crash, a rollout strategy, and a Pod template — with far less conceptual overhead. The two things you must still get right to make this safe (covered in the manifest comments too):

1. **`replicas: 1`, always** — Postgres's data directory isn't designed for two processes to write to the same files concurrently. Bumping `replicas` to 2 here wouldn't give you more throughput, it would corrupt your database.
2. **`strategy: { type: Recreate }`** — Deployments default to `RollingUpdate`, which starts the *new* Pod before killing the *old* one, so for a brief moment two Pods exist side-by-side. Our PVC is `ReadWriteOnce` (only one Pod can mount it at a time), so the new Pod would get stuck `Pending` forever, unable to attach the volume — `Recreate` avoids this by killing the old Pod first, *then* starting the new one.

If you ever genuinely need Postgres replication (a primary + read replicas, each with their own disk), **that's** when you'd reach for a StatefulSet — but that's a fundamentally different architecture, not just "more replicas of the same Deployment."

---

## 3. ConfigMap vs Secret — What Goes Where

Both are ways to inject configuration into a Pod as environment variables (or mounted files) without hardcoding values into your image — but they differ in intent:

| | ConfigMap | Secret |
|---|---|---|
| **For** | Non-sensitive config: hostnames, ports, feature flags, db names | Sensitive data: passwords, API keys, tokens |
| **Storage** | Plaintext in `etcd` | Base64-*encoded* (NOT encrypted by default!) in `etcd` |
| **Visible via `kubectl get -o yaml`?** | Yes, as plain text | Yes, but base64-encoded — trivially decoded (`base64 -d`), this is *not* real security on its own |

```yaml
# ConfigMap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: instagram-config
data:
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "instagram_clone"
  FLASK_ENV: "production"
```

```yaml
# Secrets.yml
apiVersion: v1
kind: Secret
metadata:
  name: instagram-secret
type: Opaque
data:
  DB_USER: cG9zdGdyZXM=          # base64 of "postgres"
  DB_PASSWORD: eW91cnBhc3N3b3Jk    # base64 of "yourpassword"
```

Generate the base64 values yourself rather than copying these example ones into anything real:

```bash
echo -n "postgres" | base64        # -> cG9zdGdyZXM=
echo -n "yourpassword" | base64    # -> eW91cnBhc3N3b3Jk
```

⚠️ **Important caveat the Secret object itself won't tell you:** Kubernetes Secrets are base64-*encoded*, not encrypted, by default. Anyone with `kubectl get secret instagram-secret -o yaml` access can trivially decode it. For real production security you'd enable **encryption at rest** for `etcd`, or use an external secret manager (HashiCorp Vault, AWS Secrets Manager via the External Secrets Operator, etc.). This setup is appropriate for learning/staging; treat it as a starting point, not a production security boundary.

**How both get into the running containers** — your `app.py`/`db.py` should read these as plain environment variables (e.g. `os.environ["DB_HOST"]`), exactly as if you'd set them with `export` locally:

```yaml
envFrom:
  - configMapRef:
      name: instagram-config
  - secretRef:
      name: instagram-secret
```

`envFrom` dumps *every* key in the ConfigMap/Secret in as an env var, which is why the API Deployment uses it — it's the equivalent of merging both into your shell's environment. (Postgres's own Deployment instead uses individual `env: [{ name: ..., valueFrom: ... }]` entries, because the official `postgres` image expects very specific variable names like `POSTGRES_USER`/`POSTGRES_PASSWORD`, not your app's `DB_USER`/`DB_PASSWORD` names — `valueFrom` lets you rename a key from the ConfigMap/Secret into whatever env var name the container actually expects.)

---

## 4. PVC — Persistent Storage for Postgres

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

**Why Postgres needs this at all:** containers are **ephemeral** — if a Pod is deleted, rescheduled, or crashes, anything written to its normal container filesystem is gone. A PVC is a request for a piece of real persistent storage (backed by your cloud provider's disk service, or local storage on bare-metal/minikube) that **outlives the Pod** — so when Kubernetes recreates the Postgres Pod (after a crash, a node failure, a rollout), it reattaches the *same* disk with all your data intact.

- **`ReadWriteOnce` (RWO):** the volume can be mounted as read-write by **one Node** at a time. This is the right (and most common/portable) mode for a single Postgres instance. `ReadWriteMany` (multiple Nodes simultaneously) needs special storage backends (NFS, etc.) and isn't something a normal database wants anyway — concurrent writers to the same raw data files is a corruption risk regardless of Kubernetes.
- **`requests.storage: 2Gi`** — the minimum size you're asking for; bump it for real workloads. Most cloud StorageClasses support **expanding** a PVC later (`kubectl edit pvc postgres-pvc`, raise the number) without recreating it, but check `allowVolumeExpansion` on your StorageClass first.
- **No `storageClassName` set** — this means "use whatever StorageClass is marked default on this cluster" (e.g. `gp2`/`gp3` on EKS, `standard` on GKE, the `hostpath` provisioner on minikube). Set it explicitly if your cluster has no default or you want a specific tier (e.g. SSD vs HDD).

How the Postgres Deployment actually *uses* this PVC:

```yaml
volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
containers:
  - volumeMounts:
      - name: postgres-storage
        mountPath: /var/lib/postgresql/data
        subPath: pgdata
```

`mountPath: /var/lib/postgresql/data` is exactly where the official `postgres` image expects its data directory to live. `subPath: pgdata` puts the actual data inside a `pgdata/` subfolder *within* the volume rather than the volume's bare root — this sidesteps a well-known quirk where Postgres complains about "lost+found" or other filesystem metadata directories that some volume types place at the root.

---

## 5. The Two Deployments

### Postgres Deployment (data layer — fixed at 1 replica)

Already covered in Section 2 conceptually; the manifest wires together everything from Sections 3–4: it reads `DB_NAME`/`DB_USER`/`DB_PASSWORD` from the ConfigMap/Secret, mounts the PVC, and adds **probes**:

```yaml
readinessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
livenessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
```

- **`readinessProbe`** answers "is this Pod ready to receive traffic *right now*?" — if it fails, Kubernetes stops sending it requests (via the Service) but does **not** restart the Pod. Useful for "still starting up" windows.
- **`livenessProbe`** answers "is this Pod still alive/healthy, or should it be killed and restarted?" — if it fails repeatedly, Kubernetes kills and recreates the container.
- `pg_isready` is a real Postgres CLI tool built for exactly this purpose — it checks the server can accept connections.

### Flask API Deployment (app layer — scales horizontally)

```yaml
replicas: 2
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Unlike Postgres, this app is **stateless** — every request goes straight to the database over the network, no local data of its own — so it's safe to run many identical copies, and safe to use the default `RollingUpdate` strategy: Kubernetes can bring up one new Pod, wait for it to pass its readiness probe, then kill an old one, repeating until the rollout is done, with zero downtime. `maxUnavailable: 1` / `maxSurge: 1` together mean "at most 1 Pod missing, and at most 1 extra Pod beyond the desired count, at any point during the rollout."

```yaml
readinessProbe:
  httpGet:
    path: /users
    port: 5000
```

We reuse the real `/users` endpoint as a health check — if it can successfully return a 200, the app and its DB connection are working end-to-end. (For a larger app you'd typically add a dedicated lightweight `/health` route instead of reusing a real data endpoint, to avoid health checks adding load to your actual data path.)

---

## 6. Services

A **Service** gives a stable network name/IP to a *set* of Pods (selected by label), so other things in (or outside) the cluster don't need to track individual Pod IPs, which change every time a Pod restarts.

```yaml
# postgres-service — internal only
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

No `type` is set here, which defaults to **`ClusterIP`** — only reachable *from inside* the cluster. This is intentional: Postgres should never be directly reachable from the public internet. Other Pods reach it simply via the hostname `postgres-service` (Kubernetes' internal DNS resolves Service names automatically) — which is exactly the value used for `DB_HOST` in the ConfigMap.

```yaml
# instagram-api-service — externally reachable
apiVersion: v1
kind: Service
metadata:
  name: instagram-api-service
spec:
  type: LoadBalancer
  selector:
    app: instagram-api
  ports:
    - port: 80
      targetPort: 5000
```

- **`LoadBalancer`** — on a real cloud cluster (EKS/GKE/AKS), this provisions an actual cloud load balancer with a public IP automatically. On a local cluster (minikube/kind), this type usually stays `<pending>` forever unless you run `minikube tunnel` — in that case, switch to **`NodePort`** (exposes the Service on a fixed port on every Node's own IP) or just use `kubectl port-forward svc/instagram-api-service 5000:80` for local testing.
- The Service's `selector: app: instagram-api` is what connects it to the API Deployment's Pods — note this matches the Pod template's `labels`, **not** the Deployment's own name. Services route by label, never by Deployment name directly.

---

## 7. HPA — Autoscaling the API (and Deliberately Not Postgres)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: instagram-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: instagram-api-deployment
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

**What HPA does:** continuously checks real resource usage (CPU/memory) across all Pods of a target Deployment, and automatically adjusts the `replicas` count up or down to keep average utilization near your target — e.g. "if average CPU usage across all `instagram-api` Pods goes above 60% of what each Pod *requested*, add more Pods."

**Why it only targets the API Deployment:** HPA works by adding/removing *identical, interchangeable* replicas — which is exactly what our stateless Flask Pods are. Postgres is pinned to `replicas: 1` with a single `ReadWriteOnce` PVC; if HPA tried to scale it to 2, the second Pod would be unable to mount the already-attached volume and would get stuck `Pending` forever. Scaling a single-writer database horizontally requires an entirely different architecture (read replicas, connection routing, etc.) — not something a generic HPA can safely do for you.

**Prerequisite:** HPA needs the **metrics-server** add-on running in your cluster to read CPU/memory usage at all. Most managed clusters (EKS/GKE/AKS) have it available as an add-on; on minikube, enable it with:

```bash
minikube addons enable metrics-server
```

Without metrics-server, `kubectl get hpa` will show `<unknown>` for current utilization and the HPA can't make any scaling decisions.

---

## 8. Putting It All Together — Deploy Order & Commands

Order matters here, mainly so things don't crash-loop waiting on something that doesn't exist yet (Kubernetes is fairly forgiving about ordering since Pods retry, but this order avoids unnecessary restarts):

```bash
# 1. Config & secrets first — nothing else can start correctly without these
kubectl apply -f k8s/ConfigMap.yml
kubectl apply -f k8s/Secrets.yml

# 2. Storage next — so Postgres has somewhere to mount when it starts
kubectl apply -f k8s/Pvc.yml

# 3. Both Deployments (Postgres + API are defined in the same file)
kubectl apply -f k8s/deployment.yml

# 4. Services — so Pods (and the outside world) can actually reach each other
kubectl apply -f k8s/Services.yml

# 5. Autoscaler — added last since it depends on the API Deployment already existing
kubectl apply -f k8s/Hpa.yml
```

Or, since Kubernetes manifests are declarative and order-tolerant for most cases, you can simply apply everything in the directory at once and let Kubernetes sort out dependencies via retries:

```bash
kubectl apply -f k8s/
```

---

## 9. Verifying Everything Works

```bash
# Are the Pods up?
kubectl get pods
# Expect: 1 postgres pod, 2 (or more) instagram-api pods, all STATUS=Running

# Did the PVC actually bind to real storage?
kubectl get pvc
# Expect: postgres-pvc STATUS=Bound

# Are the Services pointing at the right Pods?
kubectl get endpoints postgres-service
kubectl get endpoints instagram-api-service
# Expect: real Pod IP:port entries, not <none>

# Is the HPA tracking metrics?
kubectl get hpa
# Expect: TARGETS column showing real percentages, not <unknown>

# Tail logs from the API to confirm it can actually reach Postgres
kubectl logs -l app=instagram-api --tail=50

# Local test without a real LoadBalancer (e.g. on minikube)
kubectl port-forward svc/instagram-api-service 5000:80
curl http://localhost:5000/users
```

If `/users` returns `[]` (an empty list) rather than an error, your Flask API successfully connected to Postgres through `postgres-service`, read the ConfigMap/Secret env vars correctly, and the whole chain — Docker image → Deployment → Service → PVC-backed database — is wired correctly end to end.
