# Production Launch — school.cybe.tech:8443 on cybe-lab

Operator runbook for bringing school-prod live on the existing
cybe-lab server, exposing `https://school.cybe.tech:8443` without a
tunnel or VPS.

**Architectural note:** this deviates from Arch §2.1 ("production must
never run on internal lab infrastructure"). Accepted deviation for
early launch; plan migration to cybe-prod when one of these triggers
fires:
- Active users > 100 concurrent
- UU PDP audit begins
- Lab hardware struggles (sustained > 50% CPU or memory pressure)

## Preconditions

- Cybe-lab cluster Healthy (this is the case today)
- `prod/{blue,green,routing}` manifests + sealed secrets already in
  `Cybe-Asia/digital-school-gitops` main branch (landed)
- Argo Application templates in `argocd/school-prod-*-application.yaml`
  (landed)
- Public IP `202.165.34.138` reachable on port 8443 inbound
- You have SiteGround DNS admin access for `cybe.tech`
- Interactive SSH (with sudo) to cybe host
- Sealed-secrets master key backed up off-laptop (`runbooks` reminder)

## Order of operations

Do these in order. Each step is idempotent — safe to re-run.

## Step 1 — DNS records in SiteGround

Log into SiteGround → Client Area → Websites → cybe.tech → DNS Zone
Editor. Add three A records:

| Name | Value | TTL |
|---|---|---|
| `school` | `202.165.34.138` | 600 |
| `blue.school` | `202.165.34.138` | 600 |
| `green.school` | `202.165.34.138` | 600 |

Low TTL for first 2 weeks so rollback is fast.

Verify:

```sh
dig +short school.cybe.tech       # should return 202.165.34.138
dig +short blue.school.cybe.tech  # ditto
dig +short green.school.cybe.tech # ditto
```

## Step 2 — Obtain TLS cert via DNS-01 (wildcard)

Let's Encrypt HTTP-01 won't work (port 80 blocked). DNS-01 will.
Request a wildcard cert covering `*.cybe.tech` so one cert covers
school, skillz, and anything else later.

On the cybe host:

```sh
# Install certbot if not present
sudo apt-get install -y certbot

# Run the DNS-01 challenge interactively
sudo certbot certonly --manual \
  --preferred-challenges dns \
  --email devops@cybe.tech \
  --agree-tos --no-eff-email \
  -d 'cybe.tech' -d '*.cybe.tech'
```

Certbot will pause and print something like:

```
Please deploy a DNS TXT record under the name:
  _acme-challenge.cybe.tech
with the following value:
  abc123...xyz
```

Don't press Enter yet. In SiteGround DNS editor, add a TXT record:

| Name | Value | TTL |
|---|---|---|
| `_acme-challenge` | `abc123...xyz` (from certbot) | 60 |

Wait 30 seconds, verify:

```sh
dig +short TXT _acme-challenge.cybe.tech
```

Once it returns the value, press Enter in certbot. LE validates, saves
cert to `/etc/letsencrypt/live/cybe.tech/`.

Note the paths:

```
/etc/letsencrypt/live/cybe.tech/fullchain.pem
/etc/letsencrypt/live/cybe.tech/privkey.pem
```

**Renewal:** `certbot renew` every 60 days. Fully automated once a
SiteGround DNS API hook is configured; manual otherwise. Set a
calendar reminder.

## Step 3 — Add the nginx vhost

The cybe host nginx already serves `skillz.cybe.tech` on 8443. Add a
**new** server block for school; do NOT touch existing ones.

```sh
sudo nano /etc/nginx/sites-available/school-cybe-tech
```

Paste:

```nginx
# school-prod vhost — routes school.cybe.tech:8443 + its *.school
# subdomains into the k3s Traefik ingress.

server {
    listen 8443 ssl http2;
    listen [::]:8443 ssl http2;

    server_name school.cybe.tech blue.school.cybe.tech green.school.cybe.tech;

    ssl_certificate /etc/letsencrypt/live/cybe.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cybe.tech/privkey.pem;

    # Sane TLS defaults
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Common proxy setup
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Port 8443;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # Reasonable timeouts
    proxy_connect_timeout 10s;
    proxy_send_timeout    60s;
    proxy_read_timeout    60s;

    # Body size (Moodle uploads tune up to 200M elsewhere; school
    # does API JSON, 8M is plenty)
    client_max_body_size 8M;

    location / {
        proxy_pass http://127.0.0.1:80;
    }
}
```

Enable + reload:

```sh
sudo ln -s /etc/nginx/sites-available/school-cybe-tech /etc/nginx/sites-enabled/
sudo nginx -t       # MUST show 'syntax is ok' before reload
sudo systemctl reload nginx
```

If `nginx -t` fails, DO NOT reload. Fix the config first. If it
succeeds, the reload is graceful and does not interrupt existing
Moodle connections.

## Step 4 — Register Argo Applications for prod

```sh
# From your laptop
for app in blue green routing; do
  kubectl apply -f "argocd/school-prod-${app}-application.yaml"
done
```

Argo detects the new apps and starts deploying. blue + green come up
identical; routing deploys the live-ingress + ExternalName services.

Watch progress (takes ~5 minutes for all pods ready because of first
image pulls):

```sh
kubectl get application -n platform-system | grep school-prod
kubectl get pods -n school-prod-blue
kubectl get pods -n school-prod-green
kubectl get pods -n school-prod-routing
```

Expected end state: all three Applications Synced + Healthy, 16 pods
Running in blue (5 services × 3 replicas + neo4j), 16 pods in green,
0 pods in routing (ExternalName Services have no pods).

## Step 5 — First smoke test

From your laptop (not from inside the cluster):

```sh
# Blue-direct
curl -vk --max-time 10 https://blue.school.cybe.tech:8443/

# Green-direct
curl -vk --max-time 10 https://green.school.cybe.tech:8443/

# Live (points at blue by default per prod/routing/live-ingress.yaml)
curl -vk --max-time 10 https://school.cybe.tech:8443/
```

All three should return HTTP 200 (or a redirect that follows to 200).
The cert should be valid (green-lock in browser).

Internal validation — backend connectivity:

```sh
# From inside the cluster
kubectl exec -n school-prod-blue deploy/auth-service -- sh -c \
  'curl -sf -o /dev/null -w "%{http_code}\n" http://neo4j:7474/'
# Expected: 200
```

## Step 6 — Open traffic

You're live. Announce `https://school.cybe.tech:8443/`.

## Step 7 — Watch

Grafana (via the lab.cybe.tech ingress): `school-prod-*` should show
up in dashboards. Add an alert for:

- School-prod pod crashloop (already covered by existing
  PrometheusRule — it's a regex `school-.*`)
- Cert expiry < 30d (add via prometheus-blackbox-exporter, not in
  this runbook)

## Rollback

If something goes wrong in step 5 or shortly after:

### Rollback the nginx vhost

```sh
sudo rm /etc/nginx/sites-enabled/school-cybe-tech
sudo nginx -t && sudo systemctl reload nginx
```

Traffic to school.cybe.tech:8443 now returns whatever nginx's default
server sends (typically 404). Moodle unaffected.

### Rollback the Argo apps

```sh
kubectl delete application -n platform-system \
  school-prod-blue-services school-prod-green-services school-prod-routing
```

Argo prunes all pods in `school-prod-*` namespaces (prune: false for
blue/green per the Application manifests, but the apps themselves are
gone so nothing reconciles). Namespaces stay (kubectl delete won't
cascade to the namespace itself).

If you want to also delete the namespaces:

```sh
kubectl delete namespace school-prod-blue school-prod-green school-prod-routing
```

### DNS rollback

Remove the A records from SiteGround. Users get NXDOMAIN.

## Post-launch follow-ups

- Replace `SMTP_PASSWORD=REPLACE-WITH-REAL-SENDGRID-KEY` in both
  blue and green sealed secrets once SendGrid account is set up
- Move the LE cert renewal to fully automated (DNS-01 hook script
  for SiteGround, or switch SiteGround for a DNS provider with an
  API — Cloudflare DNS-only, Route53)
- Rotate the Grafana and Neo4j passwords you wrote down during
  sealing, once the launch stabilises
- First blue/green cutover drill — follow `runbooks/prod-blue-green-cutover.md`
  with a no-op image bump to prove the mechanics
- Add a PodDisruptionBudget for each school-prod deployment so lab
  workloads can't evict all replicas at once
- Fix the expired `skillz.cybe.tech` cert — the new wildcard covers
  it, just update that vhost to point at the same cert paths

## Related files in this repo

- `prod/blue/`, `prod/green/`, `prod/routing/` — the manifests you'll deploy
- `argocd/school-prod-*.yaml` — Argo Application templates
- `runbooks/prod-blue-green-cutover.md` — cutover procedure
- `runbooks/traefik-migrate-to-ingress-system.md` — future platform work
