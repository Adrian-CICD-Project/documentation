# DevOps Guide

Practical reference for DevOps / Platform Engineers.

---

## 1. Purpose and Scope

This document describes in one place:
- Repository and environment architecture
- Azure infrastructure (AKS, ACR, auto-shutdown)
- Multi-cloud infrastructure on AWS and GCP (EKS/GKE) – see `multicloud-infrastructure.md`
- GitOps with ArgoCD
- CI/CD process with centralized templates
- Security and secrets management
- Monitoring, alerting, and tooling

---

## 2. Repository Architecture (GitHub)

```
GitHub Organization
|
├── infra-azure                   # Terraform: AKS, ACR, network, ArgoCD
|   ├── modules/
|   └── envs/
|
├── infra-aws                     # Terraform: EKS, ECR, VPC, Secrets, auto-shutdown
├── infra-gcp                     # Terraform: GKE, Artifact Registry, VPC, Secret Manager
|                                 # (multi-cloud – patrz multicloud-infrastructure.md)
|
├── ci-cd-templates               # Centralized reusable workflow templates
|   ├── .github/workflows/
|   |   ├── java-ci-full.yml      # Full CI pipeline for Java apps
|   |   ├── promote-environment.yml # Environment promotion
|   |   ├── validate-manifests.yml  # K8s manifest validation
|   |   ├── gitleaks.yml          # Secret scanning (full git history)
|   |   └── terraform-ci.yml      # Terraform fmt/validate + Checkov
|   └── scripts/
|       └── setup-branch-protection.sh # One-time branch protection setup (gh CLI)
|
├── infrastructure-env-dev        # GitOps environment-dev
├── infrastructure-env-test       # GitOps environment-test
├── infrastructure-env-prod       # GitOps environment-prod
|
├── platform-apps                 # ArgoCD apps + Helm charts (SonarQube, DT, Prometheus)
|   └── app-of-apps.yaml
|
├── adrian-java-app               # Spring Boot application + CI trigger
|
└── documentation                 # This documentation
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
- Schedule: daily at **18:00 Central European Time**
- Implementation: Azure Automation Account + PowerShell Runbook (`Connect-AzAccount -Identity` + `Stop-AzAksCluster`)
- Configured via Terraform module `auto-shutdown`

### 3.4 Azure Key Vault
- Stores sensitive data: GitHub App keys, tokens, credentials
- AKS clusters access via Managed Identity (RBAC role: `Key Vault Secrets User`)
- Provisioned via Terraform module `key-vault`

---

## 4. ArgoCD / GitOps

### 4.1 Installation
- ArgoCD installation via Terraform or Helm provider in namespace `argocd`

### 4.2 App-of-apps
- Bootstrap files: `platform-apps/bootstrap/app-of-apps-test.yaml` (TEST), `app-of-apps-prod.yaml` (PROD)
- PROD uses separate `values-prod.yaml` with production-appropriate settings
- Manages:
  - Platform apps (SonarQube, Dependency-Track, Prometheus, Alertmanager, Grafana)
  - Argo Rollouts controller (PROD cluster only)
  - External Secrets Operator
  - Environment applications

### 4.3 Environment Promotion
- Flow: `dev -> test -> prod`
- CI in application repo updates only `environment-dev`
- Promotion to test/prod: merge PR between GitOps repositories
- Only manual step: **Merge PR**

### 4.4 Canary Deployments on PROD (Argo Rollouts)
- `adrian-java-app` on PROD is deployed as an Argo Rollouts `Rollout` (canary strategy):
  **50% traffic → 60s pause → full rollout**
- The Rollout manifest intentionally keeps the filename `deployment.yaml`
  (`infrastructure-env-prod/k8s/adrian-java-app/deployment.yaml`) so promotion
  workflows can still rewrite the `image:` line by file path
- Scaling and availability: HPA (2–4 replicas, CPU 80%) + PodDisruptionBudget (`minAvailable: 1`)
- Watch a rollout:
  ```bash
  kubectl argo rollouts get rollout adrian-java-app -n environment-prod --watch
  ```

---

## 5. CI/CD

### 5.1 Centralized Templates

All CI/CD logic is maintained in the **ci-cd-templates** repository:

| Template | Purpose |
|----------|---------|
| `java-ci-full.yml` | Complete CI pipeline for Java Spring Boot apps |
| `promote-environment.yml` | Promote app between environments (dev->test, test->prod) |
| `validate-manifests.yml` | Validate Kubernetes manifest YAML syntax |
| `gitleaks.yml` | Secret scanning over full git history (wired into all repos) |
| `terraform-ci.yml` | Terraform `fmt -check` + `validate` + Checkov scan (infra repos) |

Application repositories only contain trigger configuration and call these centralized templates.

### 5.2 CI (repo `adrian-java-app`)

The CI workflow calls `ci-cd-templates/java-ci-full.yml` which executes:

1. Maven build
2. Tests + coverage
3. SonarQube analysis
4. SBOM generation + send to Dependency-Track (findings exported back as report)
5. Docker image build; push to ACR **only from `main`** (feature branches build-only)
6. Image vulnerability scan (Trivy) — **pipeline fails on fixable CRITICAL vulnerabilities**
7. GitHub Release creation with assets: SBOM (`bom.json`), Dependency-Track findings
   (`dtrack-findings.json`), Trivy report, JUnit results (`junit-test-results.zip`)
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
- Secrets **never** go into Git (enforced by `.gitignore`, code review and **gitleaks** secret scanning in CI on every push/PR)
- CI workflows do not log secrets or tokens to output
- All deployments run with hardened `securityContext` (runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities)
- All deployments define startup/readiness/liveness probes against Spring Actuator
- **Dependabot** keeps Maven, GitHub Actions, Docker and Terraform dependencies updated (weekly PRs)
- **Branch protection** on `main` in all repos (merge via PR only, no force-push) — one-time setup via `ci-cd-templates/scripts/setup-branch-protection.sh`; **CODEOWNERS** in every repo

### 6.2 Secret Storage Locations

| Type | Storage | Purpose |
|------|---------|---------|
| CI/CD tokens | GitHub Secrets | Pipeline authentication (Azure, SonarQube, DTrack) |
| Runtime secrets | Azure Key Vault | GitHub App keys, application credentials |
| K8s delivery | External Secrets Operator | Pulls from Key Vault → creates K8s Secrets |

### 6.3 GitHub Secrets (CI/CD)
- `AZURE_CREDENTIALS`, `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`
- `SONAR_TOKEN`, `SONAR_HOST_URL`
- `DEPENDENCY_TRACK_API_KEY`, `DTRACK_API_URL`
- `ACR_NAME`
- `ENV_REPOS_TOKEN`

### 6.4 Azure Key Vault (Runtime)
- GitHub App credentials (private key, app ID, installation ID)
- Delivered to AKS via External Secrets Operator
- AKS access through Managed Identity (no passwords in code)
- Setup guide: `infra-azure/docs/key-vault-external-secrets-setup.md`

---

## 7. Monitoring and Alerting

### 7.1 Monitoring Stack
- In test cluster: Prometheus, Alertmanager, Grafana (separate Helm charts, SLIM profile)
- Installed via ArgoCD app-of-apps (`platform-apps/charts/app-of-apps/values.yaml`)

### 7.2 What We Monitor
- Spring Boot metrics (Actuator), HTTP requests, 4xx/5xx errors
- Grafana: dashboards auto-provisioned from grafana.com — **JVM Micrometer (ID 4701)**
  and **Spring Boot Statistics (ID 11378)**, Prometheus datasource preconfigured

**Grafana access** (namespace `monitoring`, service type LoadBalancer):
```bash
kubectl get svc grafana -n monitoring        # external IP
# login: admin, password:
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

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
| IaC | Terraform | Infrastructure as code | 1.9.x |
| IaC | AzureRM Provider | Terraform provider | ~> 3.0 |
| IaC Security | Checkov | Terraform static security scan (CI) | latest (action v12) |
| CI/CD | GitHub Actions | CI/CD pipeline | - |
| CI/CD | ci-cd-templates | Centralized workflow templates | - |
| CI/CD | Dependabot | Automated dependency update PRs | - |
| App Runtime | Java 21 | Application runtime | 21 |
| Build Tool | Maven | Application build | 3.9+ |
| Framework | Spring Boot | Application backend | 3.4.3 |
| Code Quality | SonarQube | Static analysis | chart 2025.5.0 |
| Security | Dependency-Track | Dependency analysis (SBOM) | chart 0.40.0 |
| Security | Trivy | Image scanning + CRITICAL gate | action 0.28.0 |
| Security | gitleaks | Secret scanning in CI (full git history) | 8.21.2 |
| Observability | Prometheus | Metrics | chart 25.30.0 |
| Observability | Alertmanager | Alert routing (email on HTTP 500) | chart 1.13.1 |
| Observability | Grafana | Dashboards (JVM, Spring Boot) | chart 8.8.2 |
| Deployment | Argo Rollouts | Canary deployments on PROD | chart 2.38.0 |
| K8s Tools | kubectl | CLI | - |
| K8s Tools | Helm | Chart management | 3.x |
| Secrets | Azure Key Vault | Centralized secret storage | - |
| Secrets | External Secrets Operator | K8s secret delivery from Key Vault | 0.12.1 |

---

## 9. Operational Procedures

### 9.1 New Environment
1. Add folder in `infra-azure/envs`
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
