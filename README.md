# Cloud Native CI/CD Pipeline

Abschlussprojekt mit Spring-Backend, JavaScript-Frontend, Docker und Kubernetes.

## Aktive CI/CD-Pipeline

Die produktiv genutzte Pipeline ist:

- `.github/workflows/ci-cd.yaml`

Detaillierte Ablauf- und Secret-Doku:

- `.github/workflows/README.md`

## Kurzüberblick Workflow

1. `build` (Maven + npm + Tests)
2. `push-docker` (Docker Build/Push für Backend und Frontend)
3. `deploy-dev` (Deploy nach `jan-dev`)
4. `manual-approval` (Freigabe-Gate für Produktion)
5. `deploy-prod` (Deploy nach `jan-prod`)

## Benötigte GitHub Secrets

- `DOCKERHUB_USERNAME`
- `DOCKER_PASS`
- `KUBE_CONFIG`

## Projektstruktur

```text
.
├── .github/workflows/
│   ├── ci-cd.yaml
│   └── README.md
├── backend/
├── frontend/
├── k8s/
│   ├── dev/
│   └── prod/
└── README.md
```

## Umgebungen und Kubernetes

- Dev Namespace: `jan-dev`
- Prod Namespace: `jan-prod`
- Manifeste:
  - `k8s/dev/*`
  - `k8s/prod/*`

## Spring-Profile (Backend)

- `backend/src/main/resources/application-dev.properties`
  - `server.port=8081`
- `backend/src/main/resources/application-prod.properties`
  - `logging.level.root=ERROR`
- Maven-Profile in `backend/pom.xml`:
  - `dev`
  - `prod`

## CI/CD Dokumentation (GitHub Actions)

Dieser Teil beschreibt den aktuell laufenden Workflow aus `.github/workflows/ci-cd.yaml`.

### Ziel

Der Workflow baut Backend und Frontend, pushed beide Docker Images nach Docker Hub und deployed danach nach Kubernetes:

- `jan-dev` (automatisch)
- `jan-prod` (nach manueller Freigabe)

### Trigger

Der Workflow startet bei:

- `push` auf Branch `main`

### Job-Reihenfolge

1. `build`
2. `push-docker` (needs: `build`)
3. `deploy-dev` (needs: `push-docker`)
4. `manual-approval` (needs: `deploy-dev`, Environment `prod`)
5. `deploy-prod` (needs: `manual-approval`)

### Technischer Ablauf im Detail

#### 1) Build

- `actions/checkout@v4`
- Java 17 Setup (`actions/setup-java@v4`)
- Node 20 Setup (`actions/setup-node@v4`)
- Backend Build: `mvn clean package` in `backend/`
- JAR-Artefakt Upload (`backend/target/*.jar`)
- Frontend: `npm install`, `npm run build`, `npm test` in `frontend/`

#### 2) Docker Build + Push

- Checkout
- Download des Backend-Artefakts
- Docker Login via `docker/login-action@v3`
- Build und Push:
  - `minicube78/kukuk-backend:latest`
  - `minicube78/kukuk-frontend:latest`

#### 3) Deploy Dev

- kubectl Setup (`azure/setup-kubectl@v4`)
- Kubeconfig aus Secret schreiben (`KUBE_CONFIG`, Base64)
- Deployment:
  - `kubectl apply -n jan-dev -f k8s/dev`

#### 4) Manual Approval

- Job läuft im Environment `prod`
- Freigabe erfolgt über GitHub Environment Protection Rules

#### 5) Deploy Prod

- kubectl Setup
- Kubeconfig aus Secret
- Deployment:
  - `kubectl apply -n jan-prod -f k8s/prod`

### Benötigte Secrets (GitHub)

Diese Secrets müssen im Repository gesetzt sein (`Settings > Secrets and variables > Actions`):

1. `DOCKERHUB_USERNAME`
2. `DOCKER_PASS`
3. `KUBE_CONFIG`

### Hinweise zu Docker-Hub-Credentials

- Nutze für `DOCKER_PASS` bevorzugt ein Docker Hub Access Token (nicht das Account-Passwort).
- Das Token muss Push-Rechte für die Ziel-Repositories haben.
- Der Username muss exakt zum Namespace der Image-Tags passen (hier `minicube78`).

### Hinweise zu GitLab-Credentials

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

### Environment / Approval Einstellungen

Für den manuellen Prod-Gate ist im GitHub-Repo ein Environment `prod` konfiguriert:

- Required Reviewers
- (optional) Wait Timer

Ohne diese Regeln würde der `manual-approval`-Job einfach so laufen.

### Troubleshooting

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

### Optional: Jenkins

Jenkins wurde für CI im Schulungskontext genutzt, GitHub Actions für eine vollständige CI/CD-Pipeline inklusive Docker und Kubernetes, da dort ein Docker-Daemon verfügbar ist.

## Typische Stolpersteine & Learnings

Im Verlauf des Projekts sind mehrere Probleme aufgetreten, die typisch für reale CI/CD-Umgebungen sind.  
Die folgenden Punkte dokumentieren diese Stolpersteine mit Ursache, Lösung und Learning.

### 1) Java- und Maven-Versionen nicht konsistent

**Problem:** `java -version` zeigte Java 21, Maven-Builds schlugen aber mit `release version 17 not supported` fehl; teilweise war `javac` nicht verfügbar.

**Ursache:** Runtime, Compiler und Maven liefen nicht mit derselben Toolchain; `JAVA_HOME` war nicht sauber gesetzt; teils war nur ein JRE statt JDK vorhanden.

