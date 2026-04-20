# Conventions

Single source of truth for naming + process rules across the Digital
School platform. If the code contradicts this doc, the code is wrong.

## 1. Service repo branches

Every service repo (admission-services, auth-services, otp-services,
notification-services, digital-school-frontend) follows these rules.

### Branch → environment mapping

| Branch pattern | Overlay in gitops | Final env | Cascade to prod? | Use case |
|----------------|-------------------|-----------|------------------|----------|
| `main`         | `lab/dev`         | `school-dev` → test → staging → prod | ✅ Full auto | Daily work |
| `qa/**`        | `lab/test`        | `school-test` only                   | ❌ Stops at test | Feature QA without dev pollution |
| `staging/**`   | `lab/staging`     | `school-staging` only                | ❌ Stops at staging | UAT / stakeholder demo |
| `release-*`    | `lab/staging`     | `school-staging` → prod              | ✅ Fast path | Hotfix, skip dev/test |
| `feat/**` or any other | — (no CI overlay bump) | — | — | Local feature branches before PR |

### Branch name examples

```
✅ main
✅ qa/bulk-upload-fix
✅ qa/fix-auth-timeout
✅ staging/uat-q2-demo
✅ staging/stakeholder-preview
✅ release-v1.2.3
✅ release-hotfix-2026-04-20
✅ feat/add-csv-parser        (local only, PR to main)
✅ fix/timezone-bug           (local only, PR to main)

❌ qa-bulk-upload             (missing slash)
❌ staging                    (must have subpath)
❌ release/v1                 (must use hyphen not slash)
❌ QA/foo                     (must be lowercase)
```

### Cleanup after merge

- GitHub auto-deletes the head branch (enable in repo Settings → General → Pull Requests).
- Locally: `git fetch --prune`

## 2. Commit message markers

CI and GitOps tooling reads these from commit messages.

