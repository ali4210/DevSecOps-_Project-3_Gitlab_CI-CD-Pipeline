# 🛡️ DevSecOps CI/CD Pipeline — Project 3

> **A fully automated, self-hosted 7-stage DevSecOps pipeline built on GitLab CE with SAST, DAST, SCA, and manual production gate.**







***

## 📌 Project Overview

This project implements a **complete DevSecOps CI/CD pipeline** for the intentionally vulnerable web application **OWASP NodeGoat**, deployed on a **fully self-hosted infrastructure** running on **Kali Linux**.

Every tool — GitLab CE, GitLab Runner, SonarQube, OWASP Dependency-Check, OWASP ZAP — runs locally on a single machine with no cloud dependency.

***

## 🏗️ Infrastructure Stack

| Component | Tool / Version | Role |
|---|---|---|
| OS | Kali Linux (master node) | Host machine |
| Source Control | GitLab CE (self-hosted) | Git server + CI/CD |
| CI Runner | GitLab Runner v18.11.1 | Pipeline executor |
| Containerization | Docker (privileged mode) | Job isolation |
| Target App | OWASP NodeGoat (Node.js 20) | Vulnerable test application |
| SAST | SonarQube Community | Static code analysis |
| SCA | OWASP Dependency-Check | CVE vulnerability scanning |
| DAST | OWASP ZAP | Live dynamic attack simulation |
| Registry | GitLab Container Registry | Docker image storage |

***

## 🔁 Pipeline Stages

```
┌──────────┐   ┌──────────┐   ┌──────────────────┐   ┌────────────┐
│  BUILD   │ → │   TEST   │ → │ DEPENDENCY-CHECK │ → │  SONARQUBE │
└──────────┘   └──────────┘   └──────────────────┘   └────────────┘
                                                              ↓
┌──────────────┐   ┌──────────────────┐   ┌────────────────────────┐
│  QUALITY     │ → │  DEPLOY STAGING  │ → │  ZAP DAST SCAN         │
│  GATE        │   │  (auto)          │   │  (live scan)           │
└──────────────┘   └──────────────────┘   └────────────────────────┘
                                                              ↓
                                              ┌─────────────────────┐
                                              │  DEPLOY PRODUCTION  │
                                              │  (manual approval)  │
                                              └─────────────────────┘
```

### Stage Details

| # | Stage | Tool | Type | Trigger |
|---|---|---|---|---|
| 1 | Build | Node.js 20 Alpine | Compile + Install | Auto |
| 2 | Unit Test | npm test + coverage | Testing | Auto |
| 3 | Dependency-Check | OWASP DC (346,335 CVEs) | SCA | Auto |
| 4 | SAST Analysis | SonarQube | Static Analysis | Auto |
| 5 | Quality Gate | SonarQube API | Gate Enforcement | Auto |
| 6 | Deploy Staging | Docker | Staging Deploy | Auto |
| 7 | DAST Scan | OWASP ZAP | Live Scan | Auto |
| 8 | Deploy Production | Docker | Production Deploy | **Manual** |

***

## 🔧 Key Configurations

### GitLab Runner (`/etc/gitlab-runner/config.toml`)
- Executor: **Docker (privileged)**
- Pull policy: `if-not-present`
- Volume mounts: Docker socket + Dependency-Check NVD database cache
- Extra hosts: internal GitLab hostname resolution

### CI/CD Variables (GitLab → Settings → CI/CD)
```
SONAR_TOKEN          → SonarQube authentication token
SONAR_HOST_URL       → http://<host>:9000
CI_REGISTRY_USER     → GitLab registry username
CI_REGISTRY_PASSWORD → GitLab registry password
```

### OWASP Dependency-Check
- NVD database: **228MB** local cache at `/home/kali/dependency-check-data/`
- Mounted into container: `/usr/share/dependency-check/data`
- Flag used: `--noupdate` to skip re-download on each run
- CVEs scanned: **346,335**

***

## 📁 Repository Structure

```
devsecops-demo/
├── .gitlab-ci.yml          # Main pipeline definition (7 stages)
├── app/                    # OWASP NodeGoat application source
│   ├── server.js
│   ├── package.json
│   └── ...
├── sonar-project.properties # SonarQube project config
├── Dockerfile              # Multi-stage Docker build
└── README.md
```

***

## 🚀 How to Reproduce This Project

### Prerequisites
- Kali Linux (or Ubuntu) machine with 8GB+ RAM
- Docker installed and running
- GitLab CE running at your local IP
- SonarQube running at port 9000
- OWASP Dependency-Check NVD database downloaded

### Steps
1. Clone this repo into your GitLab CE instance
2. Register a GitLab Runner with Docker executor (privileged)
3. Set CI/CD variables in GitLab project settings
4. Mount Dependency-Check data volume in runner config
5. Push to `main` branch — pipeline triggers automatically
6. Monitor stages in GitLab → CI/CD → Pipelines

***

## 📊 Security Findings Summary

| Tool | Finding Type | Result |
|---|---|---|
| OWASP Dependency-Check | Known CVEs in dependencies | Vulnerabilities found (NodeGoat is intentionally vulnerable) |
| SonarQube | Code smells, bugs, vulnerabilities | Issues detected in NodeGoat source |
| OWASP ZAP | Live attack simulation | DAST alerts generated from running app |

> ⚠️ **Note:** OWASP NodeGoat is **intentionally vulnerable** by design. All findings are expected and part of the learning exercise.

***

## 🎓 Course Context

This project was completed as **Lab Project 3** of the **AIOps program** at **Al-Nafi International College** (online).

- **Student:** Saleem Ali
- **Program:** AIOps (Artificial Intelligence Operations)
- **Institution:** Al-Nafi International College
- **GitHub:** [github.com/ali4210](https://github.com/ali4210)
- **LinkedIn:** [linkedin.com/in/saleem-ali-189719325](https://www.linkedin.com/in/saleem-ali-189719325)

***

## 📜 License

This project is for educational purposes. OWASP NodeGoat is licensed under the Apache 2.0 License.
