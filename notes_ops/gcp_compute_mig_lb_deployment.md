# Deploying APIs to GCP Compute Engine — Spring Boot, Node/Express (TS), Flask — with MIGs + Load Balancer

Three minimal APIs (one `GET`, one `POST` each), then the same GCP deployment pattern applied three times: **Instance Template → Managed Instance Group (MIG) → Health Check → Backend Service → Load Balancer**. The app layer changes; the infrastructure pattern doesn't.

---

## Table of Contents

1. The three APIs (code)
2. GCP deployment pattern, conceptually
3. Common GCP setup (project, firewall, network)
4. Deploying the Spring Boot API
5. Deploying the Node/Express + TypeScript API
6. Deploying the Flask API
7. Putting one Load Balancer in front of all three (path-based routing) — optional
8. Verifying everything, and tearing it down

---

## 1. The Three APIs

Each app exposes:
- `GET /items` — returns a static/in-memory list
- `POST /items` — accepts JSON, appends to the in-memory list, returns the created item
- `GET /health` — for the GCP health check (Section 2)

### 1a. Spring Boot (Java)

```
spring-api/
├── pom.xml
└── src/main/java/com/example/api/
    ├── ApiApplication.java
    ├── Item.java
    └── ItemController.java
```

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
  </parent>
  <groupId>com.example</groupId>
  <artifactId>api</artifactId>
  <version>1.0.0</version>
  <properties>
    <java.version>17</java.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

```java
// ApiApplication.java
package com.example.api;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiApplication.class, args);
    }
}
```

```java
// Item.java
package com.example.api;

public class Item {
    private Long id;
    private String name;

    public Item() {}
    public Item(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

```java
// ItemController.java
package com.example.api;

import org.springframework.web.bind.annotation.*;
import java.util.List;
import java.util.ArrayList;
import java.util.concurrent.atomic.AtomicLong;

@RestController
public class ItemController {

    private final List<Item> items = new ArrayList<>();
    private final AtomicLong counter = new AtomicLong(0);

    public ItemController() {
        items.add(new Item(counter.incrementAndGet(), "First item"));
    }

    @GetMapping("/health")
    public String health() {
        return "ok";
    }

    @GetMapping("/items")
    public List<Item> getItems() {
        return items;
    }

    @PostMapping("/items")
    public Item createItem(@RequestBody Item item) {
        Item created = new Item(counter.incrementAndGet(), item.getName());
        items.add(created);
        return created;
    }
}
```

Default port is `8080`. Run locally with `./mvnw spring-boot:run`.

### 1b. Node.js + Express + TypeScript

```
node-api/
├── package.json
├── tsconfig.json
└── src/
    └── server.ts
```

```json
// package.json
{
  "name": "node-api",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/server.js",
    "dev": "ts-node src/server.ts"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "ts-node": "^10.9.2",
    "@types/express": "^4.17.21",
    "@types/node": "^20.14.0"
  }
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"]
}
```

```ts
// src/server.ts
import express, { Request, Response } from "express";

const app = express();
app.use(express.json());

interface Item {
  id: number;
  name: string;
}

const items: Item[] = [{ id: 1, name: "First item" }];
let nextId = 2;

app.get("/health", (_req: Request, res: Response) => {
  res.status(200).send("ok");
});

app.get("/items", (_req: Request, res: Response) => {
  res.status(200).json(items);
});

app.post("/items", (req: Request, res: Response) => {
  const { name } = req.body;
  if (!name) {
    return res.status(400).json({ error: "name is required" });
  }
  const newItem: Item = { id: nextId++, name };
  items.push(newItem);
  res.status(201).json(newItem);
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`Node API running on port ${PORT}`);
});
```

Build with `npm run build`, run with `npm start`. Listens on `8080` by default (GCP convention — see Section 2).

### 1c. Python Flask

```
flask-api/
├── requirements.txt
└── app.py
```

```
# requirements.txt
flask
gunicorn
```

```python
# app.py
from flask import Flask, request, jsonify

