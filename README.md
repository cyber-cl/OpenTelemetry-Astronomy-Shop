# 🔭 OpenTelemetry Astronomy Shop — End-to-End DevOps Pipeline

> A production-grade DevOps implementation on the [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo) microservices application — covering **containerization**, **infrastructure-as-code**, and **fully automated GitOps CI/CD** on AWS EKS.

---

## 📌 Project Overview

The **OpenTelemetry Astronomy Shop** is a cloud-native e-commerce application composed of **15+ polyglot microservices** (Go, Python, Java, .NET, Node.js, Rust, C++, and more). I used this as the foundation to build and demonstrate a complete DevOps lifecycle:

| Phase | What I Did |
|---|---|
| **Containerization** | Wrote production-ready, multi-stage Dockerfiles for every microservice |
| **Infrastructure as Code** | Provisioned the entire AWS infrastructure (VPC, subnets, NAT, EKS) using Terraform with reusable modules and remote state |
| **CI Pipeline** | Built GitHub Actions workflows that build, lint, test, and push Docker images to DockerHub |
| **CD Pipeline** | Deployed microservices to EKS via ArgoCD using a GitOps pattern — any manifest change auto-syncs to the cluster |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DEVELOPER WORKFLOW                                │
│                                                                            │
│   Code Push ──► GitHub Actions CI ──► Build & Test ──► Docker Image Push   │
│                                              │                              │
│                                              ▼                              │
│                                   Update K8s Manifest                       │
│                                    (image tag via sed)                      │
│                                              │                              │
│                                              ▼                              │
│                                   Git Commit & Push                         │
│                                   (to same repo)                            │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            ARGOCD (GitOps)                                  │
│                                                                             │
│   Watches repo ──► Detects manifest change ──► Syncs to EKS cluster         │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          AWS EKS CLUSTER                                    │
│                     (Provisioned via Terraform)                              │
│                                                                             │
│   ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────┐        │
│   │  Product Catalog  │  │  Recommendation    │  │    Ad Service    │        │
│   │  Service (Go)     │  │  Service (Python)  │  │    (Java)        │        │
│   └──────────────────┘  └────────────────────┘  └──────────────────┘        │
│                                                                             │
│   ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────┐        │
│   │  Cart Service     │  │  Checkout Service  │  │  Currency Svc    │        │
│   └──────────────────┘  └────────────────────┘  └──────────────────┘        │
│                                                                             │
│   ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────┐        │
│   │  Payment Service  │  │  Shipping Service  │  │  Email Service   │        │
│   └──────────────────┘  └────────────────────┘  └──────────────────┘        │
│                         ... + more services                                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    AWS INFRASTRUCTURE (Terraform)                            │
│                                                                             │
│   VPC (10.0.0.0/16)                                                         │
│   ├── Public Subnets  (3 AZs) ── Internet Gateway ── NAT Gateways          │
│   └── Private Subnets (3 AZs) ── EKS Worker Nodes (t3.medium)              │
│                                                                             │
│   State: S3 Backend (opentelemetry-eks-state-bucket)                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 🐳 Phase 1 — Containerization (Docker)

Built **multi-stage, production-optimized Docker images** for the microservices. Each image follows best practices to minimize size and attack surface.

### Key Services Containerized

| Service | Language | Base Image | Highlights |
|---|---|---|---|
| **Product Catalog** | Go | `golang:1.22-alpine` → `alpine` | Multi-stage build, Go build cache mounts, ~15 MB final image |
| **Recommendation** | Python | `python:3.12-slim-bookworm` | OpenTelemetry auto-instrumentation bootstrapped at build time |
| **Ad Service** | Java | `eclipse-temurin:21-jdk` → `eclipse-temurin:21-jre` | Multi-stage build with Gradle, JDK for build / JRE for runtime |

### Dockerfile Best Practices Applied

- ✅ **Multi-stage builds** — separate build and runtime stages to exclude compilers and source
- ✅ **Build cache mounts** — `--mount=type=cache` for Go module and build caches
- ✅ **Minimal base images** — Alpine and slim variants to reduce CVE surface area
- ✅ **Dependency layer caching** — `COPY go.mod` / `requirements.txt` before source for optimal Docker layer caching
- ✅ **Non-root users** and minimal runtime footprint

### Docker Compose

A full `docker-compose.yml` is available to spin up the **entire 15+ service application** locally for development and testing, with shared networking, health checks, and resource limits.

```bash
# Spin up the full application locally
docker compose up -d
```

---

## 🏛️ Phase 2 — Infrastructure as Code (Terraform)

Provisioned the **complete AWS infrastructure** using Terraform with a modular architecture and remote state management.

### Module Architecture

