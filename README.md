# Vikunja : DevOps Infrastructure on Azure

> End-of-Training Project - DevOps Engineer 2025-2026 (Bruxelles Formation)
> TechNova Solutions (fictional company)

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Quick Start (local)](#quick-start-local)
- [Deploying to Azure](#deploying-to-azure)
- [CI/CD Pipeline](#cicd-pipeline)
- [Monitoring](#monitoring)
- [Branching Strategy](#branching-strategy)
- [Secret Management](#secret-management)
- [Author](#author)

---

## Project Overview

This repository contains the full DevOps infrastructure to deploy
[Vikunja](https://vikunja.io), an open-source task manager, on Microsoft Azure.

The project covers the complete DevOps lifecycle:

- Containerisation of the Go API and Vue.js frontend
- Automated CI/CD pipeline via GitHub Actions
- Azure infrastructure provisioning via Terraform (IaC)
- Secure secret management with Azure Key Vault
- Full observability: metrics, logs and alerting

The focus is on industry best practices: infrastructure as code, environment
separation, least-privilege principle and operational documentation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Actions                           │
│   Build → Test → Lint → Docker Build → Push ACR → Deploy      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                ┌───────────▼───────────┐
                │  Azure Container      │
                │  Registry (ACR)       │
                └───────────┬───────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼───────┐   ┌───────▼───────┐   ┌──────▼────────┐
│   Staging     │   │  Production   │   │  Monitoring   │
│   VM (B1s)    │   │  VM (B1s)     │   │  VM (B1s)     │
│               │   │               │   │               │
│ - API Go      │   │ - API Go      │   │ - Prometheus  │
│ - Frontend    │   │ - Frontend    │   │ - Grafana     │
│ - Nginx       │   │ - Nginx       │   │ - Loki        │
└───────┬───────┘   └───────┬───────┘   └───────────────┘
        │                   │
        └─────────┬─────────┘
                  │
    ┌─────────────▼─────────────┐
    │  Azure Database for       │
    │  PostgreSQL (Flexible)    │
    └───────────────────────────┘
```

The detailed architecture diagram is available in [`docs/architecture.drawio`](docs/architecture.drawio).

---

## Tech Stack

| Category | Tool | Role |
|---|---|---|
| SCM | GitHub | Code hosting, PRs, branch protection |
| CI/CD | GitHub Actions | Automated build and deployment pipeline |
| Containerisation | Docker (multi-stage) | Building and packaging services |
| Registry | Azure Container Registry | Docker image storage |
| IaC | Terraform (AzureRM provider) | Full infrastructure provisioning |
| Cloud | Microsoft Azure | Environment hosting |
| Secrets | Azure Key Vault | Secure secret management |
| Reverse proxy | Nginx + Let's Encrypt | TLS termination |
| Metrics | Prometheus + Grafana | Metrics collection and visualisation |
| Logs | Loki + Promtail | Log centralisation |
| Alerting | Alertmanager + Discord | Alert notifications |
| Image security | Trivy | Vulnerability scanning in the CI pipeline |

---

## Prerequisites

The following tools must be installed on your local machine:

```bash
# Check versions
terraform --version    # >= 1.6
docker --version       # >= 24.0
docker compose version # >= 2.0
az --version           # Azure CLI >= 2.50
git --version
```

An active Azure account is required with sufficient permissions to create
resources (Contributor role on the subscription or a dedicated Resource Group).

---

## Repository Structure

```
vikunja-infra/
│
├── .github/
│   └── workflows/
│       ├── ci-api.yml         
│       ├── ci-frontend.yml    
│       └── cd-prod.yml         
│
├── terraform/
│   ├── modules/
│   │   ├── networking/         
│   │   ├── compute/            
│   │   └── database/           
│   ├── environments/
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   └── terraform.tfvars
│   │   └── production/
│   │       ├── main.tf
│   │       └── terraform.tfvars
│   └── backend.tf           
│
├── docker/
│   ├── api/
│   │   └── Dockerfile        
│   ├── frontend/
│   │   └── Dockerfile          
│   └── docker-compose.yml      
│
├── monitoring/
│   ├── docker-compose.monitoring.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── grafana/
│   │   └── dashboards/
│   ├── loki/
│   │   └── loki-config.yml
│   └── alertmanager/
│       └── alertmanager.yml
│
├── config/
│   ├── local.env      
│   ├── staging.env
│   └── production.env
│
└── docs/
    ├── architecture.drawio    
    ├── runbook.md              
    └── branching-strategy.md   
```

---

## Quick Start (local)

Clone the Vikunja source repositories:

```bash
# Clone this infrastructure repository
git clone https://github.com/nurumisys/vikunja-infra.git
cd vikunja-infra

# Clone the API and frontend into the expected directories
git clone https://github.com/go-vikunja/vikunja.git sources/api
git clone https://github.com/go-vikunja/frontend.git sources/frontend
```

Set up local environment variables:

```bash
cp config/local.env.example config/local.env
# Edit local.env with your values (DB password, etc.)
```

Start the full local stack:

```bash
docker compose -f docker/docker-compose.yml up --build
```

The application is available at `http://localhost:3000`.  
The API is available at `http://localhost:3456`.

---

## Deploying to Azure

### 1. Azure authentication

```bash
az login
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

### 2. Create Terraform remote state storage (once)

```bash
az group create --name rg-tfstate --location westeurope
az storage account create \
  --name satfstatevikunja \
  --resource-group rg-tfstate \
  --sku Standard_LRS
az storage container create \
  --name tfstate \
  --account-name satfstatevikunja
```

### 3. Provision the staging environment

```bash
cd terraform/environments/staging
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

### 4. Provision the production environment

```bash
cd terraform/environments/production
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

> **Note:** Never commit `terraform.tfvars` files containing secrets.
> Use Azure Key Vault or CI environment variables for sensitive values.

### 5. Destroy the infrastructure

```bash
# Always destroy after the presentation to avoid unnecessary costs
terraform destroy -var-file="terraform.tfvars"
```

---

## CI/CD Pipeline

The pipeline triggers automatically on every branch push and on every Pull Request
targeting `main`.

```
Push / PR
    │
    ├── ci-api.yml
    │   ├── go test ./...
    │   ├── golangci-lint
    │   ├── gosec (SAST)
    │   ├── trivy (image scan)
    │   ├── docker build + push ACR
    │   └── deploy → staging (automatic)
    │
    └── ci-frontend.yml
        ├── npm run test:unit
        ├── eslint
        ├── docker build + push ACR
        └── deploy → staging (automatic)

Merge to main
    │
    └── cd-prod.yml (manual trigger)
        ├── Select version to deploy
        ├── Pull image from ACR
        └── Rolling update → production
```

Production deployment is **manual** (`workflow_dispatch`) to prevent accidental
releases and ensure a human review before going live.

---

## Monitoring

The monitoring stack is deployed on a dedicated VM and accessible at:

| Service | URL | Description |
|---|---|---|
| Grafana | `http://monitoring-vm-ip:3000` | Metrics and log dashboards |
| Prometheus | `http://monitoring-vm-ip:9090` | PromQL query interface |
| Alertmanager | `http://monitoring-vm-ip:9093` | Active alert management |

Three alerts are configured by default:

| Alert | Condition | Notification |
|---|---|---|
| `ApiDown` | API service unreachable for > 1 min | Discord — Critical |
| `HighCPU` | CPU > 80% for > 5 min | Discord — Warning |
| `DiskSpaceLow` | Disk usage > 85% | Discord — Warning |

To start the monitoring stack manually:

```bash
docker compose -f monitoring/docker-compose.monitoring.yml up -d
```

---

## Branching Strategy

This project uses **Trunk-Based Development**:

- `main`: the principal branch, always stable and deployable. Protected: merges only via Pull Request, CI must pass before merge.
- `feat/<name>`: short-lived feature branches (ideally < 2 days).
- `fix/<name>`: bugfix branches.

Full strategy documentation is available in [`docs/branching-strategy.md`](docs/branching-strategy.md).

---

## Secret Management

**No secret must ever appear in plain text in this repository.**

Secrets are managed as follows:

- **Azure Key Vault**: passwords, application keys, tokens. VMs access them via their Managed Identity (no static credentials).
- **GitHub Secrets**: Azure credentials for the CI/CD pipeline (`AZURE_CREDENTIALS`, `ACR_LOGIN_SERVER`, etc.).
- **`.env` files**: `config/*.env.example` files serve as templates. Actual values are never committed.

If a secret is accidentally committed, immediately revoke the affected value
and notify the project owner.

---

## Author

**Morgane Renaut**  
DevOps Engineer Training 2025-2026 (Bruxelles Formation)
End-of-Training Project
