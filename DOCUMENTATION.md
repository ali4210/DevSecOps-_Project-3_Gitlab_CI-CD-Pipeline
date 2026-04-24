# 📘 Full Project Documentation — DevSecOps CI/CD Pipeline (Project 3)
### Al-Nafi International College | AIOps Program | Student: Saleem Ali

***

## 1. Project Identity

| Field | Detail |
|---|---|
| Project Name | DevSecOps CI/CD Pipeline — Project 3 |
| Student | Saleem Ali |
| Institution | Al-Nafi International College (Online) |
| Program | AIOps (Artificial Intelligence Operations) |
| Host OS | Kali Linux (master node) |
| GitLab Repo | Self-hosted GitLab CE |
| GitHub | https://github.com/ali4210/DevSecOps-_Project-3_Gitlab_CI-CD-Pipeline |
| LinkedIn | https://www.linkedin.com/in/saleem-ali-189719325 |
| Completion Date | April 2026 |

***

## 2. Project Objective

The goal of this project was to design, implement, and validate a **fully automated DevSecOps CI/CD pipeline** that integrates security scanning at every layer of the software delivery process — from static code analysis to live dynamic scanning — all running on **100% self-hosted, local infrastructure** with no cloud dependency.

The target application was **OWASP NodeGoat** — an intentionally vulnerable Node.js application designed for security testing and education.

***

## 3. Infrastructure Setup

### 3.1 Host Machine
- **Operating System:** Kali Linux (amd64)
- **Hostname:** master
- **User:** kali
- **Local IP:** 192.168.0.150

### 3.2 Self-Hosted Services

| Service | Port | Purpose |
|---|---|---|
| GitLab CE | 8929 | Source control + CI/CD orchestration |
| GitLab Container Registry | Internal | Docker image storage |
| SonarQube Community | 9000 | SAST / Static analysis server |
| OWASP NodeGoat App | 4000 (staging) | Deployed application for DAST |
| GitLab Runner | System service | Pipeline job executor |

### 3.3 GitLab CE Installation
GitLab CE was installed and configured on the local machine. The GitLab URL was set to `http://192.168.0.150:8929` and the external URL matched this for both web access and runner communication.

### 3.4 SonarQube Setup
SonarQube Community Edition was run as a Docker container. A project token (`SONAR_TOKEN`) was generated and stored as a GitLab CI/CD variable. The project properties file `sonar-project.properties` was placed in the repository root.

***

## 4. GitLab Runner Configuration

### 4.1 Runner Registration
The runner was registered using the following command:

```bash
sudo gitlab-runner register \
  --url http://localhost:8929 \
  --token glrt-NS3epGRG4AshCCx-hYtHcG86MQpwOjIKdDozCnU6MQ8.01.1715hnuja \
  --executor docker \
  --docker-image docker:24 \
  --docker-privileged \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
  --description DevSecOps-Runner \
  --non-interactive
```

### 4.2 Final config.toml

```toml
concurrent = 1
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "DevSecOps-Runner"
  url = "http://192.168.0.150:8929"
  clone_url = "http://192.168.0.150:8929"
  id = 2
  token = "glrt-NS3epGRG4AshCCx-hYtHcG86MQpwOjIKdDozCnU6MQ8.01.1715hnuja"
  token_obtained_at = 2026-04-23T10:36:29Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
      AssumeRoleMaxConcurrency = 0
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "node:20-alpine"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = [
      "/var/run/docker.sock:/var/run/docker.sock",
      "/cache",
      "/home/kali/dependency-check-data:/usr/share/dependency-check/data"
    ]
    extra_hosts = ["gitlab:192.168.0.150"]
    shm_size = 0
    network_mtu = 0
    pull_policy = ["if-not-present"]
```

### 4.3 Key Configuration Decisions