```
terraform/
├── main.tf                  # Root module — orchestrates VPC + EKS
├── variables.tf             # Parameterized inputs (CIDR, AZs, node config)
├── outputs.tf               # Cluster endpoint, name, VPC ID
├── Modules/
│   ├── vpc/                 # Reusable VPC module
│   │   ├── main.tf          #   VPC, subnets, IGW, NAT, route tables
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/                 # Reusable EKS module
│       ├── main.tf          #   EKS cluster, node groups, IAM roles
│       ├── variables.tf
│       └── outputs.tf
└── EKS/
    └── backend/
        └── main.tf          # S3 bucket for Terraform remote state
```

### VPC Module — What It Provisions

| Resource | Details |
|---|---|
| **VPC** | `10.0.0.0/16` CIDR with DNS hostnames enabled |
| **Public Subnets** | 3 subnets across `us-east-1a`, `1b`, `1c` — tagged for external ELB |
| **Private Subnets** | 3 subnets across 3 AZs — tagged for internal ELB, hosts EKS worker nodes |
| **Internet Gateway** | Attached to VPC for public internet access |
| **NAT Gateways** | One per AZ with Elastic IPs — enables private subnet outbound traffic |
| **Route Tables** | Public routes → IGW; Private routes → NAT Gateway per AZ |

### EKS Module — What It Provisions

| Resource | Details |
|---|---|
| **EKS Cluster** | Kubernetes `v1.36`, deployed into private subnets |
| **Cluster IAM Role** | `AmazonEKSClusterPolicy` attached |
| **Node Group** | `t3.medium` ON_DEMAND instances (min: 1, desired: 2, max: 4) |
| **Node IAM Role** | `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly` |

### State Management

- **Remote Backend**: Terraform state is stored in S3 (`opentelemetry-eks-state-bucket`) with encryption enabled
- **State Locking**: Uses S3 native lock files (`use_lockfile = true`) to prevent concurrent modifications
- The S3 backend bucket itself is provisioned via a separate Terraform config (`terraform/EKS/backend/`)

### Terraform Commands Used

```bash
# Initialize modules and backend
terraform init

# Preview infrastructure changes
terraform plan

# Provision the infrastructure
terraform apply

# Tear down (lifecycle management)
terraform destroy
```

---

## 🚀 Phase 3 — CI/CD Pipeline (GitHub Actions + ArgoCD)

Implemented a **fully automated GitOps pipeline** where code changes flow from commit to production without manual intervention.

### CI Pipeline — GitHub Actions

