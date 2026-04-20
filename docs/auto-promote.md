# Auto-promote pipeline

Pushes in service repos cascade **all the way to live production** with zero human intervention. The pipeline deploys to the idle prod color, flips routing automatically, then soaks the new live color for 5 minutes and auto-rolls-back if it breaks.

## End-to-end timeline

```
t=0         git push main in a service repo
t≈2 min     service CI done → bumps gitops main lab/dev
t≈2 min     auto-promote workflow starts (.github/workflows/auto-promote.yml)
t≈5 min     lab/dev committed → lab/test; waiting school-test Healthy
t≈8 min     lab/test Healthy → lab/staging; waiting staging Healthy + smoke
t≈12 min    staging smoke passed → prod/(idle color)
t≈16 min    prod/(idle) Healthy + external smoke passed
t≈16 min    AUTO-CUTOVER: prod/routing/live-ingress.yaml flipped → Argo syncs → Traefik reloads
t≈17 min    live traffic on new color; old color goes idle
t≈17-22 min post-cutover soak: 5 min of external HTTP + Argo health probes
t≈22 min    soak passed → release complete
            (OR) soak failed → AUTO-ROLLBACK: revert cutover commit, live reverts to previous color
```

**Total: ~22 min from `git push` to validated-live, zero clicks.**

## What it does

1. Reads image tags in `lab/dev/kustomization.yaml` (whatever the service CI just bumped).
2. Commits matching tags to `lab/test/kustomization.yaml`.
3. Waits for ArgoCD `school-test-services` to report Synced + Healthy.
4. Commits to `lab/staging/kustomization.yaml`.
5. Waits for ArgoCD `school-staging-services` Healthy — this gates on the PostSync smoke Job.
6. Reads `prod/routing/live-ingress.yaml` to figure out which color is **idle** (blue or green).
7. Commits to `prod/<idle>/kustomization.yaml`.
8. Waits for ArgoCD `school-prod-<idle>-services` Healthy.
9. External smoke-test: `curl https://<idle>.school.cybe.tech:8443/` — must return 2xx/3xx.
10. **Auto-cutover**: sed-rewrites `prod/routing/live-ingress.yaml` externalName values to point at the new color; bot commits + pushes.
11. Waits 45s for Argo to sync routing + Traefik reload.
12. **Post-cutover soak**: 20 probes × 15s = 5 minutes. Measures HTTP against both `school.cybe.tech:8443` (live) and `<new-color>.school.cybe.tech:8443`. Also polls Argo health on the new-live color. Tolerates up to 3 HTTP blips + 2 Argo state blips.
13. If soak passes → success summary, release complete.
14. If soak fails → **auto-rollback**: revert the cutover commit, Argo re-syncs, live traffic returns to previous color. Failed-color pods stay running for post-mortem.

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
