# Final Project - Adrian-CICD-Project

## Table of Contents

- [1. Organization Setup](#organization-setup)
  - [1.1. Repository Creation](#repository-creation)
  - [1.2. GitHub Apps Configuration](#github-apps-configuration)
- [2. Infra-azure - Infrastructure as Code](#infra-azure-infrastructure-as-code)
  - [2.1. Project Structure](#project-structure)
  - [2.2. Main Files Description](#main-files-description)
  - [2.3. Terraform Modules](#terraform-modules)
  - [2.4. Automation Scripts](#automation-scripts)
  - [2.5. Deployment Process](#deployment-process)
  - [2.6. ArgoCD Installation](#argocd-installation)
- [3. Platform-apps - GitOps and App-of-Apps](#platform-apps-gitops-and-app-of-apps)
  - [3.1. Project Structure](#project-structure-1)
  - [3.2. Bootstrap Folder](#bootstrap-folder)
  - [3.3. Charts/app-of-apps Folder](#chartsapp-of-apps-folder)
  - [3.4. Automation Scripts](#automation-scripts-1)
  - [3.5. Deployment Results](#deployment-results)
- [4. CI/CD Tools Preparation](#cicd-tools-preparation)
  - [4.1. SonarQube Configuration](#sonarqube-configuration)
  - [4.2. DependencyTrack Configuration](#dependencytrack-configuration)
  - [4.3. kube-prometheus-stack Configuration](#kube-prometheus-stack-configuration)
- [5. CI/CD Templates - Centralized Workflows](#cicd-templates-centralized-workflows)
  - [5.1. Repository Structure](#repository-structure)
  - [5.2. Available Templates](#available-templates)
  - [5.3. Usage Pattern](#usage-pattern)
- [6. Adrian-java-app - Application and CI Pipeline](#adrian-java-app-application-and-ci-pipeline)
  - [6.1. CI Pipeline Scope](#ci-pipeline-scope)
  - [6.2. Execution Stages](#execution-stages)
  - [6.3. CI/CD Flow](#cicd-flow)
  - [6.4. GitHub Secrets Configuration](#github-secrets-configuration)
  - [6.5. CI Execution](#ci-execution)
  - [6.6. Analysis Results](#analysis-results)
- [7. Infrastructure-env-dev - Development Environment](#infrastructure-env-dev-development-environment)
  - [7.1. Repository Contents](#repository-contents)
  - [7.2. GitOps Process](#gitops-process)
  - [7.3. Promotion to TEST](#promotion-to-test)
- [8. Infrastructure-env-test - Test Environment](#infrastructure-env-test-test-environment)
  - [8.1. Repository Contents](#repository-contents-1)
  - [8.2. GitOps Process](#gitops-process-1)
  - [8.3. Promotion to PROD](#promotion-to-prod)
- [9. Infrastructure-env-prod - Production Environment](#infrastructure-env-prod-production-environment)
  - [9.1. Repository Contents](#repository-contents-2)
  - [9.2. GitOps Process](#gitops-process-2)
  - [9.3. Manifest Validation](#manifest-validation)
- [10. Application Monitoring](#application-monitoring)
  - [10.1. Key Components](#key-components)
  - [10.2. Configuration Instructions](#configuration-instructions)
  - [10.3. Verification Checklist](#verification-checklist)
  - [10.4. Monitoring Results](#monitoring-results)
- [11. Project Documentation](#project-documentation)
- [12. Multi-Cloud Infrastructure (AWS & GCP)](#multi-cloud-infrastructure-aws--gcp)

---

## Organization Setup

### Repository Creation

The first step was to create a repository in the GitHub organization, which will serve as the foundation for the entire DevOps project.

![](media/image1.png)

### GitHub Apps Configuration

To authorize the repository in ArgoCD, GitHub Apps was created for the organization. The application configuration enables secure connection between ArgoCD and organization repositories.

![](media/image28.png)

![](media/image13.png)

---

## Infra-azure - Infrastructure as Code

The **infra-azure** repository is responsible for creating all infrastructure required for the project using Terraform.

### Project Structure

```
.
├── backend.hcl
├── check-infra.sh
├── deploy.sh
├── destroy.sh
├── envs
│   ├── prod.tfvars
│   └── test.tfvars
├── install-argocd.sh
├── main.tf
├── modules
│   ├── acr
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── aks
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── auto-shutdown
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── key-vault
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── network
│   │   ├── main.tf
│   │   └── variables.tf
│   └── resource-group
│       ├── main.tf
│       └── variables.tf
├── outputs.tf
├── providers.tf
├── README.md
├── docs/
│   └── key-vault-external-secrets-setup.md
├── variables.tf
└── versions.tf
```

### Main Files Description

| File | Description |
|------|-------------|
| `main.tf` | Calls created modules from `./modules` folder |
| `variables.tf` | Variable definitions required for the entire project (e.g., project name, region) |
| `outputs.tf` | Specifies information displayed after `terraform apply` (e.g., cluster IP address) |
| `providers.tf` | Provider configuration (e.g., `azurerm` for Azure, `kubernetes`) |
| `versions.tf` | Specifies required Terraform and provider versions for compatibility |

### Terraform Modules

The `./modules` folder contains independent infrastructure units:

- **resource-group** - Creates logical container for all Azure resources
- **network** - VNET configuration, subnets, and security rules (NSG)
- **acr (Azure Container Registry)** - Private registry for Docker images
- **aks (Azure Kubernetes Service)** - Kubernetes cluster configuration (node pools, cluster version)
- **auto-shutdown** - Azure Automation Runbook that stops AKS clusters daily at 18:00 CET (Central European Time) to reduce costs
- **key-vault** - Azure Key Vault with RBAC authorization, role assignments for AKS kubelet identities (`Key Vault Secrets User`) and Terraform identity (`Key Vault Secrets Officer`). Used with External Secrets Operator to deliver secrets to Kubernetes

### Automation Scripts

| Script | Function |
|--------|----------|
| `deploy.sh` / `destroy.sh` | Automates creating and destroying entire infrastructure |
| `check-infra.sh` | Verifies correctness of created resources |
| `install-argocd.sh` | Configures ArgoCD after cluster creation (`kubectl` or `helm` commands) |

### Deployment Process

Infrastructure deployment flow:

1. Run `deploy.sh` script
2. Script calls `main.tf`, loading data from `envs/test.tfvars`
3. `main.tf` runs each module, passing variables
4. Terraform manages state (state files are excluded from Git via `.gitignore`)

**Outputs after deployment:**

![](media/image16.png)

![](media/image25.png)

### ArgoCD Installation

After running `install-argocd.sh` script:

![](media/image27.png)

![](media/image14.png)

**Summary:** At this point, complete infrastructure has been created along with ArgoCD exposed to the world with login credentials.

> **Note:** ArgoCD is installed via `install-argocd.sh` script (not a Terraform module) to avoid Helm provider dependency cycles and ensure stable AKS connectivity.

---

## Platform-apps - GitOps and App-of-Apps

The **platform-apps** repository implements the **GitOps** pattern using the **App-of-Apps** strategy. In this approach, ArgoCD manages one main application that controls all other applications in the cluster.

### Project Structure

```
platform-apps/
├── bootstrap/
│   ├── app-of-apps-prod.yaml
│   ├── app-of-apps-test.yaml
│   ├── argocd-repositories-github-app.yaml  # Template (secrets via External Secrets)
│   ├── external-secrets-config.yaml          # ClusterSecretStore + ExternalSecret CRDs
│   └── ingress-nginx.yaml
├── charts/
│   └── app-of-apps/
│       ├── templates/
│       │   └── applications.yaml
│       ├── values.yaml            # TEST environment values
│       └── values-prod.yaml       # PROD environment values (separate config)
├── scripts/
│   ├── deploy-platform.sh
│   └── get-access-info.sh
└── manuals/
```

### Bootstrap Folder

The `bootstrap/` folder contains Kubernetes manifests (Custom Resources) that instruct ArgoCD what to install initially:

- **app-of-apps-prod.yaml / app-of-apps-test.yaml** - Application resource definitions pointing to `charts/app-of-apps` folder. PROD uses `values-prod.yaml` for separate production configuration; TEST uses `values.yaml`.
- **argocd-repositories-github-app.yaml** - Template for ArgoCD repository access secrets. Private keys are **not** stored in this file — they are delivered via External Secrets Operator from Azure Key Vault.
- **external-secrets-config.yaml** - Defines `ClusterSecretStore` (connected to Azure Key Vault via Managed Identity) and `ExternalSecret` CRDs that automatically create Kubernetes Secrets for ArgoCD repository access.
- **ingress-nginx.yaml** - Manifest installing Ingress Controller, which enables exposing DependencyTrack externally (solves 405 error).

### Charts/app-of-apps Folder

This is a local Helm Chart serving as infrastructure "table of contents":

- **templates/applications.yaml** - Most important file in the repository. Contains a loop that, based on the list in `values.yaml`, generates subsequent `Application` objects for ArgoCD (Prometheus, backend application, database, etc.).
- **values.yaml** - List of all applications that ArgoCD should track, along with Git repository addresses. Includes External Secrets Operator configuration.
- **values-prod.yaml** - Production-specific overrides (separate retention policies, persistence settings, environment-prod scraping, ESO enabled).

### Automation Scripts

- **deploy-platform.sh** - Responsible for deploying entire bootstrap for test and prod environments, including External Secrets Operator configuration and Ingress error fixes
- **get-access-info.sh** - Displays access data (ArgoCD admin password, application URLs)

Detailed process and operation description in: `Platform-apps/manuals/Deploy-platform.md`

![](media/image17.png)

### Deployment Results

After running the script, the stack being created in ArgoCD:

**TEST Environment:**

![](media/image29.png)

**PROD Environment:**

![](media/image10.png)

---

## CI/CD Tools Preparation

### SonarQube Configuration

SonarQube configuration process:

1. Get application IP address and login to UI with: `admin / admin`
2. **Immediately change default password** — default credentials are a security risk
3. Generate API key:
   - Path: `Administration` -> `Access Management` -> `Teams` -> `Administrators` -> `Generate New API Key`
4. Copy API key for later use in CI

### DependencyTrack Configuration

DependencyTrack configuration process:

1. Get application IP address and login to UI with: `admin / admin`
2. **Immediately change default password** — default credentials are a security risk
3. Generate API key:
   - Path: Profile -> `My Account` -> `Security` -> enter token name -> `Generate`
4. Copy API key for later use

### kube-prometheus-stack Configuration

ArgoCD automatically deploys the entire stack along with rules and secrets used for detecting anomalies in Java application (e.g., 500 error), which will be sent to email.

**Verification via port-forward:**

```bash
# Prometheus
kubectl -n monitoring port-forward svc/prometheus-server 9090:80
# -> localhost:9090

# Alertmanager
kubectl -n monitoring port-forward svc/alertmanager 9093:9093
# -> localhost:9093
```

Detailed description in: `Platform-apps/manuals/Monitoring500Error.md`

---

## CI/CD Templates - Centralized Workflows

The **ci-cd-templates** repository contains all reusable GitHub Actions workflows used across the organization. This centralized approach ensures consistency and reduces code duplication.

### Repository Structure

```
ci-cd-templates/
└── .github/
    └── workflows/
        ├── java-ci-full.yml        # Complete CI pipeline for Java apps
        ├── promote-environment.yml  # Environment promotion workflow
        └── validate-manifests.yml   # K8s manifest validation
```

### Available Templates

| Template | Purpose | Used By |
|----------|---------|---------|
| `java-ci-full.yml` | Complete CI pipeline: build, test, SonarQube, Docker, release, promote | `adrian-java-app` |
| `promote-environment.yml` | Promote application between environments (dev->test, test->prod) | `infrastructure-env-*` |
| `validate-manifests.yml` | Validate Kubernetes manifest YAML syntax | `infrastructure-env-prod` |

### Usage Pattern

Application repositories call these templates with minimal configuration:

```yaml
# Example: adrian-java-app/.github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, feature/**]
  pull_request:

jobs:
  pipeline:
    uses: Adrian-CICD-Project/ci-cd-templates/.github/workflows/java-ci-full.yml@main
    with:
      image-name: adrian-java-app
      env-repo: Adrian-CICD-Project/infrastructure-env-dev
    secrets: inherit
```

---

## Adrian-java-app - Application and CI Pipeline

The **adrian-java-app** repository contains Java Spring Boot application code with CI pipeline trigger that calls centralized templates.

### CI Pipeline Scope

The pipeline is responsible for:

- Building and testing Java application
- Code quality analysis using SonarQube
- Security scanning with Trivy
- SBOM generation in CycloneDX format
- Building and publishing Docker image to Azure Container Registry (ACR)
- Automatic promotion to DEV environment (GitOps with ArgoCD)

### Execution Stages

Pipeline in GitHub Actions runs automatically after every push to `main` branch.

#### 1. Build and Test
- Running `mvn clean verify`
- Test artifacts available in GitHub Actions -> Artifacts

#### 2. SonarQube Quality Analysis
- Code scan and quality analysis
- Report available in SonarQube instance

#### 3. Security and SBOM
- SBOM generation (`bom.json`) in CycloneDX format
- Docker image security scan with Trivy
- SBOM upload to Dependency-Track via REST API
- Application version registration in Dependency-Track

#### 4. Docker Build and Push
- Building Docker image with semantic version tag (1.0.x)
- Publishing to Azure Container Registry (ACR)

#### 5. Promotion to DEV Environment
- Updating `values/adrian-java-app/values.yaml` in `infrastructure-env-dev` repository
- Automatic Pull Request creation
- After PR approval -> ArgoCD performs rollout to DEV

### CI/CD Flow

```
Commit to adrian-java-app/main
  |
GitHub Actions CI Pipeline (calls Adrian-CICD-Project/java-ci-full.yml)
  |
Build and Test
  |
SonarQube (Quality Analysis)
  |
Security and SBOM (Trivy + CycloneDX + Dependency-Track)
  |
Docker Build and Push (ACR)
  |
Promotion to DEV (infrastructure-env-dev + GitOps ArgoCD)
```

### GitHub Secrets Configuration

![](media/image8.png)

#### Azure (Cloud)
- **AZURE_CREDENTIALS** - Generated with `az ad sp create-for-rbac`. Service Principal data enabling GitHub to create infrastructure in Azure
- **ACR_NAME** - Container registry name (e.g., `acrfordevopspoc01adrian`)

#### SonarQube (Code Analysis)
- **SONAR_TOKEN** - Generated in: `My Account` -> `Security` -> `Generate Token`
- **SONAR_HOST_URL** - SonarQube instance URL on AKS cluster

#### Dependency Track (Library Security)
- **DEPENDENCY_TRACK_API_KEY** - Generated in: `Administration` -> `Teams` -> `Automation` -> `API Key`
- **DTRACK_API_URL** - Dependency Track instance API URL

#### GitHub (GitOps Integration)
- **ENV_REPOS_TOKEN** - Personal Access Token (PAT) generated in: `Settings` -> `Developer Settings` -> `Personal Access Tokens`. Enables CI pipeline to push changes to environment repositories.

### CI Execution

After commit and push to Git, GitHub Actions detects changes and starts CI process:

![](media/image23.png)

GitHub Actions agent goes through entire process and creates commit on `infrastructure-env-dev` repository with new image.

### Analysis Results

#### SonarQube - Code Coverage

![](media/image24.png)

#### DependencyTrack - Threat Scanning

![](media/image18.png)

![](media/image9.png)

![](media/image7.png)

---

## Infrastructure-env-dev - Development Environment

### Repository Contents

Repository contains **DEV** environment configuration:

- `values/adrian-java-app/values.yaml` - Helm/Kustomize values for DEV
- `k8s/adrian-java-app/` - Application manifests (Deployment, Service) with hardened `securityContext`
- `k8s/devops-project/` - Kubernetes manifests for devops-project with hardened `securityContext`
- `deploy-argo-dev.sh` - Runs application in ArgoCD on test environment
- Workflow calling `ci-cd-templates/promote-environment.yml`

> **Security:** All deployments include hardened `securityContext` (runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities).

### GitOps Process

After PR merge in this repository:

1. ArgoCD in DEV environment detects new commit
2. Performs auto-sync
3. Rolls out new application version to `environment-dev` namespace

### Promotion to TEST

Workflow `promote-to-test` (calls centralized template):

1. Gets current tag from `values/adrian-java-app/values.yaml`
2. Updates same tag in `infrastructure-env-test` repository
3. Creates Pull Request to TEST
4. PR merge -> ArgoCD TEST performs rollout

**Related repositories:**
- `adrian-java-app` (version source)
- `ci-cd-templates` (workflow template)
- `infrastructure-env-test` (promotion target)

**State after merge:**

![](media/image20.png)

![](media/image19.png)

---

## Infrastructure-env-test - Test Environment

### Repository Contents

Repository contains declarative **TEST** environment configuration:

- `values/adrian-java-app/values.yaml` - Application values for TEST
- `k8s/adrian-java-app/` - Kubernetes manifests for TEST with hardened `securityContext`
- `k8s/devops-project/` - Kubernetes manifests for devops-project with hardened `securityContext`
- Workflow calling `ci-cd-templates/promote-environment.yml`

### GitOps Process

After PR merge:

1. ArgoCD synchronizes repository
2. Deploys new version to `environment-test` namespace

### Promotion to PROD

Workflow `promote-to-prod` (calls centralized template):

1. Reads tag from `values/adrian-java-app/values.yaml`
2. Introduces same tag to `infrastructure-env-prod` repository
3. Creates PR to PROD
4. PR merge -> rollout on production

**Related repositories:**
- `infrastructure-env-dev`
- `ci-cd-templates` (workflow template)
- `infrastructure-env-prod`

**State after CD execution:**

![](media/image22.png)

![](media/image30.png)

---

## Infrastructure-env-prod - Production Environment

### Repository Contents

Repository contains production environment configuration:

- `values/adrian-java-app/values.yaml`
- `k8s/adrian-java-app/` - with hardened `securityContext`
- `k8s/devops-project/` - with hardened `securityContext`
- Manifest validation (syntactic check via `ci-cd-templates/validate-manifests.yml`)
- Final stage of promotion chain

### GitOps Process

After PR merge:

1. ArgoCD in PROD performs auto-sync
2. Application rollout occurs in `environment-prod` namespace

### Manifest Validation

Workflow executes syntactic validation by calling centralized template:

```yaml
jobs:
  validate:
    uses: Adrian-CICD-Project/ci-cd-templates/.github/workflows/validate-manifests.yml@main
    with:
      app-name: adrian-java-app
```

This validation does not require connecting to the cluster.

**Related repositories:**
- `infrastructure-env-test`
- `ci-cd-templates` (workflow template)

**State after CD:**

![](media/image26.png)

![](media/image5.png)

### Deployed Applications After Promotion

**TEST Environment:**

![](media/image12.png)

**PROD Environment:**

![](media/image10.png)

---

## Application Monitoring

The monitoring system detects 500 errors in Java application and sends email notifications.

### Key Components

- **Application** - Spring Boot with enabled Actuator, exposing Prometheus metrics via `/actuator/prometheus`
- **Monitoring** - Prometheus (metrics collection) + Alertmanager (notification management)
- **Orchestration** - ArgoCD (GitOps) + Helm (configuration and version management)

### Configuration Instructions

After deploying new infrastructure, perform the following steps:

#### 1. Synchronization and Configuration Memory Cleanup
- Create Secret `alertmanager-gmail` in `monitoring` namespace
- In ArgoCD perform synchronization with `Replace` option
- Restart Prometheus:
  ```bash
  kubectl rollout restart deployment prometheus-server -n monitoring
  ```

#### 2. Service Exposure (Port-Forwarding)

Run in separate terminals:

```bash
# Application
kubectl -n environment-dev port-forward svc/adrian-java-app 8080:80
# -> localhost:8080

# Prometheus
kubectl -n monitoring port-forward svc/prometheus-server 9090:80
# -> localhost:9090

# Alertmanager
kubectl -n monitoring port-forward svc/alertmanager 9093:9093
# -> localhost:9093
```

#### 3. Error Generation (Load Test)

Run test loop:

```bash
while true; do
  curl -s http://localhost:8080/api/error500 > /dev/null
  echo "Sent error request: $(date)"
  sleep 2
done
```

Alternative (Coffee Shop API): break the coffee machine so every order returns 500
with a real exception stack trace:

```bash
curl -s -X POST http://localhost:8080/api/chaos/machine/break
while true; do
  curl -s -X POST "http://localhost:8080/api/orders?itemId=latte" > /dev/null
  echo "Sent failing order: $(date)"
  sleep 2
done
# afterwards: curl -s -X POST http://localhost:8080/api/chaos/machine/repair
```

### Verification Checklist

#### Application Metrics
- URL: http://localhost:8080/actuator/prometheus
- Look for: `http_server_requests_seconds_count{status="500",...}` - counter must increase

#### Prometheus -> Alertmanager Connection
- URL: http://localhost:9090/status
- Alertmanagers section: `http://alertmanager:9093/...`
- Warning: If you see IP (e.g., `10.0.x.x`) - restart Prometheus!

#### Alert State
- URL: http://localhost:9090/alerts
- Alert `DevopsProjectHttp500` = **Firing** (red)

#### Email Sending Logs

```bash
kubectl logs alertmanager-0 -n monitoring --tail=20
```

Look for: `component=dispatcher msg="Notify success" receiver=gmail`

#### Gmail
Check inbox: `adrian.dmytryk@gmail.com` (Inbox and SPAM)

### Monitoring Results

**Screenshots after triggering 500 error:**

![](media/image6.png)

![](media/image15.png)

![](media/image21.png)

![](media/image2.png)

![](media/image4.png)

---

## Project Documentation

All documentation is located in the **Documentation** repository and is divided into:

### a) Documentation for DevOps / Architect
[DevOps-Guide.md](https://github.com/Adrian-CICD-Project/documentation/blob/main/DevOps-Guide.md)

### b) Documentation for Developer
[Developer-Guide.md](https://github.com/Adrian-CICD-Project/documentation/blob/main/Developer-Guide.md)

![](media/image3.png)

---

## Secrets Management

Secrets are **never** stored in Git repositories. The project uses the following approach:

| Layer | Tool | Purpose |
|-------|------|---------|
| Storage | **Azure Key Vault** | Central secret store (GitHub App keys, tokens, credentials) |
| Delivery | **External Secrets Operator (ESO)** | Pulls secrets from Key Vault into Kubernetes |
| CI/CD | **GitHub Secrets** | Pipeline tokens (SONAR_TOKEN, ACR credentials, PAT) |
| Runtime | **Kubernetes Secrets** | Created automatically by ESO from Key Vault data |

**Flow:** Azure Key Vault → External Secrets Operator → Kubernetes Secrets → ArgoCD / Application pods

> **Setup guide:** [infra-azure/docs/key-vault-external-secrets-setup.md](https://github.com/Adrian-CICD-Project/infra-azure/blob/main/docs/key-vault-external-secrets-setup.md)

### Security Hardening Applied

- All CI workflows sanitized — no secrets logged in pipeline output
- All deployment manifests include `securityContext` (runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities)
- Terraform state files excluded from Git (`.gitignore`)
- RSA private keys removed from repository files — delivered via External Secrets
- PROD environment uses separate `values-prod.yaml` configuration
- Git history cleaned of sensitive data (BFG Repo-Cleaner)

---

## Related Repositories

| Repository | Purpose |
|------------|---------|
| **infra-azure** | Infrastructure as Code (Terraform) – Azure |
| **infra-aws** | Infrastructure as Code (Terraform) – AWS (EKS/ECR) |
| **infra-gcp** | Infrastructure as Code (Terraform) – GCP (GKE/Artifact Registry) |
| **platform-apps** | GitOps, App-of-Apps, platform tools |
| **ci-cd-templates** | Centralized CI/CD workflow templates |
| **adrian-java-app** | Java Spring Boot Application + CI trigger |
| **infrastructure-env-dev** | ArgoCD configuration for DEV environment |
| **infrastructure-env-test** | ArgoCD configuration for TEST environment |
| **infrastructure-env-prod** | ArgoCD configuration for PROD environment |
| **Documentation** | Project documentation |

---

## Multi-Cloud Infrastructure (AWS & GCP)

The Azure infrastructure was mirrored to **AWS** (`infra-aws`) and **GCP** (`infra-gcp`) to provide
a cost-optimized, multi-cloud parity of the platform: two Kubernetes clusters (`test` + `prod`),
an image registry, network, secret store, and a daily auto-shutdown at 18:00.

| Role | Azure | AWS | GCP |
|------|-------|-----|-----|
| Kubernetes | AKS | EKS | GKE (zonal) |
| Worker nodes | VM `Standard_B4ms` | **EC2** `t3.large` | **Compute Engine** `e2-standard-2` |
| Image registry | ACR | ECR | Artifact Registry |
| Network | VNet | VPC + NAT | VPC + Cloud NAT |
| Secrets | Key Vault | Secrets Manager | Secret Manager |
| Auto-shutdown 18:00 | Automation Account | EventBridge + Lambda | Cloud Scheduler + GKE API |

Full breakdown of every component and the FinOps cost-saving strategy is documented in
[`multicloud-infrastructure.md`](multicloud-infrastructure.md).

---

*Project created by Adrian Dmytryk*