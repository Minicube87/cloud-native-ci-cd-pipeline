# Cloud Native CI/CD Pipeline

Abschlussprojekt zur Bereitstellung einer vollständigen CI/CD-Pipeline für eine Microservice-Anwendung mit Spring-Backend und JavaScript-Frontend.

## Projektübersicht

Dieses Repository enthält:

- `backend/`: Java-Anwendung (Maven, getrennte Spring-Profile für `dev` und `prod`)
- `frontend/`: JavaScript-Frontend (npm-basierter Build/Test)
- `k8s/`: Kubernetes-Manifeste für die Umgebungen `dev` und `prod` (technisch in den Namespaces `jan-dev` und `jan-prod`)
- `jenkins/`: Pipeline-Definition für Build, Test, Docker und Deployments
- `docs/`: Projektdokumentation und Anforderungs-Check

## Projektstruktur

```text
.
├── Jenkinsfile
├── jenkins/
│   └── Jenkinsfile
├── backend/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── k8s/
│   ├── namespaces.yaml
│   ├── dev/
│   │   ├── backend-deployment.yaml
│   │   ├── backend-service.yaml
│   │   ├── frontend-deployment.yaml
│   │   └── frontend-service.yaml
│   └── prod/
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── frontend-deployment.yaml
│       └── frontend-service.yaml
└── docs/
    └── requirements-check.md
```

## Spring-Konfiguration und Maven-Profile

Im Backend sind Umgebungen über Profile getrennt:

- `backend/src/main/resources/application-dev.properties`
  - `server.port=8081`
- `backend/src/main/resources/application-prod.properties`
  - `logging.level.root=ERROR`

Die Profile sind in `backend/pom.xml` definiert:

- Profil `dev` setzt `spring.profiles.active=dev`
- Profil `prod` setzt `spring.profiles.active=prod`

Beispiele:

```bash
cd backend
mvn -Pdev clean package
mvn -Pprod clean package
```

## CI/CD-Prozess (Jenkins)

Die Pipeline in `Jenkinsfile` umfasst:

1. `Checkout`: Code aus SCM laden.
2. `Build`: Backend mit Maven-Profil bauen, Frontend mit npm bauen.
3. `Test`: Backend-JUnit-Tests und Frontend-Tests ausführen.
4. `Docker Build`: Backend- und Frontend-Image bauen.
5. `Docker Push`: Images in Registry pushen.
6. `Deploy Dev`: Deployment in Namespace `jan-dev`.
7. `Manual Approval`: Manuelle Freigabe für Produktion.
8. `Deploy Prod`: Deployment in Namespace `jan-prod`.

## Kubernetes-Deployment

Verwendete Cluster-Namespaces:

- `jan-dev` (Umgebung: dev)
- `jan-prod` (Umgebung: prod)

Manifeste:

- Deployments und Services für Backend + Frontend in `k8s/dev` und `k8s/prod`
- Namespace-Definitionen in `k8s/namespaces.yaml`
- `SPRING_PROFILES_ACTIVE` wird im Backend-Deployment als Umgebungsvariable gesetzt (`dev`/`prod`)

Beispiel für manuelles Deployment:

```bash
kubectl apply -f k8s/dev/
kubectl apply -f k8s/prod/
```

Hinweis: `jan-dev` und `jan-prod` sind in der bereitgestellten Infrastruktur bereits vorhanden.  
`k8s/namespaces.yaml` ist nur fuer Cluster-Admins relevant.

## Jenkins-Voraussetzungen

Folgende Jenkins-Tools/Credentials werden erwartet:

- Tools:
  - Maven-Installation namens `Maven`
  - NodeJS-Installation namens `Node`
- Credentials:
  - `registry-creds` (Username/Password für Container Registry)
  - `kubeconfig` (Secret File mit Kubeconfig)

## Lokale Verifikation

Backend:

```bash
cd backend
mvn test
```

Frontend:

```bash
cd frontend
npm install
npm test -- --runInBand
```
