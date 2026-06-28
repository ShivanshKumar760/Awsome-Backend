# Kubernetes API SDK in Node.js — Complete Reference

The official Kubernetes client for Node.js is `@kubernetes/client-node`. It is auto-generated
from the Kubernetes OpenAPI spec, which means every resource type the Kubernetes API exposes has
a corresponding typed method in this library. This note covers every major API group, how
authentication works, how to read/create/update/delete/watch every resource type, exec into pods,
stream logs, handle errors, and build real controllers.

```
Part A — Setup, authentication, and how the client works
Part B — Core resource operations (CRUD for every major resource)
Part C — Watch / streaming / informers
Part D — Exec into pods, log streaming, port forwarding
Part E — Patch types (the three patch strategies)
Part F — Custom Resources (CRDs)
Part G — RBAC resources
Part H — Error handling
Part I — Real-world patterns (controller loop, retry, pagination)
```

---

## Part A — Setup and Authentication

### Install

```bash
npm install @kubernetes/client-node
npm install -D @types/node typescript ts-node-dev
```

### How the client is structured

The K8s API is divided into **API groups**, each with its own versioned client class. You don't
use one monolithic client — you instantiate the right client for the resource you want:

```
CoreV1Api          → Pods, Services, ConfigMaps, Secrets, Namespaces, Nodes, PersistentVolumes
AppsV1Api          → Deployments, StatefulSets, DaemonSets, ReplicaSets
BatchV1Api         → Jobs, CronJobs
NetworkingV1Api    → Ingresses, NetworkPolicies
RbacAuthorizationV1Api → Roles, RoleBindings, ClusterRoles, ClusterRoleBindings
StorageV1Api       → StorageClasses, PersistentVolumeClaims
AutoscalingV2Api   → HorizontalPodAutoscalers
PolicyV1Api        → PodDisruptionBudgets
CustomObjectsApi   → Custom Resources (CRDs)
```

Every client is instantiated from a `KubeConfig` object. The `KubeConfig` handles auth — it
reads credentials and server info from a kubeconfig file, a service account token, or explicit
configuration.

### Authentication modes

```typescript
import * as k8s from "@kubernetes/client-node";

const kc = new k8s.KubeConfig();

// Mode 1: Load from ~/.kube/config (local dev, most common)
kc.loadFromDefault();

// Mode 2: Load from a specific file
kc.loadFromFile("/path/to/kubeconfig");

// Mode 3: In-cluster — uses the mounted service account token
// Works inside a Pod that has a ServiceAccount bound to it
// Reads: /var/run/secrets/kubernetes.io/serviceaccount/token
//        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
kc.loadFromCluster();

// Mode 4: Automatically detect — in-cluster if running in a pod, otherwise ~/.kube/config
// This is what you want in code that runs both locally and in a cluster
kc.loadFromDefault(); // loadFromDefault already does this detection

// Mode 5: Explicit credentials (for testing or programmatic setup)
kc.loadFromOptions({
  clusters: [
    {
      name: "my-cluster",
      server: "https://my-k8s-api:6443",
      skipTLSVerify: false,
      caData: "base64-encoded-ca-cert",
    },
  ],
  users: [{ name: "my-user", token: "my-bearer-token" }],
  contexts: [{ name: "my-context", cluster: "my-cluster", user: "my-user" }],
  currentContext: "my-context",
});
```

### Instantiating clients

```typescript
// Always the same pattern: kc.makeApiClient(ApiClass)
const coreV1 = kc.makeApiClient(k8s.CoreV1Api);
const appsV1 = kc.makeApiClient(k8s.AppsV1Api);
const batchV1 = kc.makeApiClient(k8s.BatchV1Api);
const networkV1 = kc.makeApiClient(k8s.NetworkingV1Api);
const rbacV1 = kc.makeApiClient(k8s.RbacAuthorizationV1Api);
const customApi = kc.makeApiClient(k8s.CustomObjectsApi);
const storageV1 = kc.makeApiClient(k8s.StorageV1Api);
const hpaV2 = kc.makeApiClient(k8s.AutoscalingV2Api);
```

### How API calls actually work under the hood

Every method on these clients is a thin wrapper that:

1. Builds an HTTP request (path, method, query params, body)
2. Adds the auth header (Bearer token, client cert, or exec credential)
3. Makes an HTTPS request to the Kubernetes API server
4. Deserializes the JSON response into a typed object

```typescript
// What you write:
const response = await coreV1.readNamespacedPod({
  name: "my-pod",
  namespace: "default",
});

// What actually happens:
// GET https://<api-server>/api/v1/namespaces/default/pods/my-pod
// Authorization: Bearer <token-from-kubeconfig>
// → returns V1Pod typed object
```

All methods return `Promise<{response: http.IncomingMessage, body: T}>` in older versions, but
in v1+ they return the resource directly. Check your version — examples below use the v1+ API
where the method returns the object directly.

---

## Part B — Core Resource Operations

### Namespace operations

```typescript
import * as k8s from "@kubernetes/client-node";

const kc = new k8s.KubeConfig();
kc.loadFromDefault();
const coreV1 = kc.makeApiClient(k8s.CoreV1Api);

// ── List all namespaces ──────────────────────────────────────────────────
const namespaceList = await coreV1.listNamespace();
namespaceList.items.forEach((ns) => {
  console.log(ns.metadata?.name, ns.status?.phase);
});

// ── Create a namespace ───────────────────────────────────────────────────
const newNs = await coreV1.createNamespace({
  body: {
    apiVersion: "v1",
    kind: "Namespace",
    metadata: {
      name: "my-app",
      labels: { environment: "production", team: "backend" },
    },
  },
});
console.log("Created namespace:", newNs.metadata?.name);

// ── Read a specific namespace ────────────────────────────────────────────
const ns = await coreV1.readNamespace({ name: "my-app" });
console.log("Phase:", ns.status?.phase);

// ── Delete a namespace ───────────────────────────────────────────────────
await coreV1.deleteNamespace({ name: "my-app" });
```

---

### Pod operations

