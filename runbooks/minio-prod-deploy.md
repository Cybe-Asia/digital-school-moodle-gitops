# Deploy MinIO to prod (blue + green)

Document upload via admission-service requires MinIO. This runbook
walks you through enabling it in production. Same shape as lab/test;
the only new work is sealing two sets of secrets — one per prod
color.

## Why this exists

Document upload was never wired up in `prod/blue` or `prod/green`.
The code was implemented (admission-service commit `ff025e8`,
frontend `parent-document-upload.tsx`) but only `lab/test` got the
matching MinIO StatefulSet + sealed credentials. As of 2026-04-25,
hitting the upload endpoint on `school.cybe.tech` fails with a
connect error.

## Prerequisites

- `kubeseal` CLI installed locally (`brew install kubeseal`)
- `kubectl` context pointing at the prod cluster (cybe-lab today)
- The sealed-secrets controller running in the prod cluster
  (verify: `kubectl -n sealed-secrets get pods`)
- This repo cloned, with the staged drafts already in place:
  - `prod/blue/app-config.yaml` — MINIO_* keys added (committed in
    a separate PR or unpushed)
  - `prod/green/app-config.yaml` — same
  - `prod/blue/kustomization.yaml` — MinIO refs commented out
  - `prod/green/kustomization.yaml` — same

## Step 1 — Generate fresh credentials (per color)

Each color gets its own credentials. Don't share between blue and
green. Don't reuse lab/test creds.

```sh
# Blue color credentials
BLUE_ROOT_USER="minio-root-blue"
BLUE_ROOT_PASSWORD="$(openssl rand -base64 32 | tr -d '/+=' | head -c 24)"
BLUE_ACCESS_KEY="$(openssl rand -hex 12)"
BLUE_SECRET_KEY="$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)"

# Green color credentials
GREEN_ROOT_USER="minio-root-green"
GREEN_ROOT_PASSWORD="$(openssl rand -base64 32 | tr -d '/+=' | head -c 24)"
GREEN_ACCESS_KEY="$(openssl rand -hex 12)"
GREEN_SECRET_KEY="$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)"

# Save these somewhere secure (1Password, vault) BEFORE proceeding.
# Once sealed, you can't retrieve them from the cluster.
echo "BLUE  ROOT_USER=$BLUE_ROOT_USER ROOT_PASS=$BLUE_ROOT_PASSWORD ACCESS=$BLUE_ACCESS_KEY SECRET=$BLUE_SECRET_KEY"
echo "GREEN ROOT_USER=$GREEN_ROOT_USER ROOT_PASS=$GREEN_ROOT_PASSWORD ACCESS=$GREEN_ACCESS_KEY SECRET=$GREEN_SECRET_KEY"
```

## Step 2 — Seal `minio-secret` for both colors

Root credentials for the StatefulSet. Read by MinIO itself + the
init-job that creates the bucket and scoped service account.

```sh
# Blue
kubectl create secret generic minio-secret \
  --from-literal=MINIO_ROOT_USER="$BLUE_ROOT_USER" \
  --from-literal=MINIO_ROOT_PASSWORD="$BLUE_ROOT_PASSWORD" \
  --namespace=school-prod-blue \
  --dry-run=client -o yaml |
kubeseal --format=yaml \
  --controller-namespace=sealed-secrets \
  > prod/blue/minio-secret-sealed.yaml

# Green
kubectl create secret generic minio-secret \
  --from-literal=MINIO_ROOT_USER="$GREEN_ROOT_USER" \
  --from-literal=MINIO_ROOT_PASSWORD="$GREEN_ROOT_PASSWORD" \
  --namespace=school-prod-green \
  --dry-run=client -o yaml |
kubeseal --format=yaml \
  --controller-namespace=sealed-secrets \
  > prod/green/minio-secret-sealed.yaml
```

## Step 3 — Re-seal `app-secrets-sealed.yaml` to include MINIO_ACCESS_KEY + MINIO_SECRET_KEY

This is the trickier step because re-sealing requires every key the
existing app-secrets already contains. Get the current key list off
the cluster:

```sh
# Blue — list existing keys (values not exposed; just key names)
kubectl -n school-prod-blue get secret app-secrets -o json |
  jq -r '.data | keys[]'
# Expected output: NEO4J_PASSWORD, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET,
#                  XENDIT_SECRET_KEY, XENDIT_WEBHOOK_TOKEN, SMTP_PASSWORD, ...
```

Then read each value (you'll need cluster access to read live
secrets, OR you keep the source values in 1Password — preferred):

```sh
# Decrypt one key at a time (only do this if you don't have the
# values stashed elsewhere — the cluster IS the source of truth):
kubectl -n school-prod-blue get secret app-secrets \
  -o jsonpath='{.data.NEO4J_PASSWORD}' | base64 -d
# Repeat for each key
```