| Setting | Value | Reason |
|---|---|---|
| `privileged = true` | true | Required for Docker-in-Docker (building images inside CI jobs) |
| `pull_policy` | `if-not-present` | Avoids repeated Docker Hub pulls, uses local cached images |
| `extra_hosts` | `gitlab:192.168.0.150` | Allows CI containers to resolve GitLab hostname internally |
| Volume: docker.sock | `/var/run/docker.sock` | Enables Docker commands inside pipeline jobs |
| Volume: dependency-check | `/home/kali/dependency-check-data` | Persists 228MB NVD database across pipeline runs |
| `clone_url` | `http://192.168.0.150:8929` | Ensures runner clones from correct internal GitLab URL |

***

## 5. CI/CD Pipeline — Stage by Stage

### 5.1 Pipeline Overview

```yaml
stages:
  - build
  - test
  - dependency-check
  - sonarqube
  - quality-gate
  - deploy-staging
  - dast
  - deploy-production
```

### 5.2 Stage 1 — Build

**Purpose:** Install Node.js dependencies and compile the application.

```yaml
build:
  stage: build
  image: node:20-alpine
  script:
    - npm install
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour
```

- Docker image: `node:20-alpine`
- Installs all npm packages
- Passes `node_modules/` as artifact to subsequent stages

### 5.3 Stage 2 — Unit Test

**Purpose:** Run automated tests and generate coverage report.

```yaml
test:
  stage: test
  image: node:20-alpine
  script:
    - npm test
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
```

- Runs the NodeGoat test suite
- Coverage percentage extracted and displayed in GitLab pipeline UI

### 5.4 Stage 3 — OWASP Dependency-Check (SCA)

**Purpose:** Scan all npm dependencies against the NVD (National Vulnerability Database) for known CVEs.

```yaml
dependency-check:
  stage: dependency-check
  image: owasp/dependency-check:latest
  script:
    - /usr/share/dependency-check/bin/dependency-check.sh
        --scan /builds/$CI_PROJECT_PATH
        --format HTML
        --format JSON
        --out /builds/$CI_PROJECT_PATH/reports
        --noupdate
  artifacts:
    paths:
      - reports/
    expire_in: 1 week
  allow_failure: true
```

**Critical flags:**
- `--noupdate` — skips NVD re-download, uses the 228MB cached database mounted via volume
- `--format HTML --format JSON` — generates both human-readable and machine-readable reports
- `allow_failure: true` — pipeline continues even if vulnerabilities are found (NodeGoat is intentionally vulnerable)

**NVD Database:**
- Location on host: `/home/kali/dependency-check-data/odc.mv.db`
- Size: **228MB**
- CVEs indexed: **346,335**
- Mounted into container at: `/usr/share/dependency-check/data`

### 5.5 Stage 4 — SonarQube SAST

**Purpose:** Perform static application security testing and code quality analysis.

```yaml
sonarqube-analysis:
  stage: sonarqube
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner
        -Dsonar.projectKey=nodegoat
        -Dsonar.sources=app
        -Dsonar.host.url=$SONAR_HOST_URL
        -Dsonar.login=$SONAR_TOKEN
  allow_failure: true
```

- Uses `sonar-scanner-cli` Docker image
- Scans the `app/` directory
- Credentials passed via CI/CD variables (never hardcoded)

**sonar-project.properties:**
```properties
sonar.projectKey=nodegoat
sonar.projectName=OWASP NodeGoat
sonar.sources=app
sonar.language=js
sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

### 5.6 Stage 5 — Quality Gate

**Purpose:** Poll SonarQube API and fail the pipeline if code quality gate is not passed.

```yaml
quality-gate:
  stage: quality-gate
  image: alpine:latest
  script:
    - apk add --no-cache curl jq
    - |
      STATUS=$(curl -s -u $SONAR_TOKEN: \
        "$SONAR_HOST_URL/api/qualitygates/project_status?projectKey=nodegoat" \
        | jq -r '.projectStatus.status')
      echo "Quality Gate Status: $STATUS"
      if [ "$STATUS" != "OK" ]; then
        echo "Quality Gate FAILED"
        exit 1
      fi
  allow_failure: true
