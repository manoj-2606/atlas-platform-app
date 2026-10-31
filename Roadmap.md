# Monarch's 11-Project Cloud Portfolio Roadmap

> **Rule:** Every project is created and destroyed in ONE DAY.
> **Rule:** Every project runs on Azure Pay-As-You-Go. Cost-controlled.
> **Rule:** Every project is a standalone GitHub repo with clean README + architecture diagram.
> **Rule:** No hello-world. Real workloads. Real problems. Real solutions.
> **Rule:** FinOps and DevSecOps are baked INTO every project — not separate learning tracks.

---

## The Standard Workload (Build This First)

Before any project, build **one reusable app** that every project deploys. This is what real platform engineers do — the app stays the same, the platform changes.

**`monarch-platform-app`** — A Python FastAPI REST API with:

```
GET  /health          → {"status": "healthy", "version": "1.0.0"}
GET  /metrics          → Prometheus-format metrics (request count, latency histogram, error rate)
GET  /info             → {"app": "monarch-platform-app", "env": "dev|prod"}
POST /simulate/failure → Triggers OOMKill / CrashLoop / slow response (for troubleshooting projects)
POST /simulate/load    → Generates CPU/memory load (for HPA/scaling projects)
```

**Why this design:**
- `/health` → liveness/readiness probes in K8s
- `/metrics` → Prometheus scraping without sidecar hacks
- `/simulate/failure` → you can demo troubleshooting CrashLoopBackOff, OOMKilled live
- `/simulate/load` → you can demo HPA scaling live

**Dockerfile:** Multi-stage build. Final image runs as non-root. Under 100MB.

**Time to build:** 2-3 hours once. Reused across all 11 projects.

**Repo:** `github.com/manoj-2606/monarch-platform-app`

---

## The Shared Base Module

Every project that needs AKS uses the same Terraform base. Not copy-pasted — imported as a module.

**`monarch-infra-base`** — Terraform module that provisions:

```
- Resource Group
- VNet (10.0.0.0/16) + AKS Subnet (10.0.1.0/24)
- AKS Cluster (1 node, Standard_B2s — cheapest, FinOps: right-sized for non-prod)
- ACR (Basic SKU)
- Default cost tags on EVERY resource: { team, project, env, cost-center, created-by }
- Outputs: kube_config, acr_login_server, resource_group_name
```

**FinOps built in from Day 1:** The default_tags block ensures every resource is tagged for cost attribution. This is the #1 thing missing in 90% of Azure environments. You do it by default.

**DevSecOps built in from Day 1:** Every repo includes a `.pre-commit-config.yaml` with `gitleaks` to prevent secrets from ever reaching Git history. Set this up once, copy to every project.

**Cost per day (8 hours):**
- AKS control plane: **FREE**
- 1x Standard_B2s node: **~$0.04/hr = ~$0.32/day**
- ACR Basic: **~$0.17/day**
- VNet/NSG: **FREE**
- **Total: ~₹45-50 per project day**

**Critical:** Every project README includes a `## Cleanup` section with:
```bash
terraform destroy -auto-approve
# Verify in Azure Portal → Resource Groups → confirm empty
```

**Repo:** `github.com/manoj-2606/monarch-infra-base`

---

## The 11 Projects

---

### Project 1: Infrastructure as Code Foundation
**`monarch-aks-terraform`**

**What you build:**
Provision a production-style AKS cluster with Terraform using Azure Verified Modules (AVM). Hub-spoke network topology (simplified). Remote state in Azure Blob with lease locking. Feature flags via Terraform variables to toggle components on/off.

**Skills covered:**
- Terraform (modules, state management, AVM, variables, outputs, feature flags)
- Azure Networking (VNet, Subnets, NSG)
- AKS provisioning
- Remote state (azurerm backend, blob lease locking)
- [FinOps] Cost tagging strategy — default_tags on every resource
- [FinOps] Right-sizing — why B2s over D2s, spot nodes for non-prod
- [DevSecOps] gitleaks pre-commit hook — secrets never reach Git

**What goes in the repo:**
```
├── main.tf                  # Root module
├── variables.tf             # All configurable inputs
├── outputs.tf               # Cluster config, ACR server
├── backend.tf               # Remote state config
├── environments/
│   ├── dev.tfvars
│   └── prod.tfvars
├── modules/
│   └── aks-cluster/         # Reusable AKS module
├── docs/
│   └── architecture.png     # Hub-spoke diagram (draw.io)
├── .pre-commit-config.yaml  # gitleaks — secrets never reach Git
├── TROUBLESHOOTING.md       # State recovery story documented
├── COST.md                  # Why B2s, why Basic ACR, tag strategy explained
└── README.md
```

**Interview story unlocked:** "I delivered an Azure Landing Zone in 1 week using AVM. Here's the architecture. Here's how I handle state recovery. Every resource is tagged for cost attribution from Day 1."