```typescript
// ── List pods in a namespace ─────────────────────────────────────────────
const podList = await coreV1.listNamespacedPod({ namespace: "default" });
podList.items.forEach((pod) => {
  console.log(
    pod.metadata?.name,
    pod.status?.phase, // Pending | Running | Succeeded | Failed | Unknown
    pod.status?.podIP,
    pod.spec?.nodeName
  );
});

// ── List with label selector ─────────────────────────────────────────────
const filteredPods = await coreV1.listNamespacedPod({
  namespace: "default",
  labelSelector: "app=nginx,environment=production",
});

// ── List with field selector ──────────────────────────────────────────────
const runningPods = await coreV1.listNamespacedPod({
  namespace: "default",
  fieldSelector: "status.phase=Running",
});

// ── Read a specific pod ───────────────────────────────────────────────────
const pod = await coreV1.readNamespacedPod({
  name: "my-pod",
  namespace: "default",
});
console.log(
  "Containers:",
  pod.spec?.containers.map((c) => c.name)
);
console.log(
  "Container statuses:",
  pod.status?.containerStatuses?.map((cs) => ({
    name: cs.name,
    ready: cs.ready,
    restarts: cs.restartCount,
    state: Object.keys(cs.state ?? {})[0], // running | waiting | terminated
  }))
);

// ── Create a pod ──────────────────────────────────────────────────────────
const newPod = await coreV1.createNamespacedPod({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "Pod",
    metadata: {
      name: "my-pod",
      labels: { app: "my-app" },
    },
    spec: {
      containers: [
        {
          name: "main",
          image: "nginx:1.25-alpine",
          ports: [{ containerPort: 80 }],
          resources: {
            requests: { cpu: "100m", memory: "128Mi" },
            limits: { cpu: "500m", memory: "256Mi" },
          },
          env: [
            { name: "ENV_VAR", value: "hello" },
            {
              name: "SECRET_VAR",
              valueFrom: {
                secretKeyRef: { name: "my-secret", key: "my-key" },
              },
            },
          ],
          livenessProbe: {
            httpGet: { path: "/health", port: 80 },
            initialDelaySeconds: 10,
            periodSeconds: 15,
          },
        },
      ],
      restartPolicy: "Always",
      serviceAccountName: "default",
    },
  },
});
console.log("Created pod:", newPod.metadata?.name);

// ── Delete a pod ──────────────────────────────────────────────────────────
await coreV1.deleteNamespacedPod({
  name: "my-pod",
  namespace: "default",
  body: {
    gracePeriodSeconds: 30, // 0 = force delete
    propagationPolicy: "Foreground", // Foreground | Background | Orphan
  },
});

// ── Read pod status only (lighter than full pod read) ─────────────────────
const podStatus = await coreV1.readNamespacedPodStatus({
  name: "my-pod",
  namespace: "default",
});
console.log("Phase:", podStatus.status?.phase);
```

---

### Deployment operations

```typescript
const appsV1 = kc.makeApiClient(k8s.AppsV1Api);

// ── List deployments ──────────────────────────────────────────────────────
const deployments = await appsV1.listNamespacedDeployment({
  namespace: "default",
});
deployments.items.forEach((d) => {
  console.log(
    d.metadata?.name,
    `${d.status?.readyReplicas ?? 0}/${d.spec?.replicas} ready`
  );
});

// ── Create a deployment ───────────────────────────────────────────────────
const deployment = await appsV1.createNamespacedDeployment({
  namespace: "default",
  body: {
    apiVersion: "apps/v1",
    kind: "Deployment",
    metadata: {
      name: "my-deployment",
      labels: { app: "my-app" },
    },
    spec: {
      replicas: 3,
      selector: { matchLabels: { app: "my-app" } },
      strategy: {
        type: "RollingUpdate",
        rollingUpdate: { maxSurge: 1, maxUnavailable: 0 },
      },
      template: {
        metadata: { labels: { app: "my-app", version: "v1.0.0" } },
        spec: {
          containers: [
            {
              name: "app",
              image: "my-registry/my-app:1.0.0",
              ports: [{ containerPort: 3000 }],
              resources: {
                requests: { cpu: "100m", memory: "128Mi" },
                limits: { cpu: "500m", memory: "256Mi" },
              },
              readinessProbe: {
                httpGet: { path: "/health", port: 3000 },
                initialDelaySeconds: 5,
                periodSeconds: 10,
              },
            },
          ],
        },
      },
    },
  },
});

// ── Read a deployment ─────────────────────────────────────────────────────
const dep = await appsV1.readNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
});
console.log("Desired:", dep.spec?.replicas);
console.log("Ready:", dep.status?.readyReplicas);
console.log("Updated:", dep.status?.updatedReplicas);

// ── Scale a deployment ─────────────────────────────────────────────────────
// Option 1: patch the replicas field directly
await appsV1.patchNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
  body: { spec: { replicas: 5 } },
});

// Option 2: use the scale subresource (cleaner, more explicit)
await appsV1.replaceNamespacedDeploymentScale({
  name: "my-deployment",
  namespace: "default",
  body: {
    apiVersion: "autoscaling/v1",
    kind: "Scale",
    metadata: { name: "my-deployment", namespace: "default" },
    spec: { replicas: 5 },
  },
});

// ── Update image (rolling update) ─────────────────────────────────────────
await appsV1.patchNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
  body: {
    spec: {
      template: {
        spec: {
          containers: [{ name: "app", image: "my-registry/my-app:2.0.0" }],
        },
      },
    },
  },
});

// ── Restart a deployment (patch annotation to trigger rolling restart) ─────
await appsV1.patchNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
  body: {
    spec: {
      template: {
        metadata: {
          annotations: {
            "kubectl.kubernetes.io/restartedAt": new Date().toISOString(),
          },
        },
      },
    },
  },
});

// ── Delete a deployment ───────────────────────────────────────────────────
await appsV1.deleteNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
});
```

---

### Service operations

```typescript
// ── List services ─────────────────────────────────────────────────────────
const services = await coreV1.listNamespacedService({ namespace: "default" });
services.items.forEach((svc) => {
  console.log(
    svc.metadata?.name,
    svc.spec?.type, // ClusterIP | NodePort | LoadBalancer | ExternalName
    svc.spec?.clusterIP,
    svc.status?.loadBalancer?.ingress?.[0]?.ip
  );
});

// ── Create a ClusterIP service ────────────────────────────────────────────
const clusterIpSvc = await coreV1.createNamespacedService({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "Service",
    metadata: { name: "my-service", labels: { app: "my-app" } },
    spec: {
      type: "ClusterIP",
      selector: { app: "my-app" }, // matches pod labels
      ports: [{ name: "http", port: 80, targetPort: 3000, protocol: "TCP" }],
    },
  },
});

// ── Create a LoadBalancer service ─────────────────────────────────────────
const lbSvc = await coreV1.createNamespacedService({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "Service",
    metadata: { name: "my-lb-service" },
    spec: {
      type: "LoadBalancer",
      selector: { app: "my-app" },
      ports: [{ port: 443, targetPort: 3000, protocol: "TCP" }],
    },
  },
});

// Wait for LoadBalancer IP to be assigned
async function waitForLoadBalancerIP(
  name: string,
  namespace: string
): Promise<string> {
  while (true) {
    const svc = await coreV1.readNamespacedService({ name, namespace });
    const ip = svc.status?.loadBalancer?.ingress?.[0]?.ip;
    if (ip) return ip;
    await new Promise((r) => setTimeout(r, 3000));
  }
}

// ── Read a service ────────────────────────────────────────────────────────
const svc = await coreV1.readNamespacedService({
  name: "my-service",
  namespace: "default",
});
console.log("ClusterIP:", svc.spec?.clusterIP);
console.log("Ports:", svc.spec?.ports);
```

---

### ConfigMap operations

```typescript
// ── Create a ConfigMap ────────────────────────────────────────────────────
await coreV1.createNamespacedConfigMap({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "ConfigMap",
    metadata: { name: "app-config" },
    data: {
      "database-url": "postgres://db:5432/myapp",
      "log-level": "info",
      "feature-flags": JSON.stringify({ darkMode: true, betaFeatures: false }),
      "nginx.conf": `server { listen 80; location / { proxy_pass http://app:3000; } }`,
    },
  },
});

