# kubectl — Full Command Reference

Practical, copy-pasteable `kubectl` commands, organized by what you're trying to do — creating resources, generating YAML without applying it, inspecting/debugging, and managing rollouts. Examples use the resource names from your Instagram API setup (`instagram-api-deployment`, `postgres-service`, `instagram-api-hpa`, `postgres-pvc`, etc.) wherever it makes the example concrete.

---

## Table of Contents

1. Context & cluster basics
2. Creating resources imperatively (Pod, Deployment, Service)
3. Generating YAML without creating anything (`--dry-run`)
4. `get` — listing resources of every kind
5. `describe` — deep inspection
6. `logs` — reading container output
7. `exec` — running commands inside a container
8. `apply` vs `create` vs `replace`
9. `rollout` — status, history, undo, restart
10. Restarting things (Pods, Deployments)
11. Scaling
12. `edit`, `patch`, `delete`
13. Metrics server & `top`
14. Storage: PV, PVC, StorageClass
15. Port-forwarding & temporary debug access
16. Deleting/replacing images inside a kind cluster (crictl)
17. Real debugging session walkthrough — commands used, with their purpose
18. Quick cheat sheet

---

## 1. Context & Cluster Basics

```bash
kubectl version                       # client + server version
kubectl cluster-info                   # control plane / DNS endpoint info
kubectl get nodes                       # list all Nodes in the cluster
kubectl get nodes -o wide                # + extra columns (internal IP, OS, kernel, container runtime)

kubectl config get-contexts              # list all known clusters/contexts
kubectl config current-context           # which one you're pointed at right now
kubectl config use-context <name>         # switch context (e.g. switch between minikube/kind/cloud cluster)

kubectl get namespaces                    # list all namespaces
kubectl get pods -n kube-system             # -n / --namespace targets a specific namespace
kubectl config set-context --current --namespace=<ns>   # change your DEFAULT namespace for this context
```

---

## 2. Creating Resources Imperatively

These are quick one-off commands — useful for learning/debugging, but **manifests (`kubectl apply -f file.yml`) are the right way to manage anything real**, since imperative commands aren't reproducible/version-controlled.

```bash
# Pod
kubectl run my-pod --image=nginx:latest --port=80

# Deployment
kubectl create deployment instagram-api-deployment --image=instagram-api:latest --replicas=2 --port=5000

# Service — exposes an existing Deployment/Pod
kubectl expose deployment instagram-api-deployment --port=80 --target-port=5000 --type=ClusterIP --name=instagram-api-service

# ConfigMap (from literals, or from a file)
kubectl create configmap instagram-config --from-literal=DB_HOST=postgres-service --from-literal=DB_PORT=5432
kubectl create configmap instagram-config --from-env-file=.env

# Secret
kubectl create secret generic instagram-secret --from-literal=DB_USER=postgres --from-literal=DB_PASSWORD=yourpassword
```

---

## 3. Generating YAML Without Creating Anything — `--dry-run`

This is the single most useful trick for _learning_ manifest structure, or for quickly scaffolding a YAML file you'll then hand-edit, instead of writing one from scratch.

```bash
# Pod YAML, printed to stdout, NOTHING actually created
kubectl run my-pod --image=nginx:latest --port=80 --dry-run=client -o yaml

# Deployment YAML
kubectl create deployment instagram-api-deployment \
  --image=instagram-api:latest --replicas=2 --port=5000 \
  --dry-run=client -o yaml

# Service YAML
kubectl create service clusterip instagram-api-service --tcp=80:5000 --dry-run=client -o yaml

# Redirect straight into a file you can then edit by hand
kubectl create deployment instagram-api-deployment --image=instagram-api:latest --dry-run=client -o yaml > deployment.yml
```

**`--dry-run=client` vs `--dry-run=server`:**

| Mode     | What it does                                                                                                                                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client` | Builds the object **entirely locally** — never even talks to the API server. Fastest, purely for generating YAML.                                                                                                                                       |
| `server` | Sends the request to the API server, which validates/runs admission webhooks as if it were really happening, but **doesn't persist it**. Useful for catching real validation errors (e.g. a bad field name) that client-side generation wouldn't catch. |

```bash
kubectl apply -f deployment.yml --dry-run=server   # validate an existing file against the live API, without applying it
```

---

## 4. `get` — Listing Resources

```bash
kubectl get pods                          # Pods in current namespace
kubectl get pods -A                        # -A / --all-namespaces — across the WHOLE cluster
kubectl get pods -o wide                    # + Node, IP, etc.
kubectl get pods --show-labels               # see every label on each Pod
kubectl get pods -l app=instagram-api          # filter by label selector
kubectl get pods --field-selector=status.phase=Running   # filter by a field instead of a label
kubectl get pods -w                          # watch — stream live updates as things change

