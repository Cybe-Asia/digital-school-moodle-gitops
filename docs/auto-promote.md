# Auto-promote pipeline

Pushes in service repos cascade **all the way to live production** with zero human intervention. The pipeline is split across two workflows that chain together:

- `.github/workflows/auto-promote-lab.yml` — cascades lab/dev → lab/test → lab/staging, stops at lab/staging
- `.github/workflows/auto-promote-prod.yml` — cascades lab/staging → prod/(idle color), auto-cutover, 5-min soak, auto-rollback

The lab workflow's final commit to `lab/staging/kustomization.yaml` triggers the prod workflow. Both carry `[skip promote]` on their own commits to avoid recursion.

## Two push paths, both end at prod

### Path A — push to service repo `main` (daily work)

1. Service CI builds image, bumps `lab/dev/kustomization.yaml`
2. **auto-promote-lab** fires → cascades to lab/test → lab/staging
3. Final lab/staging commit triggers **auto-promote-prod**
4. auto-promote-prod → prod/(idle) → cutover → soak → live

Full cascade, ~22 min end-to-end.

### Path B — push to service repo `release-*` branch (hotfix / staging-first)

1. Service CI builds image, bumps `lab/staging/kustomization.yaml` directly (skipping lab/dev + lab/test)
2. **auto-promote-prod** fires on that lab/staging commit
3. Same prod → cutover → soak → live

Faster path (~10 min), bypasses lab validation. Use for emergencies.

### Path C — push to service repo `qa/**` branch (isolated test-only)

1. Service CI builds image, bumps `lab/test/kustomization.yaml` directly (skipping lab/dev and lab/staging)
2. ArgoCD syncs `school-test-services` → image lands in namespace `school-test` only
3. **No auto-promote cascade** — `auto-promote-lab.yml` is scoped to `paths: lab/dev/kustomization.yaml`, so a commit to `lab/test` does not trigger it. The commit message still carries a defensive `[skip promote]` marker.

Use this for:
- QA validation of a feature before merging to `main`
- Reproducing a bug in the test namespace without contaminating dev
- Running integration tests against a specific image without promoting further

**Caveat:** the next push to `main` of any service will cascade through `lab/dev → lab/test`, overwriting the QA image in `lab/test`. Treat QA pins as ephemeral — merge to `main` when validation is done.

Branch naming is required: the prefix must be `qa/` (e.g. `qa/bulk-upload`, `qa/fix-auth-race`). Anything else falls back to no overlay update.

### Path D — push to service repo `staging/**` branch (isolated staging, no prod)

1. Service CI builds image, bumps `lab/staging/kustomization.yaml` directly (skipping lab/dev and lab/test)
2. Commit message carries `[skip promote]` so `auto-promote-prod.yml` short-circuits (it gates on that marker — see line 29 of that workflow)
3. ArgoCD syncs `school-staging-services` → image lands in namespace `school-staging` only

Use this for:
- UAT / business validation at staging before sign-off
- Demo-ing a feature on staging to stakeholders without shipping to prod
- Load-testing an image in the staging namespace (which has prod-like quotas) without exposing users

**Differs from `release-*` purely by the `[skip promote]` marker.** Same overlay, same namespace. When UAT passes, three paths to prod:

1. **Merge to `main` (recommended)** — full lab-cascade rebuilds the image and re-verifies through dev → test → staging → prod. ~22 min. Best audit trail.
2. **Cut `release-x.y.z` from the staging branch** — rebuild, then staging → prod (skips dev/test). ~10 min. Fast path when you want a fresh build but not lab re-verification.
3. **Manual `workflow_dispatch` on `auto-promote-prod`** — re-uses the EXACT image already in `lab/staging` (no rebuild). ~7 min. Use this when you need the byte-identical image that was UAT'd to be what ships. Requires a `reason` input for audit. Trigger from GitHub Actions tab → `auto-promote-prod` → "Run workflow".

