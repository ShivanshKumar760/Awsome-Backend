# Kubernetes Stateful Deployments — Beginner to Advanced

Most Kubernetes tutorials teach Deployments first — and that's fine for stateless apps. But the
moment you need a database, a message queue, a cache cluster, or anything that needs to remember
who it is across restarts, you need **StatefulSets**. This note builds from zero to production-
grade stateful workloads.

```
Part A — Why stateful is different from stateless
Part B — StatefulSet basics: what it gives you
Part C — Persistent storage: PV, PVC, StorageClass
Part D — Headless Services: stable network identity
Part E — StatefulSet YAML: full breakdown of every field
Part F — Scaling, updates, and rollbacks
Part G — Init containers and pod ordering
Part H — Real-world examples: PostgreSQL, Redis, MongoDB
Part I — Backup, restore, and disaster recovery
Part J — Advanced patterns: sharding, sidecars, anti-affinity
Part K — Common mistakes and how to fix them
```

---

## Part A — Why Stateful Is Different From Stateless

### Stateless app (Deployment)

```
User Request
    │
    ▼
┌─────────────────────────────────────────┐
│  Load Balancer                          │
└──────┬──────────────┬───────────────────┘
       │              │
       ▼              ▼
  ┌─────────┐    ┌─────────┐
  │  Pod-1  │    │  Pod-2  │    ← identical, interchangeable
  │  nginx  │    │  nginx  │    ← any pod can serve any request
  └─────────┘    └─────────┘    ← if pod-1 dies, pod-2 takes over
                                ← no data lost (stateless)
```

Stateless pods are **cattle** — identical, replaceable, no individual identity.

### Stateful app (StatefulSet)

```
┌─────────────────────────────────────────────────────────┐
│  StatefulSet: postgres                                  │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────┐          │
│  │  postgres-0       │    │  postgres-1       │          │
│  │  PRIMARY (writes) │    │  REPLICA (reads)  │          │
│  │  PVC: data-0     │    │  PVC: data-1      │          │
│  └──────────────────┘    └──────────────────┘          │
│                                                         │
│  postgres-0 ≠ postgres-1  — NOT interchangeable        │
│  Each has its own:                                      │
│    • Stable hostname:  postgres-0.postgres-headless     │
│    • Stable storage:   data-postgres-0 (PVC)           │
│    • Stable identity:  ordinal index (0, 1, 2...)      │
└─────────────────────────────────────────────────────────┘
```

Stateful pods are **pets** — each one has a name, a personality (role), and its own data.

### The three things a stateful app needs that a Deployment can't provide

```
1. STABLE NETWORK IDENTITY
   Pod name stays the same across restarts
   postgres-0 is always postgres-0, not some random hash
   DNS name: postgres-0.postgres-headless.default.svc.cluster.local

2. STABLE STORAGE
   PVC is attached to a specific pod by name
   When postgres-0 restarts, it gets its OWN PVC back (not postgres-1's)
   Data survives pod deletion and rescheduling

3. ORDERED OPERATIONS
   Pods start in order: 0 → 1 → 2
   Pods stop in reverse: 2 → 1 → 0
   Primary DB always starts before replicas
```

---

## Part B — StatefulSet Basics

### What a StatefulSet gives you vs. a Deployment

| Feature      | Deployment                        | StatefulSet                          |
| ------------ | --------------------------------- | ------------------------------------ |
| Pod names    | Random hash (`nginx-7d8f9-xkz2p`) | Ordered index (`nginx-0`, `nginx-1`) |
| Pod identity | None — all identical              | Stable — each has unique identity    |
| Storage      | Shared or none                    | Each pod gets its own PVC            |
| DNS          | Single service DNS                | Per-pod DNS via headless service     |
| Start order  | All at once (parallel)            | Ordered (0, 1, 2...)                 |
| Stop order   | Any order                         | Reverse (2, 1, 0)                    |
| Use case     | Web servers, APIs                 | Databases, queues, caches            |

### The core components of a stateful deployment

```
StatefulSet
    │
    ├── Headless Service     (stable DNS for each pod)
    │       postgres-0.postgres.default.svc.cluster.local
    │       postgres-1.postgres.default.svc.cluster.local
    │
    ├── Pod Template         (what each pod looks like)
    │       containers, resources, probes, env vars
    │
    └── VolumeClaimTemplates (each pod gets its own PVC)
            data-postgres-0   → 10Gi
            data-postgres-1   → 10Gi
            data-postgres-2   → 10Gi
```