kubectl get deployments                       # short: kubectl get deploy
kubectl get replicasets                        # short: kubectl get rs
kubectl get services                            # short: kubectl get svc
kubectl get configmaps                           # short: kubectl get cm
kubectl get secrets
kubectl get hpa
kubectl get pvc
kubectl get pv
kubectl get storageclass                          # short: kubectl get sc
kubectl get events --sort-by=.lastTimestamp           # cluster-wide events, oldest first by default — sort to see latest

kubectl get all                                # everything common (pods, svc, deploy, rs) in the current namespace
kubectl api-resources                           # list EVERY resource type the cluster knows about, with short names

# Get a resource's full YAML back out (handy after creating something imperatively)
kubectl get deployment instagram-api-deployment -o yaml
kubectl get pod <pod-name> -o json
kubectl get svc instagram-api-service -o jsonpath='{.spec.clusterIP}'   # extract a single field
```

---

## 5. `describe` — Deep Inspection

`describe` gives you a **human-readable summary** including the live **Events** feed at the bottom — almost always the first command to run when something's broken (exactly what we used to find the `gunicorn not found` error earlier in this conversation).

```bash
kubectl describe pod instagram-api-deployment-5c6bf849bf-45vhg
kubectl describe deployment instagram-api-deployment
kubectl describe service instagram-api-service
kubectl describe hpa instagram-api-hpa
kubectl describe pvc postgres-pvc
kubectl describe node test-cluster-worker2
```

**What to look at in the output:**

- **`Status` / `Conditions`** — current high-level state (`Running`, `Pending`, `CrashLoopBackOff`...).
- **`Events` (bottom section)** — the actual timeline of what the scheduler/kubelet did and why, including pull errors, OOMKills, failed mounts, failed probes. This is where you find the _real_ root cause, not just the symptom.
- For a Pod specifically: **`Last State`** under each container shows the previous exit code/reason if it's restarted.

---

## 6. `logs` — Reading Container Output

```bash
kubectl logs <pod-name>                          # current container's stdout/stderr
kubectl logs <pod-name> -f                        # follow live, like `tail -f`
kubectl logs <pod-name> --previous                  # logs from the PREVIOUS crashed instance of this container
                                                       # (essential for CrashLoopBackOff — the live container has no logs yet)
kubectl logs <pod-name> -c <container-name>           # if the Pod has multiple containers, specify which one
kubectl logs <pod-name> --since=10m                    # only the last 10 minutes
kubectl logs <pod-name> --tail=50                       # only the last 50 lines

# Logs across an entire Deployment's Pods at once, by label
kubectl logs -l app=instagram-api --tail=50 --prefix
#                                              ^^^^^^^ prefixes each line with the source Pod's name
```

---

## 7. `exec` — Running Commands Inside a Container

```bash
kubectl exec -it <pod-name> -- bash                 # interactive shell (note the `--` before the command!)
kubectl exec -it <pod-name> -- sh                     # if bash isn't installed (common in slim/alpine images)
kubectl exec <pod-name> -- env                          # one-off command, non-interactive — check env vars
kubectl exec <pod-name> -- cat /etc/hosts
kubectl exec <pod-name> -c <container-name> -- bash       # target a specific container in a multi-container Pod

# Through a Deployment instead of a specific Pod name (picks one matching Pod automatically)
kubectl exec -it deploy/instagram-api-deployment -- bash
```

**Important limitation you already hit:** `exec` only works on a **currently running** container. If the container has already crashed (e.g. `CrashLoopBackOff`), there's nothing to attach to — that's when you reach for `kubectl logs --previous` and `kubectl describe pod` instead.

---

## 8. `apply` vs `create` vs `replace`

| Command                       | Behavior                                                                                                                                            |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kubectl create -f file.yml`  | Creates a NEW resource. **Fails if it already exists.**                                                                                             |
| `kubectl apply -f file.yml`   | Creates if missing, or **intelligently patches** an existing resource to match the file (only changes the diff). What you should use almost always. |
| `kubectl replace -f file.yml` | Fully **overwrites** an existing resource with the file's content — fails if it doesn't already exist (use `--force` to delete+recreate if needed). |