// ── Read a ConfigMap ──────────────────────────────────────────────────────
const cm = await coreV1.readNamespacedConfigMap({
  name: "app-config",
  namespace: "default",
});
console.log("database-url:", cm.data?.["database-url"]);

// ── List ConfigMaps ───────────────────────────────────────────────────────
const cmList = await coreV1.listNamespacedConfigMap({ namespace: "default" });

// ── Update a ConfigMap (replace all data) ─────────────────────────────────
await coreV1.replaceNamespacedConfigMap({
  name: "app-config",
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "ConfigMap",
    metadata: {
      name: "app-config",
      resourceVersion: cm.metadata?.resourceVersion,
    },
    data: { ...cm.data, "log-level": "debug" },
  },
});

// ── Patch a ConfigMap (update only specific keys) ─────────────────────────
await coreV1.patchNamespacedConfigMap({
  name: "app-config",
  namespace: "default",
  body: { data: { "log-level": "warn" } },
});

// ── Delete a ConfigMap ────────────────────────────────────────────────────
await coreV1.deleteNamespacedConfigMap({
  name: "app-config",
  namespace: "default",
});
```

---

### Secret operations

```typescript
// ── Create a Secret ───────────────────────────────────────────────────────
// Secret data values MUST be base64 encoded
function b64(s: string): string {
  return Buffer.from(s).toString("base64");
}

await coreV1.createNamespacedSecret({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "Secret",
    metadata: { name: "app-secrets" },
    type: "Opaque", // Opaque | kubernetes.io/tls | kubernetes.io/dockerconfigjson | ...
    data: {
      "db-password": b64("super-secret-password"),
      "jwt-secret": b64("my-jwt-signing-key"),
    },
  },
});

// ── Create a TLS secret ───────────────────────────────────────────────────
await coreV1.createNamespacedSecret({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "Secret",
    metadata: { name: "my-tls-cert" },
    type: "kubernetes.io/tls",
    data: {
      "tls.crt": b64(certPem),
      "tls.key": b64(keyPem),
    },
  },
});

// ── Read and decode a secret ──────────────────────────────────────────────
const secret = await coreV1.readNamespacedSecret({
  name: "app-secrets",
  namespace: "default",
});
const dbPassword = Buffer.from(
  secret.data?.["db-password"] ?? "",
  "base64"
).toString("utf-8");
console.log("DB Password:", dbPassword);

// ── Patch a secret (add/update a key) ────────────────────────────────────
await coreV1.patchNamespacedSecret({
  name: "app-secrets",
  namespace: "default",
  body: { data: { "new-key": b64("new-value") } },
});
```

---

### Node operations

```typescript
// ── List all nodes ────────────────────────────────────────────────────────
const nodes = await coreV1.listNode();
nodes.items.forEach((node) => {
  const conditions = node.status?.conditions ?? [];
  const ready = conditions.find((c) => c.type === "Ready")?.status === "True";
  console.log(
    node.metadata?.name,
    ready ? "Ready" : "NotReady",
    node.status?.capacity?.cpu,
    node.status?.capacity?.memory,
    node.metadata?.labels?.["kubernetes.io/arch"],
    node.metadata?.labels?.["node.kubernetes.io/instance-type"]
  );
});

// ── Cordon a node (mark unschedulable) ───────────────────────────────────
await coreV1.patchNode({
  name: "my-node",
  body: { spec: { unschedulable: true } },
});

// ── Uncordon ──────────────────────────────────────────────────────────────
await coreV1.patchNode({
  name: "my-node",
  body: { spec: { unschedulable: false } },
});

// ── Taint a node ─────────────────────────────────────────────────────────
const existingNode = await coreV1.readNode({ name: "my-node" });
await coreV1.patchNode({
  name: "my-node",
  body: {
    spec: {
      taints: [
        ...(existingNode.spec?.taints ?? []),
        { key: "dedicated", value: "gpu", effect: "NoSchedule" },
      ],
    },
  },
});
```

---

### StatefulSet, DaemonSet, Job, CronJob

```typescript
// ── StatefulSet ───────────────────────────────────────────────────────────
const statefulSets = await appsV1.listNamespacedStatefulSet({
  namespace: "default",
});
statefulSets.items.forEach((ss) => {
  console.log(
    ss.metadata?.name,
    `${ss.status?.readyReplicas}/${ss.spec?.replicas}`
  );
});

await appsV1.createNamespacedStatefulSet({
  namespace: "default",
  body: {
    apiVersion: "apps/v1",
    kind: "StatefulSet",
    metadata: { name: "my-db" },
    spec: {
      serviceName: "my-db",
      replicas: 3,
      selector: { matchLabels: { app: "my-db" } },
      template: {
        metadata: { labels: { app: "my-db" } },
        spec: {
          containers: [
            {
              name: "db",
              image: "postgres:16-alpine",
              volumeMounts: [
                { name: "data", mountPath: "/var/lib/postgresql/data" },
              ],
            },
          ],
        },
      },
      volumeClaimTemplates: [
        {
          metadata: { name: "data" },
          spec: {
            accessModes: ["ReadWriteOnce"],
            resources: { requests: { storage: "10Gi" } },
            storageClassName: "fast-ssd",
          },
        },
      ],
    },
  },
});

// ── DaemonSet ─────────────────────────────────────────────────────────────
const daemonSets = await appsV1.listNamespacedDaemonSet({
  namespace: "kube-system",
});

// ── Job ───────────────────────────────────────────────────────────────────
const batchV1 = kc.makeApiClient(k8s.BatchV1Api);

await batchV1.createNamespacedJob({
  namespace: "default",
  body: {
    apiVersion: "batch/v1",
    kind: "Job",
    metadata: { name: "db-migration" },
    spec: {
      backoffLimit: 3,
      completions: 1,
      template: {
        spec: {
          restartPolicy: "OnFailure",
          containers: [
            {
              name: "migrate",
              image: "my-app:latest",
              command: ["node", "dist/migrate.js"],
            },
          ],
        },
      },
    },
  },
});

// Poll until job completes
async function waitForJob(name: string, namespace: string): Promise<boolean> {
  while (true) {
    const job = await batchV1.readNamespacedJob({ name, namespace });
    if (job.status?.succeeded && job.status.succeeded > 0) return true;
    if (
      job.status?.failed &&
      job.status.failed >= (job.spec?.backoffLimit ?? 3)
    )
      return false;
    await new Promise((r) => setTimeout(r, 5000));
  }
}

