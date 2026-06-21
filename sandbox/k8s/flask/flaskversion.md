# Build Notes: "Replit for Flask APIs" — Full System (Auth, Projects, Shared PVC, Neon Postgres, K8s Executor Pods)

This extends the ephemeral-pod code-runner you already have into a full multi-tenant platform: users sign up, log in with JWT, create projects, get auto-generated `requirements.txt` + `api.py` scaffolding written to a shared PVC, and can run their project in an isolated child pod that mounts the **same PVC** as the main API.

---

## 1. Architecture Overview

```
                        ┌─────────────────────────────┐
   Browser/Client  ───► │   Main API (Flask, k8s Pod)  │
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
                                    │  pip install -r reqs.txt      │
                                    │  python api.py  (stays up,    │
                                    │   self-heals on crash)        │
                                    └────────────┬───────────────────┘
                                                 │
                              both mount ──┐     │
                                           ▼     ▼
                              ┌──────────────────────────────┐
                              │   Shared PVC (ReadWriteMany)  │
                              │  /projects/<project_id>/      │
                              │    ├── api.py                 │
                              │    └── requirements.txt       │
                              └──────────────────────────────┘
```

**Key design decision:** the Main API mounts the PVC at its **root** (it needs to read/write _any_ user's project folder). Each ephemeral executor pod mounts the **same PVC**, but scoped with `subPath: projects/<project_id>` — so it can only ever see _its own_ project's folder, never anyone else's. Same PVC resource, different mount scope. This satisfies "main API and child pod share the same PVC" while keeping tenants isolated from each other.

---

## 2. Tech Stack

| Layer                     | Choice                                                                            |
| ------------------------- | --------------------------------------------------------------------------------- |
| Web framework             | Flask                                                                             |
| Auth                      | `flask-jwt-extended` (JWT access tokens) + `werkzeug.security` (password hashing) |
| ORM                       | `flask-sqlalchemy` (for the platform's own `users`/`projects` tables)             |
| Database                  | Postgres via **Neon** (serverless Postgres)                                       |
| Container orchestration   | Kubernetes Python client (`kubernetes` package)                                   |
| Generated project's stack | Flask + `psycopg2-binary` (kept simple/raw SQL for the user's own app)            |
| Production server         | `gunicorn`                                                                        |

---

## 3. Database Schema (Neon Postgres)

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- needed for gen_random_uuid()

CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at    TIMESTAMP DEFAULT NOW()
);

