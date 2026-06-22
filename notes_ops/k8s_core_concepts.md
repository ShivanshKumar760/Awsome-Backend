# Kubernetes Notes — Pods, Deployments, Services, Storage & StatefulSets

Builds directly on the Instagram API deployment you already wrote manifests for — every concept below is illustrated using `instagram-api-deployment`, `postgres-deployment`, `postgres-pvc`, etc. from that setup.

---

## Table of Contents

1. Pod
2. ReplicaSet
3. Deployment (and how it owns a ReplicaSet)
4. Service
5. Types of Service (ClusterIP, NodePort, LoadBalancer, ExternalName)
6. Headless vs Headful (ClusterIP) Service
7. HPA — HorizontalPodAutoscaler
8. Volumes & Storage in Kubernetes (types, PV, PVC, StorageClass)
9. StatefulSet (and why our Postgres setup deliberately avoided it)

---

## 1. Pod

A **Pod** is the smallest deployable unit in Kubernetes — **not** a single container, but a wrapper around **one or more containers that are always scheduled together, on the same Node, sharing the same network namespace and (optionally) storage volumes.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: instagram-api-pod
  labels:
    app: instagram-api
spec:
  containers:
    - name: instagram-api
      image: your-dockerhub-username/instagram-api:latest
      ports:
        - containerPort: 5000
```

**Key properties:**

- **Shared network:** every container in a Pod shares one IP address and one port space — they talk to each other via `localhost`. This is why a common pattern is a "sidecar" container (e.g. a log shipper) living alongside your main app container in the same Pod.
- **Shared storage (optional):** Volumes defined at the Pod level can be mounted into multiple containers in that Pod.
- **Ephemeral and disposable:** Pods are **not** meant to be created directly and babysat by hand. If a Pod dies, it's gone — nothing recreates it on its own. This is precisely why you essentially never write a bare `kind: Pod` manifest for a real app; you let a **controller** (ReplicaSet, Deployment, StatefulSet) manage Pods for you, so they get recreated automatically.
- **Gets its own IP**, but that IP is **not stable** — if the Pod is deleted and recreated, it gets a brand-new IP. This is exactly why **Services** (Section 4) exist — to give a stable address in front of Pods whose individual IPs keep changing.

In our Instagram API setup, you never see a bare `kind: Pod` — every Pod is created and supervised indirectly, via the `template:` section inside a Deployment.

---

## 2. ReplicaSet

A **ReplicaSet's** only job: **ensure a specified number of identical Pod replicas are running at all times.** If a Pod backed by a ReplicaSet crashes or is deleted, the ReplicaSet notices the count dropped below the desired number and immediately creates a replacement.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: instagram-api-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: instagram-api
  template:
    metadata:
      labels:
        app: instagram-api
    spec:
      containers:
        - name: instagram-api
          image: your-dockerhub-username/instagram-api:latest
          ports:
            - containerPort: 5000
```

**How matching works:** `selector.matchLabels` tells the ReplicaSet "any Pod with this label belongs to me, and I'm responsible for keeping exactly `replicas` of them alive." The `template` is the blueprint used to stamp out new Pods whenever the actual count is below the desired count.

**What a ReplicaSet does NOT do:**
- It does not give you rolling updates. If you change the `image:` in a ReplicaSet's template, **existing Pods are not updated** — only newly created ones (e.g. after a crash) would use the new image. There's no controlled rollout.
- This gap is exactly why you almost never write a `kind: ReplicaSet` manifest directly — you use a **Deployment** instead, which manages ReplicaSets *for* you and adds rollout/rollback behavior on top.

---

## 3. Deployment (and How It Owns a ReplicaSet)

A **Deployment** is a higher-level controller that manages **ReplicaSets**, which in turn manage **Pods**. The chain of ownership is:

```
Deployment  →  manages  →  ReplicaSet  →  manages  →  Pod(s)
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: instagram-api-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: instagram-api
  template:
    metadata:
      labels:
        app: instagram-api
    spec:
      containers:
        - name: instagram-api
          image: your-dockerhub-username/instagram-api:latest
          ports:
            - containerPort: 5000
```

Notice this manifest is **nearly identical** to the ReplicaSet example above — the difference isn't really in the YAML shape, it's in what the controller *does* with it.

**What a Deployment adds on top of a ReplicaSet — rolling updates:**

