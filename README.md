# DevSecOps Spring Boot Pipeline (Jenkins · Docker · Nexus · SonarQube · OWASP · Prometheus/Grafana)

This repository contains a complete **DevSecOps** example for a **Spring Boot** application. It includes:

- **Jenkins CI/CD** pipeline (`Jenkinsfile`)
- **Containerization** with **Docker** (`Dockerfile`)
- **Artifact management** via **Nexus Repository** (conditional upload)
- **SAST** with **SonarQube** (conditional analysis)
- **SCA** with **OWASP Dependency-Check** (Maven plugin)
- **DAST** with **OWASP ZAP** against a staging URL (conditional)
- **Observability** using **Spring Boot Actuator + Micrometer Prometheus**, **Prometheus** and **Grafana**
- **Local demo stack** with `docker-compose` (app, SonarQube, Nexus, Prometheus, Grafana)

All files run out-of-the-box locally without secrets. CI steps that need credentials are **skipped automatically** unless the corresponding environment variables are present in Jenkins.

---

## 1) Quick start (Local)

### Prerequisites
- Docker & Docker Compose
- Java 17 (only if building outside Docker)

### Run everything locally
```bash
# Build the app image
docker build -t devsecops-spring-boot:local .

# Start services (app, SonarQube, Nexus, Prometheus, Grafana)
docker compose up -d

# App available
#   http://localhost:8080/                      (Hello endpoint)
#   http://localhost:8080/actuator              (Actuator)
#   http://localhost:8080/actuator/prometheus   (metrics)
# SonarQube:  http://localhost:9000  (default admin/admin; create a token in UI)
# Nexus:      http://localhost:8081
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000  (admin/admin; set new password when prompted)
```

Stop all services:
```bash
docker compose down -v
```

---

## 2) Jenkins CI/CD pipeline

The pipeline in [`Jenkinsfile`](Jenkinsfile) performs:

1. **Build, Test & SCA**: Maven build with unit tests and **OWASP Dependency-Check**
2. **SAST (SonarQube)**: Runs if `SONAR_HOST_URL` and `SONAR_TOKEN` are set in Jenkins
3. **Docker Build & Push**: Builds image always; pushes if `DOCKER_REGISTRY`, `DOCKER_USERNAME`, `DOCKER_PASSWORD` are set
4. **Upload JAR to Nexus**: Runs if `NEXUS_URL`, `NEXUS_REPO_PATH`, `NEXUS_USER`, `NEXUS_PASSWORD` are set
5. **DAST (OWASP ZAP)**: Baseline scan against `STAGING_URL` if set
6. **Archives**: JUnit, Dependency-Check, ZAP reports

### Suggested Jenkins env/credentials
- `SONAR_HOST_URL` (e.g., `http://localhost:9000`), `SONAR_TOKEN`
- `DOCKER_REGISTRY` (e.g., `ghcr.io/haytem-benkhaled`), `DOCKER_USERNAME`, `DOCKER_PASSWORD`
- `NEXUS_URL` (e.g., `http://localhost:8081`), `NEXUS_REPO_PATH` (e.g., `repository/raw-releases`), `NEXUS_USER`, `NEXUS_PASSWORD`
- `STAGING_URL` (e.g., `https://staging.example.com`)

---

## 3) SonarQube (SAST)
- Start SonarQube locally: `docker compose up -d sonarqube`
- Log in at `http://localhost:9000` (admin/admin), create a token
- Configure Jenkins environment: `SONAR_HOST_URL=http://localhost:9000` and `SONAR_TOKEN` to the generated token

---

## 4) Dependency Scanning (SCA)
- **OWASP Dependency-Check** is configured in `pom.xml` and produces `target/dependency-check-report.html` and `.xml`.

---

## 5) DAST with OWASP ZAP
- Set `STAGING_URL` in Jenkins to the staging base URL; the pipeline will run the **ZAP Baseline** Docker scan and publish `zap-report.html` and `zap-report.xml`.

---

## 6) Observability
- Prometheus metrics are exposed at `/actuator/prometheus`.
- Prometheus is pre-configured in `prometheus/prometheus.yml` to scrape the app.
- Grafana is provisioned with a Prometheus datasource.

---

## 7) Project structure

```
.
├── Jenkinsfile
├── Dockerfile
├── README.md
├── pom.xml
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml
├── src/
│   ├── main/
│   │   ├── java/com/haytem/devsecops/
│   │   │   ├── Application.java
│   │   │   └── HelloController.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/com/haytem/devsecops/
│           └── HelloControllerTest.java
└── .gitignore
```

---

## 8) Local build & tests (without Jenkins)
```bash
docker run --rm -v "$PWD":/wrk -w /wrk maven:3.9.9-eclipse-temurin-17 mvn -B clean verify
```

Run Dependency-Check locally:
```bash
docker run --rm -v "$PWD":/usr/src --entrypoint mvn maven:3.9.9-eclipse-temurin-17 -B org.owasp:dependency-check-maven:check
```

---

## 9) Notes
- Default ports: app `8080`, SonarQube `9000`, Nexus `8081`, Prometheus `9090`, Grafana `3000`
- The repository contains **no secrets**. CI uses environment variables when available.
