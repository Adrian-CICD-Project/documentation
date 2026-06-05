# Multi-Cloud Infrastructure – AWS & GCP

Dokument opisuje rozszerzenie infrastruktury projektu DevOps z **Azure** na **AWS** i **GCP**.
Celem było odwzorowanie tej samej architektury (2 klastry Kubernetes: `test` + `prod`, rejestr obrazów,
sieć, sekrety, auto-shutdown) przy **maksymalnym ograniczeniu kosztów**.

Kod IaC znajduje się w osobnych repozytoriach organizacji `Adrian-CICD-Project`:

- [`infra-azure`](https://github.com/Adrian-CICD-Project/infra-azure) – istniejąca baza (Azure)
- [`infra-aws`](https://github.com/Adrian-CICD-Project/infra-aws) – **nowe** (AWS)
- [`infra-gcp`](https://github.com/Adrian-CICD-Project/infra-gcp) – **nowe** (GCP)

---

## 1. Mapowanie usług Azure → AWS → GCP

| Rola w projekcie | Azure (baza) | AWS (`infra-aws`) | GCP (`infra-gcp`) |
|---|---|---|---|
| Sieć | VNet `10.0.0.0/16` + subnet | VPC `10.0.0.0/16` + 2× public/2× private subnet | VPC custom + subnet z secondary ranges (VPC-native) |
| Wyjście do internetu | (domyślne) | 1× NAT Gateway | Cloud NAT |
| Rejestr obrazów | ACR (Basic) | ECR + lifecycle policy | Artifact Registry + cleanup policy |
| Klaster K8s `test` | AKS `devops-poc01-test` | EKS `devops-poc01-test` | GKE `devops-poc01-test` (zonal) |
| Klaster K8s `prod` | AKS `devops-poc01-prod` | EKS `devops-poc01-prod` | GKE `devops-poc01-prod` (zonal) |
| Węzły robocze | VM `Standard_B4ms` ×1 | **EC2** `t3.large` ×1 (managed node group) | **Compute Engine** `e2-standard-2` ×1 (node pool) |
| Tożsamość dla podów | Managed Identity | IRSA (OIDC) | Workload Identity |
| Pull obrazów | rola `AcrPull` | IAM `AmazonEC2ContainerRegistryReadOnly` | rola `artifactregistry.reader` |
| Sekrety | Key Vault | Secrets Manager | Secret Manager |
| Dostarczanie sekretów | External Secrets Operator | External Secrets Operator (przez IRSA) | External Secrets Operator (przez Workload Identity) |
| Auto-shutdown 18:00 | Automation Account + Runbook | EventBridge Scheduler + Lambda | Cloud Scheduler + GKE API |
| Region | `westeurope` | `eu-west-1` (Irlandia) | `europe-west1` (Belgia) |

---

## 2. Za co odpowiada każdy komponent w naszym projekcie

### Warstwa obliczeniowa (węzły Kubernetes)

- **EC2 (AWS)** – maszyny wirtualne pełniące rolę **węzłów roboczych klastra EKS**. To na nich
  uruchamiają się wszystkie pody: aplikacja `adrian-java-app`, ArgoCD oraz narzędzia platformowe
  (SonarQube, Dependency-Track, Prometheus, Grafana). Odpowiednik VM `Standard_B4ms` z AKS.
  Typ `t3.large` (2 vCPU / 8 GB) – mała, tania instancja wystarczająca dla POC.
- **Compute Engine (GCP)** – analogicznie: **węzły robocze klastra GKE**. Typ `e2-standard-2`.
- W obu przypadkach grupa węzłów ma `min = 0`, co umożliwia **scale-to-zero** (auto-shutdown).

### Warstwa zarządzania Kubernetes (control plane)

- **EKS / GKE control plane** – zarządzany Kubernetes („master"), odpowiednik **AKS**. Odpowiada za
  API Kubernetes, scheduler, etcd. Jest rozliczany osobno (ok. 0,10 USD/h za klaster) i **nie można go
  zatrzymać** – dlatego oszczędzamy, wyłączając węzły, a nie control plane.

### Warstwa sieciowa

- **VPC (AWS/GCP)** – wirtualna sieć izolująca klaster, odpowiednik **Azure VNet**. Definiuje adresację
  IP węzłów, podów i usług.
- **Podsieci (subnets)** – AWS: 2× publiczna (LoadBalancery ArgoCD, NAT) + 2× prywatna (węzły EC2).
  GCP: jedna subnet z **secondary ranges** dla podów i usług (wymóg VPC-native GKE).
- **NAT Gateway (AWS) / Cloud NAT (GCP)** – daje węzłom w podsieci prywatnej **wyjście do internetu**
  (pobieranie obrazów z rejestru, dostęp do GitHub, Helm). W AWS to **największy stały koszt sieci**,
  dlatego użyto **tylko jednego** NAT (nie per-AZ).
- **Internet Gateway / Cloud Router** – brama internetowa VPC.

### Rejestr obrazów

- **ECR (AWS) / Artifact Registry (GCP)** – rejestr obrazów Docker, odpowiednik **ACR**. Tu pipeline CI
  wypycha zbudowane obrazy aplikacji. Skonfigurowane polityki czyszczenia (lifecycle/cleanup) usuwają
  stare obrazy, aby nie rosły koszty storage. ECR ma włączony `scan_on_push` (skan podatności – wymóg projektu).

### Tożsamość i sekrety

- **Secrets Manager (AWS) / Secret Manager (GCP)** – bezpieczne przechowywanie sekretów (klucz GitHub App,
  tokeny), odpowiednik **Azure Key Vault**. Sekrety **nie są** trzymane w Git.
- **IRSA (AWS) / Workload Identity (GCP)** – mechanizm, dzięki któremu pody (np. **External Secrets Operator**)
  pobierają sekrety, używając tożsamości chmury **bez statycznych kluczy**. Odpowiednik powiązania
  Managed Identity ↔ Key Vault na Azure.
- **External Secrets Operator (ESO)** – komponent w klastrze synchronizujący sekrety z menedżera chmury
  do `Secret` Kubernetes. Wspólny dla wszystkich trzech chmur.

### Auto-shutdown (oszczędność)

- **EventBridge Scheduler + Lambda (AWS)** – odpowiednik **Azure Automation Account + Runbook**.
  Codziennie o **18:00** EventBridge uruchamia funkcję Lambda, która skaluje grupę węzłów EKS do **0**
  (`desiredSize = 0, minSize = 0`) – wyłącza maszyny EC2.
- **Cloud Scheduler + GKE API (GCP)** – Cloud Scheduler codziennie o 18:00 wywołuje bezpośrednio API GKE
  `nodePools.setSize = 0` (bez Cloud Function – taniej). Uwierzytelnia się tokenem OAuth dedykowanego SA.

---

## 3. Koszty / FinOps – co generuje koszt i jak go ograniczamy

| Źródło kosztu | AWS | GCP | Mechanizm oszczędności |
|---|---|---|---|
| Węzły (godzinowo) | EC2 `t3.large` | Compute Engine `e2-standard-2` | **scale-to-zero 18:00**, 1 węzeł, opcja SPOT/Spot VM |
| Control plane | EKS ~73 USD/mc/klaster | GKE ~73 USD/mc (1. zonal w free tier) | GKE **zonal**, trzymanie węzłów = 0 poza godzinami pracy |
| Sieć | 1× NAT Gateway | Cloud NAT | tylko **jeden** NAT zamiast per-AZ |
| Storage obrazów | ECR | Artifact Registry | lifecycle/cleanup policy (10 ostatnich / 30 dni) |

**Dźwignie oszczędności (priorytetowo):**

1. **Auto-shutdown o 18:00** – węzły schodzą do 0 → brak opłat za EC2/Compute Engine poza godzinami pracy.
2. **Małe instancje** (`t3.large` / `e2-standard-2`), 1 węzeł na klaster.
3. **Tryb SPOT / Spot VM** – ustawienie `capacity_type = "SPOT"` (AWS) lub `use_spot = true` (GCP) daje
   dodatkowe ~70–90% oszczędności na węzłach.
4. **GKE zonal** zamiast regional – brak potrojenia węzłów; 1. klaster zonal w free tier.
5. **Jeden NAT Gateway** na AWS – eliminuje największy stały koszt sieci.
6. **Polityki czyszczenia rejestru** – ograniczają koszt storage obrazów.

> **Rekomendacja:** klastry uruchamiać tylko na czas demonstracji/testów. Po pracy zostawić węzły = 0
> (auto-shutdown robi to automatycznie). Pełny `destroy.sh` usuwa całość, gdy środowisko nie jest potrzebne.

---

## 4. Wdrożenie

Obie chmury używają tego samego wzorca co `infra-azure`:

```bash
# AWS
cd infra-aws && ./scripts/deploy.sh        # Terraform (test+prod) + ArgoCD + weryfikacja

# GCP
cd infra-gcp && export GCP_PROJECT=<id> && ./scripts/deploy.sh
```

Jeden `terraform apply` tworzy **oba** klastry (`test` + `prod`) – jak na Azure. ArgoCD instaluje skrypt
`install-argocd.sh` (te same namespace'y: `environment-dev/test/prod`, `argocd`, `sonarqube`,
`dependency-track`, `monitoring`, `external-secrets`).

Szczegóły modułów: `infra-aws/README.md` oraz `infra-gcp/README.md`.

---

## 5. Wersje narzędzi

| Narzędzie | Wersja |
|---|---|
| Terraform | >= 1.6 |
| AWS provider | ~> 5.0 |
| Google provider | ~> 5.0 |
| EKS Kubernetes | 1.30 |
| GKE Kubernetes | wersja domyślna kanału (zonal) |
| Helm | >= 3.x |
| ArgoCD | chart `argo/argo-cd` (Helm) |