```bash
kubectl apply -f deployment.yml
kubectl apply -f k8s/                  # apply every manifest in a directory at once
kubectl apply -f k8s/ --recursive        # also descend into subdirectories
kubectl diff -f deployment.yml            # see exactly what `apply` WOULD change, without changing anything
```

This is exactly the `apply`-driven workflow used throughout the earlier Instagram API setup — and `apply`'s "patch, don't fully overwrite" behavior is also precisely why the PVC resize attempt failed earlier: it tried to _patch_ an immutable field instead of erroring out as cleanly as `replace` would have.

---

## 9. `rollout` — Status, History, Undo, Restart

Applies to Deployments, DaemonSets, and StatefulSets — anything that manages a rollout via ReplicaSets.

```bash
kubectl rollout status deployment instagram-api-deployment      # watch a rollout until it completes (or fails)
kubectl rollout history deployment instagram-api-deployment       # list past revisions
kubectl rollout history deployment instagram-api-deployment --revision=2   # full details of one specific revision

kubectl rollout undo deployment instagram-api-deployment           # roll back to the PREVIOUS revision
kubectl rollout undo deployment instagram-api-deployment --to-revision=2   # roll back to a SPECIFIC revision

kubectl rollout pause deployment instagram-api-deployment           # pause a rollout mid-way (e.g. for canary-style manual control)
kubectl rollout resume deployment instagram-api-deployment            # resume a paused rollout

kubectl rollout restart deployment instagram-api-deployment             # see Section 10
```

**Why `rollout history` entries appear at all:** every time you change a Deployment's Pod template (image, env vars, etc.), Kubernetes creates a **new ReplicaSet** under the hood (as covered in the k8s concepts notes) and records it as a new revision — `rollout undo` works by scaling the _previous_ ReplicaSet back up and the current one down, not by "undoing" individual field changes.

---

## 10. Restarting Things

There is **no direct "restart a Pod" command** — Pods are meant to be disposable, so the idiomatic way to restart is to either delete the Pod (letting its controller recreate it) or trigger a fresh rollout at the Deployment level.

```bash
# Restart a single Pod — delete it, the Deployment/ReplicaSet immediately creates a replacement
kubectl delete pod <pod-name>

# Restart EVERY Pod in a Deployment, one at a time, with zero downtime
# (does a rolling replacement using the SAME image/config — useful after fixing
#  something external, like reloading a ConfigMap-derived env var, or recovering
#  from a one-off bad state, without actually changing the manifest)
kubectl rollout restart deployment instagram-api-deployment

# Restart by scaling down to 0 then back up (heavier-handed, causes a brief full outage —
# generally avoid this for anything with >0 minReplicas under an HPA)
kubectl scale deployment instagram-api-deployment --replicas=0
kubectl scale deployment instagram-api-deployment --replicas=2
```

**`rollout restart` is what you used earlier** after running `kind load docker-image` — it's the right tool specifically because `imagePullPolicy: IfNotPresent` means simply re-applying the same YAML wouldn't force the kubelet to notice a freshly loaded image; `rollout restart` forces new Pods (and therefore a fresh image-presence check) regardless of whether the manifest itself changed.

---

## 11. Scaling

```bash
kubectl scale deployment instagram-api-deployment --replicas=4         # manual scale
kubectl autoscale deployment instagram-api-deployment --min=2 --max=8 --cpu-percent=60   # imperative HPA creation
                                                                                            # (equivalent to applying Hpa.yml)
```

---

## 12. `edit`, `patch`, `delete`

```bash
kubectl edit deployment instagram-api-deployment        # opens the live object in $EDITOR, applies on save
                                                            # — convenient for a quick one-off tweak, but NOT
                                                            #   tracked anywhere; prefer editing the YAML file + apply

kubectl patch deployment instagram-api-deployment -p '{"spec":{"replicas":3}}'   # scriptable, surgical field change

kubectl delete -f deployment.yml                  # delete everything defined in a file
kubectl delete deployment instagram-api-deployment   # delete by name directly
kubectl delete pods --all                            # delete every Pod in the current namespace (controllers recreate them!)
kubectl delete pod <pod-name> --grace-period=0 --force   # force-delete a stuck/Terminating Pod (use as a last resort)
```

---

## 13. Metrics Server & `top`

