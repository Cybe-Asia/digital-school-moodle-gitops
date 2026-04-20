# Phase 1 Status — Digital School

Snapshot of what's landed vs. what's outstanding for the `school`
domain, mapped against the three authoritative docs:

- Cybe DevOps Roadmap (Phase 1) — §2.1 – §2.5 staging/epics
- Cybe DevOps Phase 1 Epics & Stories Plan — §1 – §9 per-epic stories
- Cybe DevOps Architecture & Environment Plan (v1.1) — §1 – §15

## Coverage matrix

| Epic | Story | State | Where |
|---|---|---|---|
| L1 | S1 Namespace registry | ✅ | `lab/{dev,test,staging}/namespace.yaml`, `platform/ingress-system/namespace.yaml` |
| L1 | S2 ResourceQuota + LimitRange | ✅ | `base/governance/`, per-env quota in `lab/*/resource-quota.yaml` |
| L1 | S3 NetworkPolicy | ✅ enforced | `base/governance/network-policy.yaml` |
| L2 | S1 Deploy ArgoCD | ✅ (pre-existing) | `platform-system` namespace |
| L2 | S2 Argo Projects (school) | ⏳ | Currently using default project; domain-scoped project can be added |
| L2 | S3 Traefik + cert-manager | ⏳ partial | Traefik in `kube-system` (k3s default), migration runbook in `runbooks/traefik-migrate-to-ingress-system.md`. cert-manager ✅ installed via `argocd/cert-manager-application.yaml` with `lab-ca` self-signed ClusterIssuer. |
| L3 | S1 Standard StorageClass + PVC template | ✅ | `local-path` in lab; `base/neo4j` uses it |
| L3 | S2 DB backup CronJob | ✅ | `base/neo4j/backup-cronjob.yaml`, 7-day retention |
| L3 | S3 moodledata snapshot | ⬜ N/A | school has no Moodle; this is for skillz/moodle domain |
| L4 | S1 `sha-<gitsha>` tags | ✅ | Service repo CI workflows |
| L4 | S2 Trivy scan gating | ✅ (CRITICAL) | Service repo CI workflows |
| L4 | S3 SBOM artifact | ✅ | Anchore Syft, 90-day retention per build |
| L5 | S1 Reusable CI template | ✅ canonical + inlined copies | `.github/workflows/build-scan-push.yml` in gitops; identical inlined `deploy.yml` in each of 5 service repos |
| L5 | S2 GitOps-only RBAC | ✅ | ValidatingAdmissionPolicy `gitops-only-workloads`, applied cluster-wide; scoped to `domain=school` |
| L6 | S1 Env repo paths `/lab/*` | ✅ | `lab/{dev,test,staging}` |
| L6 | S2 Branch protection + CODEOWNERS | ✅ CODEOWNERS / ⏳ branch rule | `.github/CODEOWNERS` committed; `gh api` call still needs admin auth — see `docs/branch-protection.md` |
| L7 | S1 Smoke test | ✅ | `lab/staging/smoke-test-job.yaml`, PostSync hook |
| L7 | S2 Staging replicas ≥ 2 | ✅ | `lab/staging/kustomization.yaml` replicas block |
| L8 | S1 Prometheus stack | ✅ | `argocd/monitoring-stack-application.yaml` |
| L8 | S2 Alerts + dashboards | ✅ | `argocd/school-crashloop-alerts.yaml`, `platform/monitoring-dashboards/school-overview-dashboard.yaml` |
| L9 | S1 Restore drill | ✅ proven | `runbooks/neo4j-restore-drill.md` + pod template; PASSED on school-dev |
| Arch §9 | Domain isolation | ✅ | network-enforced via NetworkPolicy, RBAC-enforced via VAP |
| Arch §11 | Secrets at rest | ✅ | SealedSecrets per environment |
| Arch §8 | Blue/Green prod (Phase 2) | ⏳ scaffold | `prod/{blue,green,routing}` manifests + `runbooks/prod-*.md`; **not deployed** pending cybe-prod cluster |

## Outstanding operator actions (non-code)

| # | Action | Blocker | Risk if delayed |
|---|---|---|---|
| 1 | Move `~/.cybe-secrets-backup/cybe-lab-sealed-secrets-master-key-*.yaml` to a vault | Needs your password manager / encrypted USB | If laptop disk dies, every SealedSecret in the repo is permanently unreadable |
| 2 | Run `gh api PUT .../branches/main/protection` | Needs admin GH auth | CODEOWNERS is advisory until this lands |
| 3 | Traefik migration to `ingress-system` namespace | Needs sudo + maintenance window | Low — current location works, NetworkPolicy already compatible |
| 4 | Rotate Grafana admin password | Already rotated to random via SealedSecret | — see session notes for the one-time plaintext |
| 5 | Rotate Neo4j + JWT passwords per environment | Data migration complexity (Neo4j stores credentials) | Lab data is synthetic; low urgency |

## Cluster snapshot (lab)

| Namespace | Argo Sync | Pods | Purpose |
|---|---|---|---|
| `platform-system` | — | ArgoCD, metrics | GitOps control plane |
| `cert-manager` | Synced/Healthy | 3 | TLS automation |
| `monitoring` | Synced/Healthy | 7 | Prometheus + Grafana + Alertmanager + exporters |
| `ingress-system` | — | 0 | Declared; Traefik move pending |
| `kube-system` | — | Traefik, coredns, svclb | k3s system |
| `school-dev` | Synced/Healthy | 6 | dev |
| `school-test` | Synced/Healthy | 6 | test |
| `school-staging` | Synced/Healthy | 11 (2x per svc) | staging release gate |
| `school-prod-{blue,green,routing}` | — | 0 | Scaffolded, not deployed — Phase 2 |

## GitOps repo layout

```
.github/
  CODEOWNERS
  workflows/build-scan-push.yml      # canonical CI template
argocd/                               # Argo Application manifests
  cert-manager-application.yaml
  cert-manager-issuers-application.yaml
  grafana-admin-sealed.yaml           # SealedSecret, safe in git
  monitoring-dashboards-application.yaml
  monitoring-stack-application.yaml
  school-crashloop-alerts.yaml
  school-prod-{blue,green,routing}-application.yaml  # templates; don't apply yet
  school-test-application.yaml
base/                                 # shared service manifests
  admission/, auth/, frontend/, neo4j/, notification/, otp/
  governance/  (LimitRange, NetworkPolicy)
cluster-policies/                     # cluster-wide policy (apply manually)
  gitops-only.yaml                    # ValidatingAdmissionPolicy
docs/                                 # narrative documentation
lab/                                  # dev/test/staging overlays
prod/                                 # Phase 2 blue/green scaffolding
platform/                             # platform-layer manifests
  cert-manager/, ingress-system/, monitoring-dashboards/
runbooks/                             # step-by-step operator procedures
```

## What Phase 2 unlocks

- Actual `cybe-prod` cluster provisioning
- Deploy `prod/{blue,green,routing}` against it (runbook: `runbooks/prod-first-deploy.md`)
- First cutover (`runbooks/prod-blue-green-cutover.md`)
- Replicate the pattern to `skillz` and `bi` domains
- Production SLO definitions
- WAF / CDN in front of Traefik