app = Flask(__name__)

items = [{"id": 1, "name": "First item"}]
next_id = 2

@app.route("/health", methods=["GET"])
def health():
    return "ok", 200

@app.route("/items", methods=["GET"])
def get_items():
    return jsonify(items), 200

@app.route("/items", methods=["POST"])
def create_item():
    global next_id
    data = request.get_json()
    if not data or "name" not in data:
        return jsonify({"error": "name is required"}), 400
    item = {"id": next_id, "name": data["name"]}
    next_id += 1
    items.append(item)
    return jsonify(item), 201

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

Run in production with `gunicorn --bind 0.0.0.0:8080 app:app`.

---

## 2. GCP Deployment Pattern, Conceptually

All three apps get deployed the **same way**, regardless of language:

```
1. Instance Template   — "blueprint": machine type, OS image, startup script that installs + runs the app
2. Managed Instance Group (MIG) — a controller that creates N VM instances from that template, auto-heals
                                    crashed ones, and (optionally) autoscales based on load
3. Health Check         — an HTTP probe (GET /health) the Load Balancer uses to decide which instances
                            are actually ready to receive traffic
4. Backend Service      — groups the MIG + health check together as one logical "backend"
5. Load Balancer        — the public entry point; routes incoming traffic to healthy instances
                            in the backend service
```

This mirrors the Kubernetes Deployment → ReplicaSet → Pod chain from your k8s notes, except here the "Pod" is a real VM and the "Deployment" is the MIG. A **MIG is GCP's equivalent of a Kubernetes Deployment for VMs** — it's the thing that guarantees "N instances of this exact template are always running," recreating any that crash or get deleted.

**Why each app must listen on `0.0.0.0`, not `127.0.0.1`:** Compute Engine VMs have their own internal IP — binding to `localhost`/`127.0.0.1` would make the app reachable only from *inside that one VM*, invisible to the Load Balancer. All three apps above already do this correctly (Flask explicitly, Node/Spring Boot default to all interfaces).

**Why `/health` matters so much here:** unlike Kubernetes (which has built-in liveness/readiness probes baked into the Pod spec), on GCP the Load Balancer's health check is the *only* signal deciding whether an instance receives traffic at all. An instance that's "up" but whose app crashed will keep receiving traffic forever without a correctly configured health check pointing at a real app endpoint.

---

## 3. Common GCP Setup (Do Once)

```bash
# Set your project (replace with your actual project ID)
gcloud config set project YOUR_PROJECT_ID

# Enable required APIs
gcloud services enable compute.googleapis.com

# Firewall rule: allow the Load Balancer's health checker IP ranges to reach your VMs
# (these two ranges are fixed, documented GCP health-check source ranges)
gcloud compute firewall-rules create allow-health-check \
  --network=default \
  --action=ALLOW \
  --direction=INGRESS \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --rules=tcp:8080

# Firewall rule: allow SSH for debugging (optional but recommended while setting up)
gcloud compute firewall-rules create allow-ssh \
  --network=default \
  --action=ALLOW \
  --direction=INGRESS \
  --source-ranges=0.0.0.0/0 \
  --rules=tcp:22
```

We'll repeat **Instance Template → MIG → Health Check → Backend Service → Load Balancer** three times below, once per app, using **different ports per app** (`8080` Spring Boot, `8081` Node, `8082` Flask) so all three can coexist on the same project without colliding, each behind its own Load Balancer IP. (Section 7 shows an alternative: one Load Balancer, path-based routing to all three.)

---

## 4. Deploying the Spring Boot API

### Step 1 — Build a deployable artifact and put it somewhere the VM can fetch it

Simplest approach for this walkthrough: build the JAR locally, upload it to a Cloud Storage bucket, and have the VM's startup script download and run it.