**Estimated time:** 4-5 hours (including README + diagram)
**Estimated cost:** ~₹50

---

### Project 2: Containerization & Image Security
**`monarch-container-security`**

**What you build:**
Take the `monarch-platform-app`, write a production-grade multi-stage Dockerfile. Scan with Trivy for CVEs. Push to ACR. Run as non-root user. Layer caching optimized. Snyk scan for code-level vulnerabilities.

**Skills covered:**
- Docker (multi-stage builds, layer caching, non-root, .dockerignore)
- Trivy (image scanning, severity filtering, CI integration)
- Snyk (code/IaC scanning)
- ACR (push, tagging strategy)
- [DevSecOps] SBOM generation via Trivy (trivy image --format spdx-json)
- [DevSecOps] gitleaks pre-commit hook
- [DevSecOps] Image provenance — what's inside your container, auditable

**What goes in the repo:**
```
├── app/                     # FastAPI app source
├── Dockerfile               # Multi-stage, non-root, < 100MB
├── .dockerignore
├── .trivyignore             # Known acceptable CVEs with justification
├── scripts/
│   ├── scan.sh              # Trivy scan with threshold (CRITICAL=0)
│   ├── generate-sbom.sh     # trivy image --format spdx-json → sbom.json
│   └── snyk-scan.sh         # Snyk IaC scan
├── .pre-commit-config.yaml  # gitleaks
├── docs/
│   ├── image-layers.png     # Screenshot: docker history showing layer sizes
│   ├── trivy-report.png     # Screenshot: clean scan output
│   └── sbom-sample.json     # Sample SBOM output — shows supply chain awareness
└── README.md                # Includes "Why multi-stage" + SBOM explanation
```

**Interview story unlocked:** "I enforce zero critical CVEs in every image before it reaches any registry. Here's how the scanning pipeline works."

**Estimated time:** 3-4 hours
**Estimated cost:** ~₹30 (ACR push only, no AKS needed)

---

### Project 3: Kubernetes Deployment with Helm
**`monarch-helm-deployment`**

**What you build:**
Create a Helm chart for `monarch-platform-app`. Deploy to AKS with environment-specific values (dev/prod). NGINX Ingress controller. Liveness and readiness probes pointing at `/health`. Resource requests and limits set. ConfigMaps for app config. Secrets for sensitive values.

**Skills covered:**
- Helm (chart structure, values files, templates, helpers)
- Kubernetes (Deployments, Services, Ingress, ConfigMaps, Secrets)
- NGINX Ingress Controller
- Liveness/Readiness Probes
- Resource Limits & Requests

**What goes in the repo:**
```
├── terraform/               # Uses monarch-infra-base module
├── charts/
│   └── monarch-app/
│       ├── Chart.yaml
│       ├── values.yaml           # Defaults
│       ├── values-dev.yaml       # Dev overrides
│       ├── values-prod.yaml      # Prod overrides (higher replicas, stricter limits)
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── configmap.yaml
│           ├── secret.yaml
│           ├── hpa.yaml
│           └── _helpers.tpl
├── docs/
│   └── helm-architecture.png
└── README.md
```

**Interview story unlocked:** "I use Helm with per-environment values files. Here's how I structure charts for dev vs prod, and why I chose Ingress over LoadBalancer for cost."

**Estimated time:** 5-6 hours
**Estimated cost:** ~₹50

---

### Project 4: CI/CD with Azure DevOps
**`monarch-ado-pipeline`**

**What you build:**
Full Azure DevOps pipeline: Build Docker image → Trivy scan (fail on CRITICAL) → Push to ACR → Helm deploy to AKS (dev) → Smoke test (curl /health) → Manual approval gate → Helm deploy to AKS (prod) → Notify via webhook.

**Skills covered:**
- Azure DevOps (YAML pipelines, stages, jobs, tasks)
- CI/CD (build → scan → push → deploy → test → promote)
- Approval Gates
- Variable Groups + Azure Key Vault integration
- Pipeline triggers (branch-based)

**What goes in the repo:**
```
├── azure-pipelines.yml       # Multi-stage pipeline
├── pipelines/
│   ├── templates/
│   │   ├── build-scan.yml    # Reusable build + Trivy template
│   │   ├── deploy.yml        # Reusable Helm deploy template
│   │   └── smoke-test.yml    # curl /health + assert 200
│   └── variables/
│       ├── dev.yml
│       └── prod.yml
├── terraform/                # Infra provisioning
├── charts/                   # Helm chart (from Project 3)
├── docs/
│   ├── pipeline-flow.png     # Pipeline stages diagram
│   └── approval-gate.png     # Screenshot of approval in action
└── README.md
```