**Lösung:** `openjdk-17-jdk` installieren, `JAVA_HOME` auf Java 17 setzen, dann prüfen, dass `java`, `javac` und `mvn` konsistent auf Java 17 zeigen.

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

**Learning:** CI/CD scheitert häufig an inkonsistenten Toolchains, nicht am Anwendungscode.

### 2) Spring Boot JAR nicht startbar (`no main manifest attribute`)

**Problem:** Start der gebauten JAR schlug mit `no main manifest attribute` fehl.

**Ursache:** Spring Boot Maven Plugin war falsch konfiguriert bzw. wurde als Dependency statt als Build-Plugin behandelt.

**Lösung:** `spring-boot-maven-plugin` korrekt unter `build/plugins` verwenden; Version durch den Parent verwalten lassen.

**Learning:** Maven trennt strikt zwischen Runtime-Dependencies und Build-Plugins.

### 3) Spring Profiles falsch eingeordnet (Build vs. Runtime)

**Problem:** Unklarheit, ob `spring.profiles.active` in Maven/Jenkins oder zur Laufzeit gesetzt werden soll.

**Ursache:** Build-Logik und Laufzeit-Konfiguration wurden vermischt.

**Lösung:** Profile als Laufzeitkonfiguration behandeln: lokal per Startparameter, in Kubernetes per Umgebungsvariable im Deployment.

**Learning:** Spring-Profile gehören zur Runtime-Umgebung, nicht in den CI-Build.

### 4) Jenkins findet kein `pom.xml`

**Problem:** Jenkins-Fehler `There is no POM in this directory`.

**Ursache:** Jenkins startet im Repository-Root; `pom.xml` lag im Unterordner `backend`.

**Lösung:** Build explizit im korrekten Verzeichnis ausführen:

```groovy
dir('backend') {
  sh 'mvn clean package'
}
```

**Learning:** Der Speicherort des Jenkinsfiles ist nicht automatisch das Arbeitsverzeichnis.

### 5) `mvn: not found` im Jenkins-Build

**Problem:** Maven wurde im Agent nicht gefunden.

**Ursache:** Maven war nicht im PATH des Jenkins-Agents.

**Lösung:** Jenkins-Tooling im Pipeline-`tools`-Block definieren.

```groovy
tools {
  maven 'Maven'
}
```

**Learning:** Jenkinsfiles sollten keine globalen Tool-Installationen voraussetzen.

### 6) Jenkinsfile am falschen Ort im Repository

**Problem:** Jenkinsfile lag in `backend/`; dadurch gab es Probleme beim Zugriff auf Frontend-Dateien.

**Ursache:** Das Jenkinsfile steuert immer den Build für das gesamte Repository.

**Lösung:** Jenkinsfile ins Repo-Root legen und Teilprojekte per `dir()` adressieren.

**Learning:** In Monorepos gehört das Jenkinsfile in der Regel ins Root-Verzeichnis.

### 7) Docker Build schlägt in Jenkins fehl

**Problem:** `error during connect: docker daemon not available`.

**Ursache:** Schul-Jenkins stellte keinen Docker-Daemon bereit; kein Zugriff auf `/var/run/docker.sock`; keine Admin-Rechte.

**Lösung:** Docker Build/Push außerhalb dieser Jenkins-Umgebung ausführen oder auf GitHub Actions ausweichen; Einschränkung dokumentieren.

**Learning:** Ein Jenkinsfile kann fachlich korrekt sein, auch wenn die bereitgestellte Infrastruktur es nicht vollständig ausführen kann.

### 8) Docker Build im Container falsch gedacht

**Problem:** Versuch, Maven innerhalb des Runtime-Dockerfiles auszuführen.

**Ursache:** CI-Build und Runtime-Image-Aufbau wurden vermischt.

**Lösung:** Artefakt zuerst in CI bauen (Maven), danach im Dockerfile nur verpacken.

**Learning:** Maven erstellt Artefakte, Docker paketiert sie für die Ausführung.

### 9) Kubernetes Service findet keine Pods

**Problem:** Service funktionierte nicht bzw. routete nicht auf Pods.

**Ursache:** Service und Deployment lagen in unterschiedlichen Namespaces oder Namespace war nicht sauber gesetzt.

**Lösung:** Deployment und Service im selben Namespace deployen und Labels/Selector konsistent halten.

**Learning:** Kubernetes-Objekte arbeiten namespace-lokal.

### 10) Frontend-Build läuft „zu schnell“

**Problem:** Frontend-Build dauerte nur wenige Sekunden.

**Ursache:** `npm run build` enthielt nur ein `echo` statt eines echten Build-Kommandos.

**Lösung:** Build-Scriptinhalt prüfen und echte Build-Tools/Kommandos hinterlegen.

**Learning:** CI führt exakt das aus, was im Script definiert ist.

### 11) Jenkins vs. GitHub Actions

**Problem:** Docker funktionierte in GitHub Actions, aber nicht im Schul-Jenkins.

**Ursache:** GitHub-Runner haben standardmäßig Docker-Unterstützung; der Schul-Jenkins-Agent nicht.

**Lösung:** Jenkins für Grundlagen nutzen, vollständige Build/Push/Deploy-Kette über GitHub Actions abbilden.

**Learning:** CI/CD-Design hängt immer von den Plattform-Restriktionen ab.

### 12) Gesamt-Learning

Die meisten CI/CD-Probleme entstehen nicht durch falschen Code, sondern durch falsche Annahmen über Umgebung, Tooling und Verantwortlichkeiten.
