# Atlas Platform App

A purpose-built Python FastAPI workload designed for platform engineering portfolio projects. This is **not** a hello-world app — it is a standardized test workload that gets deployed, monitored, broken, and scaled across 11 different infrastructure configurations.

Every endpoint in this app exists because a downstream project needs it. Nothing is decorative.

---

## Table of Contents

- [Why This App Exists](#why-this-app-exists)
- [Architecture Overview](#architecture-overview)
- [Endpoints — What They Do and Why](#endpoints--what-they-do-and-why)
- [Project File Structure](#project-file-structure)
- [File-by-File Breakdown](#file-by-file-breakdown)
- [How to Run Locally](#how-to-run-locally)
- [How to Run with Docker](#how-to-run-with-docker)
- [How to Test Each Endpoint](#how-to-test-each-endpoint)
- [Where This App Gets Used (Project Map)](#where-this-app-gets-used-project-map)
- [Image Specifications](#image-specifications)
- [Tech Stack](#tech-stack)
- [Design Decisions and Why](#design-decisions-and-why)
- [Troubleshooting](#troubleshooting)
- [Author](#author)

---

## Why This App Exists

In a real platform engineering team, you don't build a new app for every infrastructure change. You have **one workload** that your platform serves. The app stays the same — the platform changes around it.

This app is that workload. It's the **patient** that every project (P1–P11) treats with a different infrastructure setup:

- P1 provisions the cluster it runs on (Terraform)
- P2 containerizes it and scans for vulnerabilities (Docker + Trivy)
- P3 deploys it with Helm charts
- P4 builds a CI/CD pipeline around it (Azure DevOps)
- P5 does the same with GitHub Actions + OIDC
- P6 monitors it with Prometheus + Grafana
- P7 deploys it via GitOps (ArgoCD)
- P8 enforces policies on it (Kyverno)
- P9 injects secrets into it (Azure Key Vault + CSI Driver)
- P10 controls its network traffic (NetworkPolicy + CNI)
- P11 combines everything into one production-grade platform

One app. Eleven platforms. That's the portfolio.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Atlas Platform App                   │
│                   (FastAPI + Uvicorn)                 │
│                                                       │
│  ┌─────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │ main.py │  │metrics.py│  │   simulator.py     │   │
│  │         │  │          │  │                     │   │
│  │ Routes: │  │ Counter  │  │ simulate_failure()  │   │
│  │ /health │  │ Histogram│  │  - crash (exit 1)   │   │
│  │ /info   │  │          │  │  - oom (mem alloc)   │   │
│  │ /metrics│  │ Labels:  │  │  - slow (30s delay)  │   │
│  │ /sim/*  │  │ method   │  │                     │   │
│  │         │  │ endpoint │  │ simulate_load()     │   │
│  │ Middle- │  │ status   │  │  - CPU burn threads  │   │
│  │ ware:   │  │          │  │  - configurable      │   │
│  │ auto-   │  │ Exposed  │  │    duration + %      │   │
│  │ tracks  │  │ at       │  │                     │   │
│  │ every   │  │ /metrics │  │                     │   │
│  │ request │  │          │  │                     │   │
│  └─────────┘  └──────────┘  └───────────────────────┘   │
│                                                       │
│  Middleware intercepts ALL requests automatically      │
│  → increments Counter (total requests by status)      │
│  → records Histogram (latency distribution)           │
│                                                       │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
              Runs on port 8000
              Non-root user: atlas
              Image size: < 100MB
```

---

## Endpoints — What They Do and Why

### `GET /health`

**Returns:**
```json
{"status": "healthy", "version": "1.0.0"}
```

**Why it exists:** Kubernetes uses this for **liveness probes** (is the container alive?) and **readiness probes** (is it ready to receive traffic?). If this endpoint stops returning 200, K8s kills and restarts the pod.

**Used in:** P3 (Helm — probe config), P4 (ADO — smoke test after deploy), P5 (GHA — smoke test), P11 (Capstone)

---

### `GET /info`

**Returns:**
```json
{"app": "atlas-platform-app", "env": "dev"}
```

**Why it exists:** Quick verification of which environment you're hitting. The `env` value comes from the `APP_ENV` environment variable, which gets set via Kubernetes ConfigMap in P3. If `APP_ENV` isn't set, defaults to `"dev"`.

**Used in:** P3 (ConfigMap → env var mapping), P4/P5 (post-deploy verification), P11

---

### `GET /metrics`

**Returns:** Prometheus text format (not JSON). Example:
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/health",status="200"} 42.0

# HELP http_request_duration_seconds Request latency in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",endpoint="/health",le="0.01"} 38.0
http_request_duration_seconds_bucket{method="GET",endpoint="/health",le="0.05"} 42.0
...
```

**Why it exists:** Prometheus scrapes this endpoint on a configured interval (default 15s). It collects two metrics:

1. **`http_requests_total`** (Counter) — Total requests, labeled by method, endpoint, and status code. Only goes up. Used for:
   - Request rate: `rate(http_requests_total[5m])`
   - Error rate: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100`

2. **`http_request_duration_seconds`** (Histogram) — Latency distribution across buckets [10ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s]. Used for:
   - P95 latency: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`

**Used in:** P6 (Prometheus scraping + Grafana dashboards + AlertManager rules), P11

---

### `POST /simulate/failure`

**Query param:** `mode` — one of `crash`, `oom`, `slow`

| Mode | What happens | K8s result | What you demo |
|---|---|---|---|
| `crash` | Process exits with code 1 | `CrashLoopBackOff` status | Troubleshooting a crashing pod |
| `oom` | Allocates memory infinitely | `OOMKilled` status | Container hitting memory limits |
| `slow` | Sleeps for 30 seconds | Readiness probe fails, pod removed from Service endpoints | Service degradation, probe tuning |

**Example:**
```bash
curl -X POST "http://localhost:8000/simulate/failure?mode=slow"
```

**Why it exists:** You can't demo troubleshooting skills with an app that never breaks. This endpoint lets you **deliberately cause** the exact failure scenarios interviewers ask about.

**Used in:** P6 (trigger alerts → verify AlertManager routing), P10 (test pod behavior under NetworkPolicy), P11 (live troubleshooting demo)

---

### `POST /simulate/load`

**Query params:**
- `duration` — seconds to run (5–300, default 30)
- `cpu_percent` — CPU burn intensity (10–100, default 80)

**Returns:**
```json
{"status": "load_generating", "duration": 30, "threads": 2}
```

**Why it exists:** Horizontal Pod Autoscaler (HPA) scales pods when CPU exceeds a threshold (e.g., 70%). This endpoint generates real CPU load so you can **demo HPA scaling live** — pods going from 1 → 3 → 5 while load is active, then scaling back down after cooldown.

**Example:**
```bash
curl -X POST "http://localhost:8000/simulate/load?duration=60&cpu_percent=90"
```

**Used in:** P3 (HPA demo), P6 (Grafana CPU panel spike + alert firing), P11 (scaling demo)

---

## Project File Structure

```
atlas-platform-app/
├── app/
│   ├── __init__.py              # Python package marker (empty file)
│   ├── main.py                  # FastAPI app — all routes + middleware
│   ├── metrics.py               # Prometheus Counter + Histogram setup
│   └── simulator.py             # Failure + load simulation logic
├── requirements.txt             # Pinned Python dependencies
├── Dockerfile                   # Multi-stage build, non-root, < 100MB
├── .dockerignore                # Excludes .git, __pycache__, docs from image
├── .gitignore                   # Standard Python ignores
├── .pre-commit-config.yaml      # gitleaks — blocks secrets from reaching Git
└── README.md                    # This file
```

---

## File-by-File Breakdown

### `requirements.txt`
```
fastapi==0.115.0
uvicorn==0.30.6
prometheus-client==0.21.0
```
- **fastapi** — Web framework. Handles routing, request parsing, response serialization.
- **uvicorn** — ASGI server. Runs the FastAPI app. Think of it like nginx but for Python async apps.
- **prometheus-client** — Exposes metrics in the text format Prometheus expects.
- **Why pinned versions (`==`):** Reproducible builds. `>=` or no pin means a future install could pull a breaking change. Production always pins.

### `app/__init__.py`
Empty file. Tells Python that `app/` is a package, enabling imports like `from app.metrics import REQUEST_COUNT`.

### `app/metrics.py`
Defines two Prometheus metric objects:
- **Counter (`http_requests_total`)** — Incremented on every request. Labels: method, endpoint, status. Counters only go up — Prometheus uses `rate()` to calculate per-second values.
- **Histogram (`http_request_duration_seconds`)** — Records request duration into predefined buckets. Labels: method, endpoint. This is how you get percentile latencies (p50, p95, p99) via `histogram_quantile()`.

These objects are imported by `main.py` and updated via middleware.

### `app/simulator.py`
Two functions:
- **`simulate_failure(mode)`** — Three failure modes:
  - `crash`: `os._exit(1)` kills the process immediately. Kubernetes sees the container exit and enters restart loop → CrashLoopBackOff.
  - `oom`: Infinite memory allocation. Hits the container memory limit → OOMKilled.
  - `slow`: 30-second sleep. Readiness probe (checking `/health`) times out → pod removed from Service endpoints → traffic stops flowing to it.

- **`simulate_load(duration, cpu_percent)`** — Spawns CPU-burning threads proportional to available cores. HPA monitors CPU metrics → triggers scale-out when threshold exceeded.

### `app/main.py`
The core application file. Contains:
- **Middleware (`track_metrics`)** — Wraps every incoming request. Before the response is sent, it records the method, endpoint, status code, and latency into the Prometheus metrics. You don't need to instrument each route individually — the middleware handles all of them.
- **5 route handlers** — One per endpoint. Each is minimal because the heavy logic lives in `metrics.py` and `simulator.py`.
- **`APP_ENV` environment variable** — Read at runtime via `os.getenv()`. In Kubernetes, this gets injected via ConfigMap (Project 3). Defaults to `"dev"` if not set.

### `Dockerfile`
Multi-stage build:
- **Stage 1 (builder):** Installs Python dependencies into `/install`. This stage includes pip, build tools, cache — things you don't want in production.
- **Stage 2 (production):** Copies only the installed packages from Stage 1. Creates a non-root user (`atlas`) with no shell and no home directory. Runs the app as this user.
- **Result:** Clean image under 100MB. No pip. No cache. No root.

### `.dockerignore`
Excludes `.git`, `__pycache__`, `.env`, IDE configs, and docs from the Docker build context. Smaller context = faster builds, smaller image.

### `.gitignore`
Standard Python ignores: `__pycache__`, virtual environments, IDE folders, `.env` files.

### `.pre-commit-config.yaml`
Configures gitleaks as a pre-commit hook. On every `git commit`, gitleaks scans staged files for API keys, passwords, tokens, and other secrets. If anything is detected, the commit is blocked. This is your DevSecOps baseline — secrets never reach Git history.

**Setup (one-time):**
```bash
pip install pre-commit
pre-commit install
```

---

## How to Run Locally

**Prerequisites:** Python 3.12+

```bash
# Clone the repo
git clone git@github.com:manoj-2606/atlas-platform-app.git
cd atlas-platform-app

# Install dependencies
pip install -r requirements.txt

# Run the app
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

App is now live at `http://localhost:8000`

`--reload` watches for file changes and auto-restarts (dev only, never in production).

---

## How to Run with Docker

```bash
# Build the image
docker build -t atlas-platform-app:1.0.0 .

# Run the container
docker run -d -p 8000:8000 --name atlas-app atlas-platform-app:1.0.0

# Verify
curl http://localhost:8000/health

# Check image size (should be < 100MB)
docker images atlas-platform-app:1.0.0

# Check it runs as non-root
docker exec atlas-app whoami
# Output: atlas

# Stop and remove
docker stop atlas-app && docker rm atlas-app
```

---

## How to Test Each Endpoint

```bash
# Health check — should return {"status": "healthy", "version": "1.0.0"}
curl http://localhost:8000/health

# App info — should return {"app": "atlas-platform-app", "env": "dev"}
curl http://localhost:8000/info

# Prometheus metrics — should return text-format metrics
curl http://localhost:8000/metrics

# Simulate slow response (safe — just delays 30s, then recovers)
curl -X POST "http://localhost:8000/simulate/failure?mode=slow"

# Simulate CPU load for 10 seconds at 50%
curl -X POST "http://localhost:8000/simulate/load?duration=10&cpu_percent=50"

# ⚠️ WARNING: These will kill/crash the process
# Only use in containers or K8s pods, not on your local machine
curl -X POST "http://localhost:8000/simulate/failure?mode=crash"
curl -X POST "http://localhost:8000/simulate/failure?mode=oom"
```

---

## Where This App Gets Used (Project Map)

| Project | What it uses from this app |
|---|---|
| P1: IaC Foundation | Deploys this app on the provisioned AKS cluster |
| P2: Container Security | Builds Docker image, scans with Trivy, pushes to ACR |
| P3: Helm Deployment | Deploys via Helm chart, configures probes on `/health`, HPA on `/simulate/load` |
| P4: ADO Pipeline | CI/CD builds, scans, deploys this app. Smoke test hits `/health` |
| P5: GHA + OIDC | Same as P4 but GitHub Actions with OIDC auth |
| P6: Observability | Prometheus scrapes `/metrics`. Grafana dashboards. Alerts on `/simulate/failure` |
| P7: GitOps ArgoCD | ArgoCD watches Helm chart changes and syncs this app |
| P8: Policy Kyverno | Validates this app's pods have resource limits, labels, non-root |
| P9: Secrets Management | Injects Key Vault secrets into this app via CSI driver |
| P10: K8s Networking | Tests NetworkPolicy rules between this app's frontend/backend |
| P11: Capstone | All of the above — full production platform running this app |

---

## Image Specifications

| Spec | Value |
|---|---|
| Base image | `python:3.12-slim` |
| Build type | Multi-stage (builder → production) |
| Final image size | < 100MB |
| Runs as | Non-root user (`atlas`) |
| Exposed port | 8000 |
| CVE policy | Zero critical vulnerabilities (Trivy scanned) |

---

## Tech Stack

| Component | Tool | Purpose |
|---|---|---|
| Language | Python 3.12 | Application runtime |
| Framework | FastAPI | REST API routing + request handling |
| Server | Uvicorn | ASGI server (runs FastAPI) |
| Metrics | prometheus-client | Exposes Prometheus-format metrics |
| Secrets scanning | gitleaks | Pre-commit hook — blocks secrets from Git |

---

## Design Decisions and Why

| Decision | Why |
|---|---|
| FastAPI over Flask | Async by default, auto-generates OpenAPI docs, better performance |
| Prometheus client over StatsD | Prometheus pull model is K8s-native. No sidecar needed — just scrape `/metrics` |
| `os._exit(1)` for crash simulation | Kills process immediately without cleanup. Mimics real crash behavior K8s sees |
| Threading for CPU load | Scales with available cores. `cpu_percent` controls thread count relative to `os.cpu_count()` |
| Non-root container user | Security baseline. Kyverno (P8) blocks root containers — this is compliant from Day 1 |
| Pinned dependency versions | Reproducible builds. Same `pip install` produces same result today and 6 months from now |
| Multi-stage Docker build | Final image has no pip, no cache, no build tools. Smaller attack surface, faster pulls |
| Separate metrics.py/simulator.py | Single responsibility. main.py stays clean. Each concern is isolated and testable |

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'app'` | Running from wrong directory | `cd` into repo root (where `app/` folder is) |
| `/metrics` returns empty | No requests made yet | Hit `/health` a few times first, then check `/metrics` |
| `simulate/failure?mode=crash` kills local Python | Expected — `os._exit(1)` terminates the process | Only use crash/oom modes inside Docker or K8s |
| Docker image > 100MB | Likely caching issue | Rebuild with `docker build --no-cache -t atlas-platform-app:1.0.0 .` |
| `pre-commit` not running on commit | Hook not installed | Run `pre-commit install` in repo root |
| Port 8000 already in use | Another process on 8000 | `lsof -i :8000` to find it, kill it, or use `--port 8001` |

---

## Author

**Manoj Kumar R**
- GitHub: [github.com/manoj-2606](https://github.com/manoj-2606)
- LinkedIn: [linkedin.com/in/manoj2606](https://linkedin.com/in/manoj2606)

---

*This app is part of the Atlas Platform portfolio — 11 production-grade Azure cloud projects demonstrating infrastructure, security, observability, policy, and cost management.*