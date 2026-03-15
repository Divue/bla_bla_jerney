# ☸️ Jerney — Cloud Native DevSecOps

> A 3-tier blog platform (React + Node.js + PostgreSQL) taken from bare-metal EC2 to a **fully automated, security-scanned, Kubernetes-native deployment on AWS EKS** — built the way a DevOps engineer would approach a real production system.

<br/>

![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS_Auto_Mode-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Containers-Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![Security](https://img.shields.io/badge/Security-DevSecOps-DC2626?style=flat-square&logo=shieldsdotio&logoColor=white)
![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?style=flat-square&logo=amazonaws&logoColor=white)


---

## 📐 Architecture

```
                    ┌───────────────────────────────────────────────┐
                    │                    GitHub                     │
                    │                                               │
                    │   Developer ──push──▶ GitHub Actions          │
                    │                            │                  │
                    │              ┌─────────────┴──────────────┐   │
                    │              │  7-Stage DevSecOps Pipeline │   │
                    │              │  Lint → SCA → Build →       │   │
                    │              │  Scan → IaC → Dockerfile    │   │
                    │              │  → Manifest Update          │   │
                    │              └──────┬─────────────┬────────┘   │
                    │                     │             │            │
                    │              push image     commit new tag     │
                    │                     ▼             ▼            │
                    │              ┌──────────┐  ┌────────────────┐  │
                    │              │  GHCR    │  │ k8s/jerney.yaml│  │
                    │              │(Registry)│  │  (GitOps src)  │  │
                    │              └──────────┘  └────────────────┘  │
                    └───────────────────────────────────────────────┘
                           │ pull image              │ kubectl apply
                           ▼                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  AWS  (ap-south-1)                                                       │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  VPC  10.0.0.0/16              (Terraform managed)              │    │
│  │                                                                 │    │
│  │  Public Subnets  ×3   ──NAT──▶  Private Subnets ×3             │    │
│  │  (Load Balancers)               (EKS Worker Nodes)             │    │
│  │                                                                 │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │  EKS Cluster  "jerney-eks"  (Auto Mode / Karpenter)       │  │    │
│  │  │                                                           │  │    │
│  │  │   ┌─────────── Namespace: jerney ─────────────────────┐  │  │    │
│  │  │   │                                                    │  │  │    │
│  │  │   │  ┌──────────┐  NetworkPolicy  ┌──────────┐        │  │  │    │
│  │  │   │  │ Frontend │────────────────▶│ Backend  │        │  │  │    │
│  │  │   │  │  Nginx   │                 │ Node.js  │        │  │  │    │
│  │  │   │  │  :8080   │                 │  :5000   │        │  │  │    │
│  │  │   │  └──────────┘                 └────┬─────┘        │  │  │    │
│  │  │   │                    NetworkPolicy   │              │  │  │    │
│  │  │   │                                    ▼              │  │  │    │
│  │  │   │                             ┌──────────┐          │  │  │    │
│  │  │   │                             │PostgreSQL│          │  │  │    │
│  │  │   │                             │  :5432   │          │  │  │    │
│  │  │   │                             │ EBS PVC  │          │  │  │    │
│  │  │   │                             └──────────┘          │  │  │    │
│  │  │   └────────────────────────────────────────────────────┘  │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 🗂️ Repository Structure

```
jerney-cloud-native-devsecops/              (devops branch)
│
├── .github/workflows/
│   ├── ci-cd.yml                    # 7-stage DevSecOps pipeline definition
│   └── notes.md                     # CI/CD pipeline concepts & every stage explained
│
├── frontend/
│   ├── src/                         # React components & pages
│   ├── nginx.conf                   # Custom Nginx config (non-root, port 8080)
│   ├── Dockerfile                   # Multi-stage: Node build → Nginx serve
│   ├── dockerfile_notes.md          # Every line explained
│   └── package.json
│
├── backend/
│   ├── src/                         # Express routes, DB connection, health check
│   ├── Dockerfile                   # Multi-stage: prod deps only + dumb-init
│   ├── dockerfile_notes.md          # Every line explained
│   └── package.json
│
├── k8s/
│   ├── namespace/                   # jerney namespace
│   ├── secrets/                     # PostgreSQL credentials (base64)
│   ├── storage/                     # StorageClass (EBS gp3) + PVC
│   ├── database/                    # PostgreSQL Deployment + ClusterIP Service
│   ├── backend/                     # Backend Deployment + ClusterIP Service
│   ├── frontend/                    # Frontend Deployment + NodePort Service
│   ├── network/                     # NetworkPolicy (zero-trust between pods)
│   ├── jerney.yaml                  # Single combined manifest (all of the above)
│   └── notes.md                     # Kubernetes from scratch — every concept explained
│
├── terraform/
│   ├── providers.tf                 # AWS provider, version constraints, S3 backend config
│   ├── variables.tf                 # Input variable declarations
│   ├── terraform.tfvars             # Environment values (ap-south-1, dev)
│   ├── main.tf                      # VPC module + EKS Auto Mode module
│   ├── outputs.tf                   # Cluster name, endpoint, VPC ID
│   └── notes.md                     # Terraform + AWS concepts explained
│
├── deploy/                          # (from main branch) EC2 bare-metal deployment
│   ├── setup.sh                     # One-click EC2 setup script
│   └── jerney-nginx.conf            # Nginx reverse proxy config
│
├── docker-compose.yml               # Local development stack (all 3 services + network)
├── steps.md                         # Full implementation walkthrough — start to finish
└── README.md                        # You are here
```

---

## 🔒 Security at Every Layer

Security is not a final step — it is woven into every layer of the stack.

| Layer | Threat | Control |
|---|---|---|
| **Code** | Bugs, bad practices | ESLint on every push |
| **Dependencies** | Known CVEs in npm packages | `npm audit` — SCA stage |
| **Dockerfile** | Insecure instructions, bad practices | Hadolint lint |
| **Container Image** | Vulnerable OS + library packages | Trivy image scan |
| **IaC** | Terraform/K8s misconfigurations | Checkov scan |
| **Container Runtime** | Root escalation, filesystem tampering | `runAsNonRoot`, `readOnlyRootFilesystem`, `drop: ALL` capabilities |
| **Pod Communication** | Lateral movement between services | NetworkPolicy — Frontend→Backend→DB only, nothing else |
| **Secrets** | Plaintext credentials | K8s Secrets + EKS envelope encryption via KMS |
| **AWS Network** | Direct node access from internet | Worker nodes in private subnets — no public IPs |

---

## 🚀 CI/CD Pipeline

**7 stages** run automatically on every push and pull request.

```
Push to any branch
        │
        ├──▶  Stage 1 — Lint (ESLint)              ←─ parallel
        ├──▶  Stage 6 — Dockerfile Lint (Hadolint)  ←─ parallel
        │
        ▼  needs: lint
        Stage 2 — Dependency Audit (npm audit / SCA)
        │
        ▼  needs: sca
        Stage 3 — Build & Push (Docker Buildx → GHCR)
        │
        ▼  needs: build
        ├──▶  Stage 4 — Image Scan (Trivy)          ←─ parallel
        ├──▶  Stage 5 — IaC Scan (Checkov)          ←─ parallel
        │
        ▼  push events only (not PRs)
        Stage 7 — Update K8s Manifest (GitOps commit)
```

**Live run history:** [Actions tab →](../../actions)

---

## 🛠️ Tech Stack

| Category | Technology |
|---|---|
| **Frontend** | React 18, Vite, Nginx Alpine |
| **Backend** | Node.js 20, Express |
| **Database** | PostgreSQL 16 |
| **Containers** | Docker, Docker Buildx, multi-stage builds |
| **Orchestration** | Kubernetes, AWS EKS Auto Mode |
| **IaC** | Terraform, `terraform-aws-modules/vpc`, `terraform-aws-modules/eks` |
| **CI/CD** | GitHub Actions |
| **Registry** | GitHub Container Registry (GHCR) |
| **Security Scanning** | Trivy, Checkov, Hadolint, ESLint, npm audit |
| **Cloud** | AWS — EKS, VPC, EBS, KMS, CloudWatch Logs |

---

## ⚡ Quick Start

### Run locally with Docker Compose

```bash
git clone <repo-url>
cd jerney-cloud-native-devsecops
git checkout devops

docker compose up -d
# Frontend → http://localhost:80
# Backend  → http://localhost:5000
```

### Deploy to AWS EKS

```bash
# 1. Provision infrastructure (~10-15 min)
cd terraform/
terraform init
terraform plan
terraform apply

# 2. Connect kubectl to the new cluster
aws eks update-kubeconfig --name jerney-eks --region ap-south-1

# 3. Verify cluster is ready
kubectl get nodes

# 4. Deploy the application
kubectl apply -f k8s/ -R

# 5. Access the frontend
kubectl port-forward svc/jerney-frontend 8080:80 -n jerney
# Open: http://localhost:8080
```

### Destroy when done (avoid AWS charges)

```bash
kubectl delete -f k8s/ -R
cd terraform/ && terraform destroy
```

---

## 📖 Notes & Documentation

Each component has dedicated notes written to explain every concept from scratch — useful for interviews and for anyone reimplementing the project.

| File | What it covers |
|---|---|
| [`steps.md`](./steps.md) | Full implementation walkthrough — every decision in order |
| [`k8s/notes.md`](./k8s/notes.md) | Kubernetes — Pods, Deployments, Services, Storage, NetworkPolicy, Security |
| [`terraform/notes.md`](./terraform/notes.md) | Terraform + AWS — VPC, subnets, EKS Auto Mode, modules, state |
| [`frontend/dockerfile_notes.md`](./frontend/dockerfile_notes.md) | Frontend Dockerfile — multi-stage build, layer caching, non-root Nginx |
| [`backend/dockerfile_notes.md`](./backend/dockerfile_notes.md) | Backend Dockerfile — dumb-init, custom user, prod-only dependencies |
| [`.github/workflows/notes.md`](./.github/workflows/notes.md) | CI/CD pipeline — all 7 stages, GitOps pattern, security tools |

---

*Application originally built by the development team on the `main` branch. This `devops` branch documents the full DevSecOps transformation applied on top of it.*