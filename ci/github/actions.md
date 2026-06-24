# GitHub Actions — Complete Notes

---

## 1. What is GitHub Actions?

GitHub Actions is a CI/CD platform built directly into GitHub. It lets you automate workflows triggered by events in your repository — push, pull request, release, cron schedule, etc.

Everything is defined in `.yml` files inside `.github/workflows/`.

---

## 2. Core Concepts

| Term         | What it means                                                                   |
| ------------ | ------------------------------------------------------------------------------- |
| **Workflow** | The entire automation file (`.yml`). One repo can have many workflows.          |
| **Event**    | What triggers the workflow (`push`, `pull_request`, `schedule`, etc.)           |
| **Job**      | A group of steps that run on the same machine. Jobs run in parallel by default. |
| **Step**     | A single task inside a job — either a shell command or a pre-built Action.      |
| **Action**   | A reusable unit of work (from Marketplace or your own). Used with `uses:`       |
| **Runner**   | The machine that executes the job (`ubuntu-latest`, `windows-latest`, etc.)     |
| **Secret**   | Encrypted variable stored in GitHub Settings → used as `${{ secrets.NAME }}`    |
| **Context**  | Built-in variables like `${{ github.sha }}`, `${{ github.ref }}`, etc.          |

---

## 3. Anatomy of a Workflow File — Every Line Explained

```yaml
name: CI Pipeline # Display name shown in GitHub Actions UI

on: # Defines what triggers this workflow
  push: # Trigger on git push
    branches: # But only for these branches
      - main
      - develop
  pull_request: # Also trigger on pull requests
    branches:
      - main
  workflow_dispatch: # Allows manual trigger from GitHub UI

env: # Global environment variables available to ALL jobs
  IMAGE_NAME: my-app # Can be referenced as $IMAGE_NAME in shell or ${{ env.IMAGE_NAME }} in yml

jobs: # All jobs live under this key
  build: # Job ID — you name this yourself (used for dependencies)
    name: Build and Test # Human-readable name shown in UI
    runs-on: ubuntu-latest # Runner OS — the virtual machine GitHub spins up

    steps: # Ordered list of tasks for this job
      - name: Checkout code # Human label for this step
        uses:
          actions/checkout@v4 # Uses the official GitHub checkout Action
          # `uses` = run a pre-built Action (from Marketplace or local)
          # @v4 = pinned version (always pin versions!)

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3 # Sets up advanced Docker builder (needed for multi-platform builds)

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with: # `with` passes inputs/arguments to an Action
          username: ${{ secrets.DOCKERHUB_USERNAME }} # ${{ secrets.X }} reads encrypted GitHub Secret
          password: ${{ secrets.DOCKERHUB_TOKEN }} # Never hardcode credentials!

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: . # Docker build context (the directory)
          push: true # Actually push to registry
          tags: myuser/myapp:latest # Image tag

      - name: Run a shell command # `run` executes raw shell commands
        run: echo "Hello World" # Single line command

      - name: Multi-line shell commands
        run: | # `|` means multi-line block — each line runs in sequence
          echo "Line 1"
          echo "Line 2"
          ls -la

      - name: Step with env override
        env: # Step-level env vars (override global ones)
          MY_VAR: hello
        run: echo $MY_VAR

  deploy: # Second job
    needs:
      build # `needs` = wait for `build` job to succeed before starting
      # Without this, jobs run in parallel
    runs-on: ubuntu-latest

    if: github.ref == 'refs/heads/main' # Conditional — only run this job on main branch

    steps:
      - name: Deploy
        run: echo "Deploying..."
```

---

## 4. Key GitHub Contexts (Built-in Variables)

```yaml
${{ github.sha }}           # Full commit SHA — e.g. a3f2c1d...  (great for image tags)
${{ github.ref }}           # Full ref — e.g. refs/heads/main
${{ github.ref_name }}      # Just the branch/tag name — e.g. main
${{ github.repository }}    # owner/repo — e.g. john/my-app
${{ github.actor }}         # Username who triggered the workflow
${{ github.event_name }}    # What triggered it — push, pull_request, etc.
${{ github.run_number }}    # Auto-incrementing run number
${{ secrets.MY_SECRET }}    # Encrypted secret from repo/org settings
${{ env.MY_VAR }}           # Environment variable defined in workflow
```

---

## 5. Setting Up Secrets

Go to: **GitHub Repo → Settings → Secrets and variables → Actions → New repository secret**

Secrets you'll commonly need:

- `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN` (for Docker Hub)
- `GHCR_TOKEN` (for GitHub Container Registry — use `${{ secrets.GITHUB_TOKEN }}` which is auto-provided)
- `SSH_PRIVATE_KEY` (for SSH into EC2/Droplet)
- `EC2_HOST`, `DROPLET_HOST`, `GCP_HOST` (server IP or hostname)
- `KUBECONFIG` or `KUBE_TOKEN` (for k8s)

---

## 6. Docker Image Tagging Strategy

```yaml
# Using commit SHA — unique, traceable, immutable
tags: myuser/myapp:${{ github.sha }}

# Using short SHA (first 7 chars) — cleaner
tags: myuser/myapp:${{ github.sha[:7] }}

# Using branch name + SHA
tags: myuser/myapp:main-${{ github.sha }}

# Using run number
tags: myuser/myapp:${{ github.run_number }}

# Multiple tags at once
tags: |
  myuser/myapp:latest
  myuser/myapp:${{ github.sha }}
```

**Best practice:** Always tag with SHA (never only `latest`) so you can roll back to an exact image.

---

## 7. Workflow: Flask API → Docker → Deploy to VM (EC2 / GCP / DigitalOcean)

This is the pattern: **Push to main → Build Docker image → Push to registry → SSH into VM → Pull and run new container**

```yaml
# .github/workflows/flask-deploy-vm.yml
name: Flask API — Build & Deploy to VM

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/flask-api
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    name: Build Docker Image
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: . # Dockerfile is in root
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

  deploy:
    name: Deploy to VM
    needs: build-and-push # Only runs if build succeeded
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3 # Popular Action for SSH commands
        with:
          host: ${{ secrets.VM_HOST }} # Your EC2/GCP/Droplet public IP
          username: ${{ secrets.VM_USER }} # e.g. ubuntu (EC2), root (Droplet)
          key: ${{ secrets.SSH_PRIVATE_KEY }} # Contents of your .pem or id_rsa private key
          script: |
            docker pull ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            docker stop flask-api || true              # Stop old container (ignore error if not running)
            docker rm flask-api || true                # Remove old container
            docker run -d \
              --name flask-api \
              --restart unless-stopped \
              -p 5000:5000 \
              -e SECRET_KEY=${{ secrets.SECRET_KEY }} \
              -e DB_URL=${{ secrets.DB_URL }} \
              ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```

**VM setup prerequisites** (do this once on your server):

```bash
# On the VM
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker ubuntu   # Allow ubuntu user to run docker without sudo
# Then log out and back in
```

---

## 8. Workflow: Node.js → Docker → Deploy to VM

```yaml
# .github/workflows/nodejs-deploy-vm.yml
name: Node.js — Build & Deploy to VM

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/nodejs-api
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: SSH Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            docker stop nodejs-api || true
            docker rm nodejs-api || true
            docker run -d \
              --name nodejs-api \
              --restart unless-stopped \
              -p 3000:3000 \
              -e NODE_ENV=production \
              -e DATABASE_URL=${{ secrets.DATABASE_URL }} \
              ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```

---

## 9. Workflow: Django → Docker → Deploy to VM

```yaml
# .github/workflows/django-deploy-vm.yml
name: Django — Build & Deploy to VM

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/django-app
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: SSH Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            docker stop django-app || true
            docker rm django-app || true
            docker run -d \
              --name django-app \
              --restart unless-stopped \
              -p 8000:8000 \
              -e DJANGO_SECRET_KEY=${{ secrets.DJANGO_SECRET_KEY }} \
              -e DATABASE_URL=${{ secrets.DATABASE_URL }} \
              -e DJANGO_ALLOWED_HOSTS=${{ secrets.VM_HOST }} \
              ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            # Run migrations inside the running container
            docker exec django-app python manage.py migrate --noinput
            docker exec django-app python manage.py collectstatic --noinput
```

---

## 10. Workflow: Spring Boot → Docker → Deploy to VM

```yaml
# .github/workflows/springboot-deploy-vm.yml
name: Spring Boot — Build & Deploy to VM

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/spring-api
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4 # Official Java setup Action
        with:
          java-version: "21"
          distribution: "temurin" # Eclipse Temurin (formerly AdoptOpenJDK)

      - name: Build JAR with Maven
        run: mvn clean package -DskipTests # Build the fat JAR

      # Alternative if using Gradle:
      # - name: Build JAR with Gradle
      #   run: ./gradlew bootJar

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: SSH Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            docker stop spring-api || true
            docker rm spring-api || true
            docker run -d \
              --name spring-api \
              --restart unless-stopped \
              -p 8080:8080 \
              -e SPRING_DATASOURCE_URL=${{ secrets.DB_URL }} \
              -e SPRING_DATASOURCE_USERNAME=${{ secrets.DB_USER }} \
              -e SPRING_DATASOURCE_PASSWORD=${{ secrets.DB_PASSWORD }} \
              ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```