When you change the Pod template (e.g. bump the image tag), a Deployment doesn't edit the existing ReplicaSet's Pods in place. Instead, it:

1. Creates a **brand-new ReplicaSet** with the updated template (`replicas: 0` initially).
2. Gradually scales the **new** ReplicaSet up while scaling the **old** ReplicaSet down — governed by `maxSurge`/`maxUnavailable` (exactly as configured in the Instagram API's `RollingUpdate` strategy from the earlier deployment guide).
3. Once the new ReplicaSet is fully scaled up and the old one is fully scaled down to 0, the old ReplicaSet is kept around (but empty) — this is what enables `kubectl rollout undo`, an instant rollback to the previous ReplicaSet's template if something goes wrong.

**Seeing the ownership chain yourself:**

```bash
kubectl get deployments
# NAME                         READY   UP-TO-DATE   AVAILABLE
# instagram-api-deployment     2/2     2            2

kubectl get replicasets
# NAME                                    DESIRED   CURRENT   READY
# instagram-api-deployment-7d8f9c6b5d     2         2         2

kubectl get pods
# NAME                                          READY   STATUS
# instagram-api-deployment-7d8f9c6b5d-abc12     1/1     Running
# instagram-api-deployment-7d8f9c6b5d-xyz89     1/1     Running
```

Notice the **naming pattern**: the ReplicaSet's name is `<deployment-name>-<hash>`, and each Pod's name is `<replicaset-name>-<random-suffix>`. That hash in the ReplicaSet's name is generated from the Pod template — every time you change the template meaningfully (new image, new env var, etc.), Kubernetes computes a new hash, which is precisely how it knows to spin up a *new* ReplicaSet rather than reuse the old one.

**Rule of thumb:** You write Deployments. Kubernetes writes (and rewrites) ReplicaSets for you, as an implementation detail of how rollouts work. You almost never touch a ReplicaSet directly.

---

## 4. Service

Pods are disposable and their IPs change constantly (every restart/reschedule = new IP). A **Service** solves this by giving a stable, unchanging **virtual IP and DNS name** in front of a *set* of Pods, selected by label — and load-balances traffic across whichever Pods currently match.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: instagram-api-service
spec:
  selector:
    app: instagram-api   # routes to any Pod with this label, regardless of which Deployment/ReplicaSet created it
  ports:
    - port: 80            # the Service's own port
      targetPort: 5000      # the port the container actually listens on
```

**How traffic actually flows:** `Client → Service (stable virtual IP, name = instagram-api-service) → one of the matching Pods (real, changing IPs)`. Other Pods in the cluster reach this purely by DNS name — `instagram-api-service` resolves automatically via Kubernetes' built-in DNS (CoreDNS), with zero manual IP tracking required, which is exactly how the Flask API found Postgres via `DB_HOST=postgres-service` in the earlier setup.

**Service vs Deployment — these are two completely separate concerns:**

| | Deployment | Service |
|---|---|---|
| Manages | Which Pods exist, how many, how they're updated | How traffic gets routed to whichever Pods currently exist |
| Selector matches | Pods it's responsible for creating | Pods it's responsible for routing to (created by anyone) |
| Without it | No Pods would exist at all | Pods would exist but have no stable address |

A Service's `selector` matching a Deployment's Pod labels is a **coincidental connection by label**, not a direct reference — a Service has no idea which Deployment (if any) created the Pods it's routing to. This is intentional and flexible: a single Service could front Pods from multiple different Deployments, as long as they share the matching label.

---

## 5. Types of Service

```yaml
spec:
  type: ClusterIP   # <- this field controls everything below
```

| Type | Reachable from | Typical use |
|---|---|---|
| **`ClusterIP`** (default) | Inside the cluster only | Internal services — exactly what `postgres-service` uses. Postgres should never be reachable from outside the cluster. |
| **`NodePort`** | Outside the cluster, via `<any-node-ip>:<a-fixed-port-30000-32767>` | Quick external access for local clusters (minikube/kind) without a cloud load balancer available. |
| **`LoadBalancer`** | Outside the cluster, via a real public IP | Production external access on a cloud provider (EKS/GKE/AKS) — provisions an actual cloud load balancer automatically. This is what `instagram-api-service` uses. |
| **`ExternalName`** | N/A — it's a DNS alias, not a proxy | Maps a Service name to an external DNS name (e.g. `my-external-db.example.com`) so in-cluster code can use a clean internal name instead of hardcoding an external hostname. |

**`ClusterIP` (internal only):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP    # this is also the default if `type` is omitted entirely
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**`NodePort` (exposes a fixed port on every Node):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: instagram-api-nodeport
spec:
  type: NodePort
  selector:
    app: instagram-api
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30080   # optional — Kubernetes picks one in the 30000-32767 range if omitted
```

Reachable at `http://<any-node-ip>:30080` — useful for local development clusters where you don't have a real cloud load balancer to provision.

**`LoadBalancer` (provisions a real external load balancer):**

```yaml
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

On a managed cloud cluster, this automatically creates a cloud load balancer (an AWS ELB, GCP Load Balancer, etc.) with a real public IP — `kubectl get svc` shows it under `EXTERNAL-IP` once provisioning finishes. Note that `LoadBalancer` is really `NodePort` + an extra cloud-provisioned layer on top, not a separate mechanism — every `LoadBalancer` Service also has a `NodePort` under the hood.

**`ExternalName` (DNS alias, no proxying):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-postgres
spec:
  type: ExternalName
  externalName: my-managed-db.cloudprovider.com
```

No `selector`, no `ports`, no actual traffic proxying — this Service is purely a DNS CNAME record. Any Pod doing a lookup for `external-postgres` gets redirected to `my-managed-db.cloudprovider.com` by the cluster's DNS. Useful when migrating from an external managed database to an in-cluster one (or vice versa) without changing application code.

---

## 6. Headless vs Headful (Normal `ClusterIP`) Service

This distinction is the one most people skip past, and it matters a lot for StatefulSets (Section 9).

### Headful Service (a normal `ClusterIP` Service)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: instagram-api-service
spec:
  type: ClusterIP   # "headful" = has a real cluster-assigned virtual IP
  selector:
    app: instagram-api
  ports:
    - port: 80
      targetPort: 5000
```

When you do a DNS lookup for `instagram-api-service` from inside the cluster, you get back **one single virtual IP** — the Service's own ClusterIP. Kubernetes' internal load-balancing (via `kube-proxy`) then transparently spreads traffic across whichever real Pods currently match the selector. The client never sees individual Pod IPs — it just sees one stable address, and load balancing happens invisibly behind it. This is the right behavior when your Pods are **interchangeable** — any Pod can serve any request equally well (exactly true of our stateless Flask API Pods).

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None    # <- THIS is what makes it headless
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Setting `clusterIP: None` tells Kubernetes: **don't allocate a virtual IP or do load balancing at all.** Instead, a DNS lookup for `postgres-headless` returns the **individual IP addresses of every matching Pod directly** (an "A record" per Pod, rather than one IP for a virtual load balancer).

**Why you'd ever want this:** when Pods are **not interchangeable** — when a client genuinely needs to reach **one specific Pod by its own identity**, not "any Pod, load balanced." This matters when:
- Pods in a database cluster have different roles (e.g. one primary that accepts writes, several replicas that only serve reads) — a client (or another Pod) needs to specifically reach the primary, not a random one.
- Peer-to-peer cluster software (Cassandra, Elasticsearch, Kafka, etcd, ZooKeeper) where each node needs to discover and talk to every *other specific* node by identity, to form a cluster — not be load-balanced to "some node."

This is also exactly why **StatefulSets are almost always paired with a headless Service**: StatefulSet gives each Pod a stable, predictable hostname (`postgres-0`, `postgres-1`, `postgres-2`), and a headless Service is what makes those individual hostnames resolvable via DNS in the form `postgres-0.postgres-headless`, `postgres-1.postgres-headless`, etc. — see Section 9.

| | Headful (`ClusterIP`) | Headless (`clusterIP: None`) |
|---|---|---|
| DNS lookup returns | One virtual IP (the Service's own) | Every matching Pod's real IP, individually |
| Load balancing | Yes, automatic, invisible to the client | No — caller chooses/discovers which Pod to use |
| Right for | Interchangeable, stateless Pods (our Flask API) | Pods with distinct identity/role (database cluster nodes) |
| Typically paired with | Deployment | StatefulSet |

---

## 7. HPA — HorizontalPodAutoscaler

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

**What it does:** continuously watches real resource usage (via the **metrics-server** add-on) across all Pods belonging to a target Deployment/ReplicaSet/StatefulSet, and automatically edits the target's `replicas` field up or down to keep usage near your target — "horizontal" because it changes the **number of Pods**, as opposed to "vertical" scaling (giving each Pod more CPU/memory, which is what a `VerticalPodAutoscaler` does instead).

**The actual mechanism, end to end:**

```
metrics-server reports: avg CPU across Pods = 85%
        ↓
HPA controller compares: 85% > target (60%)
        ↓
HPA computes new desired replica count, e.g. scales replicas: 3 -> 5
        ↓
HPA edits the Deployment's `replicas` field directly
        ↓
Deployment's ReplicaSet creates 2 new Pods to match
```

This is exactly why HPA only targets things that manage **interchangeable** Pods (Deployments, ReplicaSets, StatefulSets where it makes sense) — it works purely by changing a replica *count*, which is meaningless for a single-instance resource like our `postgres-deployment` (`replicas: 1`, pinned to one PVC). Scaling that to 2 wouldn't add capacity, it would just create a second Pod that can't mount the same `ReadWriteOnce` disk and gets stuck `Pending` forever.

**`minReplicas`/`maxReplicas`** set the floor/ceiling HPA will never go outside of, regardless of load — protecting you from scaling to 0 (outage) or runaway scaling (cost explosion from a bug causing a CPU spike).

---

## 8. Volumes & Storage in Kubernetes

### The core problem volumes solve

A container's own filesystem is **ephemeral** — anything written there disappears the moment the container restarts (even just a normal restart, not a crash). **Volumes** are how you give a Pod storage that survives beyond a single container's lifetime.

### Types of volumes, by how long they live

| Volume type | Lifetime | Example use |
|---|---|---|
| **`emptyDir`** | Tied to the **Pod** — survives container restarts within the same Pod, but deleted when the Pod itself is deleted | Scratch space, caching, sharing files between containers in the same Pod |
| **`hostPath`** | Tied to a specific **Node's** local disk | Local development/testing only — ties your Pod to one specific Node, breaks if rescheduled elsewhere. Avoid in production. |
| **`persistentVolumeClaim` (PVC)** | Independent of any Pod *or* Node — survives Pod deletion, Pod rescheduling, even cluster upgrades | Real persistent data: our Postgres database files. This is the one that matters for stateful apps. |
| **`configMap` / `secret`** | Read-only, tied to the ConfigMap/Secret object | Mounting config files or credentials into a container as actual files (alternative to env vars) |

```yaml
# emptyDir example — scratch space, gone when the Pod is deleted
volumes:
  - name: scratch-space
    emptyDir: {}
```

### `PersistentVolume` (PV) and `PersistentVolumeClaim` (PVC) — the real persistence layer

This is a two-sided system, deliberately decoupling "what storage actually exists" from "what a Pod is asking for":

- **`PersistentVolume` (PV):** represents an actual piece of real storage in the cluster — a cloud disk (AWS EBS, GCP Persistent Disk), an NFS share, etc. Usually **not** written by app developers directly — it's provisioned automatically by a `StorageClass` (below), or manually by a cluster admin for static/manual provisioning.
- **`PersistentVolumeClaim` (PVC):** a **request** for storage, made by you/your app — "I need 2Gi, mountable read-write by one Node." Kubernetes finds (or dynamically creates) a PV that satisfies the claim and **binds** them together.

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

```yaml
# In the Pod template:
volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc   # binds this Pod to whatever PV satisfies postgres-pvc
containers:
  - volumeMounts:
      - name: postgres-storage
        mountPath: /var/lib/postgresql/data
```

**Access modes** (how many Nodes can mount the volume, and how):

| Access mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | One Node can mount it read-write at a time. The right choice for a single-instance database (our Postgres setup). |
| `ReadOnlyMany` (ROX) | Many Nodes can mount it, but read-only. |
| `ReadWriteMany` (RWX) | Many Nodes can mount it read-write simultaneously. Needs a storage backend that supports this (NFS, some cloud filesystems) — most basic cloud block storage (EBS, etc.) does **not** support RWX. |
| `ReadWriteOncePod` (RWOP, newer) | Stricter than RWO — guarantees only a single **Pod** (not just single Node) can mount it read-write, even if multiple Pods are scheduled on the same Node. |

### `StorageClass` — dynamic provisioning

Without a `StorageClass`, someone (a cluster admin) would have to **manually** create a PV ahead of time for every PVC that needs one. A `StorageClass` automates this: it defines *how* to dynamically provision storage on demand, so a PVC creates its own matching PV automatically, on the fly.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com    # cloud-specific plugin that knows how to create real disks
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# Referencing it from a PVC:
spec:
  storageClassName: fast-ssd
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 2Gi
```

If you omit `storageClassName` entirely (as our `postgres-pvc` does), Kubernetes uses whatever StorageClass is marked **default** on your cluster — convenient for getting started, but worth being explicit about once you care about performance tiers (SSD vs HDD) or cost.

---

## 9. StatefulSet (and Why We Deliberately Avoided It)

A **StatefulSet** is the controller you reach for when you need **multiple replicas of a stateful app, where each replica needs a stable, unique identity and its own separate, persistent storage that follows it specifically — not shared, and not interchangeable with any other replica's storage.**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
spec:
  serviceName: postgres-headless   # MUST point to a headless Service (Section 6)
  replicas: 3
  selector:
    matchLabels:
      app: postgres-cluster
  template:
    metadata:
      labels:
        app: postgres-cluster
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # <- THE key difference from a Deployment
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
```

**What's genuinely different from a Deployment:**

| | Deployment | StatefulSet |
|---|---|---|
| Pod naming | Random suffix (`instagram-api-deployment-7d8f9c6b5d-abc12`) | Predictable, ordered (`postgres-cluster-0`, `postgres-cluster-1`, `postgres-cluster-2`) |
| Pod identity across restarts | None — any replacement Pod is identical/interchangeable | Stable — `postgres-cluster-0` is always recreated as `postgres-cluster-0`, with the *same* PVC reattached |
| Storage | One PVC, optionally shared/referenced by the whole Deployment | `volumeClaimTemplates` — Kubernetes automatically creates **one separate PVC per replica** (`data-postgres-cluster-0`, `data-postgres-cluster-1`, ...), and that exact PVC always reattaches to that exact replica |
| Scale-up/down order | Unordered — Kubernetes can create/destroy Pods in any order, all at once | Ordered — Pods are created `0, 1, 2, ...` in sequence (each waiting for the previous to be `Ready`) and scaled down in reverse order `2, 1, 0` |
| Pairs with | A normal (headful) Service | A **headless** Service, giving each Pod a stable DNS name like `postgres-cluster-0.postgres-headless` |

**Why our actual setup uses a plain `Deployment` with `replicas: 1` instead:** all of StatefulSet's value comes from managing **multiple** replicas that each need their **own** separate identity/storage. We run exactly **one** Postgres instance. There's no "which replica is this" question to answer (there's only one), no ordering concern (nothing to order), and no need for `volumeClaimTemplates` to auto-generate per-replica PVCs (we just reference one PVC directly, as covered in the earlier deployment guide). A StatefulSet here would add real conceptual and operational overhead for zero practical benefit.

**When you actually *would* reach for a StatefulSet in this same project:** if you outgrew a single Postgres instance and moved to a real **Postgres replication cluster** — one primary that accepts writes, plus several read replicas, each needing its own disk and a stable identity so the rest of the system can tell them apart (and specifically route writes to the primary via its predictable hostname) — that's the point where the table above stops being theoretical and starts being the only sane way to model the problem.

---

## Quick Reference Summary

| Concept | One-line purpose |
|---|---|
| Pod | Smallest deployable unit; one or more co-located, co-networked containers |
| ReplicaSet | Keeps N identical Pods running; no rollout logic of its own |
| Deployment | Manages ReplicaSets to add rolling updates/rollbacks on top of "keep N Pods alive" |
| Service | Stable virtual address in front of a changing set of Pods |
| ClusterIP | Internal-only Service (default) |
| NodePort | Exposes a fixed port on every Node — simple external access |
| LoadBalancer | Provisions a real cloud load balancer — production external access |
| ExternalName | Pure DNS alias to an external hostname, no proxying |
| Headful Service | One virtual IP, automatic load balancing — for interchangeable Pods |
| Headless Service | DNS returns each Pod's real IP individually — for Pods with distinct identity |
| HPA | Auto-scales replica count based on real CPU/memory usage |
| emptyDir | Pod-lifetime scratch storage |
| hostPath | Node-local storage — dev/testing only |
| PV / PVC | Decoupled "real storage that exists" vs "a request for storage" |
| StorageClass | Automates dynamic PV provisioning on demand |
| StatefulSet | Multiple replicas, each with a stable identity AND its own separate persistent storage, paired with a headless Service |
