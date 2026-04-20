# Trunk branch is `main`

This repo uses a **single-trunk, path-per-environment** model (Option A
in the doctrine — see Architecture §9). The trunk branch is `main`.
`dev` is historical and will be deleted once this migration is stable.

## What lives where

```
main (trunk)
├── base/                  shared manifests
├── lab/dev/               school-dev overlay
├── lab/test/              school-test overlay
├── lab/staging/           school-staging overlay  ← CODEOWNERS
├── prod/blue/             school-prod-blue overlay  ← CODEOWNERS
├── prod/green/            school-prod-green overlay  ← CODEOWNERS
├── prod/routing/          live Ingress + ExternalName  ← CODEOWNERS
├── base/                  ← CODEOWNERS (cross-env)
├── argocd/                ← CODEOWNERS (wire-the-system)
├── cluster-policies/      ← CODEOWNERS (cluster-wide VAP)
├── platform/              ← CODEOWNERS (platform add-ons)
├── .github/               ← CODEOWNERS (CI + codeowners)
├── runbooks/              operator docs (free-push)
└── docs/                  narrative docs (free-push)
```

## Review policy (per `.github/CODEOWNERS`)

| Path | Direct push allowed? |
|---|---|
| `/lab/dev/*` | ✅ yes |
| `/lab/test/*` | ✅ yes |
| `/runbooks/*` | ✅ yes |
| `/docs/*` | ✅ yes |
| README.md etc | ✅ yes |
| `/lab/staging/*` | ❌ PR + owner review |
| `/prod/blue/*` | ❌ PR + owner review |
| `/prod/green/*` | ❌ PR + owner review |
| `/prod/routing/*` | ❌ PR + owner review |
| `/base/*` | ❌ PR + owner review |
| `/argocd/*` | ❌ PR + owner review |
| `/cluster-policies/*` | ❌ PR + owner review |
| `/platform/*` | ❌ PR + owner review |
| `/.github/*` | ❌ PR + owner review |

CI bots are exempted from branch protection so they can direct-push
image-tag bumps into `/lab/dev/` on every service-repo merge-to-main.

## Promotion flow (same digest through envs)

```
service repo (main branch)
    │ push
    ▼
GitHub Actions: Trivy + SBOM + build + push ghcr.io/...:sha-<gitsha>
    │
    ▼ auto-commit [skip ci] on this repo's main branch
gitops main:  lab/dev/kustomization.yaml  (image tag bumped)
    │
    ▼ ArgoCD auto-sync
school-dev cluster

─── manual PR bumps ───
lab/dev → lab/test (auto-sync)
lab/test → lab/staging (CODEOWNERS review + smoke test Job)
lab/staging → prod/green (CODEOWNERS review)
    │
    ▼ manual `argocd app sync school-prod-green-services`
school-prod-green cluster (idle, validated via green.school.cybe.tech)

prod/routing/live-ingress.yaml: flip ExternalName blue → green
    │
    ▼ ArgoCD auto-sync
school.cybe.tech:8443 now live on green (atomic cutover)
```

## Why single-trunk, not branch-per-env

- Architecture §9 documents exactly this layout (paths, not branches)
- ArgoCD best practice is "separate by path not by branch"
- Base manifests change once, all envs pick up atomically (no cherry-picking)
- Cross-env refactors = one PR (not N PRs merged in order)
- `git log -p` tells the whole environment history
- CODEOWNERS gives per-env review gating without branch-boundary merge conflicts

## What about `dev`?

The branch `dev` was the working branch before this migration. It
remains pointing at the last-shared commit for rollback safety but
receives no new commits. Delete it after ~1 week of stable operation
on `main`.