```

- Uses `curl` + `jq` to query SonarQube REST API
- Exits with code 1 if quality gate is not `OK`

### 5.7 Stage 6 — Deploy to Staging

**Purpose:** Build Docker image of NodeGoat and deploy it to the staging environment automatically.

```yaml
deploy-staging:
  stage: deploy-staging
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t nodegoat:staging .
    - docker stop nodegoat-staging || true
    - docker rm nodegoat-staging || true
    - docker run -d
        --name nodegoat-staging
        -p 4000:4000
        nodegoat:staging
```

- Builds the Docker image using the project `Dockerfile`
- Stops and removes any existing staging container
- Starts a fresh container on port 4000
- This is where ZAP scans the live application

### 5.8 Stage 7 — OWASP ZAP DAST

**Purpose:** Run a live dynamic attack simulation against the deployed staging application.

```yaml
dast:
  stage: dast
  image: ghcr.io/zaproxy/zaproxy:stable
  script:
    - mkdir -p /zap/wrk
    - zap-baseline.py
        -t http://192.168.0.150:4000
        -r zap-report.html
        -J zap-report.json
        -I
  artifacts:
    paths:
      - zap-report.html
      - zap-report.json
    expire_in: 1 week
  allow_failure: true
```

- Uses `ghcr.io/zaproxy/zaproxy:stable` image
- `-t` points to the staging app deployed in the previous stage
- `-I` flag: ignore warnings, do not fail on alerts
- `-r` and `-J` generate both HTML and JSON reports
- `allow_failure: true` — allows pipeline to continue since NodeGoat has intentional vulnerabilities

### 5.9 Stage 8 — Deploy to Production (Manual)

**Purpose:** Manually approved production deployment after all security gates pass.

```yaml
deploy-production:
  stage: deploy-production
  image: docker:24
  when: manual
  script:
    - docker build -t nodegoat:production .
    - docker stop nodegoat-prod || true
    - docker rm nodegoat-prod || true
    - docker run -d
        --name nodegoat-prod
        -p 5000:4000
        nodegoat:production
