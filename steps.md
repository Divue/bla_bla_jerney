# 🛤️ Implementation Steps
### How a DevOps Engineer Approaches This Project — Start to Finish

> This is the full walkthrough of how the `devops` branch was built on top of the existing application.
> Each section links to deeper notes where every concept is explained from scratch.

---

## The Starting Point

The development team has already delivered:
- A working React + Node.js + PostgreSQL application
- A GitHub repo with source code on the `main` branch
- A bare-metal EC2 deployment script (`deploy/setup.sh`)

**This is where the DevOps engineer's journey begins.**

The job: take this application and make it production-grade — containerised, orchestrated, infrastructure-as-code, and fully automated with a security-first CI/CD pipeline.

```
main branch (given)          devops branch (built here)
───────────────────          ──────────────────────────
React source code      ───▶  Docker containers
Node.js source code    ───▶  Kubernetes on EKS
EC2 setup script       ───▶  Terraform VPC + EKS
Manual deployment      ───▶  GitHub Actions CI/CD
No security scanning   ───▶  Trivy + Checkov + Hadolint
```

---

## Step 0 — Understand the Application

Before touching any DevOps tooling, read the codebase and run it locally.

```bash
git clone <repo-url>
cd jerney-cloud-native-devsecops
git checkout main

# Read the setup script — it tells you everything about how the app runs
cat deploy/setup.sh
```

Key things to note from the setup script:
- Backend runs on **port 5000** with Node.js
- Frontend is built with `npm run build` and served by Nginx
- Database is PostgreSQL 16 with specific credentials
- Nginx acts as a reverse proxy — forwards `/api` calls to the backend

### Run on EC2 (bare-metal, the way it was)

```bash
# On an EC2 instance (t2.medium, Ubuntu 22.04, 16GB volume)
# Security Group: allow port 22 (SSH) and 80 (HTTP)

git clone <repo-url> ~/Jerney
cd ~/Jerney
chmod +x deploy/setup.sh
./setup.sh

# App is live at http://<EC2_PUBLIC_IP>
```

### Run locally (no Docker)

```bash
# Backend
cd backend
npm install
export DB_HOST=localhost DB_PORT=5432 DB_USER=jerney_user
export DB_PASSWORD=jerney_pass_2026 DB_NAME=jerney_db PORT=5000
npm start

# Frontend (separate terminal)
cd frontend
npm install
npm run dev
# http://localhost:3000
```

---

## Step 1 — Provision AWS Infrastructure with Terraform

Now that you understand the app, create the cloud infrastructure it will run on.

> 📖 Deep dive: [`terraform/notes.md`](./terraform/notes.md)

### Why Terraform?

Clicking through the AWS Console is fine for learning but doesn't scale. Terraform defines infrastructure as code — version-controlled, repeatable, and destroyable with one command.

### Setup on EC2 (where Terraform is run from)

```bash
# Install Terraform
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor \
  | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt-get install terraform

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure AWS credentials
# AWS Console → IAM → Security Credentials → Create Access Key
aws configure
# AWS Access Key ID:     <your-key>
# AWS Secret Access Key: <your-secret>
# Default region:        ap-south-1
# Default output format: json
```

### Write the Terraform files (in order)

```
terraform/
├── providers.tf     ← 1st: declare AWS provider + version constraints
├── variables.tf     ← 2nd: declare all input variables with types + defaults
├── terraform.tfvars ← 3rd: actual values for this environment (overrides defaults)
├── main.tf          ← 4th: the real infrastructure — VPC module + EKS module
└── outputs.tf       ← 5th: values to print after apply (endpoint, VPC ID, etc.)
```

**Key decisions in `main.tf`:**
- **VPC module** — `terraform-aws-modules/vpc/aws` creates subnets, NAT Gateway, Internet Gateway, and route tables automatically. Don't write these from scratch.
- **EKS Auto Mode** — AWS manages node scaling, patching, and replacement. No node group configuration needed. Karpenter runs under the hood.
- **Private subnets for nodes** — worker nodes have no public IPs, reducing attack surface.
- **Single NAT Gateway** — cost-saving for dev (~$32/month vs ~$96/month for one per AZ).