**Interview story unlocked:** "Here's my full CI/CD pipeline — build, scan, deploy, test, promote with manual approval. Zero critical CVEs reach production."

**Estimated time:** 5-6 hours
**Estimated cost:** ~₹50

---

### Project 5: CI/CD with GitHub Actions + OIDC
**`monarch-gha-oidc-pipeline`**

**What you build:**
Same pipeline as Project 4 but in GitHub Actions. The critical difference: authenticate to Azure using OIDC Workload Identity Federation — no stored secrets. Federated credential setup via Terraform.

**Skills covered:**
- GitHub Actions (workflows, jobs, reusable workflows)
- OIDC / Workload Identity Federation (zero stored credentials)
- Terraform (federated identity credential provisioning)
- CI/CD (same pattern, different tool)

**What goes in the repo:**
```
├── .github/
│   └── workflows/
│       ├── ci.yml            # Build + scan + push
│       ├── cd-dev.yml        # Deploy to dev
│       ├── cd-prod.yml       # Deploy to prod (requires approval)
│       └── reusable/
│           ├── docker-build.yml
│           └── helm-deploy.yml
├── terraform/
│   ├── main.tf               # AKS + ACR + OIDC federated credential
│   └── oidc.tf               # App registration + federated identity
├── docs/
│   ├── oidc-flow.png         # 5-box OIDC token exchange diagram
│   └── gha-vs-ado.md         # Comparison: when to use which
└── README.md
```

**Interview story unlocked:** "I replaced service principal secrets with OIDC federation. Here's the token exchange flow. Zero credentials stored anywhere."

**Estimated time:** 5-6 hours
**Estimated cost:** ~₹50

---

### Project 6: Observability Stack
**`monarch-observability-stack`**

**What you build:**
Deploy Prometheus + Grafana + AlertManager on AKS using Helm (kube-prometheus-stack). Write 5 custom PromQL queries. Build a Grafana dashboard with panels for: request rate, error rate, latency p95, pod restarts, node CPU/memory. AlertManager routes alerts to a webhook (Slack-style). Define SLI/SLO for the app.

**Skills covered:**
- Prometheus (deployment, PromQL, alert rules, scrape configs)
- Grafana (dashboards, panels, variables, data sources)
- AlertManager (routing, receivers, inhibition)
- SRE Concepts (SLI, SLO, Error Budgets)
- Kubernetes monitoring
- [FinOps] Kubecost — cost per namespace, deployment, pod. The FinOps tool for K8s.
- [FinOps] Azure Cost Management integration — budget alerts via Azure Monitor

**What goes in the repo:**
```
├── terraform/                # AKS infra
├── monitoring/
│   ├── values-prometheus.yaml    # kube-prometheus-stack overrides
│   ├── alerts/
│   │   ├── pod-alerts.yaml       # CrashLoopBackOff, OOMKilled, high restart
│   │   ├── app-alerts.yaml       # Error rate > 1%, latency p95 > 500ms
│   │   ├── node-alerts.yaml      # CPU > 80%, memory > 85%
│   │   └── cost-alerts.yaml      # Azure budget threshold alert
│   ├── dashboards/
│   │   ├── monarch-app.json      # Grafana dashboard JSON
│   │   └── kubecost-summary.png  # Kubecost cost-per-namespace screenshot
│   ├── finops/
│   │   ├── kubecost-values.yaml  # Kubecost Helm install overrides
│   │   └── budget-alert.json     # Azure Cost Management budget alert config
│   └── slo/
│       └── slo-definition.md     # SLI: availability, latency. SLO: 99.9%, p95 < 200ms
├── docs/
│   ├── grafana-dashboard.png     # Dashboard screenshot
│   ├── alertmanager-routing.png  # Alert routing diagram
│   └── promql-cheatsheet.md      # Your 5 queries explained
├── scripts/
│   └── generate-load.sh          # Hit /simulate/load to trigger alerts
└── README.md
```

**5 PromQL queries to include:**
```promql
# 1. Request rate per second
rate(http_requests_total[5m])

# 2. Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# 3. P95 latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 4. Pod restart count (last hour)
increase(kube_pod_container_status_restarts_total[1h])

# 5. Error budget burn rate
1 - (sum(rate(http_requests_total{status!~"5.."}[1h])) / sum(rate(http_requests_total[1h])))
```

**Interview story unlocked:** "I can define SLIs, set SLOs, calculate error budget burn rate, and build alerting that routes to the right team."

**Estimated time:** 6-7 hours
**Estimated cost:** ~₹60 (slightly longer cluster uptime)

---

### Project 7: GitOps with ArgoCD
**`monarch-gitops-argocd`**

**What you build:**
Install ArgoCD on AKS. Configure it to watch a Git repo for Helm chart changes. Deploy `monarch-platform-app` via ArgoCD Application CRD. Demo: push a change to the chart → ArgoCD auto-syncs → new version deployed. Demo: rollback to previous version via ArgoCD UI.

