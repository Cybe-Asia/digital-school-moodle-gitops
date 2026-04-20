# Auto-promote pipeline

Pushes in service repos cascade through **lab/dev → lab/test → lab/staging → prod/(idle)** automatically. Only the final live-traffic cutover is manual.

## End-to-end timeline

```
t=0         git push main in a service repo
t≈2 min     service CI done → bumps gitops main lab/dev
t≈2 min     auto-promote workflow starts (.github/workflows/auto-promote.yml)
t≈5 min     lab/dev committed → lab/test; waiting school-test Healthy
t≈8 min     lab/test Healthy → lab/test committed → lab/staging; waiting staging Healthy + smoke
t≈12 min    staging smoke passed → committed → prod/(idle color)
t≈16 min    prod/(idle) Healthy + external smoke passed
t≈16 min    YOU get a green summary in GH Actions with a ready-to-merge cutover note
t≈??        YOU merge a 5-line PR flipping prod/routing/live-ingress.yaml
t≈??+1min   live traffic on new color
```

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
10. Posts a summary with the exact cutover PR instructions.

Any step failure → workflow fails → promotion halts at that point → previous envs keep running on their previous image. Safe.

## What's still manual

**Cutover.** You still merge a PR to `prod/routing/live-ingress.yaml` to switch live traffic from current color to the new color. This preserves:
- Human-gated production traffic switch
- Ability to sit on prod/(idle) for a while and smoke-test more thoroughly
- Easy rollback by reverting the cutover PR

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