**Caveat:** the next `release-*` push or any `main` merge that cascades to lab/staging will overwrite the staging pin. Treat staging/* pins as ephemeral too.

Branch naming is required: the prefix must be `staging/` (e.g. `staging/uat-bulk-upload`, `staging/demo-q2`). Anything else falls back to no overlay update.

## End-to-end timeline (Path A)

```
t=0         git push main in a service repo
t≈2 min     service CI done → bumps gitops main lab/dev
t≈2 min     auto-promote-lab starts
t≈5 min     lab/dev committed → lab/test; waiting school-test Healthy
t≈8 min     lab/test Healthy → lab/staging; waiting staging Healthy + smoke
t≈12 min    staging smoke passed → lab workflow DONE
t≈12 min    lab/staging commit triggers auto-promote-prod
t≈15 min    prod/(idle) Healthy + external smoke passed
t≈15 min    AUTO-CUTOVER: prod/routing/live-ingress.yaml flipped → Argo syncs → Traefik reloads
t≈16 min    live traffic on new color; old color goes idle
t≈16-21 min post-cutover soak: 5 min of external HTTP + Argo health probes
t≈21 min    soak passed → release complete
            (OR) soak failed → AUTO-ROLLBACK: revert cutover commit, live reverts
```

**Total: ~22 min from `git push` to validated-live, zero clicks.**

## What each workflow does

### auto-promote-lab
1. Reads image tags in `lab/dev/kustomization.yaml`
2. Commits matching tags to `lab/test/kustomization.yaml`
3. Waits for ArgoCD `school-test-services` Synced + Healthy
4. Commits to `lab/staging/kustomization.yaml` (this commit INTENTIONALLY omits `[skip promote]` so the prod workflow picks it up)
5. Waits for ArgoCD `school-staging-services` Healthy (gates on the PostSync smoke Job)

### auto-promote-prod
6. Reads `prod/routing/live-ingress.yaml` to figure out which color is **idle**
7. Reads image tags in `lab/staging/kustomization.yaml` (whether set by auto-promote-lab or by a `release-*` push)
8. Commits to `prod/<idle>/kustomization.yaml`
9. Waits for ArgoCD `school-prod-<idle>-services` Healthy
10. External smoke-test: `curl https://<idle>.school.cybe.tech:8443/` must return 2xx/3xx
11. **Auto-cutover**: sed-rewrites `prod/routing/live-ingress.yaml` externalName values → bot commits + pushes
12. Waits 45s for Argo to sync routing + Traefik to reload
13. **Post-cutover soak**: 20 probes × 15s = 5 min. HTTP against both `school.cybe.tech:8443` and `<new-color>.school.cybe.tech:8443`, plus Argo health. Tolerates ≤3 HTTP blips + ≤2 Argo blips.
14. If soak passes → success summary
15. If soak fails → **auto-rollback**: revert cutover commit, Argo re-syncs, live reverts to previous color

## What's still manual

**Nothing, on the happy path.** Operator clicks = 0.

Human intervention is only required when:
- Earlier pipeline steps fail (smoke test, Argo health) — pipeline halts before cutover. Debug, fix, push again.
- Soak fails and auto-rollback triggers — inspect logs, figure out what broke, fix, push again.

## Required GitHub secrets

| Secret | What | How to get |
|---|---|---|
| `GITOPS_TOKEN` | PAT with `repo` on this repo | Already exists (same one service CI uses) |
| `ARGOCD_URL` | `https://argo.cybe.tech:8443` | Your ArgoCD public URL |
| `ARGOCD_USERNAME` | `admin` | ArgoCD account |
| `ARGOCD_PASSWORD` | ArgoCD admin password | From `argocd-initial-admin-secret` in platform-system namespace |

If `ARGOCD_*` secrets are missing, the workflow degrades to sleep-based waits (5 minutes per env). Less safe — add the secrets for real gating.

## How to add the secrets

From the gitops repo in GitHub:

```
Settings → Secrets and variables → Actions → New repository secret

ARGOCD_URL       = https://argo.cybe.tech:8443
ARGOCD_USERNAME  = admin
ARGOCD_PASSWORD  = <value from `kubectl -n platform-system get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`>
```

## Disabling auto-promote temporarily

If you need to pause the auto-promotion (e.g., during an incident):

```
GitHub → Actions tab → "auto-promote" workflow → ... → Disable workflow
```

Re-enable when ready. Any bumps made while disabled can be re-triggered by pushing an empty commit that changes `lab/dev/kustomization.yaml`.

## Emergency skip

To commit to `lab/dev` WITHOUT triggering auto-promote (e.g. rollback), include `[skip promote]` in the commit message. The workflow's `if:` condition checks for this.

## Branch protection compatibility

Once `main` branch protection is enabled with `require_code_owner_reviews`, the workflow's push needs to be allowed to bypass. Options:

1. In branch protection settings, list the CI bot user (owner of `GITOPS_TOKEN`) under "Allow specified actors to bypass required pull requests".
2. Use a GitHub App token instead of a user PAT — GitHub Apps can have their own bypass rules.
3. Keep branch protection off and rely on CODEOWNERS-advisory-only — simplest but weaker.

Recommended: option 1 with a dedicated `cybe-ci-bot` user. Don't use your personal PAT long-term.

## Bypassing CODEOWNERS on auto-promote paths

The workflow pushes commits directly to `main` (no PR). CODEOWNERS only enforces on PRs, so bot commits don't trigger the review gate. This is intentional — the bot is an approved promotion actor.

Human-authored changes to `lab/staging/*`, `prod/*`, etc. still require CODEOWNERS review if made via PR.