// ── CronJob ───────────────────────────────────────────────────────────────
await batchV1.createNamespacedCronJob({
  namespace: "default",
  body: {
    apiVersion: "batch/v1",
    kind: "CronJob",
    metadata: { name: "cleanup-job" },
    spec: {
      schedule: "0 2 * * *", // 2am every day
      concurrencyPolicy: "Forbid", // Forbid | Allow | Replace
      successfulJobsHistoryLimit: 3,
      failedJobsHistoryLimit: 1,
      jobTemplate: {
        spec: {
          template: {
            spec: {
              restartPolicy: "OnFailure",
              containers: [
                {
                  name: "cleanup",
                  image: "my-app:latest",
                  command: ["node", "dist/cleanup.js"],
                },
              ],
            },
          },
        },
      },
    },
  },
});
```

---

### PersistentVolume and PersistentVolumeClaim

```typescript
// ── Create a PVC ──────────────────────────────────────────────────────────
await coreV1.createNamespacedPersistentVolumeClaim({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "PersistentVolumeClaim",
    metadata: { name: "my-data" },
    spec: {
      accessModes: ["ReadWriteOnce"],
      storageClassName: "standard",
      resources: { requests: { storage: "20Gi" } },
    },
  },
});

// ── List PVCs and check their status ──────────────────────────────────────
const pvcs = await coreV1.listNamespacedPersistentVolumeClaim({
  namespace: "default",
});
pvcs.items.forEach((pvc) => {
  console.log(
    pvc.metadata?.name,
    pvc.status?.phase, // Pending | Bound | Lost
    pvc.spec?.volumeName, // which PV it's bound to
    pvc.spec?.resources?.requests?.storage
  );
});
```

---

### Ingress

```typescript
const networkV1 = kc.makeApiClient(k8s.NetworkingV1Api);

await networkV1.createNamespacedIngress({
  namespace: "default",
  body: {
    apiVersion: "networking.k8s.io/v1",
    kind: "Ingress",
    metadata: {
      name: "my-ingress",
      annotations: {
        "nginx.ingress.kubernetes.io/rewrite-target": "/",
        "cert-manager.io/cluster-issuer": "letsencrypt-prod",
      },
    },
    spec: {
      ingressClassName: "nginx",
      tls: [{ hosts: ["api.example.com"], secretName: "api-tls" }],
      rules: [
        {
          host: "api.example.com",
          http: {
            paths: [
              {
                path: "/",
                pathType: "Prefix",
                backend: {
                  service: { name: "api-service", port: { number: 80 } },
                },
              },
            ],
          },
        },
      ],
    },
  },
});

// ── List Ingresses ────────────────────────────────────────────────────────
const ingresses = await networkV1.listNamespacedIngress({
  namespace: "default",
});
ingresses.items.forEach((ing) => {
  const lb = ing.status?.loadBalancer?.ingress?.[0]?.ip;
  console.log(ing.metadata?.name, "→", lb ?? "pending");
});
```

---

## Part C — Watch, Streaming, Informers

### Watch — low-level event stream

`Watch` opens a long-running HTTP connection with `?watch=true` to the Kubernetes API server.
The server sends newline-delimited JSON events as resources change.

```typescript
import * as k8s from "@kubernetes/client-node";

const kc = new k8s.KubeConfig();
kc.loadFromDefault();

const watch = new k8s.Watch(kc);

// ── Watch pods in a namespace ─────────────────────────────────────────────
const req = await watch.watch(
  "/api/v1/namespaces/default/pods", // the URL path to watch
  { labelSelector: "app=my-app" }, // optional query params
  (type, obj) => {
    // type: "ADDED" | "MODIFIED" | "DELETED" | "BOOKMARK"
    // obj:  the V1Pod (or whatever resource you're watching)
    const pod = obj as k8s.V1Pod;
    console.log(`[${type}] Pod:`, pod.metadata?.name, "→", pod.status?.phase);

    if (type === "ADDED") {
      /* pod created or existing pod on initial sync */
    }
    if (type === "MODIFIED") {
      /* pod status/spec changed */
    }
    if (type === "DELETED") {
      /* pod was deleted */
    }
  },
  (err) => {
    // Called when the watch ends (error or server closed)
    if (err) console.error("Watch error:", err);
    else console.log("Watch ended cleanly");
  }
);

// Stop the watch after 30 seconds
setTimeout(() => req.abort(), 30000);

// ── Watch deployments ─────────────────────────────────────────────────────
await watch.watch(
  "/apis/apps/v1/namespaces/default/deployments",
  {},
  (type, obj: k8s.V1Deployment) => {
    const ready = obj.status?.readyReplicas ?? 0;
    const desired = obj.spec?.replicas ?? 0;
    console.log(`[${type}] ${obj.metadata?.name}: ${ready}/${desired} ready`);
  },
  (err) => {
    if (err) console.error(err);
  }
);

// ── Watch across ALL namespaces ───────────────────────────────────────────
await watch.watch(
  "/api/v1/pods",
  {},
  (type, pod: k8s.V1Pod) => {
    console.log(`[${type}] ${pod.metadata?.namespace}/${pod.metadata?.name}`);
  },
  () => {}
);
```

### Watch URL path reference

```
All pods (all namespaces):           /api/v1/pods
Pods in namespace:                   /api/v1/namespaces/{ns}/pods
Specific pod:                        /api/v1/namespaces/{ns}/pods/{name}
Deployments:                         /apis/apps/v1/namespaces/{ns}/deployments
StatefulSets:                        /apis/apps/v1/namespaces/{ns}/statefulsets
DaemonSets:                          /apis/apps/v1/namespaces/{ns}/daemonsets
Jobs:                                /apis/batch/v1/namespaces/{ns}/jobs
CronJobs:                            /apis/batch/v1/namespaces/{ns}/cronjobs
Services:                            /api/v1/namespaces/{ns}/services
ConfigMaps:                          /api/v1/namespaces/{ns}/configmaps
Secrets:                             /api/v1/namespaces/{ns}/secrets
Nodes:                               /api/v1/nodes
Namespaces:                          /api/v1/namespaces
Ingresses:                           /apis/networking.k8s.io/v1/namespaces/{ns}/ingresses
Custom resources:                    /apis/{group}/{version}/namespaces/{ns}/{plural}
```

### Informer — production-grade watch with local cache

An **Informer** is a higher-level abstraction over Watch. It maintains a **local in-memory
cache** of all resources of a type, handles reconnection automatically (watches expire after ~5
minutes — informers reconnect with the last `resourceVersion` so they don't miss events), and
supports event handlers (`ADD`, `UPDATE`, `DELETE`).

```typescript
import * as k8s from "@kubernetes/client-node";

const kc = new k8s.KubeConfig();
kc.loadFromDefault();
const coreV1 = kc.makeApiClient(k8s.CoreV1Api);

// ── Build an informer for Pods ────────────────────────────────────────────
const listFn = () => coreV1.listNamespacedPod({ namespace: "default" });

const informer = k8s.makeInformer(
  kc,
  "/api/v1/namespaces/default/pods",
  listFn
);

informer.on("add", (pod: k8s.V1Pod) => {
  console.log("Pod added:   ", pod.metadata?.name, pod.status?.phase);
});
informer.on("update", (pod: k8s.V1Pod) => {
  console.log("Pod updated: ", pod.metadata?.name, pod.status?.phase);
});
informer.on("delete", (pod: k8s.V1Pod) => {
  console.log("Pod deleted: ", pod.metadata?.name);
});
informer.on("error", (err: Error) => {
  console.error("Informer error:", err);
  // Informer will auto-restart after an error — no manual restart needed
});

