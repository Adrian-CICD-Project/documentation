# Developer Guide

## 1. Introduction

This document describes how developers can:

- Work with the `devops-project` application (Spring Boot)
- Run the application locally
- Create and test changes
- Trigger the CI pipeline
- Understand the deployment process to Kubernetes environments (`environment-dev`, `environment-test`, `environment-prod`)

Developers **do not need to know Kubernetes or Azure**.  
The entire deployment process is automated through GitHub Actions + ArgoCD.

---

## 2. Repository Architecture

Main repositories in the organization:

| Repository | Purpose |
|------------|---------|
| **devops-project** | Application code, tests, CI pipeline trigger |
| **ci-cd-templates** | Centralized reusable CI/CD workflow templates |
| **platform-apps** | Helm charts for SonarQube, Dependency-Track, Prometheus, Grafana |
| **infrastructure-env-dev / test / prod** | GitOps repositories for each environment |
| **infrastructure** | Terraform for infrastructure (AKS, ACR, network, ArgoCD) |
| **Documentation** | Technical documentation (what you're reading now) |

As a developer, you primarily work with the **devops-project** repository.

> **Note:** CI/CD pipeline logic is maintained centrally in `ci-cd-templates`. Application repositories only contain trigger configuration and variables.

---

## 3. Local Requirements

To run the project locally, you need:

- Java 21 (e.g., Temurin JDK)
- Maven 3.9+
- Git
- Docker (optional)
- IntelliJ IDEA / VS Code

### Cloning the Repository

```bash
git clone https://github.com/<org>/devops-project.git
cd devops-project
```

---

## 4. Project Structure

```
devops-project
├── src/main/java           # application logic
├── src/test/java           # unit tests
├── pom.xml                 # Maven configuration, SBOM, Sonar
├── charts/java-app-template
└── .github/workflows       # CI pipeline trigger (calls ci-cd-templates)
```

### Required Maven Configuration

To ensure the CI pipeline functions correctly (SonarQube analysis, SBOM generation), your `pom.xml` must include the following plugins:

#### 1. JaCoCo Plugin (Required for SonarQube Coverage)

SonarQube requires JaCoCo reports to show code coverage.

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

#### 2. CycloneDX Plugin (Required for SBOM/Trivy)

The pipeline generates a Software Bill of Materials (SBOM) to track dependencies and vulnerabilities.

```xml
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
    <version>2.7.11</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>makeAggregateBom</goal>
            </goals>
        </execution>
    </executions>
</plugin>


### Main Endpoints

- `GET /api/hello` - returns "Hello World"
- `GET /api/error500` - returns HTTP 500 error (for alert testing)
- `GET /api/error400` - returns HTTP 400 error

---

## 5. Running the Application Locally

### 5.1 Starting the Application

```bash
mvn spring-boot:run
```

Application runs at: **http://localhost:8080**

### Testing Endpoints

```bash
curl http://localhost:8080/api/hello
curl http://localhost:8080/api/error400
curl http://localhost:8080/api/error500
```

---

## 6. Tests and Coverage

Running tests:

```bash
mvn clean test
```

**Minimum code coverage: 80%**

The CI pipeline will reject commits that do not meet this criterion.

---

## 7. CI/CD Flow for Developers

After pushing changes to GitHub and creating a Pull Request, the pipeline executes:

1. Build the application
2. Run unit tests + generate reports
3. SonarQube analysis (all branches)
4. Generate SBOM and send to Dependency-Track
5. Build Docker image + push to Azure Container Registry (ACR)
6. Scan image for vulnerabilities
7. Create GitHub Release (with reports)
8. Update `environment-dev` repository with new image tag

**ArgoCD detects the change and deploys to AKS automatically.**

> You do not deploy anything manually.

> **Note:** All pipeline logic is defined in [ci-cd-templates](https://github.com/Adrian-CICD-Project/ci-cd-templates). The workflow file in `devops-project` only triggers these centralized templates.

---

## 8. Development Process

### 8.1 Making Changes

1. Create a new branch:

```bash
git checkout -b feature/change-name
```

2. Make your code changes

3. Test locally:

```bash
mvn clean test
```

4. Push changes:

```bash
git push origin feature/change-name
```

5. Open a Pull Request to `main`

### 8.2 Deployment to environment-dev

After merging to `main`, the following happens automatically:

- Pipeline builds a new Docker image
- Updates values in `infrastructure-env-dev` repository
- ArgoCD deploys the new version to the dev environment

> Deployment is fully automatic - no manual intervention required.

### 8.3 Promotion to test and prod

Promotion occurs exclusively through merging Pull Requests between GitOps repositories:

```
environment-dev -> environment-test -> environment-prod
```

**Process:**

1. Merge PR in `infrastructure-env-dev` triggers workflow that creates PR to `infrastructure-env-test`
2. ArgoCD automatically deploys to test environment
3. After verification: Merge PR in `infrastructure-env-test` creates PR to `infrastructure-env-prod`
4. ArgoCD automatically deploys to production environment

> The only manual step is clicking **Merge** in GitHub.

> **Note:** Promotion workflow logic is defined in [ci-cd-templates/promote-environment.yml](https://github.com/Adrian-CICD-Project/ci-cd-templates/blob/main/.github/workflows/promote-environment.yml)

---

## 9. Configuration and Secrets

### 9.1 Application Configuration

Application configuration is stored in:

- **Helm Values files** in environment repositories (`infrastructure-env-*`)
- **ConfigMap** in Kubernetes (created automatically from Helm Values)

Modifying configuration is done by:

1. Editing `values.yaml` files in the appropriate environment repository
2. Commit and push changes
3. ArgoCD automatically synchronizes changes to Kubernetes

### 9.2 Secrets

> **Important:** Secrets are **never** stored in Git repositories.

**Where secrets are stored:**

- **GitHub Secrets** — for CI/CD pipeline (tokens, passwords, API keys)
- **Azure Key Vault** — for runtime secrets (GitHub App keys, credentials), delivered to clusters via External Secrets Operator
- **Kubernetes Secrets** — created automatically by External Secrets Operator from Key Vault data

**Usage in application:**

The application reads environment variables through Spring Boot. Example configuration:

```properties
my.service.token=${MY_SERVICE_TOKEN}
my.service.url=${MY_SERVICE_URL}
```

Values are injected through Kubernetes Secrets as environment variables.

---

## 10. Troubleshooting

### 10.1 CI/CD Pipeline Error

If the CI/CD pipeline fails, check in order:

1. **Maven logs** - whether compilation was successful
2. **Unit tests** - whether all tests passed (min. 80% coverage)
3. **SonarQube** - whether there are code quality errors
4. **SBOM and Dependency-Track** - whether there are critical vulnerabilities in dependencies
5. **Docker image scan** - whether the image was correctly built and scanned

**Where to check:**

- GitHub Actions logs in the "Actions" tab of the repository
- SonarQube dashboard (link in PR comment)
- Dependency-Track dashboard

### 10.2 Application Error on Environment

If the application doesn't work correctly on an environment (dev/test/prod), verify:

1. **Container logs** - whether the application started without errors
2. **PR status** - whether the Pull Request was correctly merged to `main`
3. **ArgoCD status** - whether ArgoCD synchronized changes (check in ArgoCD UI)
4. **Configuration** - whether Helm Values are correct
5. **Secrets** - whether all required secrets are available in Kubernetes
6. **Pod health** - whether the pod is running (`kubectl get pods`, `kubectl describe pod`)

**Access to logs:**

- Through `kubectl logs <pod-name>` (if you have cluster access)
- Through ArgoCD UI (in the application logs tab)

---

## 11. Tools Used by Developers

### Local Tools

- **Java 21** - JDK (recommended: Temurin JDK)
- **Maven 3.9+** - build tool and dependency management
- **Spring Boot** - application framework
- **Git** - version control
- **Docker** (optional) - for local container testing
- **IntelliJ IDEA / VS Code** - code editors

### Cloud / CI/CD Tools

- **GitHub** - code repository and CI/CD (GitHub Actions)
- **SonarQube** - code quality analysis
- **Dependency-Track** - dependency vulnerability management (SBOM)
- **Azure Container Registry (ACR)** - Docker image registry
- **ArgoCD** - automatic deployment to Kubernetes (GitOps)
- **Kubernetes (AKS)** - runtime platform (Azure Kubernetes Service)

> **Full list of tools with versions and detailed configuration can be found in `DevOps-Guide.md`**.
