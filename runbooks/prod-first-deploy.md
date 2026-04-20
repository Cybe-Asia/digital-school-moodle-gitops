# Production First-Deploy Bring-up

One-time procedure for standing up `cybe-prod` and deploying the school
domain for the first time.

Doc refs: Architecture §2.2 (prod infra), §3.2 (cluster B cybe-prod),
§11 (secrets boundary — no shared creds between clusters).

## Preconditions

- Separate prod cluster provisioned (not cybe-lab). Hardware or managed
  K8s, doesn't matter which — just must be isolated from lab.
- Kubeconfig context `cybe-prod` on your laptop.
- DNS records prepared:
  - `school.cybe.tech`        → prod cluster ingress IP
  - `blue.school.cybe.tech`   → prod cluster ingress IP
  - `green.school.cybe.tech`  → prod cluster ingress IP

## 1. Platform layer

Install in order, each via Helm:

```
kubectl --context cybe-prod create ns platform-system
kubectl --context cybe-prod create ns monitoring
kubectl --context cybe-prod create ns ingress-system

# ArgoCD
helm install argocd argo/argo-cd -n platform-system \
  -f configs/argocd-prod-values.yaml

# cert-manager (TLS automation)
helm install cert-manager jetstack/cert-manager -n platform-system \
  --set crds.enabled=true

# Traefik
helm install traefik traefik/traefik -n ingress-system

# sealed-secrets
kubectl --context cybe-prod apply -f \
  https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml

# Prometheus stack
kubectl --context cybe-prod apply -f argocd/monitoring-stack-application.yaml
```

## 2. Seal the prod secrets

**Critical:** prod cluster has a *different* sealed-secrets public key
than lab. You cannot just copy `lab/staging/*-sealed.yaml` into
`prod/blue/`.

Fetch the prod controller's cert:

```
kubectl --context cybe-prod \
  -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o jsonpath='{.items[0].data.tls\.crt}' | base64 -d > /tmp/prod-sealed-cert.pem
```

Prepare plaintext Secret YAML files with prod-grade passwords
(generate with `openssl rand -base64 32`, never reuse lab creds — Arch §11):

```
cat > /tmp/blue-app-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  NEO4J_PASSWORD: "$(openssl rand -base64 32)"
  JWT_SECRET: "$(openssl rand -base64 48)"
  SMTP_PASSWORD: "<actual-sendgrid-api-key>"
EOF

kubeseal --cert /tmp/prod-sealed-cert.pem --format yaml \
  --namespace school-prod-blue \
  < /tmp/blue-app-secrets.yaml \
  > prod/blue/app-secrets-sealed.yaml
```

Repeat for `neo4j-secret` and for the green color. Destroy the plain
files after sealing:

```
shred -u /tmp/blue-app-secrets.yaml /tmp/prod-sealed-cert.pem
```

Commit the sealed files to `digital-school-gitops`:

```
git add prod/blue/app-secrets-sealed.yaml prod/blue/neo4j-secret-sealed.yaml
git add prod/green/app-secrets-sealed.yaml prod/green/neo4j-secret-sealed.yaml
git commit -m "chore(prod): seal initial secrets for cybe-prod cluster"
```

## 3. Back up the prod sealed-secrets master key

```
kubectl --context cybe-prod -n kube-system \
  get secret -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > ~/.cybe-secrets-backup/cybe-prod-sealed-secrets-master-key-$(date +%Y%m%d).yaml
chmod 600 ~/.cybe-secrets-backup/cybe-prod-sealed-secrets-master-key-*
```

Store off-laptop. Losing this file permanently bricks every sealed
secret in prod.

## 4. Register Argo Applications

```
kubectl --context cybe-prod apply -f argocd/school-prod-blue-application.yaml
kubectl --context cybe-prod apply -f argocd/school-prod-green-application.yaml
kubectl --context cybe-prod apply -f argocd/school-prod-routing-application.yaml
```

Argo starts syncing. Blue + green come up identical (same image digest).
Routing initially targets blue (per `prod/routing/live-ingress.yaml`).

## 5. First smoke test

```
curl -s https://blue.school.cybe.tech/api/v1/auth-service/health
curl -s https://green.school.cybe.tech/api/v1/auth-service/health
curl -s https://school.cybe.tech/  # routed to blue by default
```

All three should return 2xx. If any fail, do NOT proceed to open
customer traffic — debug first.

## 6. DNS cutover

Point the public `school.cybe.tech` DNS record at the prod cluster's
ingress IP. TTL low (~60s) for first week in case of rollback.

## 7. Post-bring-up

- Enable branch protection on `digital-school-gitops` main branch
  (see `docs/branch-protection.md`).
- Add the prod sealed-secrets controller key to off-site storage.
- Enable Grafana alert routing for prod (PagerDuty / Slack integration).
- Run a restore drill against prod Neo4j (`runbooks/neo4j-restore-drill.md`
  — adjust namespace for school-prod-blue).