// Start the informer — begins listing and watching
await informer.start();

// The informer runs indefinitely — it reconnects automatically
// To stop it:
// await informer.stop();

// ── Informer with label selector ──────────────────────────────────────────
const labeledListFn = () =>
  coreV1.listNamespacedPod({
    namespace: "default",
    labelSelector: "app=my-app",
  });

const labeledInformer = k8s.makeInformer(
  kc,
  "/api/v1/namespaces/default/pods",
  labeledListFn
);

// ── Deployment informer ───────────────────────────────────────────────────
const appsV1 = kc.makeApiClient(k8s.AppsV1Api);
const deployInformer = k8s.makeInformer(
  kc,
  "/apis/apps/v1/namespaces/default/deployments",
  () => appsV1.listNamespacedDeployment({ namespace: "default" })
);

deployInformer.on("add", (d: k8s.V1Deployment) =>
  console.log("Deploy added:", d.metadata?.name)
);
deployInformer.on("update", (d: k8s.V1Deployment) => {
  const ready = d.status?.readyReplicas ?? 0;
  const desired = d.spec?.replicas ?? 0;
  console.log(`Deploy updated: ${d.metadata?.name} ${ready}/${desired}`);
});
await deployInformer.start();
```

---

## Part D — Exec into Pods, Log Streaming, Port Forwarding

### Stream pod logs

```typescript
import * as k8s from "@kubernetes/client-node";
import * as stream from "stream";

const kc = new k8s.KubeConfig();
kc.loadFromDefault();

const logStream = new k8s.Log(kc);

// ── Tail logs (follow=true streams in real time) ───────────────────────────
const logOut = new stream.PassThrough();
logOut.on("data", (chunk: Buffer) => process.stdout.write(chunk));

await logStream.log(
  "default", // namespace
  "my-pod", // pod name
  "main", // container name (optional if pod has one container)
  logOut, // writable stream
  (err) => {
    if (err) console.error("Log error:", err);
  },
  {
    follow: true, // stream in real time (like kubectl logs -f)
    tailLines: 100, // last N lines first
    timestamps: true, // prefix each line with its timestamp
    sinceSeconds: 3600, // only logs from the last hour
    pretty: false,
  }
);

// ── Capture logs as a string (no follow) ─────────────────────────────────
async function getPodLogs(
  namespace: string,
  podName: string,
  container?: string,
  tailLines = 200
): Promise<string> {
  return new Promise((resolve, reject) => {
    const chunks: Buffer[] = [];
    const out = new stream.PassThrough();
    out.on("data", (chunk: Buffer) => chunks.push(chunk));
    out.on("end", () => resolve(Buffer.concat(chunks).toString("utf-8")));
    out.on("error", reject);

    logStream.log(
      namespace,
      podName,
      container ?? "",
      out,
      (err) => {
        if (err) reject(err);
      },
      { follow: false, tailLines }
    );
  });
}

const logs = await getPodLogs("default", "my-pod", "main");
console.log("Logs:", logs);
```

### Exec into a pod

```typescript
const exec = new k8s.Exec(kc);

// ── Run a command and capture output ─────────────────────────────────────
async function execInPod(
  namespace: string,
  podName: string,
  container: string,
  command: string[]
): Promise<{ stdout: string; stderr: string; exitCode: number }> {
  return new Promise((resolve, reject) => {
    const stdoutChunks: Buffer[] = [];
    const stderrChunks: Buffer[] = [];

    const stdout = new stream.PassThrough();
    const stderr = new stream.PassThrough();
    stdout.on("data", (c: Buffer) => stdoutChunks.push(c));
    stderr.on("data", (c: Buffer) => stderrChunks.push(c));

    exec
      .exec(
        namespace,
        podName,
        container,
        command,
        stdout, // stdout stream
        stderr, // stderr stream
        null, // stdin stream (null = no stdin)
        false, // tty
        (status: k8s.V1Status) => {
          const exitCode = status.status === "Success" ? 0 : 1;
          resolve({
            stdout: Buffer.concat(stdoutChunks).toString(),
            stderr: Buffer.concat(stderrChunks).toString(),
            exitCode,
          });
        }
      )
      .catch(reject);
  });
}

// Run a command in a pod
const result = await execInPod("default", "my-pod", "main", [
  "ls",
  "-la",
  "/app",
]);
console.log("stdout:", result.stdout);
console.log("stderr:", result.stderr);
console.log("exit code:", result.exitCode);

// Check if a file exists
const fileCheck = await execInPod("default", "my-pod", "main", [
  "test",
  "-f",
  "/app/dist/server.js",
]);
console.log("File exists:", fileCheck.exitCode === 0);

// ── Interactive exec with stdin (like kubectl exec -it) ───────────────────
const interactive = new stream.PassThrough(); // stdin

exec.exec(
  "default",
  "my-pod",
  "main",
  ["/bin/sh"],
  process.stdout, // pipe pod stdout → terminal
  process.stderr,
  process.stdin, // pipe terminal stdin → pod
  true, // tty = true for interactive
  (status) => console.log("Session ended:", status)
);
```

### Port forwarding

```typescript
const portForward = new k8s.PortForward(kc);
import * as net from "net";

// ── Forward local port 8080 → pod port 3000 ───────────────────────────────
const server = net.createServer((socket) => {
  portForward.portForward(
    "default",
    "my-pod",
    [3000], // pod port(s) to forward
    socket, // outgoing (from pod to local)
    null, // stderr stream (optional)
    socket // incoming (from local to pod)
  );
});

server.listen(8080, "127.0.0.1", () => {
  console.log("Port forward active: localhost:8080 → pod:3000");
});

// Stop the server when done
// server.close();
```

---

## Part E — Patch Types (The Three Strategies)

Kubernetes supports three distinct patch types. Each has different merge semantics, and using
the wrong one causes silent data loss or unexpected behavior.

```typescript
import * as k8s from "@kubernetes/client-node";

const kc = new k8s.KubeConfig();
kc.loadFromDefault();
const appsV1 = kc.makeApiClient(k8s.AppsV1Api);
const coreV1 = kc.makeApiClient(k8s.CoreV1Api);
```

### 1. JSON Merge Patch (`application/merge-patch+json`) — default

The default when you call `.patch*()` methods. Rules:

- Keys in the patch **overwrite** matching keys on the server
- `null` value **deletes** a key
- Keys not in the patch are **left unchanged**
- Arrays are **replaced entirely** — not merged element by element

```typescript
// ── Update an image — merge patch ────────────────────────────────────────
// Sends: PATCH /apis/apps/v1/namespaces/default/deployments/my-dep
//        Content-Type: application/merge-patch+json
//        Body: {"spec":{"template":{"spec":{"containers":[{"name":"app","image":"new:2.0"}]}}}}
//
// ⚠️ The containers array is REPLACED entirely — include ALL containers or others vanish
await appsV1.patchNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
  body: {
    spec: {
      template: {
        spec: {
          containers: [
            { name: "app", image: "my-app:2.0.0" },
            { name: "sidecar", image: "sidecar:1.0.0" }, // must include or it disappears
          ],
        },
      },
    },
  },
});