```bash
# Build the JAR
cd spring-api
./mvnw clean package
# produces target/api-1.0.0.jar

# Create a bucket (once) and upload the JAR
gcloud storage buckets create gs://YOUR_PROJECT_ID-artifacts --location=us-central1
gcloud storage cp target/api-1.0.0.jar gs://YOUR_PROJECT_ID-artifacts/api-1.0.0.jar
```

### Step 2 — Write the startup script

```bash
# startup-spring.sh
#!/bin/bash
apt-get update
apt-get install -y openjdk-17-jre-headless
mkdir -p /opt/app
gsutil cp gs://YOUR_PROJECT_ID-artifacts/api-1.0.0.jar /opt/app/api.jar
cat <<EOF > /etc/systemd/system/spring-api.service
[Unit]
Description=Spring Boot API
After=network.target

[Service]
ExecStart=/usr/bin/java -jar /opt/app/api.jar --server.port=8080
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable spring-api
systemctl start spring-api
```

### Step 3 — Instance Template

```bash
gcloud compute instance-templates create spring-api-template \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=startup-spring.sh \
  --tags=http-server
```

### Step 4 — Managed Instance Group (MIG)

```bash
gcloud compute instance-groups managed create spring-api-mig \
  --base-instance-name=spring-api \
  --template=spring-api-template \
  --size=2 \
  --zone=us-central1-a

# Optional: autoscaling, same spirit as the HPA from the k8s notes
gcloud compute instance-groups managed set-autoscaling spring-api-mig \
  --zone=us-central1-a \
  --max-num-replicas=5 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90
```

### Step 5 — Health Check

```bash
gcloud compute health-checks create http spring-api-health-check \
  --port=8080 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3
```

### Step 6 — Backend Service + Attach the MIG

```bash
# Named port: tells GCP which port on each instance actually serves traffic
gcloud compute instance-groups managed set-named-ports spring-api-mig \
  --zone=us-central1-a \
  --named-ports=http:8080

gcloud compute backend-services create spring-api-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=spring-api-health-check \
  --global

gcloud compute backend-services add-backend spring-api-backend \
  --instance-group=spring-api-mig \
  --instance-group-zone=us-central1-a \
  --global
```

### Step 7 — Load Balancer (URL map + target proxy + forwarding rule)

```bash
gcloud compute url-maps create spring-api-lb \
  --default-service=spring-api-backend

gcloud compute target-http-proxies create spring-api-proxy \
  --url-map=spring-api-lb

gcloud compute addresses create spring-api-ip --global

gcloud compute forwarding-rules create spring-api-forwarding-rule \
  --address=spring-api-ip \
  --global \
  --target-http-proxy=spring-api-proxy \
  --ports=80
```

```bash
# Get the public IP
gcloud compute addresses describe spring-api-ip --global --format="get(address)"

# Test it (after a minute or two for health checks to pass and propagate)
curl http://<THE_IP>/items
curl -X POST http://<THE_IP>/items -H "Content-Type: application/json" -d '{"name":"New item"}'
```

---

## 5. Deploying the Node/Express + TypeScript API

Same five resources, different startup script — Node apps deploy faster using a simple `git clone` + `npm install` + `npm run build` pattern instead of a prebuilt artifact (works well since `npm install` is fast and reliable on a fresh VM).

### Step 1 — Push your code somewhere fetchable (GitHub, or a Cloud Storage zip — GitHub shown here)

```bash
git init && git add . && git commit -m "init"
git remote add origin https://github.com/your-username/node-api.git
git push -u origin main
```

### Step 2 — Startup script

```bash
# startup-node.sh
#!/bin/bash
apt-get update
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs git
git clone https://github.com/your-username/node-api.git /opt/app
cd /opt/app
npm install
npm run build
cat <<EOF > /etc/systemd/system/node-api.service
[Unit]
Description=Node Express API
After=network.target

[Service]
WorkingDirectory=/opt/app
Environment=PORT=8081
ExecStart=/usr/bin/node dist/server.js
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable node-api
systemctl start node-api
```

### Step 3 — Instance Template

