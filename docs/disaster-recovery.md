# Disaster Recovery — what you must back up off-cluster

If the cybe server catches fire, these are the items that determine
whether you can rebuild from scratch vs. start over. Each has a specific
location, an off-site requirement, and a restore procedure.

## Critical off-cluster backups

Keep each in **at least two separate locations**. The cybe server
counts as 0. Your laptop counts as 1. A password manager counts as 1.
Aim for 2+.

### 1. Sealed-secrets controller private key (P0 — highest criticality)

**What**: the RSA key that encrypts/decrypts every SealedSecret in
this repo. Without it, all SealedSecrets in git become permanently
unreadable.

**Location on cluster**:
```sh
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml
```

**Currently**: backed up locally at
`~/.cybe-secrets-backup/cybe-lab-sealed-secrets-master-key-<date>.yaml`
(chmod 600).

**Off-cluster requirement**: YES. Copy to password manager secure
note / encrypted USB. See sc-289.

**Restore**: `kubectl apply -f` the saved yaml, then restart the
sealed-secrets-controller deployment.

### 2. Grafana admin password

**What**: admin account for https://grafana.cybe.tech:8443/

**Currently**: stored as SealedSecret `grafana-admin` in `monitoring`
namespace.

**Off-cluster requirement**: YES. Password-manager entry.

**Restore**: if forgotten, `kubectl exec` into grafana pod and run
`grafana-cli admin reset-admin-password <new>`.

### 3. ArgoCD admin password

**What**: admin account for https://argo.cybe.tech:8443/

**Currently**: stored in `argocd-initial-admin-secret` on-cluster.

**Off-cluster requirement**: optional — can always re-read from
cluster if cluster is reachable.

**Restore**: delete the `argocd-initial-admin-secret` and restart
`argocd-server`; it regenerates.

### 4. Let's Encrypt wildcard cert + private key

**What**: `/etc/letsencrypt/live/cybe.tech/{fullchain,privkey}.pem` on
the cybe host.

**Off-cluster requirement**: NICE-TO-HAVE. If lost, re-issue within
10 minutes via `certbot certonly --manual --preferred-challenges dns`
(needs SiteGround DNS access).

### 5. rclone config for off-site backups

**What**: credentials to access the Neo4j off-site backup destination.

**Currently**: SealedSecret `neo4j-offsite-rclone` in prod namespaces
(once activated). Plaintext lives in operator's local rclone config.

**Off-cluster requirement**: YES. Without this, you cannot READ the
off-site backups even if they exist.

### 6. Neo4j passwords (per env, per color)

**What**: 4 passwords — blue + green × (neo4j + jwt).

**Currently**: sealed at `prod/{blue,green}/app-secrets-sealed.yaml`
and `...neo4j-secret-sealed.yaml`. Plaintext held by operator.

**Off-cluster requirement**: YES. Password manager entry per color.

**Restore**: re-seal from plaintext, commit, merge. SealedSecrets
unseals. Neo4j data in PVC is encrypted with this password — losing it
= must restore from backup.

### 7. GitHub tokens (PATs)

- `GITOPS_TOKEN` — CI bot's PAT
- `GHCR_TOKEN` — CI bot's PAT for GHCR

**Off-cluster requirement**: NO — regenerate on demand.

### 8. SiteGround DNS panel access

Ensure the login is in your password manager and at least one other
person on the team has access (bus factor).

## Cluster-level DR — can you rebuild from `main` alone?

If cybe cluster is destroyed tomorrow, bringing it back requires:

1. New Kubernetes cluster (k3s reinstall on new hardware)
2. Platform layer: ArgoCD, Traefik, cert-manager, sealed-secrets-controller, kube-prometheus-stack, loki-stack
3. Re-seed sealed-secrets private key (item #1)
4. Apply Argo Applications → Argo pulls everything from `main`
5. Restore Neo4j data from the latest off-site backup (item #5)

Without item #1 off-site: all SealedSecrets become dead weight → must
regenerate every password → re-seal → commit. Painful but possible.

Without items #5 + #6: no prod data. Start over.

Everything else (code, manifests, CI config) is safely in GitHub.

## Recovery time estimate

With all off-site backups available:

| Task | Time |
|---|---|
| New k3s cluster | 30 min |
| Platform layer install | 30 min |
| Re-seed sealed-secrets + apply Argo apps | 15 min |
| Argo sync everything | 20 min |
| Neo4j restore from off-site | 15 min |
| DNS cutover / TLS re-issue | 20 min |
| Smoke-test | 30 min |
| **Total** | **~2.5 hours** |

## Action list (NOW)

1. [ ] **Copy sealed-secrets key** to password manager — 2 min (sc-289)
2. [ ] Copy **Grafana / ArgoCD / Neo4j passwords** to password manager — 5 min
3. [ ] Make sure **SiteGround credentials** are in the same password manager — 2 min
4. [ ] Complete **off-site backup** setup (sc-312) + verify restore once — 1 hour
5. [ ] Document **bus factor**: second person with same off-site access — ongoing

## Testing DR

Quarterly drill:
1. Stand up a second disposable k3s (at home / on a VPS)
2. Follow this doc to rebuild
3. Measure actual recovery time; update this doc with surprises

## See also

- `runbooks/neo4j-restore-drill.md` — single-backup restore
- `runbooks/neo4j-offsite-backup-setup.md` — activate off-site upload
- `runbooks/prod-first-deploy.md` — greenfield prod cluster bring-up
