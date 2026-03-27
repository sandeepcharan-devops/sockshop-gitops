# Sock Shop — Production-Grade DevOps Portfolio Project

A real-world DevOps implementation built on the [Weaveworks Sock Shop](https://github.com/microservices-demo/microservices-demo) microservices demo application. This project demonstrates end-to-end CI/CD, GitOps, security scanning, and Kubernetes failure scenario debugging — skills directly applicable to production DevOps roles.

---

## Architecture Overview

```
Developer pushes code to front-end repo
        ↓
Jenkins CI Pipeline (on EC2)
  ├── Build Docker image
  ├── Trivy security scan (CRITICAL CVE gate)
  ├── Push to Docker Hub (only if scan passes)
  └── Update image tag in sockshop-gitops repo
        ↓
Argo CD detects change in sockshop-gitops
        ↓
Argo CD deploys to Kubernetes (sock-shop namespace)
        ↓
Rolling update with zero downtime
```

---

## Infrastructure

| Component | Technology | Details |
|---|---|---|
| Kubernetes Cluster | kubeadm on AWS EC2 | 1 Master + 1 Worker, t2.medium, Ubuntu 22.04 |
| Container Runtime | containerd | v1.29 |
| CNI Plugin | Calico | Pod network CIDR 192.168.0.0/16 |
| CI Pipeline | Jenkins | v2.541.3, running on master node |
| CD Pipeline | Argo CD | v3.3.4, GitOps continuous delivery |
| Image Registry | Docker Hub | Public registry |
| Security Scanning | Trivy | CRITICAL CVE gate in pipeline |
| Metrics | metrics-server | Required for HPA |

---

## Repository Structure

```
sockshop-gitops/                  ← This repo (GitOps manifests)
manifests/
  └── front-end/
      └── deployment.yaml         ← Front-end Kubernetes deployment

front-end/                        ← Separate repo (application source)
  ├── Dockerfile                  ← Hardened, production-grade
  ├── Jenkinsfile                 ← CI pipeline definition
  ├── package.json                ← Dependencies with security resolutions
  └── .dockerignore               ← Prevents sensitive file inclusion
```

### Two-Repo GitOps Strategy

| Repo | Purpose | Watched By |
|---|---|---|
| `front-end` | Source code, Dockerfile, Jenkinsfile | Jenkins |
| `sockshop-gitops` | Kubernetes manifests | Argo CD |

This separation ensures:
- Git is the single source of truth for cluster state
- Every deployment is a traceable git commit
- Rollback = `git revert`, not re-running a pipeline
- Argo CD continuously reconciles cluster state with repo

---

## CI/CD Pipeline

### Jenkins Pipeline Stages

```
Checkout → Unit test → Build Docker Image → Trivy Scan → Push to Docker Hub → Update GitOps Repo → Cleanup
```

| Stage | Description |
|---|---|
| Checkout | Pulls latest code from front-end repo |
| Unit Tests | Runs 22 mocha tests with istanbul coverage — fails pipeline if any test fails |
| Build Docker Image | Builds image tagged with Jenkins build number e.g. `v1.5` |
| Trivy Scan | Scans for CRITICAL CVEs — pipeline fails and blocks push if found |
| Push to Docker Hub | Pushes clean image only after scan passes |
| Update GitOps Repo | Updates `deployment.yaml` image tag automatically via `sed` + git push |
| Cleanup | Removes local image to free disk space |

### Security Gate

The Trivy scan stage uses `--exit-code 1` which fails the pipeline on any CRITICAL vulnerability. This means:
- No CRITICAL CVE image ever reaches the cluster
- The push stage is completely blocked until the image is clean
- Every build has a documented security scan result

---

## Dockerfile Improvements

The original Sock Shop front-end Dockerfile had multiple production issues. All were identified and fixed:

| Issue | Original | Fixed |
|---|---|---|
| Base image | `node:10-alpine` (EOL April 2021) | `node:22-alpine3.22` (LTS) |
| ENV syntax | Legacy `ENV key value` | Modern `ENV key="value"` |
| Package manager | Mixed yarn/npm | Consistent yarn throughout |
| Health check | None | `HEALTHCHECK` with proper intervals |
| Build context | No `.dockerignore` | `.dockerignore` excludes node_modules, .git, .env |

### CVE Remediation

Initial Trivy scan found **13 CRITICAL vulnerabilities** in node_modules dependencies.

| Package | CVE | Type | Fix |
|---|---|---|---|
| `morgan 1.7.0` | CVE-2019-5413 | Unescaped input injection | Upgraded to `1.9.1` in package.json |
| `bson 1.0.9` | CVE-2020-7610 | Code injection via deserialization | Pinned to `1.1.4` via yarn resolutions |
| `handlebars 4.5.1` | CVE-2021-23369 | Remote code execution | Pinned to `4.7.7` via yarn resolutions |
| `minimist 0.0.10` | CVE-2021-44906 | Prototype pollution | Pinned to `1.2.6` via yarn resolutions |
| `growl 1.9.2` | CVE-2017-16042 | Command injection | Pinned to `1.10.0` via yarn resolutions |
| `json-schema 0.2.3` | CVE-2021-3918 | Prototype pollution | Pinned to `0.4.0` via yarn resolutions |
| `form-data 1.0.0` | CVE-2025-7783 | Unsafe random function | Pinned to `4.0.4` via yarn resolutions |

**Result: 13 CRITICAL → 0 CRITICAL**

---

## Break and Fix Scenarios

Each scenario was deliberately triggered, debugged, and resolved to demonstrate real-world troubleshooting competency.

---

### Scenario 1 — HPA Not Scaling (Missing Resource Requests)

**What was broken:**
Removed `resources.requests.cpu` from the front-end deployment directly via `kubectl edit`.

**Symptom:**
```bash
kubectl get hpa -n sock-shop
# TARGETS shows <unknown>/50% instead of actual CPU%
```

**Root cause:**
HPA calculates CPU utilization as `actual usage / requested CPU × 100`. Without `requests.cpu` defined, the denominator is zero — HPA cannot calculate utilization and makes no scaling decisions regardless of actual CPU load.

**Debug commands used:**
```bash
kubectl get hpa -n sock-shop
kubectl describe hpa front-end -n sock-shop   # Check Conditions section
kubectl describe deployment front-end -n sock-shop | grep -A5 Requests
```

**Fix:**
Restored `resources.requests.cpu: 100m` via Argo CD sync from GitOps repo.

**Bonus — Argo CD Drift Detection:**
The direct `kubectl edit` change caused Argo CD to show `OutOfSync` — demonstrating that any manual cluster change is automatically detected. Fix was applied by syncing through Argo CD, not by running another kubectl command.

---

### Scenario 2 — CrashLoopBackOff (Bad Command)

**What was broken:**
Injected `command: ["/bin/sh", "-c", "exit 1"]` into `deployment.yaml` and pushed via GitOps.

**Symptom:**
```bash
kubectl get pods -n sock-shop
# front-end showing CrashLoopBackOff with increasing restart count
```

**Root cause:**
Container was instructed to exit immediately with error code 1 on every startup. Kubernetes kept restarting it — each restart hit the same exit — creating the CrashLoopBackOff loop.

**Debug commands used:**
```bash
kubectl describe pod <pod-name> -n sock-shop   # Showed bad command + Exit Code 1
kubectl logs <pod-name> -n sock-shop           # Empty — container exits too fast
kubectl logs <pod-name> -n sock-shop --previous  # Previous container instance logs
```

**Key insight:**
`kubectl logs --previous` is critical for CrashLoopBackOff debugging — current logs are often empty because the container has already crashed. Always check previous container logs.

**Fix:**
Removed bad command from `deployment.yaml`, pushed to GitOps repo. Argo CD auto-synced and redeployed clean version.

---

### Scenario 3 — OOMKilled (Memory Limit Too Low)

**What was broken:**
Set memory limit to `10Mi` and request to `8Mi` in `deployment.yaml`.

**Symptom:**
```bash
kubectl describe pod <pod-name> -n sock-shop
# Last State: Terminated
# Reason: OOMKilled
# Exit Code: 137
```

**Root cause:**
Node.js requires significantly more than 10Mi to run. The moment the process tried to allocate memory beyond the limit, the Linux kernel OOM killer terminated the container. Kubernetes restarted it — same result — CrashLoopBackOff.

**Debug commands used:**
```bash
kubectl describe pod <pod-name> -n sock-shop | grep -A5 "Last State"
kubectl top pod -n sock-shop   # Check actual memory consumption
```

**How to distinguish OOMKilled from other crashes:**

| Exit Code | Reason | Cause |
|---|---|---|
| 137 | OOMKilled | Memory limit exceeded |
| 1 | Error | Application crash or bad command |
| 0 | Completed | App exited cleanly (wrong for a server) |

**Fix:**
Restored memory limit to `256Mi` — 2x observed actual usage of ~120Mi. Pushed via GitOps, Argo CD synced.

---

### Scenario 4 — ImagePullBackOff (Wrong Image Tag)

**What was broken:**
Deployment manifest referenced `v1.1` but only `v1.2` existed on Docker Hub (build `v1.1` had failed in Jenkins).

**Symptom:**
```bash
kubectl describe pod <pod-name> -n sock-shop
# Events: Failed to pull image — not found
```

**Root cause:**
Tag mismatch between manifest and registry. `imagePullPolicy: IfNotPresent` with a non-existent tag will always fail — the image simply doesn't exist to pull.

**Fix:**
Updated image tag in `deployment.yaml` to match the actual pushed tag. Automated this permanently by adding an `Update GitOps Repo` stage to the Jenkins pipeline that uses `sed` to update the tag automatically after every successful build.

---

## Key Learnings

1. **Resource requests are mandatory for HPA** — without them HPA is completely blind
2. **`kubectl logs --previous`** is the most important CrashLoopBackOff debug command
3. **OOMKilled = Exit Code 137** — always check Last State in describe output
4. **GitOps drift detection** — any manual kubectl change is automatically flagged by Argo CD
5. **Trivy in CI** — security scanning belongs in the pipeline as a gate, not as a post-incident tool
6. **Two-repo strategy** — separating app code from manifests gives clean audit trail and proper separation of concerns

---

## Tools & Versions

| Tool | Version |
|---|---|
| Kubernetes | v1.29.15 |
| kubeadm | v1.29.15 |
| Jenkins | v2.541.3 |
| Argo CD | v3.3.4 |
| Trivy | Latest |
| Docker | v28.x |
| Node.js | v22 (LTS) |
| Calico CNI | v3.26.0 |
| metrics-server | Latest |

---

## Author

**Sandeep Charan**
DevOps Engineer | AWS | Kubernetes | Jenkins | Argo CD | Docker | Terraform | Grafana
[GitHub](https://github.com/sandeepcharan-devops)