```

- `when: manual` — requires a human to click "Deploy" in GitLab UI
- Runs production on port 5000
- Final gate before code reaches "production"

***

## 6. CI/CD Variables Configured

All sensitive values were stored as **masked CI/CD variables** in GitLab → Project → Settings → CI/CD → Variables. Nothing was hardcoded in `.gitlab-ci.yml`.

| Variable Name | Purpose |
|---|---|
| `SONAR_TOKEN` | SonarQube authentication token |
| `SONAR_HOST_URL` | SonarQube server URL (http://192.168.0.150:9000) |
| `CI_REGISTRY_USER` | GitLab container registry login |
| `CI_REGISTRY_PASSWORD` | GitLab container registry password |

***

## 7. Problems Encountered & Solutions

This section documents every major challenge faced during the project and the exact solution applied.

### 7.1 OWASP Dependency-Check — NVD Database Download Failure

**Problem:** The NVD database download (346,335 CVEs, 228MB) kept failing mid-way. The machine would sleep, the download would time out, lock files would be left behind, and the next run would crash. This caused repeated pipeline failures and wasted hours of time.

**Root cause:** Default runner configuration did not mount a persistent volume for the database. Every pipeline run tried to re-download the entire 228MB database from scratch.

**Solution:**
1. Downloaded the NVD database once manually on the host:
   ```bash
   docker run --rm \
     -v /home/kali/dependency-check-data:/usr/share/dependency-check/data \
     owasp/dependency-check \
     --updateonly
   ```
2. Added the volume mount to `/etc/gitlab-runner/config.toml`:
   ```toml
   volumes = ["/home/kali/dependency-check-data:/usr/share/dependency-check/data"]
   ```
3. Added `--noupdate` flag to the dependency-check script in `.gitlab-ci.yml`
4. Result: Pipeline runs in minutes instead of timing out

### 7.2 OWASP ZAP — Report Generation & Networking Failure

**Problem:** ZAP could not reach the staging application. The DAST stage kept failing with connection refused errors. Even after fixing networking, the report was not being saved correctly.

**Root cause 1:** ZAP container did not know the IP of the staging app running on the host.
**Root cause 2:** The `/zap/wrk` directory was not created before ZAP tried to write the report.

**Solution:**
1. Used the actual host IP (`192.168.0.150`) instead of `localhost` in the `-t` flag
2. Added `mkdir -p /zap/wrk` before the ZAP scan command
3. Added `-I` flag to ignore failures on known vulnerabilities
4. Added `allow_failure: true` to allow the pipeline to continue

### 7.3 GitLab Runner — Clone URL Resolution

**Problem:** The runner container could not resolve the GitLab hostname when cloning the repository. Jobs would fail at the "Getting source from Git repository" step.

**Solution:** Added `extra_hosts` and `clone_url` to the runner config:
```toml
clone_url = "http://192.168.0.150:8929"
extra_hosts = ["gitlab:192.168.0.150"]
```

### 7.4 Docker-in-Docker — Privileged Mode

**Problem:** Docker build commands inside pipeline jobs were failing with "permission denied" errors.

**Solution:** Enabled `privileged = true` in runner config and mounted the Docker socket:
```toml
privileged = true
volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
```

### 7.5 GitHub Push — Secret Scanning Block

**Problem:** When pushing the repository to GitHub, the push was blocked due to AWS Access Key and Secret Key found inside `node_modules/forever/node_modules/winston/node_modules/request/tests/test-s3.js`. These were **fake test keys** hardcoded in an old version of the `winston` library — not real credentials.

**Solution:**
1. Added `node_modules/` to `.gitignore`
2. Installed `git-filter-repo`:
   ```bash
   sudo apt install git-filter-repo -y
   ```
3. Removed `node_modules` from entire git history:
   ```bash
   git filter-repo --path node_modules --invert-paths --force
   ```
4. Re-added GitHub remote and force pushed:
   ```bash
   git remote add origin git@github.com:ali4210/DevSecOps-_Project-3_Gitlab_CI-CD-Pipeline.git
   git push origin main --force
   ```

***

## 8. Tools & Versions Used

| Tool | Version | Source |
|---|---|---|
| GitLab CE | Latest (self-hosted) | packages.gitlab.com |
| GitLab Runner | 18.11.1 | packages.gitlab.com |
| Docker | 24 | docker.io |
| Node.js | 20 (Alpine) | hub.docker.com/node |
| OWASP Dependency-Check | Latest | hub.docker.com/owasp/dependency-check |
| SonarQube | Community Edition | hub.docker.com/sonarqube |
| Sonar Scanner CLI | Latest | hub.docker.com/sonarsource/sonar-scanner-cli |
| OWASP ZAP | Stable | ghcr.io/zaproxy/zaproxy |
| git-filter-repo | Latest apt | apt package |
| OWASP NodeGoat | Main branch | github.com/OWASP/NodeGoat |

***

## 9. Key Learnings

1. **Self-hosted infrastructure teaches what cloud hides.** Running every tool yourself forces you to understand networking, storage, permissions, and configuration at a deep level.

2. **Persistent volumes are critical for heavy security tools.** OWASP Dependency-Check's 228MB NVD database must be cached — downloading it on every pipeline run is not practical.

3. **Security scanning must live inside the pipeline, not after it.** Integrating SAST, SCA, and DAST into CI/CD means every commit is automatically scanned.

4. **`allow_failure: true` is a strategic decision.** On intentionally vulnerable apps like NodeGoat, setting this flag prevents the pipeline from blocking on expected findings while still generating reports.

5. **CI/CD variables protect secrets.** Never hardcode tokens, passwords, or URLs in `.gitlab-ci.yml`. GitLab masked variables ensure credentials never appear in logs.

6. **Docker socket mounting is powerful but carries risk.** Mounting `/var/run/docker.sock` gives containers full Docker access — acceptable in a controlled lab, but must be evaluated carefully in production.

7. **Git history is permanent unless explicitly rewritten.** Even test/fake secrets in old dependencies can block GitHub pushes. `git-filter-repo` is the right tool to clean history.

***

## 10. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Kali Linux Host (192.168.0.150)          │
│                                                             │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  GitLab CE  │    │  SonarQube   │    │  NVD Database │  │
│  │  :8929      │    │  :9000       │    │  228MB Cache  │  │
│  └──────┬──────┘    └──────────────┘    └───────────────┘  │
│         │                                                   │
│  ┌──────▼──────────────────────────────────────────────┐   │
│  │            GitLab Runner (Docker executor)           │   │
│  │                                                      │   │
│  │  Job 1: node:20-alpine    → Build                   │   │
│  │  Job 2: node:20-alpine    → Unit Test               │   │
│  │  Job 3: owasp/dep-check   → SCA (346K CVEs)         │   │
│  │  Job 4: sonar-scanner-cli → SAST                    │   │
│  │  Job 5: alpine + curl/jq  → Quality Gate            │   │
│  │  Job 6: docker:24         → Deploy Staging :4000    │   │
│  │  Job 7: zaproxy:stable    → DAST Live Scan          │   │
│  │  Job 8: docker:24         → Deploy Production :5000 │   │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

***

## 11. How to Re-Run This Project

If you need to rebuild this entire setup from scratch, follow these steps in order:

### Step 1 — Install GitLab CE
```bash
sudo apt install curl openssh-server ca-certificates
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://192.168.0.150:8929" apt install gitlab-ce
```

### Step 2 — Install and Start Docker
```bash
sudo apt install docker.io -y
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
```

### Step 3 — Run SonarQube
```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  sonarqube:community
```

### Step 4 — Download NVD Database Once
```bash
mkdir -p /home/kali/dependency-check-data
docker run --rm \
  -v /home/kali/dependency-check-data:/usr/share/dependency-check/data \
  owasp/dependency-check \
  --updateonly