### Minimal StatefulSet example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
spec:
  serviceName: postgres-headless # must match the headless service name
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_PASSWORD
              value: "mysecretpassword"
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates: # each pod gets its own PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## Part C — Persistent Storage Deep Dive

### The three storage objects you need to know

```
StorageClass  →  defines HOW to provision storage (which disk type, which cloud provider)
     │
     ▼
PersistentVolume (PV)  →  the actual disk (created manually or auto by StorageClass)
     │
     ▼
PersistentVolumeClaim (PVC)  →  a request for storage by a Pod (the pod's "lease")
```

### StorageClass — defines the type of disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs # AWS
# provisioner: kubernetes.io/gce-pd    # GCP
# provisioner: disk.csi.azure.com      # Azure
# provisioner: docker.io/hostpath      # local/kind for dev
parameters:
  type: gp3 # AWS EBS SSD type
  fsType: ext4
reclaimPolicy:
  Retain # Retain | Delete | Recycle
  # Retain = keep disk when PVC deleted (production)
  # Delete = delete disk when PVC deleted (dev/test)
allowVolumeExpansion: true # allow resizing PVC later
volumeBindingMode: WaitForFirstConsumer # don't create disk until pod is scheduled
```

### PersistentVolume — the actual disk (manual provisioning)

```yaml
# Only needed when NOT using dynamic provisioning (StorageClass)
# If you have a StorageClass, PVs are created automatically
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce # RWO: one node can mount for read/write
    # ReadWriteMany     # RWX: multiple nodes (NFS, EFS)
    # ReadOnlyMany      # ROX: multiple nodes, read only
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  hostPath: # for local dev/kind only
    path: /mnt/data/postgres-0
```

### PersistentVolumeClaim — what a pod requests

```yaml
# In StatefulSets this is defined in volumeClaimTemplates
# For standalone pods you create PVCs manually:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-0
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 10Gi
```

### How volumeClaimTemplates works in StatefulSet

```
StatefulSet with replicas: 3 and volumeClaimTemplate named "data"
         ↓ Creates automatically:
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  postgres-0  │    │  postgres-1  │    │  postgres-2  │
│      +       │    │      +       │    │      +       │
│ data-postgres│    │ data-postgres│    │ data-postgres│
│      -0      │    │      -1      │    │      -2      │
│   (10Gi PVC) │    │   (10Gi PVC) │    │   (10Gi PVC) │
└──────────────┘    └──────────────┘    └──────────────┘

PVC naming formula:  {volumeClaimTemplate.name}-{statefulset.name}-{ordinal}
                      data                    -postgres             -0
```

> ⚠️ **Critical:** When you delete a StatefulSet, **PVCs are NOT deleted automatically**.
> This is intentional — your data is safe. You must manually delete PVCs if you want to
> remove the data.

### Check PVC status

```bash
# List all PVCs
kubectl get pvc -n default

# Output:
# NAME               STATUS   VOLUME           CAPACITY   STORAGECLASS
# data-postgres-0    Bound    pvc-abc123...    10Gi       fast-ssd
# data-postgres-1    Bound    pvc-def456...    10Gi       fast-ssd

# PVC must be "Bound" before the pod can start
# "Pending" means no PV available or StorageClass not working

# Describe PVC to debug
kubectl describe pvc data-postgres-0 -n default
```

---

## Part D — Headless Services: Stable Network Identity

### Why StatefulSets need a Headless Service

A normal Service load-balances across all pods — you get one IP, traffic goes to any pod.
A **Headless Service** gives each pod its own individual DNS entry.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: default
spec:
  clusterIP: None # ← THIS makes it headless. "None" = no cluster IP assigned
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

### DNS entries created by a Headless Service

```
Normal Service DNS:
  postgres-svc.default.svc.cluster.local  →  10.96.x.x (virtual IP, load balanced)

