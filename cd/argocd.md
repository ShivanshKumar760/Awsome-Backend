# Argo CD — Installation & Configuration Guide

A generalized, end-to-end guide for installing Argo CD on any Kubernetes cluster, accessing the UI/CLI, connecting Git repositories, and deploying applications via GitOps. Applicable to any project — including the multi-region Flask deployments covered earlier — as the continuous-delivery layer on top of your existing manifests/Helm charts.

---

## 0. What Argo CD does (quick context)

Argo CD is a GitOps continuous-delivery tool for Kubernetes. You point it at a Git repo containing your manifests (raw YAML, Kustomize, or Helm), and it continuously:

- **Syncs** the live cluster state to match what's declared in Git
- **Detects drift** — if someone manually changes something in the cluster, Argo CD flags it as "OutOfSync"
- **Rolls back** easily, since every change is just a Git commit
- Gives you a **UI/CLI** to visualize, diff, and manually or automatically sync deployments

Instead of running `kubectl apply -f` yourself per cluster (like in the earlier multi-region guides), you'd commit your manifests to Git and let Argo CD apply them — including across multiple clusters.

---

## 1. Prerequisites

- A running Kubernetes cluster (any: EKS, GKE, AKS, kind, k3s, self-managed)
- `kubectl` installed and pointed at the target cluster (`kubectl config current-context`)
- Cluster-admin access (Argo CD needs to create CRDs, namespaces, RBAC)
- (Optional but recommended) `helm` if you prefer installing via Helm chart instead of raw manifests

---

## 2. Install Argo CD on the cluster

### Option A — Official install manifests (simplest, most common)

```bash
# Create a dedicated namespace
kubectl create namespace argocd

# Install Argo CD components (CRDs, controllers, API server, UI, repo server, etc.)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to be ready:

```bash
kubectl get pods -n argocd -w
```

You should see pods like:

```
argocd-application-controller-...
argocd-applicationset-controller-...
argocd-dex-server-...
argocd-notifications-controller-...
argocd-redis-...
argocd-repo-server-...
argocd-server-...
```

### Option B — Helm chart (better if you manage everything else via Helm)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace
```

Customize via a `values.yaml` and `helm upgrade` if you need HA mode, custom resource limits, ingress config, etc. — see the chart's `values.yaml` for all options:

```bash
helm show values argo/argo-cd > argocd-values.yaml
# edit as needed, then:
helm upgrade argocd argo/argo-cd -n argocd -f argocd-values.yaml
```

### Option C — Argo CD CLI for managing Argo CD itself (not the cluster install)

The Argo CD **CLI** (`argocd`) is separate from installing the server — it's how you interact with it afterward. Install it locally:

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Windows (via Scoop)
scoop install argocd
```

Verify:

```bash
argocd version
```

---

## 3. Access the Argo CD UI

By default, the `argocd-server` Service is `ClusterIP` — not exposed outside the cluster. Pick one of the following depending on your environment.

### Option A — Port-forward (quickest, for local/testing)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open: `https://localhost:8080`

### Option B — Expose via LoadBalancer (cloud clusters)

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd   # note the EXTERNAL-IP
```

### Option C — Expose via Ingress (recommended for production)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" # argocd-server serves HTTPS internally
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: ["argocd.myapp.com"]
      secretName: argocd-server-tls
  rules:
    - host: argocd.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

```bash
kubectl apply -f argocd-ingress.yaml
```

---

## 4. Log in (UI and CLI)

Argo CD auto-generates an initial admin password on install, stored in a Secret:

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Log in via CLI

```bash
argocd login localhost:8080 --username admin --password <password-from-above> --insecure
# --insecure only needed if using a self-signed cert (e.g. via port-forward)
```

### Log in via UI

- Go to `https://localhost:8080` (or your Ingress/LB address)
- Username: `admin`
- Password: from the command above

### Change the admin password (do this immediately after first login)

```bash
argocd account update-password
```

---

## 5. Connect a Git repository

Argo CD needs read access to whichever repo holds your Kubernetes manifests/Helm charts.

### Public repo (no auth needed)

Nothing extra required — just reference the repo URL when creating an Application (§7).

### Private repo via HTTPS (username/token)

```bash
argocd repo add https://github.com/yourorg/your-manifests-repo.git \
  --username <git-username> \
  --password <personal-access-token>
```

### Private repo via SSH

