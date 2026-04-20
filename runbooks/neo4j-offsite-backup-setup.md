# Neo4j Off-site Backup Setup

Activates the `neo4j-offsite-upload` CronJob for one environment. The
CronJob ships by default in every namespace but stays dormant until a
`neo4j-offsite-rclone` Secret is present.

**Do this at minimum for `school-prod-blue` and `school-prod-green`.**
Lab envs don't need off-site copies.

## Pick a destination

Any rclone-supported backend works. Cheapest realistic options:

| Provider | Cost | Type | Notes |
|---|---|---|---|
| [Hetzner Storage Box](https://www.hetzner.com/storage/storage-box) | €3.45/mo for 1TB | SFTP / WebDAV | EU latency, great $/GB |
| [Backblaze B2](https://www.backblaze.com/b2) | $6/TB/mo | S3-compatible | Cheap S3 alt, globally available |
| [Wasabi](https://wasabi.com/) | $6.99/TB/mo | S3-compatible | Like B2, no egress fees |
| [AWS S3 + Glacier](https://aws.amazon.com/s3/) | ~$23/TB + cheap Glacier | S3 | Familiar, enterprise-grade |
| SFTP on another VPS you own | $5/mo | SFTP | Full control, same cost as Storage Box |

Whatever you pick, get:
- Endpoint / hostname
- Access credentials (key + secret, or user/password)
- Bucket / folder name (suggested: `cybe-school-backups`)

## Generate `rclone.conf` locally

On your laptop (NOT the cybe server):

```sh
# Install rclone
brew install rclone          # macOS
# or: sudo apt install rclone

# Interactive setup
rclone config
```

Walk through the wizard:
1. `n` for new remote
2. Name it **exactly** `offsite` (the CronJob script hardcodes this name)
3. Pick the storage provider matching what you chose
4. Fill in credentials / endpoint / bucket

Verify it works:

```sh
rclone mkdir offsite:cybe-school-backups/test-folder
rclone lsd offsite:cybe-school-backups/
rclone delete offsite:cybe-school-backups/test-folder
```

The generated config file lives at `~/.config/rclone/rclone.conf`.

## Seal the config per environment

**Per environment** (`school-prod-blue`, `school-prod-green`, repeat):

```sh
NS=school-prod-blue

# Build the plain Secret
kubectl create secret generic neo4j-offsite-rclone \
  --from-file=rclone.conf="$HOME/.config/rclone/rclone.conf" \
  --namespace "$NS" \
  --dry-run=client -o yaml > /tmp/plain-secret.yaml

# Seal it with kubeseal (requires the lab sealed-secrets cert)
kubeseal --cert /tmp/sealed-secrets-cert.pem --format yaml \
  < /tmp/plain-secret.yaml \
  > "prod/blue/neo4j-offsite-rclone-sealed.yaml"

# Wipe the plaintext
shred -u /tmp/plain-secret.yaml

# Repeat for green:
NS=school-prod-green
kubectl create secret generic neo4j-offsite-rclone \
  --from-file=rclone.conf="$HOME/.config/rclone/rclone.conf" \
  --namespace "$NS" --dry-run=client -o yaml > /tmp/plain-secret.yaml
kubeseal --cert /tmp/sealed-secrets-cert.pem --format yaml \
  < /tmp/plain-secret.yaml \
  > "prod/green/neo4j-offsite-rclone-sealed.yaml"
shred -u /tmp/plain-secret.yaml
```

## Wire into kustomization

Add the new sealed-secret files to `prod/blue/kustomization.yaml` and
`prod/green/kustomization.yaml` in the `resources:` list:

```yaml
resources:
  ...
  - neo4j-secret-sealed.yaml
  - neo4j-offsite-rclone-sealed.yaml    # NEW
  - pdbs.yaml
```

## Commit + merge

```sh
git add prod/blue/neo4j-offsite-rclone-sealed.yaml \
        prod/green/neo4j-offsite-rclone-sealed.yaml \
        prod/blue/kustomization.yaml \
        prod/green/kustomization.yaml
git commit -m "chore(prod): activate off-site Neo4j backup"
git push origin main
```

ArgoCD auto-syncs prod apps, Secret lands, next CronJob run (02:15 UTC)
uploads the latest backup.

## Verify it worked

First nightly run after the secret lands (~02:15 UTC next day):

```sh
# See the upload Job's log
kubectl logs -n school-prod-blue -l job-name=neo4j-offsite-upload-<suffix> | tail
```

Or on-demand — trigger the CronJob manually:

```sh
kubectl create job -n school-prod-blue --from=cronjob/neo4j-offsite-upload manual-test
# watch the pod, then check off-site inventory:
rclone ls offsite:cybe-school-backups/school-prod-blue/
```

## Retention policy

- **Local** (`neo4j-backups` PVC on the cybe node): 7 days
- **Off-site**: 30 days

Adjust in:
- Primary CronJob: `base/neo4j/backup-cronjob.yaml` — `-mtime +7 -delete`
- Off-site upload: `base/neo4j/offsite-upload-cronjob.yaml` — `rclone delete --min-age 30d`

## Disaster recovery test

Every quarter: pull a random backup file from the off-site, pipe into
the restore-drill procedure (`runbooks/neo4j-restore-drill.md`). Log
result in that runbook's evidence section.

This is the ultimate validation — proves the entire chain (Neo4j →
local backup → off-site copy → download → restore into scratch Neo4j →
node count match) works end-to-end.
