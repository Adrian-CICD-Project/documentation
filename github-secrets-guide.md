# GitHub Actions Secrets Guide

This guide explains how to obtain the values for the secrets required in the CI/CD pipelines of this project.

## Azure & ACR (Azure Container Registry)

### ACR_NAME
- **Value:** The globally unique name of your Azure Container Registry.
- **Where to find:** 
  - Azure Portal -> Container Registries.
  - CLI: `az acr list --query "[].name" -o tsv`
  - Example: `acrfordevopspoc01adrian`

### ACR_LOGIN_SERVER
- **Value:** The login URL for your registry.
- **Where to find:** 
  - Usually formatted as `<ACR_NAME>.azurecr.io`.
  - CLI: `az acr show --name <ACR_NAME> --query "loginServer" -o tsv`
  - Example: `acrfordevopspoc01adrian.azurecr.io`

### AZURE_CREDENTIALS
- **Value:** A JSON object containing service principal credentials.
- **How to generate:**
  Run the following command in Azure CLI (replace placeholders):
  ```bash
  az ad sp create-for-rbac --name "github-actions-sp" --role contributor --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME> --sdk-auth
  ```
- **Note:** Copy the entire JSON output `{ "clientId": "...", "clientSecret": "...", ... }`.

---

## SonarQube

### SONAR_HOST_URL
- **Value:** The public URL of your SonarQube instance.
- **Where to find:** 
  - Get the LoadBalancer IP of the SonarQube service: `kubectl -n sonarqube get svc`
  - Format: `http://<EXTERNAL_IP>:9000`

### SONAR_TOKEN
- **Value:** Analysis token for authenticating scans.
- **How to generate:**
  1. Log in to SonarQube.
  2. Go to **My Account** (top right) -> **Security**.
  3. Under **Generate Token**, enter a name (e.g., `github-ci`) and click **Generate**.

---

## Dependency-Track

### DTRACK_API_URL
- **Value:** The public URL of your Dependency-Track API.
- **Where to find:** 
  - Get the LoadBalancer IP of the Dependency-Track API service: `kubectl -n dependency-track get svc`
  - Format: `http://<EXTERNAL_IP>:8081`

### DEPENDENCY_TRACK_API_KEY
- **Value:** API key for uploading SBOMs.
- **How to generate:**
  1. Log in to Dependency-Track.
  2. Go to **Administration** -> **Access Management** -> **Teams**.
  3. Select the **Automation** team.
  4. Copy the **API Key** or generate a new one.

---

## GitHub Tokens

### ENV_REPOS_TOKEN / GH_PAT_GITOPS
- **Value:** A Personal Access Token (PAT) used for cross-repository operations (e.g., promoting code from dev to test).
- **How to generate:**
  1. Go to GitHub **Settings** -> **Developer settings** -> **Personal access tokens** -> **Tokens (classic)**.
  2. Click **Generate new token (classic)**.
  3. Select the `repo` scope (full control of private repositories).
  4. Copy the generated token immediately.

---

## Summary Table

| Secret Name | Source | Description |
|-------------|--------|-------------|
| `ACR_NAME` | Azure | Name of the ACR |
| `ACR_LOGIN_SERVER` | Azure | `<ACR_NAME>.azurecr.io` |
| `AZURE_CREDENTIALS` | Azure | Service Principal JSON |
| `SONAR_HOST_URL` | Cluster | URL of SonarQube |
| `SONAR_TOKEN` | SonarQube | User/Project Token |
| `DTRACK_API_URL` | Cluster | URL of Dependency-Track API |
| `DEPENDENCY_TRACK_API_KEY` | DT | Team API Key |
| `ENV_REPOS_TOKEN` | GitHub | Personal Access Token (PAT) |
| `GH_PAT_GITOPS` | GitHub | Personal Access Token (PAT) |