```bash
argocd repo add git@github.com:yourorg/your-manifests-repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

### Verify connection

```bash
argocd repo list
```

---

## 6. Register target clusters

If Argo CD is only deploying to the cluster it's running on, this step is automatic (it registers `in-cluster` by default). To deploy to **additional/external clusters** (relevant if you're managing the 4-region setup from earlier from one central Argo CD instance):

```bash
# Make sure your kubeconfig has a context for the target cluster
kubectl config get-contexts

# Register it with Argo CD
argocd cluster add <context-name>
```

This creates a ServiceAccount + ClusterRoleBinding on the target cluster and stores its credentials as a Secret in the `argocd` namespace, letting one Argo CD instance manage multiple clusters centrally.

```bash
# Verify registered clusters
argocd cluster list
```

---

## 7. Create your first Application

An "Application" in Argo CD is the core object — it maps a Git source (path/branch) to a destination (cluster + namespace).

### Option A — Declarative (recommended, itself stored in Git — "app of apps" pattern)

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/your-manifests-repo.git
    targetRevision: main
    path: k8s/overlays/production # path within the repo containing manifests/kustomize/helm
  destination:
    server: https://kubernetes.default.svc # or the registered external cluster's API server URL
    namespace: api
  syncPolicy:
    automated:
      prune: true # remove resources deleted from Git
      selfHeal: true # auto-revert manual cluster changes back to match Git
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd-app.yaml -n argocd
```

### Option B — Via CLI

```bash
argocd app create my-app \
  --repo https://github.com/yourorg/your-manifests-repo.git \
  --path k8s/overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace api \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Option C — Via UI

`+ New App` → fill in: Application Name, Project (`default`), Sync Policy, Repository URL, Revision, Path, Cluster, Namespace → `Create`.

---

## 8. Sync, monitor, and manage Applications

```bash
# List all applications and their sync/health status
argocd app list

# Get full details on one app
argocd app get my-app

# Manually trigger a sync (if not using automated sync)
argocd app sync my-app

# View live diff between Git and cluster state
argocd app diff my-app

# Roll back to a previous sync revision
argocd app rollback my-app <history-id>

# Delete an application (and optionally its resources)
argocd app delete my-app --cascade
```

In the UI, the Application view shows a live resource tree — pods, services, deployments — color-coded by sync/health status (green = synced & healthy, yellow = progressing, red = degraded/out-of-sync).

---

## 9. Multi-cluster / multi-environment pattern (App of Apps)

For something like the earlier multi-region Flask deployment (US/SG/IN/AU clusters), the common pattern is one **parent Application** that itself points to a directory of **child Applications**, one per cluster/region:

```
repo/
├── apps/
│   ├── flask-api-us.yaml
│   ├── flask-api-sg.yaml
│   ├── flask-api-in.yaml
│   └── flask-api-au.yaml
└── root-app.yaml
```

```yaml
# root-app.yaml — the "app of apps"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/your-manifests-repo.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Each child Application file (e.g. `flask-api-sg.yaml`) targets its own registered cluster's API server and its own manifests path/overlay — one `kubectl apply -f root-app.yaml` then bootstraps and manages all four regional deployments from a single Git source of truth.

---

## 10. Useful day-2 operations

```bash
# Check Argo CD's own component health
kubectl get pods -n argocd

# View Argo CD server logs
kubectl logs -n argocd deploy/argocd-server

# Upgrade Argo CD itself (Option A install)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Upgrade via Helm
helm upgrade argocd argo/argo-cd -n argocd -f argocd-values.yaml

# Enable SSO (e.g. via Dex/OIDC) — edit the argocd-cm ConfigMap
kubectl edit configmap argocd-cm -n argocd
```

---

## 11. Notes & production hardening

- **Change the default admin password immediately** and consider disabling the local `admin` account entirely once SSO/RBAC is configured (`argocd-cm` → `accounts.admin: ...` settings, or `admin.enabled: "false"`).
- **Use projects (`AppProject`)** to scope which repos/clusters/namespaces a team's Applications are allowed to touch — avoids one team's Application accidentally deploying into another's namespace.
- **`selfHeal: true` is powerful but opinionated** — it will revert any manual `kubectl edit`/`kubectl patch` someone does directly on the cluster. Good for strict GitOps discipline; turn it off for namespaces where manual debugging tweaks are expected.
- **Use Sealed Secrets / External Secrets Operator** for any Secret manifests committed to Git — never commit plaintext credentials, even to a private repo.
- **Notifications**: the `argocd-notifications-controller` (installed by default) can be configured to post sync/health events to Slack, email, etc. — useful for alerting on failed syncs.
- **HA mode**: for production, install the HA manifests (`manifests/ha/install.yaml`) or the Helm chart with `redis-ha.enabled: true` and multiple replicas for the repo-server/application-controller.