// ── Delete a label with merge patch (null = delete) ───────────────────────
await appsV1.patchNamespacedDeployment({
  name: "my-deployment",
  namespace: "default",
  body: {
    metadata: {
      labels: {
        "old-label": null as any, // null deletes the key
        "new-label": "value", // this is added/updated
      },
    },
  },
});
```

### 2. Strategic Merge Patch (`application/strategic-merge-patch+json`)

Only supported on built-in Kubernetes resources (not CRDs). It's smarter about arrays —
Kubernetes knows the "merge key" for each array (e.g., `name` for `containers`), so it can
merge array elements by key rather than replacing the whole array.

```typescript
// To use strategic merge patch, set the Content-Type header explicitly
// The library exposes a way to set headers via the options object
const strategicMergePatch = "application/strategic-merge-patch+json";

// ── Update one container in a deployment WITHOUT replacing others ──────────
// With strategic merge patch, the containers array is merged by container name
// You only need to include the container you're changing
const response = await appsV1.patchNamespacedDeployment(
  {
    name: "my-deployment",
    namespace: "default",
    body: {
      spec: {
        template: {
          spec: {
            containers: [
              // Only mention the container you're updating
              // Other containers are preserved automatically
              { name: "app", image: "my-app:3.0.0" },
            ],
          },
        },
      },
    },
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": strategicMergePatch } }
);

// ── Add an env var to a specific container without touching others ─────────
await appsV1.patchNamespacedDeployment(
  {
    name: "my-deployment",
    namespace: "default",
    body: {
      spec: {
        template: {
          spec: {
            containers: [
              {
                name: "app", // identifies WHICH container (the merge key)
                env: [{ name: "NEW_VAR", value: "new-value" }],
                // other env vars are preserved (strategic merge merges env array by name too)
              },
            ],
          },
        },
      },
    },
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": strategicMergePatch } }
);
```

### 3. JSON Patch (`application/json-patch+json`)

RFC 6902 format. A list of explicit operations: `add`, `remove`, `replace`, `move`, `copy`,
`test`. The most precise — you specify exactly what path to change and how.

```typescript
const jsonPatch = "application/json-patch+json";

// ── Replace the replica count precisely ───────────────────────────────────
await appsV1.patchNamespacedDeployment(
  {
    name: "my-deployment",
    namespace: "default",
    body: [{ op: "replace", path: "/spec/replicas", value: 5 }] as any,
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": jsonPatch } }
);

// ── Add a label ───────────────────────────────────────────────────────────
await appsV1.patchNamespacedDeployment(
  {
    name: "my-deployment",
    namespace: "default",
    body: [
      { op: "add", path: "/metadata/labels/new-label", value: "my-value" },
    ] as any,
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": jsonPatch } }
);

// ── Remove a label ────────────────────────────────────────────────────────
await coreV1.patchNamespacedPod(
  {
    name: "my-pod",
    namespace: "default",
    body: [{ op: "remove", path: "/metadata/labels/old-label" }] as any,
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": jsonPatch } }
);

// ── Multiple operations atomically ────────────────────────────────────────
await appsV1.patchNamespacedDeployment(
  {
    name: "my-deployment",
    namespace: "default",
    body: [
      { op: "replace", path: "/spec/replicas", value: 3 },
      {
        op: "replace",
        path: "/spec/template/spec/containers/0/image",
        value: "app:4.0.0",
      },
      {
        op: "add",
        path: "/metadata/annotations/deployed-at",
        value: new Date().toISOString(),
      },
    ] as any,
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": jsonPatch } }
);
```

### Patch type summary

| Patch type                 | Array handling                                      | Use when                                           |
| -------------------------- | --------------------------------------------------- | -------------------------------------------------- |
| JSON Merge Patch (default) | **Replaces** entire arrays                          | Simple field updates, adding/removing labels       |
| Strategic Merge Patch      | **Merges** arrays by key (containers by name, etc.) | Updating one container, adding env vars            |
| JSON Patch                 | Explicit path operations                            | Precise atomic operations, multiple fields at once |

---

## Part F — Custom Resources (CRDs)

Custom Resources use `CustomObjectsApi` — a single generic API that handles all CRD types
regardless of group or version.

```typescript
const customApi = kc.makeApiClient(k8s.CustomObjectsApi);

// For a CRD like:
// apiVersion: mycompany.com/v1
// kind: MyApp
// → group: "mycompany.com", version: "v1", plural: "myapps"

const GROUP = "mycompany.com";
const VERSION = "v1";
const PLURAL = "myapps";

// ── List custom resources ─────────────────────────────────────────────────
const list = await customApi.listNamespacedCustomObject({
  group: GROUP,
  version: VERSION,
  namespace: "default",
  plural: PLURAL,
});
// list is typed as `object` — cast it
const items = (list as any).items as any[];
items.forEach((item) => console.log(item.metadata.name, item.spec));

// ── Create a custom resource ──────────────────────────────────────────────
const created = await customApi.createNamespacedCustomObject({
  group: GROUP,
  version: VERSION,
  namespace: "default",
  plural: PLURAL,
  body: {
    apiVersion: `${GROUP}/${VERSION}`,
    kind: "MyApp",
    metadata: { name: "my-app-instance" },
    spec: {
      replicas: 2,
      image: "my-registry/my-app:1.0.0",
      config: { logLevel: "info" },
    },
  },
});

// ── Read a custom resource ────────────────────────────────────────────────
const obj = await customApi.getNamespacedCustomObject({
  group: GROUP,
  version: VERSION,
  namespace: "default",
  plural: PLURAL,
  name: "my-app-instance",
});
console.log((obj as any).spec);

// ── Update status subresource ─────────────────────────────────────────────
// CRDs with a status subresource must be updated via /status, not the main endpoint
await customApi.patchNamespacedCustomObjectStatus(
  {
    group: GROUP,
    version: VERSION,
    namespace: "default",
    plural: PLURAL,
    name: "my-app-instance",
    body: {
      status: {
        phase: "Running",
        observedGeneration: 1,
        conditions: [
          {
            type: "Ready",
            status: "True",
            lastTransitionTime: new Date().toISOString(),
          },
        ],
      },
    },
  },
  undefined,
  undefined,
  undefined,
  { headers: { "Content-Type": "application/merge-patch+json" } }
);

// ── Watch custom resources ────────────────────────────────────────────────
const watch = new k8s.Watch(kc);
await watch.watch(
  `/apis/${GROUP}/${VERSION}/namespaces/default/${PLURAL}`,
  {},
  (type, obj: any) => {
    console.log(`[${type}] MyApp:`, obj.metadata.name, "→", obj.spec);
  },
  (err) => {
    if (err) console.error("Watch error:", err);
  }
);

// ── Cluster-scoped custom resources ──────────────────────────────────────
// Some CRDs are cluster-scoped (no namespace), use the Cluster variants
const clusterList = await customApi.listClusterCustomObject({
  group: GROUP,
  version: VERSION,
  plural: "myclusterresources",
});

await customApi.createClusterCustomObject({
  group: GROUP,
  version: VERSION,
  plural: "myclusterresources",
  body: {
    apiVersion: `${GROUP}/${VERSION}`,
    kind: "MyClusterResource",
    metadata: { name: "my-instance" },
    spec: {},
  },
});
```

---

## Part G — RBAC Resources

```typescript
const rbacV1 = kc.makeApiClient(k8s.RbacAuthorizationV1Api);

// ── Create a Role (namespace-scoped permissions) ──────────────────────────
await rbacV1.createNamespacedRole({
  namespace: "default",
  body: {
    apiVersion: "rbac.authorization.k8s.io/v1",
    kind: "Role",
    metadata: { name: "pod-reader" },
    rules: [
      {
        apiGroups: [""], // "" = core API group
        resources: ["pods", "pods/log"],
        verbs: ["get", "list", "watch"],
      },
      {
        apiGroups: ["apps"],
        resources: ["deployments"],
        verbs: ["get", "list", "watch", "update", "patch"],
      },
    ],
  },
});

// ── Create a RoleBinding ──────────────────────────────────────────────────
await rbacV1.createNamespacedRoleBinding({
  namespace: "default",
  body: {
    apiVersion: "rbac.authorization.k8s.io/v1",
    kind: "RoleBinding",
    metadata: { name: "pod-reader-binding" },
    roleRef: {
      apiGroup: "rbac.authorization.k8s.io",
      kind: "Role",
      name: "pod-reader",
    },
    subjects: [
      {
        kind: "ServiceAccount",
        name: "my-service-account",
        namespace: "default",
      },
      {
        kind: "User",
        name: "alice@example.com",
        apiGroup: "rbac.authorization.k8s.io",
      },
    ],
  },
});

// ── Create a ClusterRole (cluster-wide permissions) ───────────────────────
await rbacV1.createClusterRole({
  body: {
    apiVersion: "rbac.authorization.k8s.io/v1",
    kind: "ClusterRole",
    metadata: { name: "node-reader" },
    rules: [
      {
        apiGroups: [""],
        resources: ["nodes"],
        verbs: ["get", "list", "watch"],
      },
    ],
  },
});

// ── Create a ServiceAccount ───────────────────────────────────────────────
await coreV1.createNamespacedServiceAccount({
  namespace: "default",
  body: {
    apiVersion: "v1",
    kind: "ServiceAccount",
    metadata: { name: "my-controller" },
  },
});
```

---

## Part H — Error Handling

```typescript
import * as k8s from "@kubernetes/client-node";

// All API errors from @kubernetes/client-node throw an object with:
// err.response.statusCode  — HTTP status code
// err.body                 — The raw response body (may be a V1Status object)
// err.message              — Short message

async function safeGetPod(
  namespace: string,
  name: string
): Promise<k8s.V1Pod | null> {
  try {
    return await coreV1.readNamespacedPod({ name, namespace });
  } catch (err: any) {
    if (err?.response?.statusCode === 404) {
      return null; // not found — not an error in many cases
    }
    if (err?.response?.statusCode === 403) {
      throw new Error(`Permission denied reading pod ${namespace}/${name}`);
    }
    if (err?.response?.statusCode === 409) {
      throw new Error(`Conflict: ${err.body?.message}`);
    }
    throw err; // unexpected error — re-throw
  }
}

// ── Create-or-update pattern (upsert) ────────────────────────────────────
async function upsertConfigMap(
  namespace: string,
  name: string,
  data: Record<string, string>
): Promise<k8s.V1ConfigMap> {
  const body: k8s.V1ConfigMap = {
    apiVersion: "v1",
    kind: "ConfigMap",
    metadata: { name, namespace },
    data,
  };

  try {
    // Try to create
    return await coreV1.createNamespacedConfigMap({ namespace, body });
  } catch (err: any) {
    if (err?.response?.statusCode === 409) {
      // Already exists — update it instead
      return await coreV1.replaceNamespacedConfigMap({ name, namespace, body });
    }
    throw err;
  }
}

// ── Retry with exponential backoff ────────────────────────────────────────
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 5,
  baseDelayMs = 500
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err: any) {
      const status = err?.response?.statusCode;
      // Retry on 429 (too many requests) or 5xx (server errors)
      const isRetryable = status === 429 || (status >= 500 && status < 600);
      if (!isRetryable || attempt === maxAttempts) throw err;
      const delay = baseDelayMs * Math.pow(2, attempt - 1);
      console.warn(
        `Attempt ${attempt} failed (${status}), retrying in ${delay}ms`
      );
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw new Error("unreachable");
}

// Usage
const pod = await withRetry(() =>
  coreV1.readNamespacedPod({ name: "my-pod", namespace: "default" })
);
```

---

## Part I — Real-World Patterns

### Pagination with `continue` tokens

Large clusters can have thousands of pods. List calls return paginated results — always handle
pagination in production code.

```typescript
async function listAllPods(namespace: string): Promise<k8s.V1Pod[]> {
  const allPods: k8s.V1Pod[] = [];
  let continueToken: string | undefined = undefined;

  do {
    const result = await coreV1.listNamespacedPod({
      namespace,
      limit: 100, // page size
      continueToken, // token from previous page (undefined = first page)
    });

    allPods.push(...result.items);
    continueToken = result.metadata?.continue; // undefined when on last page
  } while (continueToken);

  return allPods;
}
```

### Controller loop — reconcile pattern

The fundamental pattern for any Kubernetes controller or operator:

```typescript
import * as k8s from "@kubernetes/client-node";

const kc = new k8s.KubeConfig();
kc.loadFromDefault();
const coreV1 = kc.makeApiClient(k8s.CoreV1Api);
const appsV1 = kc.makeApiClient(k8s.AppsV1Api);
const customApi = kc.makeApiClient(k8s.CustomObjectsApi);

// The reconcile function: given a resource, make the world match the desired state
async function reconcile(obj: any): Promise<void> {
  const name = obj.metadata.name as string;
  const namespace = obj.metadata.namespace as string;
  const spec = obj.spec;

  console.log(`Reconciling MyApp: ${namespace}/${name}`);

  // Check if the Deployment we manage exists
  let deployment: k8s.V1Deployment | null = null;
  try {
    deployment = await appsV1.readNamespacedDeployment({ name, namespace });
  } catch (err: any) {
    if (err?.response?.statusCode !== 404) throw err;
  }

  if (!deployment) {
    // Create it
    await appsV1.createNamespacedDeployment({
      namespace,
      body: {
        apiVersion: "apps/v1",
        kind: "Deployment",
        metadata: {
          name,
          namespace,
          ownerReferences: [
            {
              // owner reference: deployment is "owned" by this CR
              apiVersion: `mycompany.com/v1`,
              kind: "MyApp",
              name: name,
              uid: obj.metadata.uid,
              controller: true,
              blockOwnerDeletion: true,
            },
          ],
        },
        spec: {
          replicas: spec.replicas ?? 1,
          selector: { matchLabels: { app: name } },
          template: {
            metadata: { labels: { app: name } },
            spec: { containers: [{ name: "app", image: spec.image }] },
          },
        },
      },
    });
    console.log(`Created Deployment ${namespace}/${name}`);
  } else {
    // Update it if spec has changed
    const currentImage =
      deployment.spec?.template?.spec?.containers?.[0]?.image;
    const currentReplicas = deployment.spec?.replicas;

    if (currentImage !== spec.image || currentReplicas !== spec.replicas) {
      await appsV1.patchNamespacedDeployment({
        name,
        namespace,
        body: {
          spec: {
            replicas: spec.replicas,
            template: {
              spec: { containers: [{ name: "app", image: spec.image }] },
            },
          },
        },
      });
      console.log(`Updated Deployment ${namespace}/${name}`);
    }
  }

  // Update status
  await customApi.patchNamespacedCustomObjectStatus(
    {
      group: "mycompany.com",
      version: "v1",
      namespace,
      plural: "myapps",
      name,
      body: {
        status: {
          phase: "Reconciled",
          observedGeneration: obj.metadata.generation,
        },
      },
    },
    undefined,
    undefined,
    undefined,
    { headers: { "Content-Type": "application/merge-patch+json" } }
  );
}

// Run the controller — watch + reconcile
async function startController(): Promise<void> {
  const informer = k8s.makeInformer(
    kc,
    "/apis/mycompany.com/v1/namespaces/default/myapps",
    () =>
      customApi.listNamespacedCustomObject({
        group: "mycompany.com",
        version: "v1",
        namespace: "default",
        plural: "myapps",
      }) as any
  );

  informer.on("add", reconcile);
  informer.on("update", reconcile);
  // DELETE is handled automatically via ownerReferences — K8s GC cleans up child resources

  informer.on("error", (err: Error) => {
    console.error("Informer error:", err);
    // Informer auto-restarts
  });

  await informer.start();
  console.log("Controller started");
}

startController().catch(console.error);
```

### Apply pattern (server-side apply)

Server-Side Apply (SSA) is the modern way to manage resources — it tracks field ownership and
handles conflicts properly, just like `kubectl apply`.

```typescript
async function serverSideApply(
  resource: object,
  fieldManager: string = "my-controller"
): Promise<void> {
  // All resources go through a single generic HTTP call for SSA
  const client = kc.makeApiClient(k8s.CustomObjectsApi);

  // SSA uses PATCH with Content-Type: application/apply-patch+yaml
  // For built-in resources you can use the specific client with the right header
  await appsV1.patchNamespacedDeployment(
    {
      name: (resource as any).metadata.name,
      namespace: (resource as any).metadata.namespace,
      body: resource,
    },
    undefined,
    fieldManager, // fieldManager parameter
    "true", // force (overwrite conflicts from other managers)
    undefined,
    { headers: { "Content-Type": "application/apply-patch+yaml" } }
  );
}
```

### Wait for pod to be ready

```typescript
async function waitForPodReady(
  namespace: string,
  name: string,
  timeoutMs = 120_000
): Promise<void> {
  const watch = new k8s.Watch(kc);
  const deadline = Date.now() + timeoutMs;

  return new Promise((resolve, reject) => {
    let req: any;

    const checkReady = (pod: k8s.V1Pod): boolean => {
      return (
        pod.status?.conditions?.some(
          (c) => c.type === "Ready" && c.status === "True"
        ) ?? false
      );
    };

    watch
      .watch(
        `/api/v1/namespaces/${namespace}/pods`,
        { fieldSelector: `metadata.name=${name}` },
        (type, pod: k8s.V1Pod) => {
          if (Date.now() > deadline) {
            req?.abort();
            reject(
              new Error(
                `Pod ${name} did not become ready within ${timeoutMs}ms`
              )
            );
            return;
          }
          if (checkReady(pod)) {
            req?.abort();
            resolve();
          }
        },
        (err) => {
          if (err) reject(err);
        }
      )
      .then((r) => {
        req = r;
      });
  });
}

// Usage
await coreV1.createNamespacedPod({ namespace: "default", body: myPodSpec });
await waitForPodReady("default", "my-pod", 60_000);
console.log("Pod is ready!");
```

### Resource version conflicts (`resourceVersion`)

When you do a `replace` (full update), Kubernetes uses optimistic concurrency via
`resourceVersion`. If two processes try to update the same resource simultaneously, one will get
a `409 Conflict`. Always read before writing, and include the `resourceVersion`:

```typescript
async function safeUpdate(
  namespace: string,
  name: string,
  transform: (d: k8s.V1Deployment) => k8s.V1Deployment
): Promise<k8s.V1Deployment> {
  // Retry loop for optimistic concurrency conflicts
  for (let i = 0; i < 5; i++) {
    const current = await appsV1.readNamespacedDeployment({ name, namespace });
    const updated = transform(current);
    // Keep the resourceVersion from the read — required for replace
    updated.metadata = {
      ...updated.metadata,
      resourceVersion: current.metadata?.resourceVersion,
    };

    try {
      return await appsV1.replaceNamespacedDeployment({
        name,
        namespace,
        body: updated,
      });
    } catch (err: any) {
      if (err?.response?.statusCode === 409) {
        // Someone else updated it — re-read and retry
        await new Promise((r) => setTimeout(r, 100 * (i + 1)));
        continue;
      }
      throw err;
    }
  }
  throw new Error(`Failed to update ${namespace}/${name} after 5 attempts`);
}
```

---

## Quick Reference: API Groups and URL Paths

```
Resource               | API Group              | Client Class
-----------------------|------------------------|------------------------
Pod                    | core (v1)              | CoreV1Api
Service                | core (v1)              | CoreV1Api
ConfigMap              | core (v1)              | CoreV1Api
Secret                 | core (v1)              | CoreV1Api
Namespace              | core (v1)              | CoreV1Api
Node                   | core (v1)              | CoreV1Api
PersistentVolume       | core (v1)              | CoreV1Api
PersistentVolumeClaim  | core (v1)              | CoreV1Api
ServiceAccount         | core (v1)              | CoreV1Api
Deployment             | apps/v1                | AppsV1Api
StatefulSet            | apps/v1                | AppsV1Api
DaemonSet              | apps/v1                | AppsV1Api
ReplicaSet             | apps/v1                | AppsV1Api
Job                    | batch/v1               | BatchV1Api
CronJob                | batch/v1               | BatchV1Api
Ingress                | networking.k8s.io/v1   | NetworkingV1Api
NetworkPolicy          | networking.k8s.io/v1   | NetworkingV1Api
Role                   | rbac.../v1             | RbacAuthorizationV1Api
RoleBinding            | rbac.../v1             | RbacAuthorizationV1Api
ClusterRole            | rbac.../v1             | RbacAuthorizationV1Api
ClusterRoleBinding     | rbac.../v1             | RbacAuthorizationV1Api
HorizontalPodAutoscaler| autoscaling/v2         | AutoscalingV2Api
StorageClass           | storage.k8s.io/v1      | StorageV1Api
Custom Resources       | <your-group>/<version> | CustomObjectsApi
```
