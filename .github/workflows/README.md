# CI/CD Dokumentation (GitHub Actions)

Diese Doku beschreibt den aktuell laufenden Workflow aus `.github/workflows/ci-cd.yaml`.

## Ziel

Der Workflow baut Backend und Frontend, pushed beide Docker Images nach Docker Hub und deployed danach nach Kubernetes:

- `jan-dev` (automatisch)
- `jan-prod` (nach manueller Freigabe)

## Trigger

Der Workflow startet bei:

- `push` auf Branch `main`

## Job-Reihenfolge

1. `build`
2. `push-docker` (needs: `build`)
3. `deploy-dev` (needs: `push-docker`)
4. `manual-approval` (needs: `deploy-dev`, Environment `prod`)
5. `deploy-prod` (needs: `manual-approval`)

## Technischer Ablauf im Detail

### 1) Build

- `actions/checkout@v4`
- Java 17 Setup (`actions/setup-java@v4`)
- Node 20 Setup (`actions/setup-node@v4`)
- Backend Build: `mvn clean package` in `backend/`
- JAR-Artefakt Upload (`backend/target/*.jar`)
- Frontend: `npm install`, `npm run build`, `npm test` in `frontend/`

### 2) Docker Build + Push

- Checkout
- Download des Backend-Artefakts
- Docker Login via `docker/login-action@v3`
- Build und Push:
  - `minicube78/kukuk-backend:latest`
  - `minicube78/kukuk-frontend:latest`

### 3) Deploy Dev

- kubectl Setup (`azure/setup-kubectl@v4`)
- Kubeconfig aus Secret schreiben (`KUBE_CONFIG`, Base64)
- Deployment:
  - `kubectl apply -n jan-dev -f k8s/dev`

### 4) Manual Approval

- Job läuft im Environment `prod`
- Freigabe erfolgt über GitHub Environment Protection Rules

### 5) Deploy Prod

- kubectl Setup
- Kubeconfig aus Secret
- Deployment:
  - `kubectl apply -n jan-prod -f k8s/prod`

## Benötigte Secrets (GitHub)

Diese Secrets müssen im Repository gesetzt sein (`Settings > Secrets and variables > Actions`):

1. `DOCKERHUB_USERNAME`
2. `DOCKER_PASS`
3. `KUBE_CONFIG`

## Hinweise zu Docker-Hub-Credentials

- Nutze für `DOCKER_PASS` bevorzugt ein Docker Hub Access Token (nicht das Account-Passwort).
- Das Token muss Push-Rechte für die Ziel-Repositories haben.
- Der Username muss exakt zum Namespace der Image-Tags passen (hier `minicube78`).

## Hinweise zu GitLab-Credentials

Der aktuelle Workflow nutzt direkt **GitHub Actions** und **Docker Hub**.  
GitLab-Credentials werden in dieser `ci-cd.yaml` nicht direkt verwendet.

Falls mit GitLab gearbeitet wird (z. B. Mirror, Registry oder späterer Umzug), sind das die üblichen Entsprechungen:

1. GitLab CI/CD Variables (masked + protected):
   - `DOCKERHUB_USERNAME`
   - `DOCKER_PASS`
   - `KUBE_CONFIG`
2. Für GitLab Container Registry statt Docker Hub:
   - Login-Registry: `registry.gitlab.com`
   - Credentials typischerweise als Deploy Token oder Personal Access Token
   - Image-Namen dann auf `registry.gitlab.com/<group>/<project>/<image>:<tag>` umstellen

## Environment / Approval Einstellungen

Für den manuellen Prod-Gate sollte im GitHub-Repo ein Environment `prod` konfiguriert sein:

- optionale Required Reviewers
- optionaler Wait Timer

Ohne diese Regeln läuft der `manual-approval`-Job zwar, aber ohne echten Freigabe-Gate.

## Troubleshooting

1. Docker Login Fehler:
   - `DOCKERHUB_USERNAME`/`DOCKER_PASS` prüfen
   - Token nicht abgelaufen?
2. Kubernetes Auth Fehler:
   - `KUBE_CONFIG` korrekt hinterlegt (Base64 oder Plaintext)
   - Namespace-Rechte für `jan-dev`/`jan-prod` vorhanden
3. `npm test` Fehler:
   - Test-Dependencies in `frontend/package.json` prüfen
4. `kubectl apply` Fehler:
   - Manifeste in `k8s/dev` und `k8s/prod` auf Namespace und Resource-Namen prüfen