**Skills covered:**
- ArgoCD (installation, Application CRD, sync policies, rollback)
- GitOps (Git as single source of truth, declarative deployment)
- Kubernetes (declarative resource management)
- Helm (ArgoCD + Helm integration)

**What goes in the repo:**
```
├── terraform/                    # AKS infra
├── argocd/
│   ├── install/
│   │   └── values-argocd.yaml    # ArgoCD Helm values (resource limits, ingress)
│   ├── applications/
│   │   ├── monarch-app-dev.yaml  # ArgoCD Application for dev
│   │   └── monarch-app-prod.yaml # ArgoCD Application for prod
│   └── appprojects/
│       └── monarch-project.yaml  # ArgoCD AppProject with RBAC
├── charts/                       # Helm chart (same as Project 3)
├── docs/
│   ├── argocd-flow.png           # Git push → ArgoCD sync → K8s deploy diagram
│   ├── argocd-ui.png             # Screenshot of ArgoCD dashboard
│   └── gitops-vs-push.md         # Why GitOps over push-based CI/CD
└── README.md
```

**Interview story unlocked:** "I use ArgoCD for GitOps. Git is the single source of truth. I can show you a commit-to-deploy flow and a rollback in under 30 seconds."

**Estimated time:** 5-6 hours
**Estimated cost:** ~₹50

---

### Project 8: Policy Enforcement with Kyverno
**`monarch-policy-kyverno`**

**What you build:**
Install Kyverno on AKS. Write and enforce policies: no `latest` tag allowed, resource limits required on every pod, specific labels required (`app`, `team`, `env`, `cost-center`), no privileged containers, automount service account token disabled by default. Demo: try to deploy a non-compliant resource → blocked. Generate policy reports. The `cost-center` label requirement is a FinOps policy — every workload must be attributable for cost allocation.

**Skills covered:**
- Kyverno (ClusterPolicy, Policy, validate/mutate/generate rules)
- Kubernetes (admission controllers, security contexts)
- Policy-as-Code (governance, compliance)
- [FinOps] cost-center label enforcement — workloads without cost attribution are blocked
- [DevSecOps] disallow-privileged, disallow-default-sa-token — security at admission level

**What goes in the repo:**
```
├── terraform/                     # AKS infra
├── policies/
│   ├── validate/
│   │   ├── disallow-latest-tag.yaml
│   │   ├── require-resource-limits.yaml
│   │   ├── require-labels.yaml
│   │   ├── disallow-privileged.yaml
│   │   └── disallow-default-sa-token.yaml
│   ├── mutate/
│   │   └── add-default-labels.yaml       # Auto-inject team label
│   └── generate/
│       └── default-networkpolicy.yaml    # Auto-create NetworkPolicy per namespace
├── tests/
│   ├── compliant-pod.yaml                # Should pass
│   ├── non-compliant-latest.yaml         # Should be blocked
│   ├── non-compliant-no-limits.yaml      # Should be blocked
│   └── non-compliant-privileged.yaml     # Should be blocked
├── docs/
│   ├── policy-report.png                 # Kyverno policy report screenshot
│   └── blocked-deployment.png            # Screenshot of blocked non-compliant deploy
└── README.md
```

**Interview story unlocked:** "I enforce governance at the cluster level. No one deploys without resource limits, proper labels, or with the latest tag. Here's the proof."

**Estimated time:** 4-5 hours
**Estimated cost:** ~₹50

---

### Project 9: Secrets Management
**`monarch-secrets-management`**

**What you build:**
Set up Azure Key Vault with Terraform. Install Azure Key Vault CSI Driver on AKS. Mount secrets into pods as volumes (not env vars — more secure). Use Workload Identity to authenticate pod → Key Vault (no stored credentials). Demo: rotate a secret in Key Vault → pod picks up new value without restart.

**Skills covered:**
- Azure Key Vault (provisioning, access policies, secret rotation)
- CSI Secrets Store Driver (SecretProviderClass CRD)
- Workload Identity (Managed Identity → Federated Credential → Pod)
- Kubernetes (volumes, projected volumes, pod identity)

**What goes in the repo:**
```
├── terraform/
│   ├── main.tf                      # AKS + Key Vault
│   ├── keyvault.tf                  # KV + secrets + access policies
│   └── workload-identity.tf         # Managed Identity + Federation
├── k8s/
│   ├── secret-provider-class.yaml   # CSI driver config
│   ├── service-account.yaml         # Annotated with Workload Identity
│   └── deployment.yaml              # Pod mounting KV secrets as volume
├── scripts/
│   ├── rotate-secret.sh             # Rotate secret in KV + verify pod picks it up
│   └── verify-mount.sh              # kubectl exec → cat mounted secret
├── docs/
│   ├── secrets-flow.png             # Pod → Workload Identity → Key Vault diagram
│   └── why-not-env-vars.md          # Security comparison: volume vs env var
└── README.md
```