---

## 11. GitOps Pattern — ArgoCD (Update k8s Manifest Instead of Deploying Directly)

This is the **GitOps** approach. The workflow does NOT deploy anything. It only:

1. Builds and pushes the Docker image
2. Updates the image tag in your k8s manifest YAML in the repo
3. Commits and pushes that change

ArgoCD watches your repo. When it sees the manifest changed, it applies it to the cluster automatically.

```
Developer pushes code
       ↓
GitHub Action builds image → pushes to Docker Hub with tag = commit SHA
       ↓
GitHub Action updates deployment.yml: image: myapp:OLD_SHA → image: myapp:NEW_SHA
       ↓
GitHub Action commits & pushes manifest change to repo
       ↓
ArgoCD detects change in repo → kubectl apply → cluster updated ✅
```

### The manifest update step — how it works

```yaml
- name: Update image tag in k8s manifest
  run: |
    # Use sed to find the old image line and replace with new SHA
    sed -i "s|image: myuser/myapp:.*|image: myuser/myapp:${{ github.sha }}|g" k8s/deployment.yml

- name: Commit and push manifest change
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add k8s/deployment.yml
    git commit -m "ci: update image tag to ${{ github.sha }}"
    git push
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Auto-provided by GitHub, no setup needed
```

---

## 12. Flask API — GitOps with ArgoCD

```yaml
# .github/workflows/flask-gitops.yml
name: Flask API — Build & Update Manifest (GitOps)

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/flask-api

jobs:
  build-push-update:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Update k8s Manifest
        run: |
          sed -i "s|image: ${{ env.IMAGE_NAME }}:.*|image: ${{ env.IMAGE_NAME }}:${{ github.sha }}|g" \
            k8s/deployment.yml

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/deployment.yml
          git commit -m "ci: bump flask-api to ${{ github.sha }}"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Your `k8s/deployment.yml` image line should look like:

```yaml
# Before (what ArgoCD currently has deployed)
image: myuser/flask-api:abc1234

# After workflow runs (what gets committed)
image: myuser/flask-api:def5678   ← ArgoCD sees this, syncs to cluster
```

---

## 13. Node.js — GitOps with ArgoCD

```yaml
# .github/workflows/nodejs-gitops.yml
name: Node.js — Build & Update Manifest (GitOps)

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/nodejs-api

jobs:
  build-push-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Update Manifest
        run: |
          sed -i "s|image: ${{ env.IMAGE_NAME }}:.*|image: ${{ env.IMAGE_NAME }}:${{ github.sha }}|g" \
            k8s/deployment.yml

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/deployment.yml
          git commit -m "ci: bump nodejs-api to ${{ github.sha }}"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 14. Django — GitOps with ArgoCD

```yaml
# .github/workflows/django-gitops.yml
name: Django — Build & Update Manifest (GitOps)

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/django-app

jobs:
  build-push-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Update Manifest
        run: |
          sed -i "s|image: ${{ env.IMAGE_NAME }}:.*|image: ${{ env.IMAGE_NAME }}:${{ github.sha }}|g" \
            k8s/deployment.yml

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/deployment.yml
          git commit -m "ci: bump django-app to ${{ github.sha }}"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 15. Spring Boot — GitOps with ArgoCD

```yaml
# .github/workflows/springboot-gitops.yml
name: Spring Boot — Build & Update Manifest (GitOps)

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/spring-api

jobs:
  build-push-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: "21"
          distribution: "temurin"

      - name: Build JAR
        run: mvn clean package -DskipTests

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Update Manifest
        run: |
          sed -i "s|image: ${{ env.IMAGE_NAME }}:.*|image: ${{ env.IMAGE_NAME }}:${{ github.sha }}|g" \
            k8s/deployment.yml

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/deployment.yml
          git commit -m "ci: bump spring-api to ${{ github.sha }}"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 16. Separate App Repo vs Separate Manifest Repo (GitOps)

There are two common GitOps repo structures:

### Option A — Same repo (monorepo)

```
my-app/
├── src/               ← application code
├── Dockerfile
├── k8s/
│   └── deployment.yml ← manifest lives here
└── .github/workflows/
    └── deploy.yml
```

The workflow commits back to the same repo. Simple, works with `${{ secrets.GITHUB_TOKEN }}`.

### Option B — Separate manifest repo (recommended for teams)

```
my-app/           ← application code + Dockerfile + GitHub Actions
my-app-k8s/       ← only k8s manifests, watched by ArgoCD
```

When manifests are in a separate repo, you need a **Personal Access Token (PAT)** to push to it:

```yaml
- name: Checkout manifest repo
  uses: actions/checkout@v4
  with:
    repository: yourorg/my-app-k8s # Different repo
    token: ${{ secrets.PAT_TOKEN }} # PAT with repo write access
    path: manifest-repo # Check it out into a subdirectory

- name: Update image tag
  run: |
    sed -i "s|image: myuser/myapp:.*|image: myuser/myapp:${{ github.sha }}|g" \
      manifest-repo/k8s/deployment.yml

- name: Commit & Push to manifest repo
  run: |
    cd manifest-repo
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add k8s/deployment.yml
    git commit -m "ci: bump myapp to ${{ github.sha }}"
    git push
  env:
    GITHUB_TOKEN: ${{ secrets.PAT_TOKEN }}
```

---

## 17. Setting Up ArgoCD to Watch Your Repo

Once your workflow is updating the manifest, tell ArgoCD where to look:

```yaml
# argocd-app.yml — apply this once to your cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flask-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/my-app # or my-app-k8s
    targetRevision: main # branch to watch
    path: k8s # folder containing manifests
  destination:
    server: https://kubernetes.default.svc # deploy to same cluster
    namespace: default
  syncPolicy:
    automated: # auto-sync when manifest changes
      prune: true # delete resources removed from manifest
      selfHeal: true # revert manual changes to cluster
```

```bash
kubectl apply -f argocd-app.yml
```

ArgoCD polls the repo every 3 minutes by default, or you can set up a webhook for instant sync.

---

## 18. Using GitHub Container Registry (GHCR) Instead of Docker Hub

GHCR is free for public repos and uses your GitHub token — no extra signup needed.

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }} # Your GitHub username
    password: ${{ secrets.GITHUB_TOKEN }} # Auto-provided, no setup needed

- name: Build and Push to GHCR
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ghcr.io/${{ github.repository }}:latest
      ghcr.io/${{ github.repository }}:${{ github.sha }}
      # e.g. ghcr.io/john/my-app:latest
      # e.g. ghcr.io/john/my-app:abc1234
```

---

## 19. Useful Patterns

### Cache Docker layers (speeds up builds)

```yaml
- name: Build and Push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myuser/myapp:${{ github.sha }}
    cache-from: type=gha # Read cache from GitHub Actions cache
    cache-to: type=gha,mode=max # Write cache back
```

### Cache pip / npm / maven dependencies

```yaml
# pip (Python)
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

# npm (Node.js)
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

# Maven (Java)
- uses: actions/cache@v4
  with:
    path: ~/.m2
    key: ${{ runner.os }}-maven-${{ hashFiles('pom.xml') }}
```

### Run tests before building image

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt && pytest # Flask/Django
      # OR
      - run: npm ci && npm test # Node.js
      # OR
      - run: mvn test # Spring Boot

  build:
    needs: test # Only build if tests pass
    runs-on: ubuntu-latest
    steps:
      - ...
```

### Only deploy on tags (release-based deployment)

```yaml
on:
  push:
    tags:
      - "v*" # Triggers only on tags like v1.0.0, v2.3.1

# Then use the tag as the image tag
tags: myuser/myapp:${{ github.ref_name }} # e.g. myapp:v1.0.0
```

---

## 20. Quick Reference — Required Secrets per Scenario

| Scenario                      | Secrets needed                                           |
| ----------------------------- | -------------------------------------------------------- |
| Push to Docker Hub            | `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`                  |
| Push to GHCR                  | None — uses auto `GITHUB_TOKEN`                          |
| SSH to EC2                    | `VM_HOST`, `VM_USER` (ubuntu), `SSH_PRIVATE_KEY`         |
| SSH to GCP VM                 | `VM_HOST`, `VM_USER` (e.g. your-user), `SSH_PRIVATE_KEY` |
| SSH to Droplet                | `VM_HOST`, `VM_USER` (root), `SSH_PRIVATE_KEY`           |
| GitOps same repo              | None extra — uses auto `GITHUB_TOKEN`                    |
| GitOps separate manifest repo | `PAT_TOKEN` (Personal Access Token with repo scope)      |

### How to get SSH_PRIVATE_KEY for EC2

```bash
# The .pem file AWS gives you IS the private key
cat my-key.pem    # Copy this entire output into the GitHub Secret
```

### How to get SSH_PRIVATE_KEY for GCP / Droplet

```bash
# Generate a key pair
ssh-keygen -t ed25519 -C "github-actions" -f github-actions-key

# Add the PUBLIC key to your server:
# GCP: Metadata → SSH keys
# DigitalOcean: Settings → Security → SSH Keys
# Then copy the PRIVATE key into the GitHub Secret:
cat github-actions-key   # This goes in SSH_PRIVATE_KEY secret
```