The HPA (and `kubectl top`) both depend on the **metrics-server** add-on being installed — without it, there's no source of real CPU/memory usage data in the cluster at all.

```bash
# Check if it's already running
kubectl get deployment metrics-server -n kube-system

# minikube
minikube addons enable metrics-server

# kind — metrics-server isn't built in; install it manually
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# kind clusters often need this patch too, since metrics-server defaults to requiring valid TLS certs
# that kind's internal cert setup doesn't satisfy out of the box:
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Once running:
kubectl top nodes                     # CPU/memory usage per Node
kubectl top pods                       # CPU/memory usage per Pod, current namespace
kubectl top pods -A                     # across all namespaces
kubectl top pods --containers            # break down by individual container within each Pod

# Confirm the HPA can actually see metrics (won't show <unknown> once this works)
kubectl get hpa instagram-api-hpa
kubectl describe hpa instagram-api-hpa     # shows current vs target utilization, and scaling Events
```

---

## 14. Storage — PV, PVC, StorageClass

```bash
kubectl get pvc                            # list PersistentVolumeClaims, current namespace
kubectl get pvc -A                          # across all namespaces
kubectl describe pvc postgres-pvc             # see Bound status, capacity, the PV it's bound to, and Events

kubectl get pv                              # list PersistentVolumes (cluster-scoped, no namespace)
kubectl describe pv <pv-name>                  # see reclaim policy, capacity, which PVC (if any) claimed it

kubectl get storageclass                      # short: kubectl get sc
kubectl get storageclass -o yaml                # see provisioner, allowVolumeExpansion, reclaim policy, etc.
kubectl describe storageclass standard            # the "standard" class is minikube/many local clusters' default

# Mark a StorageClass as the cluster default (only one should be default at a time)
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Delete a PVC (only works if no Pod currently has it mounted)
kubectl delete pvc postgres-pvc
```

**Reminder from the earlier troubleshooting session:** if `allowVolumeExpansion` is `false` (or missing) on the relevant StorageClass, a PVC's `storage:` size genuinely cannot be patched in place — `kubectl describe storageclass <name>` is exactly where you'd confirm that before attempting a resize again.

---

## 15. Port-Forwarding & Temporary Debug Access

```bash
kubectl port-forward pod/<pod-name> 8080:5000           # forward local:8080 -> Pod's container port 5000
kubectl port-forward svc/instagram-api-service 8080:80     # forward through a Service instead of a specific Pod
kubectl port-forward deployment/instagram-api-deployment 8080:5000   # via a Deployment (picks one matching Pod)

# Temporary debug Pod, auto-deleted on exit — handy for testing DNS/connectivity from inside the cluster
kubectl run debug --rm -it --image=busybox -- sh
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash   # richer image with curl/dig/nslookup etc. preinstalled

# Run a one-off command against a Service to test it from inside the cluster
kubectl run curl-test --rm -it --image=curlimages/curl -- curl http://instagram-api-service/items
```

---

## 16. Deleting/Replacing Images Inside a `kind` Cluster (`crictl`)

A `kind` cluster doesn't use your host's Docker daemon to run containers internally — each `kind` node is itself a container, running **containerd** inside it, with its own separate image store. This is exactly why `docker build` on your host doesn't automatically make an image visible to the cluster, and why `kind load docker-image` is needed to copy it in. It also means **stale/bad images can get stuck inside that node's containerd store**, and `docker rmi` on your host does nothing to remove them — you have to reach into the node itself.