The CI workflow (`.github/workflows/ci.yaml`) triggers on **pull requests to `main`** and runs through 4 stages:

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐     ┌──────────────────────┐
│  Build   │────►│ Code Quality  │────►│ Docker Build  │────►│ Update K8s Manifest  │
│  & Test  │     │   (Linting)   │     │   & Push      │     │  (Image Tag)         │
└──────────┘     └───────────────┘     └──────────────┘     └──────────────────────┘
```

#### Stage Details

| Stage | What It Does |
|---|---|
| **Build & Test** | Sets up Go 1.24, downloads dependencies, compiles the binary, runs `go test ./...` |
| **Code Quality** | Runs `golangci-lint v2.12` for static analysis and code quality checks |
| **Docker Build & Push** | Uses `docker/build-push-action` with Buildx; pushes to DockerHub with `github.run_id` as the unique image tag |
| **Update K8s Manifest** | Uses `sed` to update the image tag in `kubernetes/productcatalog/deploy.yaml`, commits the change back to the branch |

#### Key CI Design Decisions

- **Unique image tags** — `github.run_id` ensures every build produces a unique, traceable image tag (no `latest` tag ambiguity)
- **Automated manifest update** — the CI pipeline commits the new image tag directly to the Kubernetes manifest, forming the bridge to GitOps
- **Separation of concerns** — build and code quality jobs run in parallel; Docker push only runs after build succeeds

### CD Pipeline — ArgoCD (GitOps)

ArgoCD is deployed on the EKS cluster and configured to watch this repository's `kubernetes/` directory.

#### How It Works

1. **CI pushes a new image tag** → CI updates the deployment manifest → CI commits and pushes
2. **ArgoCD detects the commit** → compares the desired state (Git) with the live state (cluster)
3. **ArgoCD syncs automatically** → applies the updated manifest to the EKS cluster
4. **Zero manual intervention** — the entire flow from code change to production deployment is automated

#### ArgoCD Benefits Realized

- 🔄 **Automatic sync** — any Git change is reflected in the cluster within seconds
- 📋 **Audit trail** — Git history serves as the single source of truth for all deployments
- ⏪ **Easy rollbacks** — revert a Git commit to roll back a deployment instantly
- 🔍 **Drift detection** — ArgoCD alerts if the live cluster state diverges from Git

---

## 📂 Kubernetes Manifests

Each microservice has its own directory under `kubernetes/` with a `Deployment` and `Service` manifest:

```
kubernetes/
├── serviceaccount.yaml              # Shared ServiceAccount for all services
├── complete-deploy.yaml             # Full application deployment (single file)
├── productcatalog/
│   ├── deploy.yaml                  # Deployment — image updated by CI pipeline
│   └── svc.yaml                     # ClusterIP Service
├── recommendation/
│   ├── deploy.yaml                  # Deployment
│   └── svc.yaml                     # ClusterIP Service
├── ad/
│   ├── deploy.yaml                  # Deployment
│   └── svc.yaml                     # ClusterIP Service
├── cart/
├── checkout/
├── currency/
├── email/
├── frontend/
├── payment/
├── shipping/
└── ... (15+ services total)
```

### Manifest Highlights

- **OpenTelemetry integration** — every pod exports traces and metrics to the OTel Collector via `OTEL_EXPORTER_OTLP_ENDPOINT`
- **Resource limits** — memory limits set per service (e.g., 20Mi for Product Catalog, 500Mi for Recommendation)
- **Kubernetes labels** — consistent labeling with `app.kubernetes.io/*` for discoverability and management
- **Service discovery** — services communicate via Kubernetes DNS (`<service-name>:<port>`)

---

## 🛠️ Tech Stack

| Category | Technologies |
|---|---|
| **Languages** | Go, Python, Java, .NET, Node.js, Rust, C++, PHP, Ruby, Kotlin |
| **Containerization** | Docker, Docker Compose, Multi-stage Builds |
| **Orchestration** | Kubernetes (AWS EKS v1.36) |
| **Infrastructure** | Terraform (Modular), AWS (VPC, EKS, S3, IAM, NAT Gateway) |
| **CI** | GitHub Actions |
| **CD** | ArgoCD (GitOps) |
| **Registry** | DockerHub |
| **Observability** | OpenTelemetry, OTel Collector |
| **Communication** | gRPC, Protocol Buffers, Kafka |

---

## 📁 Repository Structure

```
.
├── .github/workflows/
│   └── ci.yaml                # GitHub Actions CI pipeline
├── src/                       # Source code for all 15+ microservices
│   ├── product-catalog/       #   Go — with Dockerfile
│   ├── recommendation/        #   Python — with Dockerfile
│   ├── ad/                    #   Java — with Dockerfile
│   ├── cart/                  #   .NET
│   ├── checkout/              #   Go
│   ├── currency/              #   C++
│   ├── email/                 #   Ruby
│   ├── frontend/              #   TypeScript
│   ├── payment/               #   Node.js
│   ├── shipping/              #   Rust
│   └── ...
├── kubernetes/                # K8s manifests (watched by ArgoCD)
│   ├── productcatalog/
│   ├── recommendation/
│   ├── ad/
│   └── ...
├── terraform/                 # Infrastructure as Code
│   ├── main.tf                #   Root module
│   ├── Modules/
│   │   ├── vpc/               #   VPC module
│   │   └── eks/               #   EKS module
│   └── EKS/backend/           #   S3 state backend
└── docker-compose.yml         # Local development environment
```

---

## 🚀 Getting Started

### Prerequisites

- AWS CLI configured with appropriate credentials
- Terraform >= 1.0
- Docker & Docker Compose
- `kubectl` configured
- ArgoCD CLI (optional)

### 1. Provision Infrastructure

```bash
# Create the S3 backend bucket first
cd terraform/EKS/backend
terraform init && terraform apply

# Provision VPC + EKS
cd ../../
terraform init
terraform plan
terraform apply
```

### 2. Configure kubectl

```bash
aws eks update-kubeconfig --name my-eks-cluster --region us-east-1
```

### 3. Install ArgoCD on the Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Connect ArgoCD to This Repository

Point ArgoCD to the `kubernetes/` directory of this repository. ArgoCD will automatically sync all manifests to the cluster.

### 5. Trigger a Deployment

Simply open a pull request to `main` — the CI pipeline will:
1. Build and test the code
2. Push a new Docker image
3. Update the Kubernetes manifest
4. ArgoCD auto-syncs the change to the cluster ✅

---

## 💡 Key Learnings & Interview Talking Points

- **Why Terraform Modules?** — Reusability and separation of concerns. The VPC and EKS modules can be independently versioned, tested, and reused across projects. Changes to networking don't affect cluster configuration.

- **Why S3 Remote State?** — Enables team collaboration, prevents state conflicts with locking, and provides encryption at rest for sensitive infrastructure metadata.

- **Why GitOps with ArgoCD?** — Git becomes the single source of truth. Every deployment is auditable, reproducible, and reversible. No `kubectl apply` from local machines.

- **Why `github.run_id` as the image tag?** — Guarantees unique, monotonically increasing tags tied to specific CI runs. Makes it trivial to trace a running container back to the exact build that produced it.

- **Why Multi-stage Docker Builds?** — Reduces final image size by 10-20x by excluding build tools, compilers, and source code from the runtime image. Smaller images = faster pulls, less storage, reduced attack surface.

- **Why Private Subnets for Worker Nodes?** — Security best practice. Worker nodes are not directly accessible from the internet. All outbound traffic goes through NAT Gateways, and ingress is controlled via load balancers in public subnets.

---

## 📜 License

This project is based on the [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo) and is licensed under the [Apache License 2.0](./LICENSE).