**Interview story unlocked:** "I never store secrets in pipelines or env vars. Pods authenticate to Key Vault using Workload Identity — zero credentials in the cluster."

**Estimated time:** 5-6 hours
**Estimated cost:** ~₹55

---

### Project 10: Kubernetes Networking Deep Dive
**`monarch-k8s-networking`**

**What you build:**
Deploy two microservices that talk to each other via ClusterIP. Expose frontend via Ingress. Implement NetworkPolicy to restrict traffic (frontend → backend allowed, backend → frontend blocked, deny all by default). Explore CoreDNS resolution. Demonstrate CNI behavior (Azure CNI with overlay). Debug a "service not reachable" scenario step-by-step.

**Skills covered:**
- Kubernetes Networking (Services, DNS, CNI)
- NetworkPolicy (ingress/egress rules, deny-all-by-default)
- CoreDNS (service discovery, `nslookup` debugging)
- Ingress (NGINX, path-based routing)
- Azure CNI (pod IP allocation, subnet sizing)
- Troubleshooting (network debugging methodology)

**What goes in the repo:**
```
├── terraform/                        # AKS with Azure CNI
├── k8s/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── ingress.yaml                  # Path-based: /api → backend, / → frontend
│   └── network-policies/
│       ├── deny-all.yaml             # Default deny
│       ├── allow-frontend-to-backend.yaml
│       └── allow-ingress-to-frontend.yaml
├── troubleshooting/
│   └── NETWORK_DEBUG.md              # Step-by-step: "Service not reachable" runbook
│       # 1. kubectl get svc → check ClusterIP exists
│       # 2. kubectl get endpoints → check pod IPs populated
│       # 3. kubectl exec → nslookup service-name
│       # 4. kubectl exec → curl service-ip:port
│       # 5. Check NetworkPolicy → is traffic allowed?
│       # 6. Check pod labels → do they match service selector?
├── docs/
│   ├── network-topology.png          # VNet → Subnet → Pod IPs diagram
│   └── dns-resolution.png            # CoreDNS flow diagram
└── README.md
```

**Interview story unlocked:** "I can debug 'service not reachable' in K8s in under 5 minutes. Here's my runbook. I understand CNI, CoreDNS, and NetworkPolicy at the packet level."

**Estimated time:** 5-6 hours
**Estimated cost:** ~₹50

---

### Project 11: Production-Grade Platform (Capstone)
**`monarch-production-platform`**

**What you build:**
Everything from Projects 1-10 combined into ONE production-grade platform. This is your flagship repo. One `terraform apply` brings up the entire stack. One `terraform destroy` tears it all down.

**The full stack:**
- Terraform provisions: AKS (2 nodes) + ACR + Key Vault + VNet + Managed Identities
- Docker multi-stage image built and scanned with Trivy + SBOM generated
- Helm chart deploys the app with per-env values
- ArgoCD manages deployment via GitOps
- Prometheus + Grafana + AlertManager + Kubecost for observability + cost visibility
- Kyverno enforces policies (including cost-center label requirement)
- Azure Key Vault secrets mounted via CSI driver
- NetworkPolicies restrict traffic
- NGINX Ingress exposes the app
- SLI/SLO defined with error budget dashboard
- Full CI/CD pipeline (ADO or GHA) with gitleaks + Trivy + SBOM + smoke test
- Cost tagging on every Azure resource — full FinOps attribution
- Azure Cost Management budget alert configured

**What goes in the repo:**
```
├── terraform/
│   ├── main.tf
│   ├── aks.tf
│   ├── acr.tf
│   ├── keyvault.tf
│   ├── networking.tf
│   ├── identity.tf
│   └── environments/
│       ├── dev.tfvars
│       └── prod.tfvars
├── app/
│   ├── src/                          # FastAPI app
│   ├── Dockerfile
│   └── .trivyignore
├── charts/
│   └── monarch-platform/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-prod.yaml
│       └── templates/
├── platform/
│   ├── argocd/
│   ├── monitoring/
│   │   ├── prometheus-values.yaml
│   │   ├── alerts/
│   │   ├── dashboards/
│   │   └── slo/
│   ├── policies/
│   │   └── kyverno/
│   ├── secrets/
│   │   └── csi-driver/
│   └── network-policies/
├── pipelines/
│   ├── azure-pipelines.yml
│   └── .github/workflows/ci-cd.yml
├── security/
│   ├── .pre-commit-config.yaml       # gitleaks — copy to root
│   ├── sbom/
│   │   └── generate-sbom.sh          # SBOM generation script
│   └── SECURITY.md                   # Security posture summary: what's enforced and how
├── finops/
│   ├── cost-analysis.md              # Actual Azure cost breakdown per resource
│   ├── tagging-strategy.md           # Why every resource is tagged, how to query cost by tag
│   ├── kubecost-values.yaml          # Kubecost Helm config
│   └── kubecost-dashboard.png        # Cost per namespace screenshot
├── docs/
│   ├── ARCHITECTURE.md               # Full platform architecture document
│   ├── architecture-diagram.png      # The money diagram
│   ├── slo-dashboard.png
│   ├── argocd-sync.png
│   └── pipeline-flow.png
├── scripts/
│   ├── setup.sh                      # One command to deploy everything
│   ├── teardown.sh                   # One command to destroy everything
│   ├── demo-failure.sh               # Trigger CrashLoop for live troubleshooting demo
│   └── demo-scale.sh                 # Trigger HPA scaling for live demo
├── TROUBLESHOOTING.md
├── RUNBOOK.md                        # Incident response runbook
└── README.md                         # The README that gets you hired
```

