# Kubernetes Manifests — Instagram API

All six manifests used to deploy the Flask Instagram API and its Postgres database to Kubernetes, collected in one file for easy reference. Apply individually (`kubectl apply -f k8s/<file>`) or all at once (`kubectl apply -f k8s/`).

Recommended apply order: `ConfigMap.yml` → `Secrets.yml` → `Pvc.yml` → `deployment.yml` → `Services.yml` → `Hpa.yml`. See `k8s_deployment_guide.md` for the full explanation of every field.

---

## `ConfigMap.yml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: instagram-config
  labels:
    app: instagram-api
data:
  # Non-secret configuration — anything that's fine to see in plaintext.
  # Pulled into containers as environment variables (see deployment.yml).
  DB_HOST: "postgres-service"   # the Postgres Service name — Kubernetes DNS resolves this automatically
  DB_PORT: "5432"
  DB_NAME: "instagram_clone"
  FLASK_ENV: "production"
```

---

## `Secrets.yml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: instagram-secret
  labels:
    app: instagram-api
type: Opaque
data:
  # Values here MUST be base64-encoded — Kubernetes does NOT encrypt this for you,
  # it's just an encoding, not real security. Anyone who can `kubectl get secret -o yaml`
  # can decode this instantly with `echo "<value>" | base64 -d`.
  # Generate these yourself with: echo -n "postgres" | base64
  DB_USER: cG9zdGdyZXM=          # "postgres"
  DB_PASSWORD: eW91cnBhc3N3b3Jk    # "yourpassword"
```

---

## `Pvc.yml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce   # only one Node can mount this for read/write at a time — fine, since we run a single Postgres Pod
  resources:
    requests:
      storage: 2Gi
  # storageClassName omitted on purpose -> uses your cluster's default StorageClass
  # (e.g. "standard" on most managed clusters). Set it explicitly if your cluster
  # has no default, e.g.: storageClassName: standard
```

---

## `deployment.yml`

```yaml
---
# ====================================================================
# POSTGRES DEPLOYMENT
# A plain Deployment, NOT a StatefulSet — by design.
#
# StatefulSet exists to manage MULTIPLE replicas of stateful pods that each
# need a stable identity and their OWN separate volume (pod-0 gets its own
# disk, pod-1 gets its own disk, etc, and they keep that same disk across
# restarts/rescheduling). We don't need any of that here: we are running
# exactly ONE Postgres pod, with ONE PVC, that pod is mounted to. A regular
# Deployment with replicas: 1 gives us that, with far less complexity.
# ====================================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  labels:
    app: postgres
spec:
  replicas: 1   # MUST stay at 1 — Postgres's own data files are not safe to
                 # share across multiple writers pointed at the same disk.
                 # Scaling this up would NOT give you more capacity, it would
                 # give you data corruption. (Real horizontal scaling for
                 # Postgres needs a different setup entirely — replication,
                 # not duplicate pods on one PVC.)
  strategy:
    type: Recreate   # IMPORTANT: must be Recreate, not the default RollingUpdate.
                       # RollingUpdate would try to start a NEW postgres pod
                       # before killing the old one — but our PVC is ReadWriteOnce,
                       # so only one pod can mount it at a time. Recreate kills
                       # the old pod fully before starting the new one, avoiding
                       # a deadlock where two pods both want the same disk.
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
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: instagram-config
                  key: DB_NAME
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: instagram-secret
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: instagram-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: pgdata   # avoids a known issue where Postgres dislikes
                                  # writing directly into a volume's root dir
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 15
            periodSeconds: 20
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc   # binds this pod to the PVC defined in Pvc.yml

---
# ====================================================================
# FLASK API DEPLOYMENT
# This one IS meant to scale horizontally — it's stateless (no local data,
# every request talks to Postgres over the network) — which is exactly
# why it's a good fit for the HPA defined in Hpa.yml.
# ====================================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: instagram-api-deployment
  labels:
    app: instagram-api
spec:
  replicas: 2   # starting point — the HPA will adjust this up/down automatically
  strategy:
    type: RollingUpdate   # safe here: stateless pods, no shared disk, so we
                            # CAN replace them one at a time with zero downtime
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
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
          image: your-dockerhub-username/instagram-api:latest   # replace with your built+pushed image
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: instagram-config   # injects DB_HOST, DB_PORT, DB_NAME, FLASK_ENV as env vars
            - secretRef:
                name: instagram-secret    # injects DB_USER, DB_PASSWORD as env vars
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /users
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /users
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 20
```

---

## `Hpa.yml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: instagram-api-hpa
  labels:
    app: instagram-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: instagram-api-deployment   # must match the Deployment name exactly
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # scale up once average CPU across pods exceeds 60% of its request
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120   # wait 2 min of sustained low usage before scaling down
                                          # — avoids flapping (scale up, down, up, down...)
    scaleUp:
      stabilizationWindowSeconds: 0      # scale up immediately when load spikes, no hesitation

# NOTE: the HPA only targets the Flask API Deployment, never the Postgres
# Deployment. Postgres is stateful and pinned to one PVC — autoscaling it
# would try to spin up a second pod fighting over the same ReadWriteOnce
# disk, which is exactly the failure mode the `Recreate` strategy and
# replicas: 1 in deployment.yml are protecting against.
```

---

## `Services.yml`

```yaml
---
# Internal-only Service — Postgres should NEVER be exposed outside the cluster.
# Type defaults to ClusterIP, which is exactly what we want here.
apiVersion: v1
kind: Service
metadata:
  name: postgres-service   # this exact name is what DB_HOST in the ConfigMap points to
  labels:
    app: postgres
spec:
  selector:
    app: postgres            # routes traffic to any Pod with label app=postgres
  ports:
    - port: 5432              # port other Pods use to reach this Service
      targetPort: 5432         # port the container actually listens on

---
# Exposes the Flask API. NodePort/LoadBalancer/Ingress are all reasonable choices
# depending on your cluster — LoadBalancer shown here as the typical cloud-provider choice.
# On a local cluster (minikube/kind) you'd usually use NodePort or `kubectl port-forward` instead.
apiVersion: v1
kind: Service
metadata:
  name: instagram-api-service
  labels:
    app: instagram-api
spec:
  type: LoadBalancer   # change to NodePort for local clusters, e.g. minikube
  selector:
    app: instagram-api
  ports:
    - port: 80           # port external clients hit
      targetPort: 5000     # port the Flask/gunicorn container listens on
```

---