```bash
gcloud compute instance-templates create node-api-template \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=startup-node.sh \
  --tags=http-server
```

### Step 4 — MIG

```bash
gcloud compute instance-groups managed create node-api-mig \
  --base-instance-name=node-api \
  --template=node-api-template \
  --size=2 \
  --zone=us-central1-a

gcloud compute instance-groups managed set-autoscaling node-api-mig \
  --zone=us-central1-a \
  --max-num-replicas=5 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6
```

### Step 5 — Health Check + Named Port

```bash
gcloud compute health-checks create http node-api-health-check \
  --port=8081 \
  --request-path=/health

gcloud compute instance-groups managed set-named-ports node-api-mig \
  --zone=us-central1-a \
  --named-ports=http:8081
```

### Step 6 — Backend Service

```bash
gcloud compute backend-services create node-api-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=node-api-health-check \
  --global

gcloud compute backend-services add-backend node-api-backend \
  --instance-group=node-api-mig \
  --instance-group-zone=us-central1-a \
  --global
```

### Step 7 — Load Balancer

```bash
gcloud compute url-maps create node-api-lb \
  --default-service=node-api-backend

gcloud compute target-http-proxies create node-api-proxy \
  --url-map=node-api-lb

gcloud compute addresses create node-api-ip --global

gcloud compute forwarding-rules create node-api-forwarding-rule \
  --address=node-api-ip \
  --global \
  --target-http-proxy=node-api-proxy \
  --ports=80
```