### Run Terraform

```bash
cd terraform/

terraform init      # downloads AWS provider + VPC/EKS modules (~2 min)
terraform plan      # preview all resources — should show ~47 to add
                    # always read this before applying
terraform apply     # type 'yes' to confirm — takes ~10-15 min for EKS
```

### Connect kubectl to the new cluster

```bash
aws eks update-kubeconfig --name jerney-eks --region ap-south-1

# Verify connection
kubectl get nodes
# Should show nodes in Ready state

# Quick smoke test
kubectl create deployment nginx --image nginx:latest
kubectl get pods
kubectl delete deployment nginx
```

---

## Step 2 — Write Dockerfiles

> 📖 Deep dive: [`frontend/dockerfile_notes.md`](./frontend/dockerfile_notes.md) | [`backend/dockerfile_notes.md`](./backend/dockerfile_notes.md)

Both Dockerfiles use **multi-stage builds** — the key pattern for production containers.

### Multi-stage build — why it matters

```
Stage 1 (heavy — build tools)      Stage 2 (light — runtime only)
──────────────────────────         ──────────────────────────────
Full Node.js + all packages         Only what the app needs to RUN
Compile / bundle the code           No source code
Produces output artifact            No devDependencies
        │                           No build tools
        │ COPY --from=stage1         Smaller image (~25MB vs ~400MB)
        ▼                           Smaller attack surface
   only the output crosses
```

### Frontend Dockerfile

```dockerfile
# Stage 1: build the React app
FROM node:20-alpine AS build
RUN apk update && apk upgrade --no-cache     # patch OS vulnerabilities
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci && npm cache clean --force        # cacheable layer — only reruns if package.json changes
COPY . .
RUN npm run build                            # outputs compiled files to /app/dist

# Stage 2: serve with Nginx
FROM nginx:1.27-alpine AS production
RUN rm -rf /etc/nginx/conf.d/default.conf /usr/share/nginx/html/*
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html    # only the dist folder crosses stages
RUN chown -R nginx:nginx /usr/share/nginx/html \
    && chown -R nginx:nginx /var/cache/nginx \
    && chown -R nginx:nginx /var/log/nginx \
    && touch /var/run/nginx.pid \
    && chown -R nginx:nginx /var/run/nginx.pid
USER nginx                                   # non-root
EXPOSE 8080                                  # port 8080 because non-root can't use <1024
CMD ["nginx", "-g", "daemon off;"]
```

### Backend Dockerfile

```dockerfile
# Stage 1: install production dependencies only
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --only=production && npm cache clean --force   # skips devDependencies

# Stage 2: production runtime
FROM node:20-alpine AS production
RUN apk update && apk upgrade --no-cache     # patch OS in the final image (stage 1 is discarded)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup   # custom locked non-root user
RUN apk add --no-cache dumb-init=1.2.5-r3   # PID 1 signal handling — critical for graceful shutdown
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY src/ ./src/
RUN chown -R appuser:appgroup /app
USER appuser
EXPOSE 5000
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/index.js"]
```

**Why `dumb-init`?** Node.js isn't designed to be PID 1. Without it, `kubectl delete pod` sends SIGTERM which Node.js doesn't handle properly — in-flight requests get dropped and DB connections aren't closed cleanly. `dumb-init` sits as PID 1 and correctly forwards signals.

### Build and test locally

```bash
docker build -t jerney-frontend:latest ./frontend
docker build -t jerney-backend:latest ./backend

docker run -p 8080:8080 jerney-frontend:latest
docker run -p 5000:5000 jerney-backend:latest
```

---

## Step 3 — Docker Compose (Local Full Stack)

The problem: frontend, backend, and database are three separate containers. They need to be on the same network and find each other by name.

Docker Compose solves this — one file defines all services, their shared network, and persistent volumes.

