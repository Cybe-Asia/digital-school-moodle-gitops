# CI Golden Path

Describes the single reusable GitHub Actions workflow each school
service calls, and how it connects to this gitops repo.

Doc coverage: Phase 1 Epics L4 (CI/CD Artifact Discipline), L5 (Reusable
CI Workflow Template), L7 (Env Repo Enforcement).

## Flow

```
 service repo push                   digital-school-gitops
 ───────────────────                 ────────────────────
   main branch      ──build+scan+SBOM+push── lab/dev  (auto-PR)
   release-*        ──build+scan+SBOM+push── lab/staging (auto-PR)
   main             ──build+scan+SBOM+push── overlays/production (auto-PR)
```

Argo CD syncs the matching namespace whenever the overlay file
changes. Rollback = revert the overlay commit; Argo re-syncs to the
previous image.

`lab/test` is **not** auto-updated by CI. Promotion from dev to
test is a manual PR — copy the image tag line from
`lab/dev/kustomization.yaml` into `lab/test/kustomization.yaml`.
Same digest, no rebuild — satisfies L8 "Staging image digest equals
test digest".

## Reusable workflow

Lives at `.github/workflows/build-scan-push.yml` in this repo. Invoked
by service repos via:

```yaml
jobs:
  deploy:
    uses: Cybe-Asia/digital-school-gitops/.github/workflows/build-scan-push.yml@dev
    with:
      service_name: admission-services
      kustomize_image_key: ghcr.io/cybe-asia/admission-services
    secrets:
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
      GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
```

## What the workflow does

1. **Build** image from repo Dockerfile
2. **Trivy scan** — fails the pipeline on any CRITICAL or HIGH
   vulnerability that has a fix available. OS + library scope. L4-S2
   acceptance.
3. **SBOM** — SPDX JSON generated via Anchore Syft, uploaded as a
   GitHub Actions artifact, 90-day retention. L4-S3 acceptance.
4. **Push** to `ghcr.io/cybe-asia/<service>:sha-<shortsha>` only after
   scan + SBOM succeed.
5. **Auto-PR** to the matching overlay in this repo with the new
   image tag, commit message `[skip ci]` so the gitops push doesn't
   retrigger CI.

## Required GitHub secrets (per service repo)

- `GHCR_TOKEN` — PAT with `write:packages` scope.
- `GITOPS_TOKEN` — PAT with `repo` scope on
  `Cybe-Asia/digital-school-gitops`. Used only to push the overlay
  bump commit. Recommend a bot account (e.g. `cybe-ci-bot`) rather
  than a human PAT.

## Adopting the workflow in a new service

1. Add a `Dockerfile` at repo root (or set `dockerfile:` input).
2. Ensure `GHCR_TOKEN` and `GITOPS_TOKEN` are set in repo secrets.
3. Replace the service's `.github/workflows/deploy.yml` with:

```yaml
name: Build & Deploy
on:
  push:
    branches: [dev, main, 'release-*']
  workflow_dispatch:

jobs:
  deploy:
    uses: Cybe-Asia/digital-school-gitops/.github/workflows/build-scan-push.yml@dev
    with:
      service_name: <service-name>             # e.g. admission-services
      kustomize_image_key: ghcr.io/cybe-asia/<service-name>
    secrets:
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
      GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
```

4. Add the service to a kustomization overlay if not already present.

## Known gaps (tracked)

- **`release-*` / `main` overlays not yet used** — the workflow will
  auto-bump `lab/staging` / `overlays/production` when those
  branches are pushed, but the doctrinal model (L8) is that staging
  and production use the *same digest* as test, promoted via manual
  PR. The current behavior is a legacy-compatible stopgap. Remove the
  non-dev branch mappings once teams adopt promotion-via-PR.
- **No unit-test stage** — L5-S1 mentions unit tests and dep scan
  before build. Add language-specific `test` job as a dependency of
  `build-scan-push` when each service's test harness is ready.
- **Trivy + SBOM skipped for frontend** — plain-JS apps don't always
  produce meaningful OS-layer CVEs, but `npm audit` equivalent
  scanning is still worth adding. Track separately.
- **Policy-as-code (OPA/Conftest) on overlays** — a logical next step
  beyond CODEOWNERS for catching bad kustomize edits.