**Interview story unlocked:** "This is my production-grade Azure platform. Terraform provisions it. ArgoCD deploys it. Prometheus monitors it. Kyverno governs it. One command up, one command down. I've run it 20+ times — it costs ₹80 per day."

**Estimated time:** 7-8 hours (most work is integration + README + diagrams)
**Estimated cost:** ~₹80 (2-node AKS for this one to handle the full stack)

---

## Skills Coverage Matrix

| Skill | P1 | P2 | P3 | P4 | P5 | P6 | P7 | P8 | P9 | P10 | P11 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Terraform | ✅ | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Docker | | ✅ | | ✅ | ✅ | | | | | | ✅ |
| Kubernetes | ✅ | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Helm | | | ✅ | ✅ | ✅ | ✅ | ✅ | | | | ✅ |
| Azure DevOps CI/CD | | | | ✅ | | | | | | | ✅ |
| GitHub Actions | | | | | ✅ | | | | | | ✅ |
| OIDC / Workload Identity | | | | | ✅ | | | | ✅ | | ✅ |
| Prometheus | | | | | | ✅ | | | | | ✅ |
| Grafana | | | | | | ✅ | | | | | ✅ |
| AlertManager | | | | | | ✅ | | | | | ✅ |
| ArgoCD / GitOps | | | | | | | ✅ | | | | ✅ |
| Kyverno | | | | | | | | ✅ | | | ✅ |
| Trivy / Snyk | | ✅ | | ✅ | ✅ | | | | | | ✅ |
| Azure Key Vault | | | | ✅ | | | | | ✅ | | ✅ |
| Secrets (CSI Driver) | | | | | | | | | ✅ | | ✅ |
| Networking (CNI/DNS/Ingress) | ✅ | | ✅ | | | | | | | ✅ | ✅ |
| NetworkPolicy | | | | | | | | | | ✅ | ✅ |
| SRE (SLI/SLO/Error Budget) | | | | | | ✅ | | | | | ✅ |
| Bash scripting | | ✅ | | | | ✅ | | | ✅ | | ✅ |
| Linux Internals | | ✅ | | | | | | | | | ✅ |
| **--- FinOps ---** | | | | | | | | | | | |
| Cost Tagging Strategy | ✅ | | | | | | | ✅ | | | ✅ |
| Right-Sizing (node/SKU selection) | ✅ | | | | | | | | | | ✅ |
| Kubecost (K8s cost visibility) | | | | | | ✅ | | | | | ✅ |
| Azure Cost Mgmt / Budget Alerts | | | | | | ✅ | | | | | ✅ |
| **--- DevSecOps ---** | | | | | | | | | | | |
| gitleaks (pre-commit secrets scan) | ✅ | ✅ | | ✅ | ✅ | | | | | | ✅ |
| SBOM Generation (Trivy spdx-json) | | ✅ | | ✅ | ✅ | | | | | | ✅ |
| Container Security (non-root, hardened) | | ✅ | | | | | | ✅ | | | ✅ |
| Policy-as-Code (admission control) | | | | | | | | ✅ | | | ✅ |

**Every skill from your tracker is covered at least twice. FinOps and DevSecOps are woven in — not bolted on. Project 11 covers everything.**

---

## Execution Order & Timeline