```yaml
# docker-compose.yml (at project root)
services:
  frontend:
    build:
      context: ./frontend
    ports:
      - "80:8080"
    depends_on:
      - backend
    networks:
      - jerney-network

  backend:
    build:
      context: ./backend
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db             # service name = DNS name inside Docker network
      - DB_PORT=5432
      - DB_USER=jerney_user
      - DB_PASSWORD=jerney_pass_2026
      - DB_NAME=jerney_db
      - PORT=5000
    depends_on:
      - db
    networks:
      - jerney-network

  db:
    image: postgres:16-alpine  # no custom Dockerfile needed — official image works
    environment:
      - POSTGRES_USER=jerney_user
      - POSTGRES_PASSWORD=jerney_pass_2026
      - POSTGRES_DB=jerney_db
    volumes:
      - postgres-data:/var/lib/postgresql/data   # persist data between restarts
    networks:
      - jerney-network

networks:
  jerney-network:
    driver: bridge

volumes:
  postgres-data:
```

```bash
docker compose up -d         # start all services in background
docker compose ps            # check all are healthy
docker compose logs -f       # follow all logs
docker compose down          # stop everything
docker compose down -v       # stop + wipe database volume
```

**Key insight:** Inside the Docker network, containers find each other by service name. The backend uses `DB_HOST=db` — `db` resolves to the database container's IP automatically. This is exactly the same pattern Kubernetes uses with Service DNS names (`jerney-db` resolves to the DB Service ClusterIP).

---

## Step 4 — Kubernetes Manifests

> 📖 Deep dive: [`k8s/notes.md`](./k8s/notes.md)

### Push images to a registry first

Kubernetes pulls images from a registry — it can't use local Docker images.

```bash
# Tag for GHCR
docker tag jerney-frontend:latest ghcr.io/<github-username>/jerney/jerney-frontend:latest
docker tag jerney-backend:latest ghcr.io/<github-username>/jerney/jerney-backend:latest

# Login and push
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin
docker push ghcr.io/<github-username>/jerney/jerney-frontend:latest
docker push ghcr.io/<github-username>/jerney/jerney-backend:latest
```

### Manifest structure

The rule: **create storage before the database, create database before the backend.**

```
Apply order:
1. namespace/      → isolation boundary — everything else goes inside this
2. secrets/        → DB credentials — referenced by DB and backend Pods
3. storage/        → StorageClass + PVC — DB needs persistent disk before it starts
4. database/       → PostgreSQL Deployment + ClusterIP Service
5. backend/        → Backend Deployment + ClusterIP Service
6. frontend/       → Frontend Deployment + NodePort Service
7. network/        → NetworkPolicy — lock down pod communication last
```

**Key manifest decisions:**

| Resource | Decision | Why |
|---|---|---|
| DB Deployment | `strategy: Recreate` | EBS is ReadWriteOnce — only 1 Pod can mount at a time. RollingUpdate would create 2 DB Pods fighting over the same disk. |
| DB Service | `ClusterIP` | Database never reachable from outside the cluster |
| Backend replicas | `replicas: 2` | One Pod dying doesn't take down the API |
| NetworkPolicy | DB only accepts from backend | Frontend cannot directly query the database — zero-trust |
| StorageClass | `reclaimPolicy: Retain` | Accidental `kubectl delete pvc` won't destroy the EBS volume and data |
| SecurityContext | `runAsNonRoot: true`, `drop: ALL` | Same least-privilege principle as the Dockerfiles |
| DB `automountServiceAccountToken: false` | DB doesn't need to call K8s API | Remove unnecessary capability |

### Apply and verify

```bash
# Apply everything
kubectl apply -f k8s/ -R

# Watch Pods start up in real time
kubectl get pods -n jerney -w

# Check all resources
kubectl get all -n jerney

# Access frontend (from your machine)
kubectl port-forward svc/jerney-frontend 8080:80 -n jerney
# Open: http://localhost:8080

# Access from EC2 (expose to your laptop's browser)
kubectl port-forward svc/jerney-frontend -n jerney 8172:80 --address 0.0.0.0
# Open: http://<EC2_PUBLIC_IP>:8172
```

### Security layers in the K8s setup

