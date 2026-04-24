# 🛡️ DevSecOps CI/CD Pipeline — Complete Project Documentation

> **Student:** Saleem Ali | **Program:** AIOps | **Institution:** Al-Nafi International College | **April 2026**
> 
> 🔗 [GitHub](https://github.com/ali4210) -  [LinkedIn](https://www.linkedin.com/in/saleem-ali-189719325)

***

## ⚡ What Is This Project?

A **fully automated, 8-stage DevSecOps CI/CD pipeline** built entirely on **self-hosted infrastructure** running on **Kali Linux**. No cloud. No managed services. Everything installed and configured by hand — GitLab CE, GitLab Runner, SonarQube, OWASP Dependency-Check, and OWASP ZAP — all running on a single local machine.

The target application was **OWASP NodeGoat** — an intentionally vulnerable Node.js web app used for security training.

***

## 🖥️ The Infrastructure

```
╔══════════════════════════════════════════════════════════╗
║           Kali Linux Host — 192.168.0.150                ║
╠══════════════════════════════════════════════════════════╣
║  GitLab CE          → http://192.168.0.150:8929          ║
║  SonarQube          → http://192.168.0.150:9000          ║
║  NodeGoat Staging   → http://192.168.0.150:4000          ║
║  NodeGoat Prod      → http://192.168.0.150:5000          ║
║  NVD Database Cache → /home/kali/dependency-check-data/  ║
╚══════════════════════════════════════════════════════════╝
```

| Component | Tool | Version |
|---|---|---|
| OS | Kali Linux (amd64) | — |
| Source Control + CI/CD | GitLab CE (self-hosted) | Latest |
| Pipeline Executor | GitLab Runner | 18.11.1 |
| Containerization | Docker | 24 |
| Target App | OWASP NodeGoat | Node.js 20 |
| Static Analysis (SAST) | SonarQube Community | Latest |
| Dependency Scanning (SCA) | OWASP Dependency-Check | Latest |
| Live Attack Simulation (DAST) | OWASP ZAP | Stable |

***

## 🔁 The 8-Stage Pipeline

```
 ┌─────────┐    ┌─────────┐    ┌──────────────────┐    ┌────────────┐
 │  BUILD  │ →  │  TEST   │ →  │ DEPENDENCY-CHECK │ →  │  SONARQUBE │
 └─────────┘    └─────────┘    └──────────────────┘    └─────┬──────┘
                                                             ↓
 ┌──────────────────┐    ┌────────────────┐    ┌────────────────────┐
 │  DEPLOY STAGING  │ ←  │  QUALITY GATE  │ ←  │  (SonarQube API)  │
 └────────┬─────────┘    └────────────────┘    └────────────────────┘
          ↓
 ┌────────────────┐    ┌──────────────────────┐
 │   ZAP  DAST    │ →  │  DEPLOY PRODUCTION   │
 │  (live scan)   │    │  ✋ MANUAL APPROVAL  │
 └────────────────┘    └──────────────────────┘
```

| # | Stage | Docker Image Used | Type | Gate |
|---|---|---|---|---|
| 1 | Build | `node:20-alpine` | Compile + Install | Auto ✅ |
| 2 | Unit Test | `node:20-alpine` | Test + Coverage | Auto ✅ |
| 3 | Dependency-Check | `owasp/dependency-check` | SCA — 346,335 CVEs | Auto ✅ |
| 4 | SonarQube Scan | `sonarsource/sonar-scanner-cli` | SAST | Auto ✅ |
| 5 | Quality Gate | `alpine` + curl + jq | Gate Enforcement | Auto ✅ |
| 6 | Deploy Staging | `docker:24` | Auto Deploy | Auto ✅ |
| 7 | ZAP DAST | `ghcr.io/zaproxy/zaproxy:stable` | Live Scan | Auto ✅ |
| 8 | Deploy Production | `docker:24` | Production Deploy | **Manual 🖐️** |

***

## ⚙️ What We Did — Step by Step

***

### 🔷 STEP 1 — Installed GitLab CE on Kali Linux

We installed GitLab CE as a self-hosted Git server and CI/CD platform on our Kali Linux machine. This served as the single source of truth for code, pipelines, and the container registry.

```bash
sudo apt install curl openssh-server ca-certificates -y
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://192.168.0.150:8929" apt install gitlab-ce -y
sudo gitlab-ctl reconfigure
sudo gitlab-ctl status
```

***

### 🔷 STEP 2 — Installed Docker

We installed Docker on the host machine. Docker was used both to run service containers (SonarQube) and as the GitLab Runner executor so every CI job ran in an isolated container.

```bash
sudo apt install docker.io -y
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
docker --version
```

***

### 🔷 STEP 3 — Ran SonarQube as a Docker Container

We pulled and started SonarQube Community Edition as a Docker container. SonarQube was our SAST engine — it analysed the NodeGoat source code for vulnerabilities, bugs, and code smells.

```bash
docker pull sonarqube:community
docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  -p 9000:9000 \
  sonarqube:community
docker ps | grep sonarqube
```

After starting, we accessed `http://192.168.0.150:9000`, created a project called `nodegoat`, generated a **project token**, and stored it as a GitLab CI/CD variable named `SONAR_TOKEN`.

***

### 🔷 STEP 4 — Downloaded the OWASP NVD Database

We downloaded the NVD (National Vulnerability Database) locally — a 228MB file containing 346,335 known CVEs. We mounted this into the Dependency-Check container so the pipeline never re-downloads it on every run.

```bash
mkdir -p /home/kali/dependency-check-data
docker run --rm \
  -v /home/kali/dependency-check-data:/usr/share/dependency-check/data \
  owasp/dependency-check \
  --updateonly
ls -lh /home/kali/dependency-check-data/
du -sh /home/kali/dependency-check-data/odc.mv.db
```

Output confirmed: **228MB** `odc.mv.db` file downloaded successfully.

***

### 🔷 STEP 5 — Installed and Registered GitLab Runner

We installed the GitLab Runner service and registered it with our self-hosted GitLab CE instance. The runner used Docker as its executor — every pipeline job runs in a fresh Docker container.

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y

sudo gitlab-runner register \
  --url http://192.168.0.150:8929 \
  --token glrt-NS3epGRG4AshCCx-hYtHcG86MQpwOjIKdDozCnU6MQ8.01.1715hnuja \
  --executor docker \
  --docker-image node:20-alpine \
  --docker-privileged \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
  --description DevSecOps-Runner \
  --non-interactive

sudo gitlab-runner start
sudo gitlab-runner status
sudo gitlab-runner list
```

***

### 🔷 STEP 6 — Configured the Runner `config.toml`

We manually edited the runner's configuration file to add three critical things: the Docker socket volume mount, the NVD database volume mount, and the internal GitLab hostname resolution.

```bash
sudo nano /etc/gitlab-runner/config.toml
```

**Final `config.toml`:**

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

```bash
sudo gitlab-runner restart
sudo gitlab-runner verify
```

***

### 🔷 STEP 7 — Set CI/CD Variables in GitLab

We stored all sensitive values as masked variables in GitLab so they never appeared in the `.gitlab-ci.yml` file or pipeline logs.

```
GitLab → Project → Settings → CI/CD → Variables → Add Variable
```

| Variable | Value |
|---|---|
| `SONAR_TOKEN` | `<token from SonarQube project>` |
| `SONAR_HOST_URL` | `http://192.168.0.150:9000` |

***

### 🔷 STEP 8 — Created the `sonar-project.properties` File

We added this file to the root of the NodeGoat repository so SonarQube knew what to scan.

```properties
sonar.projectKey=nodegoat
sonar.projectName=OWASP NodeGoat
sonar.sources=app
sonar.language=js
sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

***

### 🔷 STEP 9 — Wrote the Full `.gitlab-ci.yml` Pipeline

We created the entire 8-stage pipeline file. Each stage runs in its own Docker container, passes artifacts forward, and scans from a different security angle.

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

# ─── STAGE 1: BUILD ───────────────────────────────────────────
build:
  stage: build
  image: node:20-alpine
  script:
    - npm install
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour

# ─── STAGE 2: UNIT TEST ───────────────────────────────────────
test:
  stage: test
  image: node:20-alpine
  script:
    - npm test
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'

# ─── STAGE 3: DEPENDENCY-CHECK (SCA) ─────────────────────────
dependency-check:
  stage: dependency-check
  image: owasp/dependency-check:latest
  script:
    - mkdir -p reports
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

# ─── STAGE 4: SONARQUBE SAST ──────────────────────────────────
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

# ─── STAGE 5: QUALITY GATE ────────────────────────────────────
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
        echo "❌ Quality Gate FAILED"
        exit 1
      fi
      echo "✅ Quality Gate PASSED"
  allow_failure: true

# ─── STAGE 6: DEPLOY STAGING ──────────────────────────────────
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
    - echo "✅ Staging deployed at http://192.168.0.150:4000"

# ─── STAGE 7: ZAP DAST ────────────────────────────────────────
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

# ─── STAGE 8: DEPLOY PRODUCTION (MANUAL) ──────────────────────
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
    - echo "🚀 Production deployed at http://192.168.0.150:5000"
```

***

### 🔷 STEP 10 — Pushed to GitLab and Triggered the Pipeline

```bash
cd /home/kali/devsecops-demo
git add .
git commit -m "feat: add complete DevSecOps CI/CD pipeline"
git push origin main
```

The pipeline triggered automatically. We monitored it live:

```
GitLab → CI/CD → Pipelines → Click the running pipeline
```

***

### 🔷 STEP 11 — Pushed to GitHub

After completing the project locally, we pushed the entire repository to GitHub for public portfolio visibility.

```bash
cd /home/kali/devsecops-demo

# Remove node_modules from git history (contained fake AWS test keys)
sudo apt install git-filter-repo -y
git filter-repo --path node_modules --invert-paths --force

# Add GitHub remote and push
git remote add origin git@github.com:ali4210/DevSecOps-_Project-3_Gitlab_CI-CD-Pipeline.git
git branch -M main
git push -u origin main --force
```

***

## 🔥 The Real Hardest Moments

### ❶ OWASP Dependency-Check + ZAP — Both Failed Together

Both tools were broken at the same time and were blocking the pipeline from completing. This was the most time-consuming phase of the entire project.

**Dependency-Check issues:**
- The 228MB NVD database was failing to download mid-pipeline
- Lock files were left behind after crashes, causing the next run to fail immediately
- The machine would sleep mid-download, corrupting the process
- Solved by downloading the database **once** on the host, mounting it as a persistent volume, and adding the `--noupdate` flag to skip re-download

**ZAP issues:**
- ZAP could not reach the staging application (connection refused)
- The `/zap/wrk` directory did not exist, causing report write failure
- Solved by using the actual host IP `192.168.0.150:4000` instead of `localhost`, adding `mkdir -p /zap/wrk`, and adding the `-I` flag

### ❷ Runner Could Not Clone the Repository

The runner container kept failing at the "Getting source from Git repository" step because it could not resolve the GitLab hostname. Solved by adding `clone_url` and `extra_hosts` to `config.toml`.

### ❸ Docker-in-Docker Permission Denied

Docker build commands inside CI jobs failed with permission errors. Solved by enabling `privileged = true` and mounting `/var/run/docker.sock`.

### ❹ GitHub Blocked the Push

GitHub's secret scanning blocked the push because old `node_modules` test files contained fake AWS keys from the `winston` library. Solved by using `git-filter-repo` to remove `node_modules` from the entire git history.

***

## 📊 Pipeline Security Outputs

| Report | Tool | Format | Where to Find |
|---|---|---|---|
| Dependency-Check Report | OWASP DC | HTML + JSON | GitLab → Pipeline → Artifacts |
| Static Analysis Dashboard | SonarQube | Web UI | http://192.168.0.150:9000 |
| ZAP DAST Report | OWASP ZAP | HTML + JSON | GitLab → Pipeline → Artifacts |

***

## 🔁 Full Rebuild Checklist

If you need to rebuild this project from zero on a fresh machine, run through these steps in order:

```
☐ 1. Install GitLab CE          → sudo apt install gitlab-ce
☐ 2. Install Docker             → sudo apt install docker.io
☐ 3. Start SonarQube            → docker run -d -p 9000:9000 sonarqube:community
☐ 4. Download NVD database      → docker run owasp/dependency-check --updateonly
☐ 5. Install GitLab Runner      → sudo apt install gitlab-runner
☐ 6. Register runner            → sudo gitlab-runner register ...
☐ 7. Edit config.toml           → add volumes + extra_hosts + clone_url
☐ 8. Set CI/CD variables        → SONAR_TOKEN + SONAR_HOST_URL in GitLab
☐ 9. Add sonar-project.properties → in repo root
☐ 10. Add .gitlab-ci.yml        → 8-stage pipeline
☐ 11. git push → pipeline runs  → monitor in GitLab CI/CD
☐ 12. Approve production deploy → manual click in GitLab
```

***

*Documentation prepared by Saleem Ali — Al-Nafi AIOps Program, April 2026*
*GitHub: https://github.com/ali4210 | LinkedIn: https://www.linkedin.com/in/saleem-ali-189719325*
