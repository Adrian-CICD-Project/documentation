# DevOps Guide

Practical reference for DevOps / Platform Engineers.

---

## 1. Purpose and Scope

This document describes in one place:
- Repository and environment architecture
- Azure infrastructure (AKS, ACR, auto-shutdown)
- GitOps with ArgoCD
- CI/CD process with centralized templates
- Security and secrets management
- Monitoring, alerting, and tooling

---

## 2. Repository Architecture (GitHub)

```
GitHub Organization
|
├── infrastructure                # Terraform: AKS, ACR, network, ArgoCD
|   ├── modules/
|   └── envs/
|
├── ci-cd-templates               # Centralized reusable workflow templates
|   └── .github/workflows/
|       ├── java-ci-full.yml      # Full CI pipeline for Java apps
|       ├── promote-environment.yml # Environment promotion
|       └── validate-manifests.yml  # K8s manifest validation
|
├── infrastructure-env-dev        # GitOps environment-dev
├── infrastructure-env-test       # GitOps environment-test
├── infrastructure-env-prod       # GitOps environment-prod
|
├── platform-apps                 # ArgoCD apps + Helm charts (SonarQube, DT, Prometheus)
|   └── app-of-apps.yaml
|
├── devops-project                # Spring Boot application + CI trigger
|
└── Documentation                 # This documentation
```

---

## 3. Azure Infrastructure

### 3.1 AKS
Environments:
- **devops-poc01-test**  
  namespaces: `environment-dev`, `environment-test`
- **devops-poc01-prod**  
  namespace: `environment-prod`

Provisioning: Terraform, module `aks`.

### 3.2 ACR
- Azure Container Registry as image registry
- Images from CI pushed directly to ACR

### 3.3 Auto-shutdown
- Requirement: shutdown at 18:00 daily
- Implementation: Automation Account or Terraform schedule

---

## 4. ArgoCD / GitOps

### 4.1 Installation
- ArgoCD installation via Terraform or Helm provider in namespace `argocd`

### 4.2 App-of-apps
- Main file: `platform-apps/app-of-apps.yaml`
- Manages:
  - Platform apps (SonarQube, Dependency-Track, Prometheus, Grafana)
  - Environment applications

### 4.3 Environment Promotion
- Flow: `dev -> test -> prod`
- CI in application repo updates only `environment-dev`
- Promotion to test/prod: merge PR between GitOps repositories
- Only manual step: **Merge PR**

---

## 5. CI/CD

### 5.1 Centralized Templates

All CI/CD logic is maintained in the **ci-cd-templates** repository:

| Template | Purpose |
|----------|---------|
| `java-ci-full.yml` | Complete CI pipeline for Java Spring Boot apps |
| `promote-environment.yml` | Promote app between environments (dev->test, test->prod) |
| `validate-manifests.yml` | Validate Kubernetes manifest YAML syntax |

Application repositories only contain trigger configuration and call these centralized templates.

### 5.2 CI (repo `devops-project`)

The CI workflow calls `ci-cd-templates/java-ci-full.yml` which executes:

1. Maven build
2. Tests + coverage
3. SonarQube analysis
4. SBOM generation + send to Dependency-Track
5. Docker image build and push to ACR
6. Image vulnerability scan
7. GitHub Release creation
8. Update `infrastructure-env-dev` repository

### 5.3 CD (repositories `infrastructure-env-*`)

Each environment repo contains:
- ArgoCD Application for the environment
- Workflow calling `ci-cd-templates/promote-environment.yml`
- Image tag updates
- ArgoCD auto-sync triggers

---

## 6. Secrets and Security

### 6.1 General Principles
- Secrets never go into Git
- Allowed locations: GitHub Secrets, Kubernetes Secrets

### 6.2 GitHub Secrets (examples)
- `AZURE_CREDENTIALS`, `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`
- `SONAR_TOKEN`, `SONAR_HOST_URL`
- `DEPENDENCY_TRACK_API_KEY`, `DTRACK_API_URL`
- `ACR_NAME`
- `ENV_REPOS_TOKEN`

### 6.3 Kubernetes Secrets
- Store application data
- Injected via Helm or created manually by DevOps

---

## 7. Monitoring and Alerting

### 7.1 Monitoring Stack
- In test cluster: Prometheus, Alertmanager, Grafana
- Version: kube-prometheus-stack (SLIM)

### 7.2 What We Monitor
- Spring Boot metrics (Actuator), HTTP requests, 4xx/5xx errors
- Grafana: ready-made dashboards

### 7.3 Alert: HTTP 500 -> email
Requirement: send email on HTTP 500.

Implementation:
1. Alert rule in Prometheus, e.g., `increase(http_server_requests_seconds_count{status="500"}[5m]) > 1`  
2. Alertmanager route: email type, SMTP from K8s secret  
3. Test: call endpoint `/api/error500`

---

## 8. Tools and Versions

| Area | Tool / Service | Purpose | Version |
|------|----------------|---------|---------|
| Cloud | Azure | Cloud environment | - |
| Kubernetes | AKS | K8s cluster | - |
| Registry | ACR | Image registry | - |
| IaC | Terraform | Infrastructure as code | - |
| IaC | AzureRM Provider | Terraform provider | - |
| CI/CD | GitHub Actions | CI/CD pipeline | - |
| CI/CD | ci-cd-templates | Centralized workflow templates | - |
| App Runtime | Java 21 | Application runtime | 21 |
| Build Tool | Maven | Application build | - |
| Framework | Spring Boot | Application backend | - |
| Code Quality | SonarQube | Static analysis | - |
| Security | Dependency-Track | Dependency analysis | - |
| Observability | Prometheus | Metrics | - |
| Observability | Alertmanager | Alert routing | - |
| Observability | Grafana | Dashboards | - |
| K8s Tools | kubectl | CLI | - |
| K8s Tools | Helm | Chart management | - |
| Security | Trivy | Image scanning | - |

---

## 9. Operational Procedures

### 9.1 New Environment
1. Add folder in `infrastructure/envs`
2. Enable modules: network, AKS, ACR, ArgoCD
3. Run Terraform
4. Create repository `infrastructure-env-<name>`
5. Add workflow calling `ci-cd-templates/promote-environment.yml`
6. Add repository to App-of-apps

### 9.2 AKS Cluster Update
1. Plan version
2. Update AKS module
3. Run Terraform
4. Test: ArgoCD, platform apps, application

### 9.3 Adding New Application
1. Create application repository with CI workflow calling `ci-cd-templates/java-ci-full.yml`
2. Add application manifests to environment repositories
3. Configure ArgoCD Application
4. Add GitHub Secrets for the new application

---

## 10. Change Management
- All changes through Pull Requests
- Mandatory validation: `terraform fmt/validate`, `kubectl apply --dry-run=client`, `helm template`
- After merge: ArgoCD + pipelines automatically update the environment

---

## 11. References
- Client requirements document
- Developer Guide
- Internal runbooks (optional)