| # | Project | When to Build | Depends On |
|---|---------|--------------|------------|
| 0 | Standard Workload (FastAPI app) | Week 1 | Nothing |
| 0 | Shared Infra Base Module | Week 1 | Nothing |
| 1 | Terraform Foundation | After K8s basics (Feb 2026) | Module 0 |
| 2 | Container Security | After Docker basics (Feb 2026) | App built |
| 3 | Helm Deployment | After Helm learning (Mar 2026) | P1 |
| 4 | ADO CI/CD Pipeline | After pipeline learning (Mar 2026) | P2, P3 |
| 5 | GHA + OIDC Pipeline | Same week as P4 or week after | P2, P3 |
| 6 | Observability Stack | After Prometheus/Grafana (Jun 2026) | P1, P3 |
| 7 | GitOps ArgoCD | After ArgoCD learning (Jul 2026) | P3 |
| 8 | Policy Kyverno | After Kyverno learning (Jul 2026) | P1 |
| 9 | Secrets Management | After KV learning (Aug 2026) | P1 |
| 10 | K8s Networking | After CNI/DNS learning (Aug 2026) | P1 |
| 11 | Capstone Platform | Oct-Nov 2026 | ALL above |

---

## Cost Summary

| Project | Azure Cost (₹) | Time (hrs) |
|---------|----------------|------------|
| Standard App + Base Module | ₹0 (local only) | 3-4 |
| P1: Terraform Foundation | ~₹50 | 4-5 |
| P2: Container Security | ~₹30 | 3-4 |
| P3: Helm Deployment | ~₹50 | 5-6 |
| P4: ADO Pipeline | ~₹50 | 5-6 |
| P5: GHA + OIDC | ~₹50 | 5-6 |
| P6: Observability | ~₹60 | 6-7 |
| P7: ArgoCD GitOps | ~₹50 | 5-6 |
| P8: Kyverno Policies | ~₹50 | 4-5 |
| P9: Secrets Management | ~₹55 | 5-6 |
| P10: K8s Networking | ~₹50 | 5-6 |
| P11: Capstone | ~₹80 | 7-8 |
| **TOTAL** | **~₹575** | **~60-65 hrs** |

**Total cost for 11 production-grade projects: under ₹600.**
That's less than one dinner out. For a GitHub portfolio that gets you to Europe.

---

## GitHub Presentation Rules

Every repo MUST have:

1. **README.md** — Problem → Solution → Architecture Diagram → How to Run → How to Destroy → Cost → What I Learned
2. **Architecture diagram** — draw.io PNG exported, embedded in README
3. **Cost section** — actual Azure spend for that project day
4. **Cleanup section** — `terraform destroy` command + verification
5. **Topics/Tags** — `azure`, `devops`, `terraform`, `kubernetes`, `helm`, `ci-cd` (for GitHub search discovery)
6. **No dead repos** — if you built it, it has a README. If it doesn't have a README, delete it.

**Pin these 4 on your GitHub profile:**
1. `monarch-production-platform` (P11 — the capstone)
2. `monarch-ado-pipeline` (P4 — CI/CD)
3. `monarch-observability-stack` (P6 — monitoring)
4. `monarch-gha-oidc-pipeline` (P5 — security/OIDC)

---

## What This Portfolio Says to a European Hiring Manager

> "This engineer doesn't just use tools — he builds platforms.
> He understands infrastructure, security, observability, policy, and cost.
> He can provision, deploy, monitor, govern, and tear down a production stack in one day.
> He documents everything. He thinks about cost. He thinks about security.
> He's ready for a senior platform role."

That's the signal. That's what gets you the interview.

---

---

## FinOps — What You Cover (Not a Separate Track)

FinOps is cost visibility and accountability for cloud. At 4-6 YOE, interviewers expect you to answer: "How do you think about cloud cost in your work?" — not to be a certified FinOps practitioner.

**What is already built into your 11 projects:**

| FinOps Concept | Where You Cover It | Interview Answer |
|---|---|---|
| Cost tagging strategy | P1 (every Terraform resource gets default_tags), P8 (Kyverno enforces cost-center label), P11 | "Every resource I provision is tagged for cost attribution. It is a Terraform default, not an afterthought." |
| Right-sizing | P1 (B2s over D2s reasoning), Base Module (cheapest viable SKU) | "I choose the cheapest SKU that meets the requirement. Dev clusters get B2s, prod gets D4s. I document why." |
| Kubecost | P6 (installed, dashboard screenshot), P11 | "I use Kubecost to see cost per namespace and deployment. I can tell you which microservice costs the most." |
| Azure Cost Management | P6 (budget alerts), P11 | "I configure budget alerts so we know before we overspend, not after." |
| Azure Advisor | Know it exists, what it recommends | "Azure Advisor flags underutilized resources. I check it monthly." |
| Destroy after use | Every project (terraform destroy) | "My dev environments are ephemeral. Spin up in the morning, tear down at night. Zero overnight cost." |

**Tools to know by name (not to master):**
- Kubecost — K8s cost allocation
- Azure Cost Management — budgets, alerts, cost analysis
- Azure Advisor — optimization recommendations
- Infracost — Terraform cost estimation before apply (optional, mention if asked)

**One-liner for interviews:** "I think about cost from Day 1. Every resource is tagged, dev environments are ephemeral, and I use Kubecost to attribute cost to teams. Cost is a first-class metric alongside latency and error rate."