Headless Service DNS (one entry PER pod):
  postgres-0.postgres-headless.default.svc.cluster.local  →  10.244.0.5 (pod-0's real IP)
  postgres-1.postgres-headless.default.svc.cluster.local  →  10.244.0.6 (pod-1's real IP)
  postgres-2.postgres-headless.default.svc.cluster.local  →  10.244.0.7 (pod-2's real IP)

DNS formula:
  {pod-name}.{headless-service-name}.{namespace}.svc.cluster.local
```

### Why this matters for databases

```
PostgreSQL primary-replica setup:
  postgres-0  →  PRIMARY  (accepts writes)
  postgres-1  →  REPLICA  (reads only, replicates from postgres-0)

Replica needs to connect to primary specifically:
  PRIMARY_HOST = postgres-0.postgres-headless.default.svc.cluster.local

This DNS name NEVER changes — even if postgres-0 pod is deleted and rescheduled,
it comes back as postgres-0 with the same DNS name and the same PVC.
Without headless service, you'd have to use an IP address which changes on every restart.
```

### Both a headless AND a regular service (common pattern)

```yaml
# Headless service — for pod-to-pod communication within the cluster
# Replicas connect to primary using this
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector: { app: postgres }
  ports:
    - port: 5432

---
# Regular service — for external connections to primary
# Your app connects to postgres using this
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
    role: primary # only route to the primary pod
  ports:
    - port: 5432

---
# Read service — routes to replicas only
apiVersion: v1
kind: Service
metadata:
  name: postgres-read
spec:
  selector:
    app: postgres
    role: replica
  ports:
    - port: 5432
```

---

## Part E — StatefulSet YAML: Full Breakdown

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres
    version: "16"

spec:
  # ── Identity ──────────────────────────────────────────────────────────────
  serviceName: postgres-headless # REQUIRED: must match headless service name
  replicas: 3 # creates postgres-0, postgres-1, postgres-2

  # ── Pod management ────────────────────────────────────────────────────────
  podManagementPolicy: OrderedReady # OrderedReady (default) | Parallel
  # OrderedReady: wait for each pod to be Running+Ready before starting the next
  # Parallel: start/stop all pods simultaneously (for apps that don't need ordering)

  # ── Update strategy ───────────────────────────────────────────────────────
  updateStrategy:
    type: RollingUpdate # RollingUpdate | OnDelete
    rollingUpdate:
      partition:
        0 # only update pods with ordinal >= partition
        # partition: 2 = only update postgres-2, not 0 or 1
        # great for canary updates on stateful apps

  # ── Pod selector ──────────────────────────────────────────────────────────
  selector:
    matchLabels:
      app: postgres # must match template labels

  # ── Pod template ──────────────────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: postgres
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9187"

    spec:
      # ── Termination ───────────────────────────────────────────────────────
      terminationGracePeriodSeconds: 60 # time to cleanly shutdown (important for DBs)

      # ── Security ──────────────────────────────────────────────────────────
      securityContext:
        runAsUser: 999 # postgres user
        runAsGroup: 999
        fsGroup: 999 # volume files owned by this group

      # ── Service account ───────────────────────────────────────────────────
      serviceAccountName: postgres-sa

      # ── Init containers (run before main containers) ──────────────────────
      initContainers:
        - name: init-permissions
          image: busybox:1.36
          command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data"]
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data

      # ── Main containers ───────────────────────────────────────────────────
      containers:
        - name: postgres
          image: postgres:16-alpine
          imagePullPolicy: IfNotPresent

          # ── Ports ──────────────────────────────────────────────────────────
          ports:
            - name: postgres
              containerPort: 5432
              protocol: TCP

          # ── Environment variables ─────────────────────────────────────────
          env:
            - name: POSTGRES_DB
              value: "myapp"
            - name: POSTGRES_USER
              value: "myapp"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: POD_NAME # inject pod name — use to detect if primary
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata # avoids "lost+found" issue

          # ── Resource limits ───────────────────────────────────────────────
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"

          # ── Volume mounts ─────────────────────────────────────────────────
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /etc/postgresql/postgresql.conf
              subPath: postgresql.conf # mount single file, not whole directory
            - name: init-scripts
              mountPath: /docker-entrypoint-initdb.d

          # ── Health checks ─────────────────────────────────────────────────
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myapp
                - -d
                - myapp
            initialDelaySeconds: 30 # wait before first check (DB takes time to start)
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myapp
                - -d
                - myapp
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          startupProbe: # gives extra time on first start
            exec:
              command: ["pg_isready", "-U", "myapp"]
            failureThreshold: 30 # 30 * 10s = 5 minutes for initial startup
            periodSeconds: 10

          # ── Lifecycle hooks ──────────────────────────────────────────────
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - pg_ctl stop -D /var/lib/postgresql/data/pgdata -m fast

      # ── Non-PVC volumes (config, scripts, etc.) ───────────────────────────
      volumes:
        - name: config
          configMap:
            name: postgres-config
        - name: init-scripts
          configMap:
            name: postgres-init-scripts

      # ── Scheduling ────────────────────────────────────────────────────────
      affinity:
        podAntiAffinity: # spread pods across nodes (not two DBs on same node)
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: ["postgres"]
              topologyKey: kubernetes.io/hostname

  # ── Storage ───────────────────────────────────────────────────────────────
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: postgres
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

---

## Part F — Scaling, Updates, and Rollbacks

### Scaling up

```bash
# Scale from 1 to 3 replicas
kubectl scale statefulset postgres --replicas=3 -n production

# Pods created in order: postgres-1 (waits for ready), then postgres-2
# Watch it happen:
kubectl get pods -w -n production
```

### Scaling down

```bash
# Scale from 3 to 1
kubectl scale statefulset postgres --replicas=1 -n production

# Pods deleted in REVERSE order: postgres-2 first, then postgres-1
# postgres-0 (primary) is LAST to be touched
# PVCs are NOT deleted — data-postgres-1 and data-postgres-2 still exist
```

### Rolling update

```bash
# Update the image
kubectl set image statefulset/postgres postgres=postgres:17-alpine -n production

# Pods updated in reverse order: postgres-2 → postgres-1 → postgres-0
# Each pod must be Running+Ready before the next one updates
# Watch the rollout:
kubectl rollout status statefulset/postgres -n production
```

### Canary update with partition

```bash
# Only update postgres-2 first (ordinal >= 2)
kubectl patch statefulset postgres -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Apply the image update
kubectl set image statefulset/postgres postgres=postgres:17-alpine

# Only postgres-2 gets the new image — test it
# If good, lower the partition to update postgres-1, then postgres-0
kubectl patch statefulset postgres -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'
kubectl patch statefulset postgres -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

### Rollback

```bash
# Check rollout history
kubectl rollout history statefulset/postgres -n production

# Rollback to previous version
kubectl rollout undo statefulset/postgres -n production

# Rollback to specific revision
kubectl rollout undo statefulset/postgres --to-revision=2 -n production
```

### OnDelete update strategy

```yaml
# Pods only update when YOU manually delete them — full control
updateStrategy:
  type: OnDelete
```

```bash
# With OnDelete, you control exactly when each pod updates
kubectl delete pod postgres-2 -n production  # it restarts with new image
kubectl delete pod postgres-1 -n production  # now update this one
kubectl delete pod postgres-0 -n production  # finally the primary
```

---

## Part G — Init Containers and Pod Ordering

### Init containers for stateful apps

Init containers run **sequentially before** the main container starts — essential for:

- Setting up replication config
- Checking if primary is ready
- Fixing file permissions
- Running schema migrations

```yaml
initContainers:
  # Step 1: Fix storage permissions
  - name: fix-permissions
    image: busybox:1.36
    command:
      - sh
      - -c
      - |
        chown -R 999:999 /var/lib/postgresql/data
        chmod 700 /var/lib/postgresql/data
    volumeMounts:
      - name: data
        mountPath: /var/lib/postgresql/data

  # Step 2: Detect role (primary or replica) based on pod ordinal
  - name: detect-role
    image: postgres:16-alpine
    command:
      - sh
      - -c
      - |
        # Extract ordinal from pod name (postgres-0 → 0)
        ORDINAL=$(echo $POD_NAME | rev | cut -d- -f1 | rev)

        if [ "$ORDINAL" = "0" ]; then
          echo "primary" > /etc/postgres-role/role
          echo "I am the PRIMARY"
        else
          echo "replica" > /etc/postgres-role/role
          echo "I am a REPLICA, will replicate from postgres-0"
          # Wait for primary to be ready
          until pg_isready -h postgres-0.postgres-headless -U myapp; do
            echo "Waiting for primary..."; sleep 2
          done
        fi
    env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
    volumeMounts:
      - name: role-config
        mountPath: /etc/postgres-role
```

### Ordered startup in practice

```
StatefulSet starts 3 replicas:

Time 0:   postgres-0 starts (init containers run, then main container)
Time 30s: postgres-0 passes readinessProbe → Ready
Time 31s: postgres-1 starts (init container waits for postgres-0 headless DNS)
Time 60s: postgres-1 passes readinessProbe → Ready
Time 61s: postgres-2 starts
Time 90s: postgres-2 → Ready
Time 91s: StatefulSet fully ready ✅

Compare with podManagementPolicy: Parallel (for apps that don't need ordering):
Time 0:   postgres-0, postgres-1, postgres-2 ALL start simultaneously
Time 30s: All ready at the same time ✅ (but no ordering guarantee)
```

---

## Part H — Real-World Examples

### Example 1: PostgreSQL Primary-Replica

```yaml
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  password: "StrongPassword123!"
  replication-password: "ReplicaPassword456!"

---
# ConfigMap with PostgreSQL config
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 200
    shared_buffers = 256MB
    wal_level = replica
    max_wal_senders = 3
    max_replication_slots = 3
    hot_standby = on

  pg_hba.conf: |
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            md5
    host    all             all             0.0.0.0/0               md5
    host    replication     replicator      0.0.0.0/0               md5

---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector: { app: postgres }
  ports:
    - port: 5432

---
# Regular Service (for app connections to primary)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - port: 5432

---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 2
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
        - name: setup-replication
          image: postgres:16-alpine
          command:
            - bash
            - -c
            - |
              ORDINAL=$(echo $POD_NAME | rev | cut -d- -f1 | rev)
              if [ "$ORDINAL" = "0" ]; then
                echo "primary" > /tmp/role
              else
                echo "replica" > /tmp/role
                # Wait for primary
                until pg_isready -h postgres-0.postgres-headless -U myapp -d myapp; do
                  sleep 2; echo "Waiting for primary..."
                done
              fi
              cp /tmp/role /etc/role/role
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: role
              mountPath: /etc/role

      containers:
        - name: postgres
          image: postgres:16-alpine
          command:
            - bash
            - -c
            - |
              ROLE=$(cat /etc/role/role)
              if [ "$ROLE" = "primary" ]; then
                # Add replica user
                export POSTGRES_INITDB_ARGS="--auth-host=md5"
                docker-entrypoint.sh postgres \
                  -c config_file=/etc/postgresql/postgresql.conf \
                  -c hba_file=/etc/postgresql/pg_hba.conf
              else
                # Setup replica from primary
                pg_basebackup -h postgres-0.postgres-headless \
                  -U replicator -p 5432 -D /var/lib/postgresql/data/pgdata \
                  -Fp -Xs -P -R
                postgres -c config_file=/etc/postgresql/postgresql.conf
              fi
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              value: myapp
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "myapp", "-d", "myapp"]
            initialDelaySeconds: 15
            periodSeconds: 5
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /etc/postgresql
            - name: role
              mountPath: /etc/role

      volumes:
        - name: config
          configMap:
            name: postgres-config
        - name: role
          emptyDir: {}

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

---

### Example 2: Redis Cluster (3 Masters + 3 Replicas)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file /data/nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    appendfsync everysec
    maxmemory 512mb
    maxmemory-policy allkeys-lru
    protected-mode no

---
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None
  selector: { app: redis }
  ports:
    - name: client
      port: 6379
    - name: gossip
      port: 16379

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 6 # 3 masters + 3 replicas
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command: ["redis-server", "/etc/redis/redis.conf"]
          ports:
            - containerPort: 6379
            - containerPort: 16379
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          readinessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 30
            periodSeconds: 10
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /etc/redis

      volumes:
        - name: config
          configMap:
            name: redis-config

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
```

---

### Example 3: MongoDB ReplicaSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-headless
spec:
  clusterIP: None
  selector: { app: mongo }
  ports:
    - port: 27017

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo-headless
  replicas: 3
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      initContainers:
        - name: wait-for-dns
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Wait for headless DNS to propagate
              until nslookup mongo-0.mongo-headless; do
                echo "Waiting for DNS..."; sleep 2
              done
      containers:
        - name: mongo
          image: mongo:7
          command:
            - mongod
            - --replSet
            - rs0
            - --bind_ip_all
            - --auth
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: admin
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
          ports:
            - containerPort: 27017
          readinessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 15
            periodSeconds: 10
          volumeMounts:
            - name: data
              mountPath: /data/db

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi
```

---

## Part I — Backup, Restore, and Disaster Recovery

### Backup strategy 1: Volume snapshots (cloud native)

```yaml
# VolumeSnapshot — point in time snapshot of a PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-backup-2024-01-15
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: data-postgres-0
```

```bash
# List snapshots
kubectl get volumesnapshot -n production

# Restore from snapshot — create a new PVC from snapshot
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
spec:
  dataSource:
    name: postgres-backup-2024-01-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
EOF
```

### Backup strategy 2: pg_dump via CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *" # 2am daily
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16-alpine
              command:
                - sh
                - -c
                - |
                  DATE=$(date +%Y%m%d-%H%M%S)
                  pg_dump -h postgres-0.postgres-headless \
                    -U myapp -d myapp \
                    -Fc -f /backup/postgres-$DATE.dump
                  echo "Backup completed: postgres-$DATE.dump"
                  # Keep only last 7 backups
                  ls -t /backup/*.dump | tail -n +8 | xargs rm -f
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: postgres-backup-pvc
```

### Restore procedure

```bash
# 1. Scale down the StatefulSet (stop writes)
kubectl scale statefulset postgres --replicas=0 -n production

# 2. Restore the dump to primary
kubectl run restore --rm -it --image=postgres:16-alpine -- bash
# Inside the pod:
pg_restore -h postgres-0.postgres-headless -U myapp -d myapp \
  -Fc /backup/postgres-20240115-020000.dump

# 3. Scale back up
kubectl scale statefulset postgres --replicas=3 -n production
```

---

## Part J — Advanced Patterns

### Pod Anti-Affinity (spread pods across nodes)

```yaml
affinity:
  podAntiAffinity:
    # HARD rule: never put two postgres pods on the same node
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: postgres
        topologyKey: kubernetes.io/hostname # "hostname" = one per node

    # SOFT rule: try to spread across availability zones
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: postgres
          topologyKey: topology.kubernetes.io/zone
```

### Resource Quotas for stateful namespaces

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: stateful-quota
  namespace: production
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    persistentvolumeclaims: "10"
    requests.storage: 500Gi
```

### Pod Disruption Budget (PDB) — protect your quorum

```yaml
# Ensure at least 2 pods always remain during voluntary disruptions
# (node drains, cluster upgrades)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2 # always keep at least 2 running
  # OR:
  # maxUnavailable: 1          # at most 1 can be down at a time
  selector:
    matchLabels:
      app: postgres
```

```bash
# Without PDB: a node drain could take down 2 of 3 postgres pods
# losing quorum and making the cluster unavailable

# With PDB: kubectl drain will wait until evicting a pod won't violate the budget
kubectl drain my-node --ignore-daemonsets --delete-emptydir-data
# This will block if draining would bring postgres below minAvailable: 2
```

### Sidecar pattern: exporter for monitoring

```yaml
containers:
  - name: postgres
    image: postgres:16-alpine
    # ... main container

  - name: postgres-exporter # sidecar: exposes metrics for Prometheus
    image: prometheuscommunity/postgres-exporter:v0.15.0
    env:
      - name: DATA_SOURCE_NAME
        value: "postgresql://myapp:$(POSTGRES_PASSWORD)@localhost:5432/myapp?sslmode=disable"
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretKeyRef:
            name: postgres-secret
            key: password
    ports:
      - containerPort: 9187
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
```

---

## Part K — Common Mistakes and Fixes

### Mistake 1: Forgetting PGDATA subpath

```yaml
# ❌ Wrong — PostgreSQL will fail: "data directory has wrong ownership"
# The /var/lib/postgresql/data directory contains a lost+found from the volume
volumeMounts:
  - name: data
    mountPath: /var/lib/postgresql/data

# ✅ Correct — use a subdirectory
env:
  - name: PGDATA
    value: /var/lib/postgresql/data/pgdata
volumeMounts:
  - name: data
    mountPath: /var/lib/postgresql/data
```

### Mistake 2: Deleting a StatefulSet removes pods but NOT PVCs

```bash
# You deleted the StatefulSet and re-created it
kubectl delete statefulset postgres
kubectl apply -f statefulset.yaml

# But now pods are stuck in Pending because old PVCs still exist
# from the previous StatefulSet and are already "Bound"

# Fix: check if old PVCs exist
kubectl get pvc -n production

# If they have correct data, it's actually fine — pods will reattach
# If you want fresh PVCs, delete the old ones first:
kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2
```

### Mistake 3: Wrong serviceName in StatefulSet

```yaml
# ❌ serviceName doesn't match the actual headless service name
spec:
  serviceName: postgres        # actual headless service is named "postgres-headless"

# Result: Pod DNS names won't be generated correctly
# postgres-0.postgres.default.svc.cluster.local → won't resolve

# ✅ Must match exactly
spec:
  serviceName: postgres-headless
```

### Mistake 4: Scaling down destroys the primary

```bash
# ❌ Wrong: scaling down from 3 to 0 directly
# K8s deletes: postgres-2, postgres-1, postgres-0 (primary last, but still deleted)
kubectl scale statefulset postgres --replicas=0

# ✅ For maintenance, keep at least 1 (the primary)
kubectl scale statefulset postgres --replicas=1
# Now only replicas are terminated, primary stays up
```

### Mistake 5: No readinessProbe causes cascading failures

```yaml
# ❌ Without readinessProbe, postgres-1 starts before postgres-0 is actually ready
# Replica tries to connect to primary that isn't ready yet → fails

# ✅ Always define readinessProbe on stateful pods
readinessProbe:
  exec:
    command: ["pg_isready", "-U", "myapp"]
  initialDelaySeconds: 15
  periodSeconds: 5
  failureThreshold: 3
# StatefulSet won't start postgres-1 until postgres-0 passes this probe
```

### Mistake 6: Using ReadWriteMany when you need ReadWriteOnce

```yaml
# ❌ Wrong for databases — two pods writing to same volume = corruption
accessModes:
  - ReadWriteMany    # multiple pods can write = data corruption for DBs

# ✅ Each pod needs its own exclusive volume
accessModes:
  - ReadWriteOnce    # only one pod mounts this for read/write

# ReadWriteMany is only correct for:
# - Shared config files (read by many)
# - NFS-backed shared storage
# - Log aggregation volumes
```

---

## Quick Reference Commands

```bash
# ── Status ────────────────────────────────────────────────────────────────
kubectl get statefulset -n production
kubectl get pods -l app=postgres -n production
kubectl get pvc -n production

# ── Describe (debug) ──────────────────────────────────────────────────────
kubectl describe statefulset postgres -n production
kubectl describe pod postgres-0 -n production
kubectl describe pvc data-postgres-0 -n production

# ── Logs ──────────────────────────────────────────────────────────────────
kubectl logs postgres-0 -n production
kubectl logs postgres-0 -n production --previous    # logs from crashed container
kubectl logs postgres-0 -n production -f            # stream live logs

# ── Exec into a pod ───────────────────────────────────────────────────────
kubectl exec -it postgres-0 -n production -- psql -U myapp -d myapp
kubectl exec -it postgres-0 -n production -- bash

# ── Scale ─────────────────────────────────────────────────────────────────
kubectl scale statefulset postgres --replicas=3 -n production

# ── Rollout ───────────────────────────────────────────────────────────────
kubectl rollout status statefulset/postgres -n production
kubectl rollout history statefulset/postgres -n production
kubectl rollout undo statefulset/postgres -n production

# ── Delete (PVCs survive) ─────────────────────────────────────────────────
kubectl delete statefulset postgres -n production
kubectl delete pvc data-postgres-0 data-postgres-1 -n production   # manual cleanup

# ── Force delete a stuck pod ─────────────────────────────────────────────
kubectl delete pod postgres-0 --grace-period=0 --force -n production
```

---

## The Mental Model

```
Deployment  = cattle  → any pod, any node, any storage, replaceable
StatefulSet = pets    → specific pod (postgres-0), specific node (via PVC binding),
                        specific storage (data-postgres-0), stable DNS name

Three guarantees StatefulSet gives you:
1. Stable identity   → postgres-0 is always postgres-0
2. Stable storage    → data-postgres-0 always attaches to postgres-0
3. Stable network    → postgres-0.postgres-headless.default.svc.cluster.local never changes

These three guarantees are why you can build
distributed databases, consensus systems, and
message queues on top of Kubernetes StatefulSets.
```
