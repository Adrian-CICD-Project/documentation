# Rozszerzenia platformy (czerwiec 2026)

Dokument opisuje usprawnienia dodane do platformy po zakończeniu remediacji
bezpieczeństwa. Wszystkie zmiany są zapisane w kodzie (IaC / GitOps / CI),
więc odtwarzają się automatycznie przy każdym wdrożeniu środowiska od zera
(`infra-azure/scripts/deploy.sh` + `platform-apps/scripts/deploy-platform.sh`).

---

## 1. Pipeline CI (`ci-cd-templates/java-ci-full.yml`)

| Zmiana | Opis |
|---|---|
| Wyniki testów w Release | Raporty JUnit (Surefire) są pakowane do `junit-test-results.zip` i dołączane jako asset do GitHub Release |
| Raport Dependency-Track w Release | Po przetworzeniu SBOM pipeline pobiera raport podatności (`/api/v1/finding/project/{uuid}/export`) i dołącza `dtrack-findings.json` do Release |
| Bramka Trivy | Pipeline **pada**, jeśli obraz zawiera podatności CRITICAL z dostępną poprawką (pełny raport nadal obejmuje wszystkie poziomy) |
| Push obrazów tylko z `main` | Buildy z feature branchy budują obraz (walidacja Dockerfile), ale nie pushują do ACR — dzięki temu każdy obraz w ACR przechodzi skan Trivy |

## 2. Skanowanie sekretów (gitleaks)

- Reusable workflow: `ci-cd-templates/.github/workflows/gitleaks.yml`
  (skan pełnej historii gita, z `--redact` — sekrety nie trafiają do logów CI).
- Podpięty w repozytoriach: `adrian-java-app`, `ci-cd-templates`,
  `platform-apps`, `infra-azure`, `infra-aws`, `infra-gcp`,
  `infrastructure-env-{dev,test,prod}`.

## 3. CI dla Terraform

- Reusable workflow: `ci-cd-templates/.github/workflows/terraform-ci.yml`:
  `terraform fmt -check` → `init -backend=false` → `validate` + skan Checkov
  (informacyjny, `soft_fail`).
- Podpięty w `infra-azure`, `infra-aws`, `infra-gcp`.
- Przy okazji naprawiono błąd walidacji: timezone w module `auto-shutdown`
  zmieniono z `Central European Standard Time` na `Europe/Warsaw` (format IANA
  wymagany przez provider azurerm).

## 4. Dependabot

- `adrian-java-app`: aktualizacje Maven, GitHub Actions i obrazów Docker (co tydzień).
- `infra-azure`: aktualizacje providerów Terraform i GitHub Actions.

## 5. Kubernetes — niezawodność aplikacji

- **Sondy** (wszystkie środowiska, obie aplikacje):
  - `startupProbe` → `/actuator/health`
  - `readinessProbe` → `/actuator/health/readiness`
  - `livenessProbe` → `/actuator/health/liveness`
  - W `application.properties` włączono `management.endpoint.health.probes.enabled=true`.
- **PROD `adrian-java-app`**:
  - uzupełniono brakujący `service.yaml` (ClusterIP 80→8080),
  - `hpa.yaml` — HorizontalPodAutoscaler (min 2, max 4 pody, CPU 80%),
  - `pdb.yaml` — PodDisruptionBudget (`minAvailable: 1`).

## 6. Canary deployment na PROD (Argo Rollouts)

- Kontroler `argo-rollouts` instalowany przez app-of-apps na klastrze PROD
  (`platform-apps/charts/app-of-apps/values-prod.yaml`).
- `infrastructure-env-prod/k8s/adrian-java-app/deployment.yaml` zawiera teraz
  `kind: Rollout` ze strategią canary: **50% ruchu → pauza 60 s → pełny rollout**.
- Plik celowo zachował nazwę `deployment.yaml` — workflow promocji podmienia
  w nim linię `image:` po ścieżce pliku.
- Obserwacja rolloutu:
  ```bash
  kubectl argo rollouts get rollout adrian-java-app -n environment-prod --watch
  ```
- Kolejność przy świeżym wdrożeniu: app-of-apps (instaluje CRD Rollout) musi
  zsynchronizować się przed aplikacją — ArgoCD ponawia sync automatycznie,
  więc nie wymaga to ręcznej interwencji.

## 7. Grafana (klaster TEST, namespace `monitoring`)

- Instalowana przez app-of-apps (`platform-apps/charts/app-of-apps/values.yaml`).
- Datasource Prometheus skonfigurowany automatycznie
  (`http://prometheus-server.monitoring.svc.cluster.local`).
- Dashboardy pobierane automatycznie z grafana.com:
  - **JVM (Micrometer)** — ID 4701,
  - **Spring Boot Statistics** — ID 11378.
- Dostęp (service typu LoadBalancer):
  ```bash
  kubectl get svc grafana -n monitoring        # zewnętrzny adres IP
  # login: admin, hasło:
  kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
  ```

## 8. Governance (ustawienia GitHub — przeżywają zaoranie infry)

- **CODEOWNERS** (`* @AdrDmy`) we wszystkich repozytoriach.
- **Branch protection**: jednorazowy skrypt
  `ci-cd-templates/scripts/setup-branch-protection.sh` (gh CLI) — wymusza
  merge przez PR, blokuje force-push na `main`. Uruchomić raz:
  ```bash
  cd ci-cd-templates && ./scripts/setup-branch-protection.sh
  ```

---

## Działania jednorazowe po wypchnięciu zmian

1. Uruchomić `setup-branch-protection.sh` (raz, ustawienia org są trwałe).
2. Dependabot i gitleaks aktywują się same po pushu plików do repo.
3. Reszta (Grafana, Argo Rollouts, sondy, HPA/PDB, canary) wdroży się
   automatycznie przez ArgoCD przy najbliższym deploymencie platformy.