```
Layer 1 — AWS Network:    Worker nodes in private subnets (no public IPs on nodes)
Layer 2 — K8s Network:    NetworkPolicy — Frontend→Backend→DB only, nothing else allowed
Layer 3 — Pod Runtime:    Non-root user, read-only filesystem, all capabilities dropped
Layer 4 — Secrets:        K8s Secret objects + EKS KMS envelope encryption at rest
Layer 5 — API Server:     Private endpoint + public endpoint (restricted to known IPs in prod)
```

---

## Step 5 — CI/CD Pipeline

> 📖 Deep dive: [`.github/workflows/notes.md`](./.github/workflows/notes.md)

### Why automate?

Steps 1–4 were done manually. Every time code changes:
- Images need to be rebuilt and pushed with a new tag
- The K8s manifest needs updating to reference that new tag
- Security scans should run to catch new vulnerabilities introduced

The pipeline automates all of this and adds security gates at every stage.

### Pipeline stages

```
Every push / pull request:
│
├── Stage 1: Lint (ESLint)             ← code quality, runs in parallel
├── Stage 6: Dockerfile Lint           ← Hadolint, runs in parallel
│
▼ after lint passes:
Stage 2: Dependency Audit              ← npm audit, checks for CVEs in packages
│
▼ after audit passes:
Stage 3: Build & Push                  ← Docker Buildx → GHCR
                                         tagged: sha + branch + latest
                                         PRs: build only (no push)
│
▼ after build:
├── Stage 4: Image Scan (Trivy)        ← scans the exact image just pushed
└── Stage 5: IaC Scan (Checkov)        ← scans terraform/ and k8s/ for misconfigs
│
▼ push events only (not PRs):
Stage 7: Update K8s Manifest           ← GitOps: commits new image tag to repo
```

### The GitOps manifest update (Stage 7)

This is the most important stage to understand. Instead of running `kubectl apply` from the pipeline:

```bash
# Stage 7 does this:
IMAGE_TAG=$(git rev-parse --short HEAD)   # e.g. "abc1234"

# Find the exact line of the image field for each container
# and replace it with the new tag
sed -i "...s|.*|image: ghcr.io/.../jerney-backend:abc1234|" k8s/jerney.yaml

# Commit back to the repo
git commit -m "ci: update k8s images to abc1234 [skip ci]"
git push
```

Result: `k8s/jerney.yaml` in Git always reflects the latest built image. Git is the single source of truth for what should be running. `[skip ci]` prevents the commit from triggering another pipeline run.

### ArgoCD (completes the GitOps loop)

ArgoCD watches the Git repo. When Stage 7 commits the updated manifest, ArgoCD detects the change and automatically applies it to the cluster — zero manual `kubectl apply` needed.

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## Step 6 — Verify the Full End-to-End Flow

With everything in place:

```
1. Developer pushes code to GitHub
            │
            ▼
2. GitHub Actions runs all 7 stages
   - Code linted ✅
   - Dependencies audited ✅
   - Images built and pushed to GHCR ✅
   - Images scanned for CVEs ✅
   - Terraform + K8s manifests scanned ✅
   - k8s/jerney.yaml updated with new image SHA ✅
            │
            ▼
3. k8s/jerney.yaml in Git now has the new image tags
            │
            ▼
4. kubectl apply -f k8s/jerney.yaml
   (or ArgoCD does this automatically)
            │
            ▼
5. Kubernetes pulls new images from GHCR
   - Frontend + Backend: rolling update (zero downtime)
   - Database: unchanged (Recreate only if DB image changes)
            │
            ▼
6. New version live — fully audited, scanned, automated
```

---

## Cleanup — Avoid AWS Charges

```bash
# 1. Remove Kubernetes resources first
kubectl delete -f k8s/ -R

# 2. Destroy all AWS infrastructure
cd terraform/
terraform destroy
# type 'yes' when prompted
# Takes ~5-10 minutes
```

> ⚠️ EKS charges ~$0.10/hour for the control plane regardless of whether Pods are running. Always destroy when not in use.

---

*For deeper explanations of every concept — see the notes files linked at the top of each section.*