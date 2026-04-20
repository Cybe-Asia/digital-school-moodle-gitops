# Unleash Secrets — admin sealing guide

Two SealedSecrets must be created before `unleash-stack` Application
can sync successfully. They are NOT committed to git here (the
Application waits for them to appear).

## 1. `unleash-postgres-credentials`

Holds the postgres DB creds. Generate and seal:

```bash
# On your laptop (has kubeseal installed + public key fetched)
cd /tmp
PG_USER="unleash"
PG_PASS="$(openssl rand -base64 32 | tr -d '=+/' | cut -c1-24)"
PG_URL="postgres://${PG_USER}:${PG_PASS}@unleash-postgres:5432/unleash"

kubectl create secret generic unleash-postgres-credentials \
  --namespace=unleash \
  --from-literal=username="${PG_USER}" \
  --from-literal=password="${PG_PASS}" \
  --from-literal=url="${PG_URL}" \
  --dry-run=client -o yaml \
  | kubeseal --format=yaml \
    --controller-name=sealed-secrets \
    --controller-namespace=kube-system \
  > unleash-postgres-credentials.sealed.yaml

# Commit to git
cp unleash-postgres-credentials.sealed.yaml \
  /path/to/digital-school-gitops/platform/unleash/
```

Then add to `kustomization.yaml`:

```yaml
resources:
  - unleash-postgres-credentials.sealed.yaml
```

## 2. `unleash-admin-credentials`

Holds the admin UI password + bootstrap admin API token. Generate:

```bash
ADMIN_PASS="$(openssl rand -base64 32 | tr -d '=+/' | cut -c1-24)"
# Admin token format: <envName>.<tokenName>.<random>
ADMIN_TOKEN="*:*.$(openssl rand -hex 32)"

kubectl create secret generic unleash-admin-credentials \
  --namespace=unleash \
  --from-literal=adminPassword="${ADMIN_PASS}" \
  --from-literal=adminToken="${ADMIN_TOKEN}" \
  --dry-run=client -o yaml \
  | kubeseal --format=yaml \
    --controller-name=sealed-secrets \
    --controller-namespace=kube-system \
  > unleash-admin-credentials.sealed.yaml
```

**Save `ADMIN_PASS` to 1Password/vault immediately — you'll need it for
first login at https://unleash.cybe.tech:8443**.

## 3. Apply + log in

```bash
cd /path/to/digital-school-gitops/platform/unleash
# Commit both .sealed.yaml files + add to kustomization.yaml

# Then in browser: https://unleash.cybe.tech:8443/
# Login: admin / <ADMIN_PASS>
# Change password immediately in UI: Profile → Change password
```

## 4. Rotating admin token later

```bash
# In Unleash UI: Admin → API access → Create new token → Admin token
# Revoke the bootstrap one after.
# No need to update secrets.yaml — INIT_ADMIN_API_TOKENS only seeds on
# empty DB. After first boot it's ignored.
```
