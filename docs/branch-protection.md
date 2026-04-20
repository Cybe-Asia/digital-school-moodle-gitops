# Branch Protection Setup

Implements Phase 1 Epic **L6-S2** ("Branch Protection & CODEOWNERS").
Must be applied once by a repo admin; GitHub does not support branch
protection rules as committed files.

## Required rules on `dev`

Doc alignment:

- Roadmap §2.4 L7: "Dev auto-updated by CI; Test requires PR approval;
  Staging requires PR approval + CODEOWNERS; No direct commits to
  staging branch"
- Epics §6.3: "Promotion only via PR; Rollback via PR revert; Audit
  logs visible"
- Epic L6-S2 AC: "Direct merge blocked without approval"

Because this repo promotes via **folder-based overlays** on a single
`dev` branch (not per-env branches), the CODEOWNERS file enforces the
per-env review gate — staging path changes require the staging owner,
prod path changes require the prod owner. Branch protection just
makes those reviews mandatory.

## Apply via `gh` CLI

Must be run by a user with `admin` permission on
`Cybe-Asia/digital-school-gitops`.

```sh
gh api -X PUT \
  /repos/Cybe-Asia/digital-school-gitops/branches/main/protection \
  -H "Accept: application/vnd.github+json" \
  -f required_status_checks=null \
  -F enforce_admins=false \
  -f 'required_pull_request_reviews[required_approving_review_count]=1' \
  -F required_pull_request_reviews[require_code_owner_reviews]=true \
  -F required_pull_request_reviews[dismiss_stale_reviews]=true \
  -F restrictions=null \
  -F allow_force_pushes=false \
  -F allow_deletions=false \
  -F required_linear_history=false \
  -F required_conversation_resolution=true
```

## What this enables

- **Direct pushes to `dev`** are blocked; all merges go through PRs.
- **Code Owner review required** — any PR touching
  `lab/staging/*` or `overlays/production/*` cannot merge without
  the owner listed in `.github/CODEOWNERS` approving.
- **Stale reviews dismissed** on new pushes — approving a PR and then
  editing it invalidates the approval.
- **No force push**, **no branch deletion** of `dev`.
- **Unresolved conversations block merge** — review comments must be
  resolved before merge.

## Verify

```sh
gh api /repos/Cybe-Asia/digital-school-gitops/branches/main/protection | jq
```

Should show `required_pull_request_reviews.require_code_owner_reviews:
true`.

## Follow-ups

- Once GitHub Actions CI runs on PRs (L4-S2 Trivy, L5-S1 reusable
  workflow), register those workflow names in `required_status_checks`
  so failing builds block merges.
- Add a second reviewer on `lab/staging/*` and
  `overlays/production/*` once the team grows — CODEOWNERS supports
  multiple handles space-separated.
- Consider protecting `main` as well (currently inactive but should
  stay in sync with `dev` via releases).