| Marker | Who sets it | Effect |
|--------|-------------|--------|
| `[skip ci]` | service CI when bumping gitops | Prevents the gitops commit from triggering service CI infinite loop |
| `[skip promote]` | gitops CI on lab/dev → test and test → staging internal steps; also set when qa/* or staging/* branches land | Prevents `auto-promote-lab.yml` and `auto-promote-prod.yml` from cascading further |

**Developers rarely write these by hand.** CI does it. Only exception: if you manually bump an overlay and want to halt the cascade, include `[skip promote]` in your commit.

### Commit subject format (humans)

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <subject>

Examples:
feat(auth): add password reset flow
fix(admission): handle empty CSV rows
chore(deps): bump tokio to 1.40
docs(auto-promote): document Path E
ci: add qa/** branch → lab/test overlay mapping
```

Types: `feat`, `fix`, `chore`, `docs`, `ci`, `refactor`, `test`, `perf`.

**No Co-Authored-By trailers** (team policy).

## 3. GitOps repo paths

```
digital-school-gitops/
├── argocd/                 # ArgoCD Application CRDs
├── base/                   # Kustomize bases per service
├── lab/
│   ├── dev/                # school-dev overlay
│   ├── test/               # school-test overlay
│   └── staging/            # school-staging overlay
├── prod/
│   ├── blue/               # school-prod-blue overlay
│   ├── green/              # school-prod-green overlay
│   └── routing/            # school-prod-routing ExternalName services
├── platform/               # cluster-wide services (Unleash, cert-manager, etc)
├── cluster-policies/       # ValidatingAdmissionPolicy etc
├── docs/                   # how-tos (this file is here)
├── runbooks/               # incident response
└── scripts/                # operator helpers
```

Never hand-edit `lab/*/kustomization.yaml` or `prod/*/kustomization.yaml`
for image tag bumps. Always drive through service CI push.

## 4. Namespace naming

| Namespace | Purpose |
|-----------|---------|
| `school-dev` | Lab dev — continuous, latest main |
| `school-test` | Lab test — integration, receives qa/* |
| `school-staging` | Lab staging — UAT, receives staging/* |
| `school-prod-blue` | Prod blue pool (live or idle) |
| `school-prod-green` | Prod green pool (live or idle) |
| `school-prod-routing` | ExternalName services — single source of "which color is live" |
| `platform-system` | ArgoCD + cluster control plane apps |
| `monitoring` | Prometheus + Grafana + Alertmanager |
| `loki` / `promtail` | Log shipping (inside monitoring) |
| `cert-manager` | cert-manager |
| `sealed-secrets` | bitnami sealed-secrets controller |
| `ingress-system` | Traefik (k3s default) |
| `unleash` | Feature flag service |

**Rule**: school-domain namespaces are always prefixed `school-`. Platform services use single-word lowercase.

## 5. ArgoCD Application naming

```
<domain>-<env>-<suffix>

✅ school-dev-services         # app pods in school-dev
✅ school-test-services
✅ school-staging-services
✅ school-prod-blue-services
✅ school-prod-green-services
✅ school-prod-routing         # ExternalName services for cutover
✅ unleash-stack
✅ loki-stack
✅ monitoring-stack
✅ cert-manager
```

Applications live in `platform-system` namespace with `spec.project: school`
for domain apps, `spec.project: default` for platform apps.

## 6. Image tag format

Service CI produces tags:

```
✅ sha-a1b2c3d        # 7-char git SHA of commit that built it
✅ sha-a1b2c3d-qa     # (future — if we add per-branch suffix)

❌ latest             # never, forbidden
❌ v1.2.3             # semver is for human releases; sha for deploys
❌ dev-abc1234        # legacy pre-migration; will be overwritten
```

Every image tag is traceable back to one git commit. `latest` is banned
because it's not traceable.

## 7. Feature flag naming (Unleash)

```
✅ bulk-upload-v2
✅ new-dashboard-ui
✅ enable-google-oauth
✅ kill-switch-payment-processing

❌ BulkUploadV2             # kebab-case only
❌ bulk_upload              # no underscores
❌ feat-123                 # descriptive names, not ticket IDs
❌ temp                     # be specific, flag will live for weeks
```

### Required flag metadata (Unleash tags)

Every flag must have:
- `owner:<github-username>` — who created it, who must remove it
- `remove-by:YYYY-MM-DD` — when this flag's job is done
- `type` — one of: `release`, `experiment`, `operational`, `kill-switch`, `permission`

Quarterly review: archive flags older than 90 days at 100% rollout.

## 8. Secret handling

### Rules

1. **Never commit plaintext secrets to git.** Ever.
2. All secrets in git are `SealedSecret` (Bitnami) or in external
   vault, never plain `Secret`.
3. Sealing uses the cluster's public key; only the cluster's private
   key (in `sealed-secrets` namespace) can decrypt.
4. The sealed-secrets master key must be backed up off-cluster (1Password
   or offline USB) — without it, a cluster disaster means re-sealing
   every secret from source.

### Naming

```
✅ ghcr-secret                    # image pull (per namespace)
✅ unleash-postgres-credentials
✅ unleash-admin-credentials
✅ grafana-admin-sealed
✅ neo4j-root-credentials

❌ secret1                        # descriptive names
```

## 9. Resource requests / limits

Every Deployment must have requests AND limits set. ResourceQuota
enforces this.

Tier defaults (tune per workload):

| Tier | CPU req | Mem req | CPU lim | Mem lim |
|------|---------|---------|---------|---------|
| Small (admin UI, CronJob) | 50m | 128Mi | 500m | 256Mi |
| Medium (typical service) | 100m | 256Mi | 1000m | 512Mi |
| Large (frontend SSR, DB) | 200m | 512Mi | 2000m | 1Gi |

Prod always has PDB `minAvailable: 2` for multi-replica deployments.

## 10. Auto-promote safety gates

| Stage | Gate |
|-------|------|
| lab/dev → lab/test | `school-test-services` must be `Synced/Healthy` in ArgoCD |
| lab/test → lab/staging | `school-staging-services` must be `Synced/Healthy` |
| lab/staging → prod/idle | `school-prod-<idle>-services` must be `Synced/Healthy` + external smoke test (HTTP 2xx/3xx) |
| cutover | Idle must pass BOTH ArgoCD Healthy AND external smoke BEFORE flipping |
| post-cutover soak | 5 min, 20 probes × 15s, tolerate ≤3 HTTP / ≤2 Argo blips |
| rollback trigger | `git revert` the cutover commit if soak fails |

No gate is manual — all automated. Manual test-on-idle is opt-in
(workflow_dispatch with `skip_cutover: true`).

## 11. Tokens + credentials inventory

Minimum set of GitHub secrets per repo:

### Service repos
- `GHCR_TOKEN` — push images
- `GITOPS_TOKEN` — push overlay bumps to digital-school-gitops

### Gitops repo
- `GITOPS_TOKEN` — for its own cross-workflow commits
- `ARGOCD_URL` — e.g. `https://argo.cybe.tech:8443`
- `ARGOCD_USERNAME` — CI bot account (TODO: sc-325 — currently uses personal)
- `ARGOCD_PASSWORD` — CI bot password

**Rotation**: every 90 days (Q1, Q2, Q3, Q4).

## 12. DNS + TLS

| Host | Purpose | Public? |
|------|---------|---------|
| `school.cybe.tech:8443` | Prod live (user-facing) | ✅ |
| `blue.school.cybe.tech:8443` | Prod blue (idle or live) | ✅ |
| `green.school.cybe.tech:8443` | Prod green (idle or live) | ✅ |
| `staging.school.cybe.tech:8443` | Staging (internal) | ⚠️ optional |
| `grafana.cybe.tech:8443` | Grafana admin | ✅ (auth-gated) |
| `argo.cybe.tech:8443` | ArgoCD admin | ✅ (auth-gated) |
| `unleash.cybe.tech:8443` | Unleash admin | ✅ (auth-gated) |

TLS: single wildcard `*.cybe.tech` issued by Let's Encrypt via DNS-01.
Renewal automatic via cert-manager every 60 days.

All public endpoints terminate TLS at host nginx on port 8443, proxy
to Traefik NodePort 32118. Port 443 is blocked at the edge (ISP).

## 13. When to break these rules

Never in production. For experiments: use your own namespace suffix
(e.g. `school-dev-arief-experiment`) and don't commit to main. Delete
the namespace when done.