`crictl` is the CLI for talking to a CRI-compatible runtime (containerd, in `kind`'s case) — think of it as "`docker` commands, but for the runtime actually being used inside the node."

### Get a shell into the kind node first

```bash
kind get clusters                          # confirm the cluster name (you saw "test-cluster" earlier)
docker exec -it test-cluster-worker2 bash    # kind nodes ARE docker containers on your host — exec straight in
```

### List images currently sitting in that node's containerd store

```bash
crictl images
# or, more detail:
crictl images -o json
```

This is the real source of truth for "does this node actually have the image I think it has" — compare it against what `kubectl describe pod <name>` shows under `Image:` if you're ever unsure whether a stale image is the problem.

### Delete a specific image

```bash
crictl rmi instagram-api:latest
```

If a Pod is currently using that image, `crictl rmi` will typically refuse or warn — stop/delete the Pod (or scale the Deployment to 0) first if needed:

```bash
kubectl scale deployment instagram-api-deployment --replicas=0
crictl rmi instagram-api:latest
kubectl scale deployment instagram-api-deployment --replicas=2
```

### Delete every unused image (containerd's version of `docker image prune`)

```bash
crictl rmi --prune
```

### Full "force a totally clean reload" sequence

This is the most reliable fix when you suspect a node is serving a stale/broken image despite rebuilding and reloading — confirm it's actually gone before reloading:

```bash
# 1. Exec into the node
docker exec -it test-cluster-worker2 bash

# 2. Confirm what's there, and remove the stale one
crictl images
crictl rmi instagram-api:latest
exit

# 3. Rebuild on your host and reload fresh into the cluster
docker build -t instagram-api:latest .
kind load docker-image instagram-api:latest --name test-cluster

# 4. Force Pods to pick it up
kubectl rollout restart deployment instagram-api-deployment

# 5. Verify the new image is actually there now
docker exec -it test-cluster-worker2 crictl images
```

**Why this matters with `imagePullPolicy: IfNotPresent`** (as in your Deployment manifest): that setting means the kubelet will happily keep using whatever image with that exact tag is _already present_ on the node, even if you've rebuilt a "better" one with the same tag on your host and reloaded it — `crictl images` is how you confirm whether the node's copy is actually current, and `crictl rmi` plus a fresh `kind load` is the guaranteed way to eliminate any ambiguity, rather than guessing whether your latest fix actually made it into the running container.

---

## 17. Real Debugging Session Walkthrough — Commands Used, With Their Purpose

This section captures the actual sequence used while debugging the Instagram API (CrashLoopBackOff → missing tables → JWT secret → multi-replica long-polling) — each command with **why** it was run at that point, not just what it does.

### Checking whether the HPA is fighting your manual scaling

```bash
kubectl get hpa                                    # is one even active? what are its min/max?
kubectl get pods -l app=instagram-api                # how many Pods exist RIGHT NOW
```

**Use case:** before assuming a bug, confirm the actual current state — an HPA silently re-scaling you back up looks identical to "my fix didn't work" if you don't check this first.

```bash
kubectl delete hpa instagram-api-hpa                  # remove the controller fighting your manual scale
kubectl scale deployment instagram-api-deployment --replicas=1   # now it actually sticks
kubectl get pods -l app=instagram-api -w                # watch it settle to exactly 1, live
```

**Use case:** isolating a multi-replica bug requires _actually_ getting down to 1 replica — deleting the HPA (not just scaling) is what makes that stick, since the HPA would otherwise immediately undo a plain `kubectl scale`.

```bash
kubectl apply -f k8s/Hpa.yml                            # restore real min/max once the test is done
```

**Use case:** don't leave the cluster without autoscaling after a debug session — re-apply from the manifest (the source of truth) rather than trying to reconstruct flags by hand.

```bash
kubectl patch hpa instagram-api-hpa -p '{"spec":{"minReplicas":1,"maxReplicas":1}}'
# ... test ...
kubectl apply -f k8s/Hpa.yml   # restore afterward
```

**Use case:** an alternative to deleting the HPA outright — pins it to exactly 1 replica without removing the object entirely, useful if you want to keep its history/events intact during the test.

### Capturing logs without losing the signal in noise

```bash
kubectl logs -l app=instagram-api -f --prefix --max-log-requests=10
```

**Use case:** tailing logs across _every_ Pod in a Deployment at once, with `--prefix` so each line is labeled by source Pod — `--max-log-requests` raises the default cap of 5 concurrent streams, needed once you're running more than 5 replicas (we hit this exact error mid-session).

```bash
kubectl scale deployment instagram-api-deployment --replicas=1
kubectl logs -l app=instagram-api -f --prefix
```

**Use case:** the simpler alternative to raising `--max-log-requests` — scale to 1 Pod first, so there's only one possible source for any log line, removing all ambiguity about which replica handled a given request.

### Confirming environment variables actually made it into a running container

```bash
kubectl exec -it deploy/instagram-api-deployment -- env | grep SECRET_KEY
```

**Use case:** after updating a Secret/ConfigMap, this is the definitive check that the **currently running** Pod actually has the value — applying a Secret does NOT hot-reload env vars into already-running containers, so this also doubles as a check for "did I actually restart the Pods after this change."

### Getting a missing schema into a live Postgres Pod

```bash
kubectl cp schema.sql postgres-deployment-858dfcf647-tqbdz:/tmp/schema.sql
kubectl exec -it postgres-deployment-858dfcf647-tqbdz -- psql -U postgres -d instagram_api -f /tmp/schema.sql
```

**Use case:** `kubectl cp` copies a local file straight into a running Pod's filesystem (no image rebuild needed) — used here to get `schema.sql` onto the Postgres Pod, then `psql -f` executes it directly against the live database. This is exactly how the `CREATE TABLE` statements got applied after discovering the database was empty.

### Inspecting what's actually running inside a container, at the process level

```bash
kubectl exec -it instagram-api-deployment-5c6bf849bf-hwzmz -- ps aux
```

**Use case:** confirms whether gunicorn (or anything) is even running inside the container right now, and how many worker processes exist — directly relevant to diagnosing the `--workers 4` long-polling bug, since each row here is a separate process with its own memory space.

```bash
kubectl exec -it instagram-api-deployment-5c6bf849bf-hwzmz -- cat /proc/1/cmdline
```

**Use case:** shows the _exact_ command line PID 1 (the container's main process) was actually started with — the definitive way to confirm what `CMD`/`ENTRYPOINT` really resolved to at runtime, useful when a Dockerfile looks right but you want to rule out a stale image or an overridden command.

---

## 18. Quick Cheat Sheet

| Goal                                                   | Command                                                                                                     |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| Generate Pod YAML without creating it                  | `kubectl run x --image=y --dry-run=client -o yaml`                                                          |
| Generate Deployment YAML                               | `kubectl create deployment x --image=y --dry-run=client -o yaml`                                            |
| Generate Service YAML                                  | `kubectl expose deployment x --port=80 --dry-run=client -o yaml`                                            |
| List everything common                                 | `kubectl get all`                                                                                           |
| Why is this Pod broken?                                | `kubectl describe pod <name>` (check Events at bottom)                                                      |
| What did the crashed container print?                  | `kubectl logs <name> --previous`                                                                            |
| Get a shell inside a running container                 | `kubectl exec -it <name> -- bash`                                                                           |
| Apply / update from YAML                               | `kubectl apply -f file.yml`                                                                                 |
| See what apply would change first                      | `kubectl diff -f file.yml`                                                                                  |
| Roll back a bad deploy                                 | `kubectl rollout undo deployment <name>`                                                                    |
| Force Pods to refresh (e.g. after loading a new image) | `kubectl rollout restart deployment <name>`                                                                 |
| Manually scale                                         | `kubectl scale deployment <name> --replicas=N`                                                              |
| Check HPA is reading real metrics                      | `kubectl describe hpa <name>`                                                                               |
| See live CPU/memory per Pod                            | `kubectl top pods`                                                                                          |
| Is my PVC actually bound?                              | `kubectl get pvc`                                                                                           |
| Can this PVC be resized?                               | `kubectl describe storageclass <name>` (check `allowVolumeExpansion`)                                       |
| Quick connectivity test from inside the cluster        | `kubectl run debug --rm -it --image=busybox -- sh`                                                          |
| List images actually inside a kind node                | `docker exec -it <node> crictl images`                                                                      |
| Delete a stale image inside a kind node                | `docker exec -it <node> crictl rmi <image:tag>`                                                             |
| Reload a fresh image into a kind cluster               | `kind load docker-image <image:tag> --name <cluster>`                                                       |
| Is the HPA fighting my manual scale?                   | `kubectl get hpa`                                                                                           |
| Stop the HPA from re-scaling during a test             | `kubectl delete hpa <name>` (or `kubectl patch hpa <name> -p '{"spec":{"minReplicas":1,"maxReplicas":1}}'`) |
| Tail logs from >5 replicas at once                     | `kubectl logs -l app=<label> -f --prefix --max-log-requests=10`                                             |
| Confirm an env var actually reached a running Pod      | `kubectl exec -it deploy/<name> -- env \| grep <VAR>`                                                       |
| Copy a local file into a running Pod                   | `kubectl cp <local-file> <pod>:<path>`                                                                      |
| Run a SQL file against a live Postgres Pod             | `kubectl exec -it <pg-pod> -- psql -U <user> -d <db> -f <path>`                                             |
| See every process running inside a container           | `kubectl exec -it <pod> -- ps aux`                                                                          |
| See the exact command PID 1 was started with           | `kubectl exec -it <pod> -- cat /proc/1/cmdline`                                                             |