CREATE TABLE projects (
    id           VARCHAR(80) PRIMARY KEY,        -- e.g. "a1b2c3d4-my-todo-api-7e21a9b4"
    user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    project_name VARCHAR(100) NOT NULL,
    folder_path  VARCHAR(255) NOT NULL,           -- relative path inside the PVC
    created_at   TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_projects_user_id ON projects(user_id);
```

**Neon connection strings:** Neon gives you two endpoints — use the **pooled** one (`...-pooler.<region>.aws.neon.tech`) for the Main API, since Flask will open many short-lived connections under concurrent requests and Neon's built-in PgBouncer pooler handles that far better than a direct connection:

```
# Pooled (use this for the app)
NEON_DATABASE_URL=postgresql://user:password@ep-xxxx-pooler.us-east-2.aws.neon.tech/dbname?sslmode=require

# Direct (use this only for one-off admin tasks / migrations)
NEON_DIRECT_URL=postgresql://user:password@ep-xxxx.us-east-2.aws.neon.tech/dbname?sslmode=require
```

---

## 4. Project Structure (Main API codebase)

```
main-api/
├── app.py
├── config.py
├── extensions.py
├── models.py
├── utils.py
├── auth/
│   ├── __init__.py
│   └── routes.py
├── projects/
│   ├── __init__.py
│   ├── routes.py
│   ├── templates.py
│   ├── executor.py
│   └── deployer.py
├── k8s/
│   ├── pvc.yaml
│   ├── deployment.yaml
│   ├── rbac.yaml
│   └── networkpolicy.yaml
├── requirements.txt
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
```

---

## 6. `requirements.txt` (Main API itself)

```
flask==3.0.3
flask-sqlalchemy==3.1.1
flask-jwt-extended==4.6.0
flask-cors==4.0.1
psycopg2-binary==2.9.9
python-dotenv==1.0.1
kubernetes==29.0.0
gunicorn==22.0.0
```

---

## 7. `config.py`

```python
import os
from datetime import timedelta

class Config:
    SQLALCHEMY_DATABASE_URI = os.environ["NEON_DATABASE_URL"]
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_ENGINE_OPTIONS = {
        "pool_pre_ping": True,   # avoids using a dead connection after Neon idles it out
        "pool_size": 5,
        "max_overflow": 5,
    }

    JWT_SECRET_KEY = os.environ["JWT_SECRET_KEY"]
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(hours=12)

    PVC_MOUNT_PATH = os.environ.get("PVC_MOUNT_PATH", "/data")
    PVC_CLAIM_NAME = os.environ.get("PVC_CLAIM_NAME", "platform-projects-pvc")
    K8S_EXECUTOR_NAMESPACE = os.environ.get("K8S_EXECUTOR_NAMESPACE", "executor")
```

---

## 8. `extensions.py`

```python
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager

db = SQLAlchemy()
jwt = JWTManager()
```

---

## 9. `models.py`

```python
import uuid
from datetime import datetime
from extensions import db

class User(db.Model):
    __tablename__ = "users"

    id            = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    email         = db.Column(db.String(255), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    created_at    = db.Column(db.DateTime, default=datetime.utcnow)

    projects = db.relationship("Project", backref="owner", lazy=True, cascade="all, delete-orphan")


class Project(db.Model):
    __tablename__ = "projects"

    id           = db.Column(db.String(80), primary_key=True)
    user_id      = db.Column(db.String(36), db.ForeignKey("users.id"), nullable=False)
    project_name = db.Column(db.String(100), nullable=False)
    folder_path  = db.Column(db.String(255), nullable=False)
    created_at   = db.Column(db.DateTime, default=datetime.utcnow)

    def to_dict(self):
        return {
            "id": self.id,
            "project_name": self.project_name,
            "folder_path": self.folder_path,
            "created_at": self.created_at.isoformat(),
        }
```

---

## 10. `utils.py` — slug helper (keeps names filesystem- and Kubernetes-safe)

Kubernetes resource names must be lowercase, alphanumeric + hyphens, and ≤ 63 characters (DNS-1123). We sanitize every piece going into a project ID or pod name.

```python
import re
import uuid

def slugify(text: str, max_len: int = 40) -> str:
    text = text.lower().strip()
    text = re.sub(r"[^a-z0-9-]", "-", text)
    text = re.sub(r"-+", "-", text).strip("-")
    return text[:max_len] or "project"

def make_project_id(user_id: str, project_name: str) -> str:
    # Shortened user_id segment keeps the combined id well under k8s's 63-char limit
    user_part = slugify(user_id, max_len=8)
    name_part = slugify(project_name, max_len=30)
    suffix = uuid.uuid4().hex[:8]
    return f"{user_part}-{name_part}-{suffix}"
```

---

## 11. `auth/routes.py`

```python
from flask import Blueprint, request, jsonify
from werkzeug.security import generate_password_hash, check_password_hash
from flask_jwt_extended import create_access_token
from extensions import db
from models import User

auth_bp = Blueprint("auth", __name__, url_prefix="/auth")

@auth_bp.route("/signup", methods=["POST"])
def signup():
    data = request.get_json() or {}
    email = data.get("email", "").strip().lower()
    password = data.get("password", "")

    if not email or not password:
        return jsonify({"error": "email and password are required"}), 400
    if len(password) < 8:
        return jsonify({"error": "password must be at least 8 characters"}), 400

    if User.query.filter_by(email=email).first():
        return jsonify({"error": "An account with that email already exists"}), 409

    user = User(email=email, password_hash=generate_password_hash(password))
    db.session.add(user)
    db.session.commit()

    token = create_access_token(identity=user.id)
    return jsonify({"access_token": token, "user": {"id": user.id, "email": user.email}}), 201


@auth_bp.route("/login", methods=["POST"])
def login():
    data = request.get_json() or {}
    email = data.get("email", "").strip().lower()
    password = data.get("password", "")

    user = User.query.filter_by(email=email).first()
    if not user or not check_password_hash(user.password_hash, password):
        return jsonify({"error": "Invalid email or password"}), 401

    token = create_access_token(identity=user.id)
    return jsonify({"access_token": token, "user": {"id": user.id, "email": user.email}}), 200
```

---

## 12. `projects/templates.py` — generates the starter files for every new project

```python
def generate_requirements_txt() -> str:
    return (
        "flask==3.0.3\n"
        "flask-cors==4.0.1\n"
        "psycopg2-binary==2.9.9\n"
        "python-dotenv==1.0.1\n"
        "gunicorn==22.0.0\n"
    )


def generate_api_py() -> str:
    return '''import os
from flask import Flask, request, jsonify
import psycopg2
from psycopg2.extras import RealDictCursor

app = Flask(__name__)
DATABASE_URL = os.environ.get("DATABASE_URL")  # injected by the platform at run time


def get_conn():
    return psycopg2.connect(DATABASE_URL, sslmode="require")


def init_db():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS items (
            id SERIAL PRIMARY KEY,
            name TEXT NOT NULL,
            description TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );
    """)
    conn.commit()
    cur.close()
    conn.close()


@app.route("/items", methods=["GET"])
def list_items():
    conn = get_conn()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute("SELECT * FROM items ORDER BY id;")
    items = cur.fetchall()
    cur.close(); conn.close()
    return jsonify(items), 200


@app.route("/items", methods=["POST"])
def create_item():
    data = request.get_json()
    if not data or "name" not in data:
        return jsonify({"error": "name is required"}), 400
    conn = get_conn()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute(
        "INSERT INTO items (name, description) VALUES (%s, %s) RETURNING *;",
        (data["name"], data.get("description")),
    )
    item = cur.fetchone()
    conn.commit()
    cur.close(); conn.close()
    return jsonify(item), 201


@app.route("/items/<int:item_id>", methods=["GET"])
def get_item(item_id):
    conn = get_conn()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute("SELECT * FROM items WHERE id = %s;", (item_id,))
    item = cur.fetchone()
    cur.close(); conn.close()
    if not item:
        return jsonify({"error": "Not found"}), 404
    return jsonify(item), 200


@app.route("/items/<int:item_id>", methods=["PUT"])
def update_item(item_id):
    data = request.get_json() or {}
    conn = get_conn()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute(
        "UPDATE items SET name = %s, description = %s WHERE id = %s RETURNING *;",
        (data.get("name"), data.get("description"), item_id),
    )
    item = cur.fetchone()
    conn.commit()
    cur.close(); conn.close()
    if not item:
        return jsonify({"error": "Not found"}), 404
    return jsonify(item), 200


@app.route("/items/<int:item_id>", methods=["DELETE"])
def delete_item(item_id):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("DELETE FROM items WHERE id = %s;", (item_id,))
    deleted = cur.rowcount
    conn.commit()
    cur.close(); conn.close()
    if deleted == 0:
        return jsonify({"error": "Not found"}), 404
    return "", 204


if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=8080)
'''
```

This is intentionally a working, generic CRUD scaffold (`/items`) — the user edits it from there into whatever their actual API needs to be.

---

## 13. `projects/routes.py` — create / list / get / edit / delete

```python
import os
import shutil
from flask import Blueprint, request, jsonify, current_app
from flask_jwt_extended import jwt_required, get_jwt_identity
from extensions import db
from models import Project
from utils import make_project_id
from projects.templates import generate_requirements_txt, generate_api_py

projects_bp = Blueprint("projects", __name__, url_prefix="/projects")


def _project_or_404(project_id, user_id):
    project = Project.query.get(project_id)
    if project is None:
        return None, (jsonify({"error": "Project not found"}), 404)
    if project.user_id != user_id:
        return None, (jsonify({"error": "Forbidden"}), 403)
    return project, None


@projects_bp.route("", methods=["POST"])
@jwt_required()
def create_project():
    user_id = get_jwt_identity()
    data = request.get_json() or {}
    project_name = data.get("project_name", "").strip()

    if not project_name:
        return jsonify({"error": "project_name is required"}), 400

    project_id = make_project_id(user_id, project_name)
    folder_path = f"projects/{project_id}"
    full_path = os.path.join(current_app.config["PVC_MOUNT_PATH"], folder_path)

    os.makedirs(full_path, exist_ok=True)

    requirements_content = generate_requirements_txt()
    api_py_content = generate_api_py()

    with open(os.path.join(full_path, "requirements.txt"), "w") as f:
        f.write(requirements_content)
    with open(os.path.join(full_path, "api.py"), "w") as f:
        f.write(api_py_content)

    project = Project(
        id=project_id,
        user_id=user_id,
        project_name=project_name,
        folder_path=folder_path,
    )
    db.session.add(project)
    db.session.commit()

    return jsonify({
        **project.to_dict(),
        "files": {
            "requirements.txt": requirements_content,
            "api.py": api_py_content,
        },
    }), 201


@projects_bp.route("", methods=["GET"])
@jwt_required()
def list_projects():
    user_id = get_jwt_identity()
    projects = Project.query.filter_by(user_id=user_id).order_by(Project.created_at.desc()).all()
    return jsonify([p.to_dict() for p in projects]), 200


@projects_bp.route("/<project_id>", methods=["GET"])
@jwt_required()
def get_project(project_id):
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error
    return jsonify(project.to_dict()), 200


@projects_bp.route("/<project_id>/files/<path:filename>", methods=["GET"])
@jwt_required()
def read_file(project_id, filename):
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error
    if filename not in ("api.py", "requirements.txt"):
        return jsonify({"error": "Unsupported file"}), 400

    full_path = os.path.join(current_app.config["PVC_MOUNT_PATH"], project.folder_path, filename)
    if not os.path.isfile(full_path):
        return jsonify({"error": "File not found"}), 404
    with open(full_path) as f:
        content = f.read()
    return jsonify({"filename": filename, "content": content}), 200


@projects_bp.route("/<project_id>/files/<path:filename>", methods=["PUT"])
@jwt_required()
def write_file(project_id, filename):
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error
    if filename not in ("api.py", "requirements.txt"):
        return jsonify({"error": "Unsupported file"}), 400

    data = request.get_json() or {}
    content = data.get("content")
    if content is None:
        return jsonify({"error": "content is required"}), 400

    full_path = os.path.join(current_app.config["PVC_MOUNT_PATH"], project.folder_path, filename)
    with open(full_path, "w") as f:
        f.write(content)

    return jsonify({"message": "Saved"}), 200


@projects_bp.route("/<project_id>", methods=["DELETE"])
@jwt_required()
def delete_project(project_id):
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error

    full_path = os.path.join(current_app.config["PVC_MOUNT_PATH"], project.folder_path)
    shutil.rmtree(full_path, ignore_errors=True)

    db.session.delete(project)
    db.session.commit()
    return "", 204
```

---

## 14. `projects/executor.py` — quick smoke-test run (optional, NOT for hosting)

This is your original `/execute` pattern, adapted so the pod runs the **persisted project from the shared PVC** (scoped via `subPath`) instead of an inline code string. It boots the app, captures startup/runtime logs for a bounded window, then **force-stops it on purpose** via `timeout 20` — it's only meant to answer "does this project start cleanly," nothing more.

> ⚠️ **This pod is designed to self-terminate.** `timeout 20` kills the process after 20 seconds, the pod has `restartPolicy: Never`, and the polling loop below is explicitly waiting for it to reach `Succeeded`/`Failed`. If you want the Flask backend to actually stay running and be reachable, skip straight to **Section 15 — Deploy a project as a persistent service**, which uses a `Deployment` + `Service` instead of a one-shot `Pod`. Keep this smoke-test endpoint around if you still want a fast "does it even boot" check before deploying — the two aren't mutually exclusive.

```python
import time
from flask import Blueprint, jsonify, current_app
from flask_jwt_extended import jwt_required, get_jwt_identity
from kubernetes import client
from kubernetes.client.rest import ApiException
from models import Project

executor_bp = Blueprint("executor", __name__, url_prefix="/projects")

v1 = client.CoreV1Api()


@executor_bp.route("/<project_id>/run", methods=["POST"])
@jwt_required()
def run_project(project_id):
    user_id = get_jwt_identity()
    project = Project.query.get(project_id)
    if project is None:
        return jsonify({"error": "Project not found"}), 404
    if project.user_id != user_id:
        return jsonify({"error": "Forbidden"}), 403

    namespace = current_app.config["K8S_EXECUTOR_NAMESPACE"]
    pvc_claim_name = current_app.config["PVC_CLAIM_NAME"]

    pod_name = f"run-{project_id}-{int(time.time() * 1000)}"[:63].rstrip("-")

    pod_manifest = {
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {"name": pod_name, "labels": {"app": "executor", "project": project_id}},
        "spec": {
            "restartPolicy": "Never",
            "automountServiceAccountToken": False,  # user code can never call the k8s API
            "volumes": [{
                "name": "project-storage",
                "persistentVolumeClaim": {"claimName": pvc_claim_name},
            }],
            "containers": [{
                "name": "python-runner",
                "image": "python:3.10-slim",
                "workingDir": "/app",
                "command": [
                    "sh", "-c",
                    "pip install --no-cache-dir -r requirements.txt && timeout 20 python api.py"
                ],
                "env": [
                    {"name": "DATABASE_URL", "valueFrom": {
                        "secretKeyRef": {"name": "neon-credentials", "key": "database_url"}
                    }},
                ],
                "volumeMounts": [{
                    "name": "project-storage",
                    "mountPath": "/app",
                    "subPath": project.folder_path,  # <-- scopes this pod to ONLY this project's folder
                }],
                "resources": {
                    "limits": {"memory": "256Mi", "cpu": "500m"},
                    "requests": {"memory": "128Mi", "cpu": "100m"},
                },
                "securityContext": {
                    "allowPrivilegeEscalation": False,
                    "readOnlyRootFilesystem": False,  # pip install needs to write site-packages
                    "runAsNonRoot": True,
                    "runAsUser": 1000,
                },
            }],
        },
    }

    try:
        v1.create_namespaced_pod(body=pod_manifest, namespace=namespace)

        timeout = 30  # a little above the in-container `timeout 20` to allow for pip install + scheduling
        start_time = time.time()
        pod_completed = False

        while time.time() - start_time < timeout:
            pod_status = v1.read_namespaced_pod_status(name=pod_name, namespace=namespace)
            if pod_status.status.phase in ("Succeeded", "Failed"):
                pod_completed = True
                break
            time.sleep(0.5)

        if not pod_completed:
            return jsonify({"error": "Execution timeout", "output": "Run took too long (max 30s)."}), 400

        logs = v1.read_namespaced_pod_log(name=pod_name, namespace=namespace)
        pod_status = v1.read_namespaced_pod_status(name=pod_name, namespace=namespace)
        container_state = pod_status.status.container_statuses[0].state
        exit_code = container_state.terminated.exit_code if container_state.terminated else None

        return jsonify({"exit_code": exit_code, "output": logs}), 200

    except ApiException as e:
        return jsonify({"error": f"Kubernetes API Error: {e.reason}"}), 500
    except Exception as e:
        return jsonify({"error": str(e)}), 500
    finally:
        try:
            v1.delete_namespaced_pod(name=pod_name, namespace=namespace, propagation_policy="Background")
        except Exception:
            pass
```

---

## 15. `projects/deployer.py` — deploy a project as a persistent service (the real "Run" button)

This is the piece that actually matches Replit's behavior: clicking **Run** deploys the project as a long-running `Deployment` + `Service`, which keeps going — and restarts itself if it crashes — until you explicitly **Stop** it.

```python
import time
from flask import Blueprint, jsonify, current_app
from flask_jwt_extended import jwt_required, get_jwt_identity
from kubernetes import client
from kubernetes.client.rest import ApiException
from models import Project

deployer_bp = Blueprint("deployer", __name__, url_prefix="/projects")

apps_v1 = client.AppsV1Api()
core_v1 = client.CoreV1Api()

APP_PORT = 8080  # must match the port api.py listens on (see Section 12's template)


def _names(project_id):
    base = project_id[:50]  # leaves room for the "deploy-"/"svc-" prefix, stays under k8s's 63-char limit
    return f"deploy-{base}", f"svc-{base}"


def _project_or_404(project_id, user_id):
    project = Project.query.get(project_id)
    if project is None:
        return None, (jsonify({"error": "Project not found"}), 404)
    if project.user_id != user_id:
        return None, (jsonify({"error": "Forbidden"}), 403)
    return project, None


def _build_deployment_manifest(project, deployment_name, pvc_claim_name):
    return {
        "apiVersion": "apps/v1",
        "kind": "Deployment",
        "metadata": {"name": deployment_name, "labels": {"app": "executor", "project": project.id}},
        "spec": {
            "replicas": 1,
            "selector": {"matchLabels": {"project": project.id}},
            "template": {
                "metadata": {
                    "labels": {"app": "executor", "project": project.id},
                    # changing this annotation on every deploy call forces a rollout,
                    # so redeploying an already-running project picks up saved file edits
                    "annotations": {"platform/deployed-at": str(time.time())},
                },
                "spec": {
                    "automountServiceAccountToken": False,
                    "volumes": [{
                        "name": "project-storage",
                        "persistentVolumeClaim": {"claimName": pvc_claim_name},
                    }],
                    "containers": [{
                        "name": "python-runner",
                        "image": "python:3.10-slim",
                        "workingDir": "/app",
                        # NOTE: no `timeout` wrapper here — this is the key difference from Section 14.
                        # restartPolicy is implicitly "Always" for Deployment-managed pods, so if
                        # api.py crashes, Kubernetes restarts the container automatically.
                        "command": ["sh", "-c", "pip install --no-cache-dir -r requirements.txt && python api.py"],
                        "ports": [{"containerPort": APP_PORT}],
                        "env": [
                            {"name": "DATABASE_URL", "valueFrom": {
                                "secretKeyRef": {"name": "neon-credentials", "key": "database_url"}
                            }},
                        ],
                        "volumeMounts": [{
                            "name": "project-storage",
                            "mountPath": "/app",
                            "subPath": project.folder_path,
                        }],
                        "resources": {
                            "limits": {"memory": "256Mi", "cpu": "500m"},
                            "requests": {"memory": "128Mi", "cpu": "100m"},
                        },
                        "securityContext": {
                            "allowPrivilegeEscalation": False,
                            "readOnlyRootFilesystem": False,
                            "runAsNonRoot": True,
                            "runAsUser": 1000,
                        },
                        "readinessProbe": {
                            "tcpSocket": {"port": APP_PORT},
                            "initialDelaySeconds": 5,
                            "periodSeconds": 5,
                        },
                    }],
                },
            },
        },
    }


@deployer_bp.route("/<project_id>/deploy", methods=["POST"])
@jwt_required()
def deploy_project(project_id):
    """The 'Run' button. Creates the Deployment+Service if they don't exist yet;
    if they already exist, patches the Deployment to trigger a fresh rollout
    (so newly saved code gets picked up) instead of erroring out."""
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error

    namespace = current_app.config["K8S_EXECUTOR_NAMESPACE"]
    pvc_claim_name = current_app.config["PVC_CLAIM_NAME"]
    deployment_name, service_name = _names(project.id)
    manifest = _build_deployment_manifest(project, deployment_name, pvc_claim_name)

    try:
        apps_v1.read_namespaced_deployment(name=deployment_name, namespace=namespace)
        apps_v1.patch_namespaced_deployment(name=deployment_name, namespace=namespace, body=manifest)
        action = "redeployed"
    except ApiException as e:
        if e.status != 404:
            return jsonify({"error": f"Kubernetes API Error: {e.reason}"}), 500
        apps_v1.create_namespaced_deployment(namespace=namespace, body=manifest)
        action = "deployed"

    try:
        core_v1.read_namespaced_service(name=service_name, namespace=namespace)
    except ApiException as e:
        if e.status != 404:
            return jsonify({"error": f"Kubernetes API Error: {e.reason}"}), 500
        service_manifest = {
            "apiVersion": "v1",
            "kind": "Service",
            "metadata": {"name": service_name, "labels": {"project": project.id}},
            "spec": {
                "selector": {"project": project.id},
                "ports": [{"port": 80, "targetPort": APP_PORT}],
                "type": "ClusterIP",
            },
        }
        core_v1.create_namespaced_service(namespace=namespace, body=service_manifest)

    internal_url = f"http://{service_name}.{namespace}.svc.cluster.local"
    return jsonify({"message": action, "internal_url": internal_url}), 202


@deployer_bp.route("/<project_id>/status", methods=["GET"])
@jwt_required()
def project_status(project_id):
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error

    namespace = current_app.config["K8S_EXECUTOR_NAMESPACE"]
    deployment_name, _ = _names(project.id)

    try:
        deployment = apps_v1.read_namespaced_deployment(name=deployment_name, namespace=namespace)
    except ApiException as e:
        if e.status == 404:
            return jsonify({"status": "stopped"}), 200
        return jsonify({"error": f"Kubernetes API Error: {e.reason}"}), 500

    ready = deployment.status.ready_replicas or 0
    desired = deployment.spec.replicas or 0
    status = "running" if desired > 0 and ready >= desired else "starting"
    return jsonify({"status": status, "ready_replicas": ready, "desired_replicas": desired}), 200


@deployer_bp.route("/<project_id>/logs", methods=["GET"])
@jwt_required()
def project_logs(project_id):
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error

    namespace = current_app.config["K8S_EXECUTOR_NAMESPACE"]
    pods = core_v1.list_namespaced_pod(namespace=namespace, label_selector=f"project={project.id}")
    if not pods.items:
        return jsonify({"error": "No running pod for this project"}), 404

    pod_name = pods.items[0].metadata.name
    try:
        logs = core_v1.read_namespaced_pod_log(name=pod_name, namespace=namespace, tail_lines=200)
    except ApiException as e:
        return jsonify({"error": f"Kubernetes API Error: {e.reason}"}), 500

    return jsonify({"pod": pod_name, "logs": logs}), 200


@deployer_bp.route("/<project_id>/stop", methods=["POST"])
@jwt_required()
def stop_project(project_id):
    """The 'Stop' / 'close project' action. Deletes the Deployment and Service,
    which actually frees the compute — unlike just closing a browser tab."""
    user_id = get_jwt_identity()
    project, error = _project_or_404(project_id, user_id)
    if error:
        return error

    namespace = current_app.config["K8S_EXECUTOR_NAMESPACE"]
    deployment_name, service_name = _names(project.id)

    for delete_fn, name in (
        (apps_v1.delete_namespaced_deployment, deployment_name),
        (core_v1.delete_namespaced_service, service_name),
    ):
        try:
            delete_fn(name=name, namespace=namespace)
        except ApiException as e:
            if e.status != 404:
                return jsonify({"error": f"Kubernetes API Error: {e.reason}"}), 500

    return jsonify({"message": "stopped"}), 200
```

**Why a `Deployment` and not a bare long-running `Pod`?** A `Deployment` gives you self-healing for free — if the container crashes or the node it's on dies, Kubernetes recreates it automatically. A standalone `Pod` you create directly just stays dead. For something meant to act like "my backend is up and running until I say otherwise," `Deployment` is the right primitive.

---

## 16. `app.py` — ties it all together

```python
from flask import Flask
from flask_cors import CORS
from kubernetes import config as k8s_config
from config import Config
from extensions import db, jwt
from auth.routes import auth_bp
from projects.routes import projects_bp
from projects.executor import executor_bp
from projects.deployer import deployer_bp


def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)

    CORS(app)
    db.init_app(app)
    jwt.init_app(app)

    with app.app_context():
        db.create_all()

    try:
        k8s_config.load_incluster_config()
    except k8s_config.ConfigException:
        k8s_config.load_kube_config()

    app.register_blueprint(auth_bp)
    app.register_blueprint(projects_bp)
    app.register_blueprint(executor_bp)
    app.register_blueprint(deployer_bp)

    return app


app = create_app()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

## 17. Kubernetes Manifests

### `k8s/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: platform-projects-pvc
  namespace: executor
spec:
  accessModes:
    - ReadWriteMany # REQUIRED: the main API and many executor pods mount this concurrently,
      # often across different nodes — a ReadWriteOnce volume will NOT work here.
  resources:
    requests:
      storage: 20Gi
  storageClassName:
    efs-sc # swap for whatever RWX-capable storage class your cluster has
    # (AWS: efs-sc, Azure: azurefile, GCP/on-prem: NFS or CephFS)
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
          image: your-registry/main-api:latest
          ports:
            - containerPort: 5000
          envFrom:
            - secretRef:
                name: main-api-secrets # NEON_DATABASE_URL, JWT_SECRET_KEY
          volumeMounts:
            - name: project-storage
              mountPath: /data # full root — the main API manages every project's folder
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

### `k8s/rbac.yaml` — least-privilege permissions for the Main API to manage executor workloads

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

### `k8s/networkpolicy.yaml` — restrict what executor pods can talk to

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
    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Allow outbound HTTPS only (covers Neon's pooler endpoint)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
```

> Plain Kubernetes `NetworkPolicy` can't match by hostname/FQDN — only IPs/CIDRs and namespace/pod selectors. For tighter "only Neon, nothing else" egress control, use a CNI that supports FQDN-based policies (e.g., Cilium), or front the egress through an explicit proxy.

---

## 18. Sequences: "Run" (smoke test) vs "Deploy"/"Stop" (persistent)

**Smoke test (`/run`, Section 14) — self-terminates on purpose:**

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
   - volumeMount: same PVC, subPath = projects/<id>   (isolation)
   - command: pip install -r requirements.txt && timeout 20 python api.py
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

**Persistent run (`/deploy` + `/stop`, Section 15) — stays up until explicitly stopped:**

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

## 19. Security Notes (worth keeping in mind as you build this out)

- **`subPath` isolation is the whole ballgame.** Never mount the PVC root into an executor pod — always scope it to `projects/<project_id>`, or one tenant's code could read or overwrite another's.
- **`automountServiceAccountToken: false`** on every executor pod — user code should never be able to reach the Kubernetes API, even by accident.
- **RBAC is scoped to the `executor` namespace only**, and only grants `pods`/`pods/log` verbs — the Main API's service account can't touch anything else in the cluster.
- **No user input goes into a shell string directly.** The executor's command is a fixed string (`pip install -r requirements.txt && timeout 20 python api.py`); the user's actual code lives in a file on disk, not interpolated into a `-c` flag (which was the pattern in your original snippet).
- **Secrets (`NEON_DATABASE_URL`, `JWT_SECRET_KEY`) belong in Kubernetes Secrets**, referenced via `secretRef`/`secretKeyRef` — never baked into the image or committed to source control.
- **Rate-limit project creation and run-frequency per user** (e.g., with Flask-Limiter) once you're past prototyping — uncapped pod creation is both a cost risk and an abuse vector.
- **Persistent deployments need a cap, not just the smoke-test endpoint.** A `Deployment` that stays up until someone clicks "Stop" is a much bigger resource/cost liability than a 20-second ephemeral pod. Enforce a max number of _simultaneously running_ deployments per user (check this in `deploy_project` before calling `create_namespaced_deployment`), and consider an idle-timeout reaper — a scheduled job that calls `/stop` on any project with no recent activity (e.g., no log/status polling, or a "last active" timestamp you update on each request to that project).

---

## 20. Natural Next Steps (not implemented above, but worth knowing about)

- **Public exposure:** the `Service` created in Section 15 is `ClusterIP` — reachable from inside the cluster (e.g., by the Main API or an internal proxy), but not from the public internet. To give users a real, shareable URL (Replit's "webview" equivalent), add an `Ingress` (or a gateway) that routes `https://yourplatform.com/p/<project_id>/*` (or a per-project subdomain) to that project's Service.
- **Per-project database isolation:** right now all projects share one Neon database (the generated `api.py` just creates an `items` table). Neon supports instant DB branching — you could call Neon's API to spin up a separate branch per project for full data isolation.
- **Build/run quotas:** track CPU-seconds, run-count, or total running-time per user in the `projects`/a new `usage` table to enforce fair-use limits.