Re-seal with the full set + new MinIO keys:

```sh
# Blue
kubectl create secret generic app-secrets \
  --from-literal=NEO4J_PASSWORD='<existing>' \
  --from-literal=GOOGLE_CLIENT_ID='<existing>' \
  --from-literal=GOOGLE_CLIENT_SECRET='<existing>' \
  --from-literal=XENDIT_SECRET_KEY='<existing>' \
  --from-literal=XENDIT_WEBHOOK_TOKEN='<existing>' \
  --from-literal=SMTP_PASSWORD='<existing-sendgrid-api-key>' \
  --from-literal=MINIO_ACCESS_KEY="$BLUE_ACCESS_KEY" \
  --from-literal=MINIO_SECRET_KEY="$BLUE_SECRET_KEY" \
  --namespace=school-prod-blue \
  --dry-run=client -o yaml |
kubeseal --format=yaml \
  --controller-namespace=sealed-secrets \
  > prod/blue/app-secrets-sealed.yaml

# Green — same pattern, swap GREEN values + namespace.
```

⚠️ **DO NOT commit until you've verified all existing keys are
preserved.** Missing a key here means the prod service that uses it
will start failing.

Quick sanity check: after re-sealing but before pushing, kustomize
build the overlay and grep for the secret keys — if any expected
key is missing, kustomize shows it.

## Step 4 — Uncomment the kustomization MinIO lines

In both `prod/blue/kustomization.yaml` and
`prod/green/kustomization.yaml`, uncomment the two lines:

```yaml
# Before:
# - ../../base/minio
# - minio-secret-sealed.yaml

# After:
- ../../base/minio
- minio-secret-sealed.yaml
```

## Step 5 — Verify locally

```sh
kustomize build prod/blue 2>&1 | grep -E "kind: (StatefulSet|SealedSecret)" | grep -i minio
# Should show: minio StatefulSet + minio-secret SealedSecret
kustomize build prod/blue 2>&1 | grep "MINIO_ACCESS_KEY\|MINIO_SECRET_KEY"
# Should NOT show plaintext — these come from the (still-sealed)
# app-secrets at apply time, not in the build output

# Repeat for prod/green.
```

If any error: stop. Don't push. Fix and re-verify.

## Step 6 — Commit + push

Single commit per color (or both together — your choice). Don't
skip-ci or skip-promote here — this IS a deploy, let ArgoCD pick it
up normally.

```sh
git add prod/blue/minio-secret-sealed.yaml \
        prod/blue/app-secrets-sealed.yaml \
        prod/blue/app-config.yaml \
        prod/blue/kustomization.yaml \
        prod/green/minio-secret-sealed.yaml \
        prod/green/app-secrets-sealed.yaml \
        prod/green/app-config.yaml \
        prod/green/kustomization.yaml \
        runbooks/minio-prod-deploy.md
git commit -m "feat(prod): deploy MinIO to blue + green for admissions document upload"
git push origin main
```

## Step 7 — Watch ArgoCD sync

```sh
argocd app get school-prod-blue
argocd app get school-prod-green
```

Both should reach `Synced/Healthy` within ~2 min. The MinIO
StatefulSet pulls its image, the init-job runs once to create the
bucket + scoped service account, then admission-service pods
naturally pick up MINIO_ENDPOINT etc. on next deploy (or you can
force a rollout):

```sh
kubectl -n school-prod-blue rollout restart deployment/admission-service
kubectl -n school-prod-green rollout restart deployment/admission-service
```

## Step 8 — Smoke test document upload

```sh
# As a parent in prod, trigger an upload via the UI, OR via curl:
curl -X POST "https://school.cybe.tech/api/v1/admission-service/documents/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/sample.pdf"
# Expect 200 + a DocumentArtifact ID in the response.
```

If you get a 500 or "connection refused" — check
`kubectl -n school-prod-blue logs deploy/admission-service` for
"MINIO_ENDPOINT" or "minio:9000" errors. Most likely cause is a
missing env var in app-secrets — re-verify Step 3.

## What's NOT in this runbook (and why)

- **Backup.** This deploys MinIO with local-path PVC and no
  off-site backup. A node disk failure loses every uploaded
  document. Add a backup CronJob (model on
  `base/neo4j/offsite-upload-cronjob.yaml`) before this is the only
  copy of any document worth keeping.
- **Per-color sync.** Documents written to `ds-documents-blue` are
  not auto-synced to `ds-documents-green`. If you cut over from
  blue to green, only documents uploaded post-cutover land in
  green's bucket. Either: (a) accept the gap (uploads were ongoing
  during cutover, lossy by design); or (b) add an `mc mirror`
  pre-cutover step (better).
- **Object retention.** Uploaded documents are kept forever today.
  GDPR/PDP-compliant deletion when an admission record is purged
  is a separate workstream.