---

## DevSecOps — What You Cover (Not a Separate Track)

DevSecOps is security built into the pipeline, not bolted on after deployment. You already cover 80% of it.

**What is already built into your 11 projects:**

| DevSecOps Concept | Where You Cover It | Interview Answer |
|---|---|---|
| Image scanning (Trivy) | P2, P4, P5, P11 | "Every image is scanned for CVEs before it reaches any registry. Zero critical CVEs allowed." |
| IaC scanning (Snyk) | P2 | "Terraform and Dockerfiles are scanned for misconfigurations before apply." |
| SBOM generation | P2, P4, P5, P11 | "I generate an SBOM for every image so we know exactly what is inside our containers." |
| Secrets scanning (gitleaks) | P1, P2, P4, P5, P11 | "Pre-commit hook prevents secrets from ever reaching Git history." |
| No stored credentials (OIDC) | P5, P9, P11 | "Pipelines authenticate via OIDC federation. Zero credentials stored anywhere." |
| Secrets management (KV + CSI) | P9, P11 | "Secrets are in Key Vault, mounted as volumes, accessed via Workload Identity. Not in env vars." |
| Non-root containers | P2, P11 | "Every container runs as non-root. No exceptions." |
| Policy enforcement (Kyverno) | P8, P11 | "Admission control blocks privileged containers, missing resource limits, and latest tags at the cluster level." |
| Network segmentation | P10, P11 | "Default deny NetworkPolicy. Only explicitly allowed traffic flows." |

**Tools to know by name (not to master):**
- Trivy — image scanning + SBOM generation (you use this)
- Snyk — code + IaC scanning (you use this)
- gitleaks — pre-commit secrets scanning (you use this)
- Kyverno — admission control policies (you use this)
- SonarQube / Semgrep — SAST (static code analysis). Know the concept. Scans code for vulnerabilities before runtime.
- OWASP ZAP — DAST (dynamic analysis). Know the concept. Scans running apps for vulnerabilities.
- Falco — runtime container security. Detects anomalous behavior (unexpected shell, file writes, network). Know the name, what it does. Do not install it.

**One-liner for interviews:** "Security is in every pipeline stage. gitleaks at commit, Trivy and Snyk at build, OIDC at deploy, Kyverno at admission, NetworkPolicy at runtime. No stored credentials anywhere."

---

## Beyond Core Stack — Interview Vocabulary

These are not things to learn deeply. These are names and concepts to mention when asked "What else are you interested in?" or "Where do you see the industry going?"

**Platform Engineering / Internal Developer Platform (IDP)**
- The hottest trend in EU DevOps hiring right now.
- Concept: instead of every dev team configuring their own CI/CD, K8s manifests, and infra, a platform team builds a self-service internal platform.
- Tool: Backstage (by Spotify). Open-source developer portal. Know the name, know that it provides a service catalog, scaffolding templates, and plugin architecture.
- Your capstone project (P11) IS a platform. You just don't call it one yet. In interviews, say: "My capstone is essentially a lightweight internal developer platform — Terraform provisions infra, Helm deploys apps, ArgoCD syncs state, Kyverno enforces policy. The developer only writes application code and a values file."
- No need to install Backstage. Just know the concept and name.

**Service Mesh (Istio / Linkerd)**
- Concept: manages service-to-service communication (mTLS, traffic splitting, observability) without changing application code.
- Know when to use it: when you have 10+ microservices and need fine-grained traffic control.
- Know when NOT to use it: when you have 2-3 services (your case). "We evaluated a service mesh and decided the complexity was not justified for our service count."
- Tool: Istio (most popular), Linkerd (simpler).
- Do NOT learn or install this. Just know the concept for interviews.

**OpenTelemetry (OTel)**
- Concept: vendor-neutral standard for collecting metrics, logs, and traces.
- Know that it replaces vendor-specific SDKs. "Instead of locking into Datadog or New Relic, OpenTelemetry lets you send telemetry to any backend."
- Relevant because Prometheus handles metrics but not distributed tracing. OTel fills that gap.
- Do NOT learn this now. Just know the name and the problem it solves.

---

## What Your Full Stack Says to a European Hiring Manager

> "This engineer builds platforms, not just pipelines.
> He provisions infrastructure with Terraform, deploys with Helm and ArgoCD,
> monitors with Prometheus and Grafana, governs with Kyverno,
> secures with OIDC, Trivy, and gitleaks, and tracks cost with tagging and Kubecost.
> He thinks about security, cost, and observability as first-class concerns.
> He documents everything. He tears down what he does not need.
> He is ready for a senior platform role in Europe."

---

*Last updated: October 2026*
*Author: Manoj Kumar R (Monarch)*
*Target: Senior DevOps / Platform Engineer — Netherlands | Finland | Sweden | Ireland*