Also open the firewall for port `8081` health checks (Section 3's rule only covered `8080`):

```bash
gcloud compute firewall-rules create allow-health-check-node \
  --network=default \
  --action=ALLOW \
  --direction=INGRESS \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --rules=tcp:8081
```

---

## 6. Deploying the Flask API

### Step 1 — Push your code somewhere fetchable

Same as Node — push `flask-api/` to a Git repo, or upload a zip to Cloud Storage. Git shown here for consistency.

### Step 2 — Startup script

```bash
# startup-flask.sh
#!/bin/bash
apt-get update
apt-get install -y python3 python3-pip python3-venv git
git clone https://github.com/your-username/flask-api.git /opt/app
cd /opt/app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cat <<EOF > /etc/systemd/system/flask-api.service
[Unit]
Description=Flask API
After=network.target

[Service]
WorkingDirectory=/opt/app
ExecStart=/opt/app/venv/bin/gunicorn --bind 0.0.0.0:8082 app:app
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable flask-api
systemctl start flask-api
```

### Step 3 — Instance Template

```bash
gcloud compute instance-templates create flask-api-template \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=startup-flask.sh \
  --tags=http-server
```

### Step 4 — MIG

```bash
gcloud compute instance-groups managed create flask-api-mig \
  --base-instance-name=flask-api \
  --template=flask-api-template \
  --size=2 \
  --zone=us-central1-a

gcloud compute instance-groups managed set-autoscaling flask-api-mig \
  --zone=us-central1-a \
  --max-num-replicas=5 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6
```

### Step 5 — Health Check + Named Port

```bash
gcloud compute health-checks create http flask-api-health-check \
  --port=8082 \
  --request-path=/health

gcloud compute instance-groups managed set-named-ports flask-api-mig \
  --zone=us-central1-a \
  --named-ports=http:8082
```

### Step 6 — Backend Service

```bash
gcloud compute backend-services create flask-api-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=flask-api-health-check \
  --global

gcloud compute backend-services add-backend flask-api-backend \
  --instance-group=flask-api-mig \
  --instance-group-zone=us-central1-a \
  --global
```

### Step 7 — Load Balancer

```bash
gcloud compute url-maps create flask-api-lb \
  --default-service=flask-api-backend

gcloud compute target-http-proxies create flask-api-proxy \
  --url-map=flask-api-lb

gcloud compute addresses create flask-api-ip --global

gcloud compute forwarding-rules create flask-api-forwarding-rule \
  --address=flask-api-ip \
  --global \
  --target-http-proxy=flask-api-proxy \
  --ports=80
```

```bash
gcloud compute firewall-rules create allow-health-check-flask \
  --network=default \
  --action=ALLOW \
  --direction=INGRESS \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --rules=tcp:8082
```

---

## 7. Optional — One Load Balancer, Three Backends (Path-Based Routing)

Instead of three separate public IPs, you can route all three APIs behind **one** Load Balancer using URL path matching — `/spring/*` → Spring Boot, `/node/*` → Node, `/flask/*` → Flask. This reuses the **same three backend services** created above; only the `url-maps` step changes.

```bash
gcloud compute url-maps create unified-api-lb \
  --default-service=spring-api-backend

gcloud compute url-maps add-path-matcher unified-api-lb \
  --path-matcher-name=api-matcher \
  --default-service=spring-api-backend \
  --path-rules="/node/*=node-api-backend,/flask/*=flask-api-backend"

gcloud compute target-http-proxies create unified-api-proxy \
  --url-map=unified-api-lb

gcloud compute addresses create unified-api-ip --global

gcloud compute forwarding-rules create unified-api-forwarding-rule \
  --address=unified-api-ip \
  --global \
  --target-http-proxy=unified-api-proxy \
  --ports=80
```

```bash
curl http://<UNIFIED_IP>/items          # -> Spring Boot (default backend)
curl http://<UNIFIED_IP>/node/items     # -> Node API
curl http://<UNIFIED_IP>/flask/items    # -> Flask API
```

Note: with this approach each app would need to actually serve its routes under the matching path prefix (or you'd add URL rewriting in the path matcher) — shown here primarily to illustrate that a Load Balancer's URL map is what decides "which backend service handles this request," entirely independent of which MIG/VMs are behind it.

---

## 8. Verifying Everything, and Tearing It Down

### Verify

```bash
# Confirm MIG instances are healthy
gcloud compute instance-groups managed list-instances spring-api-mig --zone=us-central1-a

# Confirm the backend service sees healthy backends
gcloud compute backend-services get-health spring-api-backend --global

# Hit each API directly
curl http://<SPRING_IP>/items
curl http://<NODE_IP>/items
curl http://<FLASK_IP>/items

curl -X POST http://<SPRING_IP>/items -H "Content-Type: application/json" -d '{"name":"From curl"}'
```

If `get-health` shows `HEALTHY` for all instances and `curl` returns your JSON list, the full chain — Instance Template → MIG → Health Check → Backend Service → Load Balancer — is wired correctly end to end.

### Tear down (avoid ongoing billing) — repeat per app, swapping the name prefix

```bash
gcloud compute forwarding-rules delete spring-api-forwarding-rule --global --quiet
gcloud compute target-http-proxies delete spring-api-proxy --quiet
gcloud compute url-maps delete spring-api-lb --quiet
gcloud compute addresses delete spring-api-ip --global --quiet
gcloud compute backend-services delete spring-api-backend --global --quiet
gcloud compute health-checks delete spring-api-health-check --quiet
gcloud compute instance-groups managed delete spring-api-mig --zone=us-central1-a --quiet
gcloud compute instance-templates delete spring-api-template --quiet
```

(Then repeat with `node-api-*` and `flask-api-*`, and finally remove the firewall rules from Section 3 if no longer needed.)

---

## Quick Reference — GCP Resource ↔ Kubernetes Equivalent

| GCP Compute Engine concept | Closest Kubernetes equivalent |
|---|---|
| Instance Template | Pod template (inside a Deployment's `spec.template`) |
| Managed Instance Group (MIG) | Deployment (+ ReplicaSet) — keeps N instances alive, recreates failures |
| MIG autoscaling policy | HorizontalPodAutoscaler |
| Health Check | readinessProbe / livenessProbe |
| Backend Service | Service (groups backends + health check) |
| Load Balancer (URL map, proxy, forwarding rule) | Ingress / Service of type LoadBalancer |
| Named port | `targetPort` in a Service |
