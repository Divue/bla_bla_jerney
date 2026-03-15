# 🏗️ Terraform Infrastructure Notes
### Jerney Blog Platform — AWS EKS with Terraform 

---

## Table of Contents

1. [What is Terraform?](#1-what-is-terraform)
2. [What Are We Building?](#2-what-are-we-building)
3. [Prerequisites & Setup](#3-prerequisites--setup)
4. [File Structure & What Each File Does](#4-file-structure--what-each-file-does)
5. [providers.tf — Deep Dive](#5-providerstf--deep-dive)
6. [variables.tf — Deep Dive](#6-variablestf--deep-dive)
7. [terraform.tfvars — Deep Dive](#7-terraformtfvars--deep-dive)
8. [main.tf — Deep Dive](#8-maintf--deep-dive)
9. [outputs.tf — Deep Dive](#9-outputstf--deep-dive)
10. [The Terraform Workflow](#10-the-terraform-workflow)
11. [Connecting kubectl to Your Cluster](#11-connecting-kubectl-to-your-cluster)
12. [Key Concepts Explained](#12-key-concepts-explained)
13. [Interview Questions & Answers](#13-interview-questions--answers)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)
15. [What to Improve for Production](#15-what-to-improve-for-production)

---

## 1. What is Terraform?

### 1.1 The Problem It Solves

Imagine you need to create an AWS setup: a VPC, subnets, an EKS cluster, IAM roles, security groups. You could click through the AWS Console — but then:

- How do you recreate it exactly in another region?
- How do you track what changed and who changed it?
- How do you destroy it cleanly without missing resources?
- How does your teammate create the same setup?

Doing this manually is slow, inconsistent, and error-prone. **This is the problem Terraform solves.**

> **One-Line Definition:** Terraform is an Infrastructure as Code (IaC) tool that lets you define cloud infrastructure in human-readable config files, then create, update, and destroy that infrastructure with commands.

### 1.2 How Terraform Thinks

Terraform uses a **declarative** model — you describe *what* you want, not *how* to create it:

```hcl
# You say: "I want a VPC with this CIDR"
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
# Terraform figures out the API calls to make it happen.
```

Compare this to **imperative** (step-by-step commands like AWS CLI):
```bash
# You say: "Run these commands in this order"
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 create-subnet ...
aws ec2 create-internet-gateway ...
# You manage every step and dependency yourself.
```

### 1.3 Core Terraform Concepts

| Concept | What it is |
|---|---|
| **Provider** | A plugin that knows how to talk to a cloud (AWS, GCP, Azure). Translates HCL into API calls. |
| **Resource** | A piece of infrastructure to create (VPC, EC2, EKS cluster, S3 bucket). |
| **Module** | A reusable package of resources. Like a function in programming — parameterise once, call many times. |
| **State** | A file (`terraform.tfstate`) that tracks what Terraform has created. The source of truth. |
| **Plan** | A preview of what Terraform *will* do before it does it. |
| **Apply** | Actually creates/updates/destroys infrastructure to match your config. |
| **Variables** | Inputs to make configs reusable across environments. |
| **Outputs** | Values exported after apply — like return values from a function. |
| **Data Source** | Read-only query of existing infrastructure (e.g. "what AZs exist in this region?"). |
| **Locals** | Computed values within a config file — like local variables. |

---

## 2. What Are We Building?

### 2.1 The Architecture

```
AWS Cloud (ap-south-1)
│
└── VPC (10.0.0.0/16)
    │
    ├── Public Subnets (3x, one per AZ)    ← Load balancers, NAT gateways
    │   └── NAT Gateway (single, cost-saving)
    │
    └── Private Subnets (3x, one per AZ)   ← EKS worker nodes
        └── EKS Cluster (Auto Mode)
            ├── node pool: general-purpose  ← App workloads
            └── node pool: system           ← CoreDNS, kube-proxy etc.
```

### 2.2 Key Technology Choices

| Choice | What it is | Why this choice |
|---|---|---|
| **EKS Auto Mode** | AWS manages node groups automatically — scaling, patching, replacing nodes | Zero node management overhead. AWS handles Karpenter, CoreDNS, kube-proxy. |
| **Single NAT Gateway** | One NAT Gateway shared across all AZs | Saves ~$100/month in dev. In prod: one per AZ for HA. |
| **Private Worker Nodes** | EKS nodes live in private subnets | Nodes not directly accessible from internet. Only accessible via EKS API. |
| **Public + Private Endpoint** | Cluster API accessible both publicly and privately | Public = you can `kubectl` from your laptop. Private = nodes talk to API server within VPC. |
| **Terraform Modules** | Using `terraform-aws-modules/vpc` and `terraform-aws-modules/eks` | Battle-tested community modules. Don't reinvent the wheel for standard setups. |

---

## 3. Prerequisites & Setup

### 3.1 Tools You Need

```bash
install from official site
```

### 3.2 AWS Authentication

Terraform needs AWS credentials to create resources. Configure them:

```bash
aws configure
# Prompts for:
# AWS Access Key ID:     <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region:        ap-south-1
# Default output format: json
```

This writes credentials to `~/.aws/credentials`. Terraform automatically picks them up.

> **IAM Permissions needed:** Your AWS user/role needs permissions to create VPCs, EKS clusters, IAM roles, and EC2 resources. For learning: `AdministratorAccess` is easiest. For production: use a scoped IAM policy.

---

## 4. File Structure & What Each File Does

```
terraform/
├── providers.tf      → Declares Terraform version, AWS provider, default tags
├── variables.tf      → Declares all input variables (name, type, default, description)
├── terraform.tfvars  → Actual values for variables (overrides defaults)
├── main.tf           → The real infrastructure: VPC + EKS modules
└── outputs.tf        → Values to print after apply (cluster name, endpoint, VPC ID)
```

**Why split into multiple files?**

Terraform merges all `.tf` files in a directory — splitting is purely for human organisation. Convention is:
- `providers.tf` — Terraform settings and provider config
- `variables.tf` — All variable declarations
- `main.tf` — All resources and modules
- `outputs.tf` — All output values
- `terraform.tfvars` — Environment-specific values (not committed to Git for secrets)

---

## 5. providers.tf — Deep Dive

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"   # Official AWS provider from HashiCorp registry
      version = "~> 6.0"          # Use any 6.x version (e.g. 6.1, 6.5) but not 7.0
    }
  }

  # Remote state backend (commented out — see section 15 for why this matters)
  # backend "s3" {
  #   bucket         = "jerney-terraform-state"
  #   key            = "eks/terraform.tfstate"
  #   region         = "us-east-1"
  #   dynamodb_table = "jerney-tf-lock"
  #   encrypt        = true
  # }
}

# Configure the AWS Provider
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "Jerney"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

### What each part does:

**`terraform {}` block:**
- Declares which providers are needed and their version constraints
- Optionally configures the backend (where state is stored)
- Terraform downloads providers from the registry during `terraform init`

**`required_providers`:**

| Version constraint | Meaning |
|---|---|
| `~> 6.0` | Allow 6.x but not 7.0 (pessimistic constraint) |
| `>= 6.0` | Allow 6.0 or higher (any future version) |
| `= 6.2.1` | Exactly this version only |
| `>= 6.0, < 7.0` | Explicit range |

**`default_tags`:**
All AWS resources created by this provider automatically get these tags. Crucial for:
- Cost tracking (filter bills by `Project=Jerney`)
- Identifying which resources Terraform manages
- Compliance requirements

### The commented-out S3 backend:

By default, Terraform stores state in a local file `terraform.tfstate`. The S3 backend stores it in an S3 bucket instead. Why does this matter?

| Local state | Remote state (S3) |
|---|---|
| Only on your machine | Shared — team can access |
| No locking — two people applying = corruption | DynamoDB locking prevents simultaneous applies |
| Lost if your machine dies | Durable, versioned |
| Fine for solo learning | Required for teams |

---

## 6. variables.tf — Deep Dive

```hcl
variable "aws_region" {
  description = "AWS region to deploy the EKS cluster"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
  default     = "jerney-eks"
}

variable "cluster_version" {
  description = "Kubernetes version for EKS"
  type        = string
  default     = "1.32"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}
```

### Variable anatomy:

```hcl
variable "name" {
  description = "Human-readable explanation of what this is"  # Shows in docs + plan output
  type        = string                                         # Type constraint
  default     = "some-value"                                   # Optional: used if not provided
  # validation { ... }                                         # Optional: custom rules
  # sensitive = true                                           # Hides value in plan output
}
```

### Variable types:

| Type | Example |
|---|---|
| `string` | `"us-east-1"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b", "c"]` |
| `map(string)` | `{ key = "value" }` |
| `object({...})` | Complex structured type |

### Variable precedence (highest wins):

```
1. -var flag:           terraform apply -var="cluster_name=my-cluster"
2. .tfvars file:        terraform.tfvars  ← Jerney uses this
3. Environment variable: TF_VAR_cluster_name=my-cluster
4. default in variable block
```

---

## 7. terraform.tfvars — Deep Dive

```hcl
aws_region      = "ap-south-1"   # Mumbai region (overrides default us-east-1)
environment     = "dev"
cluster_name    = "jerney-eks"
cluster_version = "1.32"
vpc_cidr        = "10.0.0.0/16"
```

This file **overrides the defaults** set in `variables.tf`. It's where you put environment-specific values.

> **Git tip:** Commit `variables.tf` (declarations with safe defaults) to Git. Be careful with `terraform.tfvars` — if it contains secrets (API keys, passwords), add it to `.gitignore`. For Jerney it's safe to commit since it only has region/name values.

### Why separate variables.tf from terraform.tfvars?

`variables.tf` = the **contract** (what inputs exist, what type, what they mean)
`terraform.tfvars` = the **values** (the actual data for this specific environment)

This means the same `variables.tf` can be reused with different `.tfvars` files:
```
terraform.tfvars          → dev environment (ap-south-1)
terraform.prod.tfvars     → prod environment (us-east-1, larger cluster)
```
```bash
terraform apply -var-file="terraform.prod.tfvars"
```

---

## 8. main.tf — Deep Dive

### 8.1 Data Source — Available AZs

```hcl
data "aws_availability_zones" "available" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}
```

A **data source** reads existing information from AWS — it doesn't create anything. This queries which Availability Zones exist in the configured region and are available by default (filters out opt-in zones like Local Zones).

> **Why not hardcode AZs?** `["ap-south-1a", "ap-south-1b", "ap-south-1c"]` would break if you change regions. The data source makes it dynamic.

### 8.2 Locals — Computing AZ List

```hcl
locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}
```

**Locals** are computed values — like local variables. `slice(list, start, end)` takes the first 3 AZs from the list returned by the data source. This gives us exactly 3 AZs regardless of region.

Usage: `local.azs` — referenced in the VPC module below.

---

### 8.3 VPC Module

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # Community module from Terraform Registry
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"    # "jerney-eks-vpc"
  cidr = var.vpc_cidr                 # "10.0.0.0/16"

  azs             = local.azs         # ["ap-south-1a", "ap-south-1b", "ap-south-1c"]

  # Private subnets: 10.0.0.0/20, 10.0.16.0/20, 10.0.32.0/20
  private_subnets = [for k, v in local.azs : cidrsubnet(var.vpc_cidr, 4, k)]

  # Public subnets: 10.0.48.0/24, 10.0.49.0/24, 10.0.50.0/24
  public_subnets  = [for k, v in local.azs : cidrsubnet(var.vpc_cidr, 8, k + 48)]

  enable_nat_gateway = true
  single_nat_gateway = true   # One NAT GW for all private subnets (dev cost-saving)

  # Tags required for EKS Auto Mode to discover which subnets to use for load balancers
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1           # EKS puts public load balancers here
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1  # EKS puts internal load balancers here
  }
}
```

#### Understanding `cidrsubnet`:

`cidrsubnet(base_cidr, newbits, netnum)` — calculates subnet CIDRs automatically.

```
cidrsubnet("10.0.0.0/16", 4, 0) = "10.0.0.0/20"    ← private subnet 1
cidrsubnet("10.0.0.0/16", 4, 1) = "10.0.16.0/20"   ← private subnet 2
cidrsubnet("10.0.0.0/16", 4, 2) = "10.0.32.0/20"   ← private subnet 3

cidrsubnet("10.0.0.0/16", 8, 48) = "10.0.48.0/24"  ← public subnet 1
cidrsubnet("10.0.0.0/16", 8, 49) = "10.0.49.0/24"  ← public subnet 2
cidrsubnet("10.0.0.0/16", 8, 50) = "10.0.50.0/24"  ← public subnet 3
```

`newbits` = how many extra bits to add to the prefix. `/16 + 4 = /20`. `/16 + 8 = /24`.

#### What the VPC module creates automatically:

- VPC with DNS hostnames and DNS resolution enabled
- 3 private subnets (for EKS worker nodes)
- 3 public subnets (for load balancers, NAT gateway)
- 1 Internet Gateway (public internet access)
- 1 NAT Gateway in a public subnet (allows private nodes to reach internet for image pulls)
- Route tables and associations for both subnet types

#### Why private subnets for worker nodes?

Worker nodes don't need to be directly reachable from the internet. They pull container images outbound through the NAT Gateway. The EKS control plane communicates with nodes through the private endpoint. Keeping nodes private significantly reduces the attack surface.

---

### 8.4 EKS Module

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.31"

  cluster_name    = var.cluster_name     # "jerney-eks"
  cluster_version = var.cluster_version  # "1.32"

  # EKS Auto Mode — AWS manages node groups, scaling, and system components
  cluster_compute_config = {
    enabled    = true
    node_pools = ["general-purpose", "system"]
  }

  # Networking — attach cluster to the VPC and private subnets
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # API server accessibility
  cluster_endpoint_public_access  = true   # kubectl from your laptop works
  cluster_endpoint_private_access = true   # nodes talk to API server inside VPC

  # Authentication — required for Auto Mode
  authentication_mode = "API"

  # Encrypt all Kubernetes secrets at rest using AWS KMS
  cluster_encryption_config = {
    resources = ["secrets"]
  }

  # Enable all control plane log types → CloudWatch Logs
  cluster_enabled_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  # Grant your current IAM identity admin access to the cluster
  enable_cluster_creator_admin_permissions = true
}
```

#### EKS Auto Mode explained:

Traditional EKS requires you to manage node groups — choosing instance types, configuring auto-scaling, patching AMIs. **EKS Auto Mode** delegates all of this to AWS:

| You manage (without Auto Mode) | AWS manages (with Auto Mode) |
|---|---|
| Node group instance types | Optimal instance selection |
| Cluster Autoscaler / Karpenter setup | Node provisioning and scaling |
| Kube-proxy, CoreDNS updates | System component lifecycle |
| AMI patching for worker nodes | Node OS patching |
| Node replacement on failures | Self-healing nodes |

**Node pools:**
- `general-purpose` — for your application workloads (frontend, backend, DB)
- `system` — for system components (CoreDNS, metrics-server, etc.)

#### Authentication Mode: API

EKS has two auth modes:
- `CONFIG_MAP` — the old way, uses the `aws-auth` ConfigMap in `kube-system`
- `API` — the new way (required for Auto Mode), uses EKS Access Entries API

`enable_cluster_creator_admin_permissions = true` automatically gives the IAM identity running Terraform full admin access to the cluster. Without this, even the creator can't `kubectl` into the cluster after creating it.

#### Cluster logging:

| Log type | What it captures |
|---|---|
| `api` | All requests to the K8s API server |
| `audit` | Who did what — critical for security |
| `authenticator` | IAM authentication attempts |
| `controllerManager` | Deployment controller, ReplicaSet controller activity |
| `scheduler` | Pod scheduling decisions |

These go to **CloudWatch Logs** → `/aws/eks/jerney-eks/cluster`. Essential for debugging and compliance.

---

## 9. outputs.tf — Deep Dive

```hcl
output "cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS cluster API endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_certificate_authority" {
  description = "EKS cluster CA certificate (base64)"
  value       = module.eks.cluster_certificate_authority_data
  sensitive   = true    # Won't be printed in terminal output
}

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "region" {
  description = "AWS region"
  value       = var.aws_region
}
```

Outputs are printed to the terminal after `terraform apply` completes. They're also accessible via `terraform output <name>`.

**`sensitive = true`** — the value is still stored in state but won't be shown in terminal output or plan. Use for anything you don't want appearing in CI/CD logs.

```bash
# Read outputs after apply
terraform output
terraform output cluster_name
terraform output -json    # Machine-readable JSON
```

---

## 10. The Terraform Workflow

### Step-by-step to deploy Jerney's infrastructure:

#### Step 1 — `terraform init`

```bash
cd terraform/
terraform init
```

What this does:
- Downloads the AWS provider (`hashicorp/aws ~> 6.0`) into `.terraform/`
- Downloads the VPC and EKS modules from the Terraform Registry
- Initialises the backend (local or S3)
- Creates `.terraform.lock.hcl` (locks exact provider versions — commit this to Git)

> Run this once when you start, and again whenever you add new providers or modules.

#### Step 2 — `terraform plan`

```bash
terraform plan
```

What this does:
- Reads your `.tf` files
- Reads current state (`terraform.tfstate`)
- Calls AWS APIs to check what actually exists
- Shows you exactly what will be **created**, **changed**, or **destroyed**

**Always read the plan output before applying.** Look for:
- Resources being destroyed unexpectedly
- The count of changes (100+ resources for a full EKS cluster is normal)
- Any errors in red

```
Plan: 47 to add, 0 to change, 0 to destroy.
```

#### Step 3 — `terraform apply`

```bash
terraform apply
```

- Shows the plan again
- Asks: `Do you want to perform these actions? yes/no`
- Type `yes` to proceed
- Creates all resources in the right order (handles dependencies automatically)
- Takes **10–15 minutes** for a full EKS cluster

```bash
# Skip the confirmation prompt (for CI/CD):
terraform apply -auto-approve
```

#### Step 4 — `terraform destroy` (cleanup)

```bash
terraform destroy
```

Destroys ALL resources Terraform created. **EKS + VPC = ~$0.10/hour when running.** Always destroy when not in use.

---

## 11. Connecting kubectl to Your Cluster

After `terraform apply` completes, your EKS cluster exists in AWS but your local `kubectl` doesn't know about it yet. Run:

```bash
# Update your local kubeconfig to point to the new cluster
aws eks update-kubeconfig --name jerney-eks --region ap-south-1

# Verify connection
kubectl get nodes

# Should output something like:
# NAME                                          STATUS   ROLES    AGE
# ip-10-0-12-34.ap-south-1.compute.internal   Ready    <none>   5m
```

**What `update-kubeconfig` does:**
- Calls the EKS API to get the cluster endpoint and CA certificate
- Adds a new context to `~/.kube/config`
- Sets it as the current context
- Configures `kubectl` to use `aws eks get-token` for authentication

After this, all `kubectl` commands (from your Kubernetes notes) work against this cluster.

### Quick smoke test:

```bash
# Deploy a test nginx pod
kubectl create deployment nginx --image nginx:latest

# Check it's running
kubectl get pods

# Clean up
kubectl delete deployment nginx
```

---

## 12. Key Concepts Explained

### 12.1 What is a VPC?

A **Virtual Private Cloud** is your own private, isolated network inside AWS. Think of it as renting a section of AWS's network that only you control.

```
Without VPC:  All AWS resources share the same network → anyone can reach anyone
With VPC:     Your resources are in your own network → you control all traffic
```

Key VPC components in Jerney:

| Component | Role |
|---|---|
| **CIDR Block** (`10.0.0.0/16`) | The IP address range for your entire VPC. Allows 65,536 IP addresses. |
| **Public Subnet** | Has a route to the Internet Gateway. Resources here can reach the internet directly. Used for NAT Gateway and (future) load balancers. |
| **Private Subnet** | No direct internet route. Resources here go through NAT Gateway for outbound. Worker nodes live here. |
| **Internet Gateway (IGW)** | Allows public subnets to communicate with the internet. |
| **NAT Gateway** | Allows private subnet resources to make OUTBOUND internet requests (to pull Docker images, reach AWS APIs) without being reachable FROM the internet. |
| **Route Table** | Rules for where network traffic goes. Public subnets: `0.0.0.0/0 → IGW`. Private subnets: `0.0.0.0/0 → NAT GW`. |

### 12.2 What is EKS?

**Elastic Kubernetes Service** is AWS's managed Kubernetes offering. AWS runs and maintains the Control Plane (API server, etcd, scheduler, controllers) for you. You just pay per hour the cluster exists.

Without EKS: you install and manage Kubernetes yourself on EC2 instances. This is complex.
With EKS: AWS maintains the control plane. You focus on your workloads.

### 12.3 Terraform Modules

A module is a reusable package of Terraform resources. The Jerney project uses two public modules:

```
terraform-aws-modules/vpc/aws   → Creates VPC, subnets, gateways, route tables
terraform-aws-modules/eks/aws   → Creates EKS cluster, IAM roles, security groups, KMS key
```

These modules are on the [Terraform Registry](https://registry.terraform.io). They're maintained by the community and handle all the complex wiring (dozens of IAM policies, security groups, etc.) that you'd otherwise have to write yourself.

Think of it like using an npm package — you declare what you want, the module handles the implementation.

### 12.4 Terraform State

The state file (`terraform.tfstate`) is how Terraform tracks what it has created. It maps your config to real AWS resource IDs.

```
Config:       module "vpc" { name = "jerney-eks-vpc" ... }
State:        { "vpc_id": "vpc-0abc1234...", "subnet_ids": ["subnet-0xyz..."] }
```

**Why state is critical:**
- If you delete the state file, Terraform loses track of what it created — it will try to create everything again, causing conflicts
- Never edit the state file manually
- For teams, store state remotely (S3 + DynamoDB lock) so everyone shares the same state

---

## 13. Core Concepts

**Q: What is Terraform and how does it differ from AWS CloudFormation?**
> Terraform is a cloud-agnostic IaC tool by HashiCorp that uses HCL. CloudFormation is AWS-native and uses JSON/YAML. Terraform works across AWS, GCP, Azure, and hundreds of other providers with the same tool and workflow. CloudFormation is tightly integrated with AWS but locked into it. Terraform has a richer module ecosystem and better state management.

**Q: What is the difference between `terraform plan` and `terraform apply`?**
> `terraform plan` is a dry run — it shows what Terraform *would* do without making any changes. It reads your config, compares it to state and actual infrastructure, and outputs a diff. `terraform apply` actually executes those changes. Best practice is always run `plan` first, review it carefully, then `apply`. In CI/CD, the plan output can be reviewed in a PR before merging triggers apply.

**Q: What is Terraform state? What happens if it's lost?**
> State is a JSON file that maps your Terraform resources to real cloud resource IDs. It's how Terraform knows "this `module.vpc` in my config corresponds to `vpc-0abc123` in AWS." If lost, Terraform has no idea what it created — running `apply` again would try to create duplicates, causing errors. You can recover with `terraform import` to re-import existing resources into state, but it's tedious. This is why remote state (S3) with versioning enabled is critical for production.

**Q: What is a Terraform module? Why use community modules?**
> A module is a reusable package of resources that can be called with parameters. Like a function. Community modules like `terraform-aws-modules/eks` bundle dozens of resources (IAM roles, security groups, KMS keys, add-ons) that are needed for a production EKS cluster — properly wired together and battle-tested. Writing this from scratch would take days and be error-prone. Modules let you stand up complex infrastructure with a few lines of code.

**Q: What is the difference between public and private subnets?**
> A public subnet has a route to an Internet Gateway — resources in it can be reached from the internet (if they have a public IP). A private subnet routes outbound internet traffic through a NAT Gateway — resources can reach the internet but are NOT reachable from it. EKS worker nodes go in private subnets for security — attackers can't directly reach them. Load balancers go in public subnets to receive external traffic.

**Q: What is EKS Auto Mode?**
> EKS Auto Mode is a feature where AWS fully manages the worker node lifecycle — provisioning, scaling, patching, and replacing nodes. Internally it uses Karpenter. Without Auto Mode, you configure managed node groups or self-managed nodes and are responsible for their maintenance. With Auto Mode, you just deploy your apps and AWS handles the underlying compute. It uses node pools (`general-purpose`, `system`) that you reference via Pod labels.

**Q: What is a NAT Gateway and why is it in a public subnet?**
> A NAT Gateway allows resources in private subnets to make outbound internet connections (like pulling Docker images from Docker Hub or calling AWS APIs) while remaining unreachable from the internet. It must be in a public subnet because it needs a public IP to communicate with the internet. Private subnet resources route `0.0.0.0/0` to the NAT Gateway, which then forwards traffic to the internet using its public IP.

**Q: Why use `single_nat_gateway = true` in dev but not production?**
> A NAT Gateway costs ~$0.045/hour + data transfer. One per AZ means 3x NAT Gateways for high availability — if one AZ goes down, other AZs still have outbound internet. In dev, we use a single NAT Gateway to save ~$70/month since HA isn't critical. In production, using `single_nat_gateway = false` creates one per AZ, so a single AZ failure doesn't kill all outbound traffic from private subnets.

**Q: What does `enable_cluster_creator_admin_permissions = true` do?**
> When you create an EKS cluster with the new `API` authentication mode, access isn't automatically granted to anyone — not even the creator. This option tells the EKS module to create an access entry granting the IAM identity that ran Terraform (your user/role) full admin access to the cluster. Without it, you'd create the cluster successfully but then be unable to `kubectl` into it.

---

## 14. Quick Reference Cheat Sheet

### Terraform Commands

| Command | What it does |
|---|---|
| `terraform init` | Download providers and modules. Run once at start. |
| `terraform plan` | Preview changes. Always run before apply. |
| `terraform apply` | Create/update infrastructure. |
| `terraform destroy` | Destroy all managed infrastructure. |
| `terraform output` | Show output values after apply. |
| `terraform state list` | List all resources tracked in state. |
| `terraform state show <resource>` | Show details of a specific resource in state. |
| `terraform fmt` | Auto-format `.tf` files. |
| `terraform validate` | Check syntax and config validity. |
| `terraform refresh` | Sync state with actual cloud state (rarely needed). |
| `terraform import` | Import existing resource into state. |

### File Reference

| File | Purpose | Commit to Git? |
|---|---|---|
| `providers.tf` | Provider + version config | ✅ Yes |
| `variables.tf` | Variable declarations | ✅ Yes |
| `main.tf` | Resources and modules | ✅ Yes |
| `outputs.tf` | Output values | ✅ Yes |
| `terraform.tfvars` | Variable values | ✅ Yes (if no secrets) |
| `.terraform/` | Downloaded providers/modules | ❌ No (`.gitignore`) |
| `terraform.tfstate` | Infrastructure state | ❌ No (use remote backend) |
| `terraform.tfstate.backup` | Previous state backup | ❌ No |
| `.terraform.lock.hcl` | Provider version lock | ✅ Yes |

### AWS Resources Created

| Resource | Count | Purpose |
|---|---|---|
| VPC | 1 | Network isolation |
| Subnets | 6 (3 public + 3 private) | AZ distribution |
| Internet Gateway | 1 | Public internet access |
| NAT Gateway | 1 | Private outbound internet |
| EIP (Elastic IP) | 1 | Static IP for NAT Gateway |
| Route Tables | 2 | Traffic routing rules |
| EKS Cluster | 1 | Managed K8s control plane |
| KMS Key | 1 | Secret encryption at rest |
| IAM Roles | several | EKS and node permissions |
| CloudWatch Log Group | 1 | Control plane logs |

---

## 15. What to Improve for Production

### Remote State Backend
```hcl
# Uncomment in providers.tf and create the S3 bucket + DynamoDB table first
backend "s3" {
  bucket         = "jerney-terraform-state"
  key            = "eks/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "jerney-tf-lock"    # Prevents concurrent applies
  encrypt        = true                # Encrypt state file at rest
}
```
Without this, state is local — lost if your machine dies, and breaks team collaboration.

### HA NAT Gateway
```hcl
single_nat_gateway = false   # One NAT GW per AZ instead of one total
```
One NAT Gateway is a single point of failure. If that AZ goes down, all private nodes lose internet access.

### Restrict Public Endpoint Access
```hcl
cluster_endpoint_public_access       = true
cluster_endpoint_public_access_cidrs = ["YOUR_OFFICE_IP/32"]  # Only your IP
```
Limiting which IPs can reach the Kubernetes API reduces the attack surface significantly.

### Separate State per Environment
```
terraform/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```
Dev and prod should have completely separate state files so a dev destroy can't affect prod.

### Pin Module Versions Exactly
```hcl
version = "= 5.8.1"   # Exact version, not ~> 5.0
```
Prevents surprise breaking changes when module maintainers release new versions.

---

*End of Notes — Infrastructure as Code means your infrastructure lives in Git. Ship it! 🚀*