```

### Step 5 — Install and Register GitLab Runner
```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y

sudo gitlab-runner register \
  --url http://192.168.0.150:8929 \
  --token YOUR_TOKEN \
  --executor docker \
  --docker-image node:20-alpine \
  --docker-privileged \
  --non-interactive
```

### Step 6 — Update config.toml
Edit `/etc/gitlab-runner/config.toml` and add:
```toml
volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache", "/home/kali/dependency-check-data:/usr/share/dependency-check/data"]
extra_hosts = ["gitlab:192.168.0.150"]
clone_url = "http://192.168.0.150:8929"
pull_policy = ["if-not-present"]
```

### Step 7 — Set CI/CD Variables in GitLab
```
SONAR_TOKEN        = <your sonarqube token>
SONAR_HOST_URL     = http://192.168.0.150:9000
```

### Step 8 — Clone NodeGoat and Push to GitLab
```bash
git clone https://github.com/OWASP/NodeGoat.git devsecops-demo
cd devsecops-demo
git remote set-url origin http://192.168.0.150:8929/root/devsecops-demo.git
git push -u origin main
```

### Step 9 — Add `.gitlab-ci.yml` and Push
Add your pipeline file and push — GitLab will automatically trigger the pipeline.

***

## 12. Security Scanning Reports

All reports are generated as pipeline artifacts and downloadable from GitLab:

| Report | Format | Stage | Retention |
|---|---|---|---|
| Dependency-Check Report | HTML + JSON | dependency-check | 1 week |
| SonarQube Dashboard | Web UI | sonarqube | Permanent |
| ZAP DAST Report | HTML + JSON | dast | 1 week |

***

*This documentation was prepared by Saleem Ali as part of the Al-Nafi AIOps program, April 2026.*
*GitHub: https://github.com/ali4210 | LinkedIn: https://www.linkedin.com/in/saleem-ali-189719325*
