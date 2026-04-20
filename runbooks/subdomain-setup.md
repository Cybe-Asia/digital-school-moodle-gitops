# Public subdomain setup

Template for exposing a `*.cybe.tech` or `*.school.cybe.tech` subdomain
through the cybe host (port 8443 TLS terminator) to a cluster service.

## Naming convention: FLAT, not nested

The wildcard TLS cert is `*.cybe.tech` which only matches **one DNS
level deep**. A host like `school-test.cybe.tech` is two levels and
would need its own cert (separate DNS-01 challenge). We avoid that
by using flat names under `cybe.tech`.

| Host | Cluster target |
|------|----------------|
| `school.cybe.tech` | prod live routing (blue/green) |
| `grafana.cybe.tech` | monitoring/grafana svc |
| `argo.cybe.tech` | argocd-server svc |
| `unleash.cybe.tech` | unleash svc (once bootstrapped) |
| `school-dev.cybe.tech` | school-dev ingress (existing) |
| **`school-test.cybe.tech`** | **school-test ingress** (new) |
| `school-staging.cybe.tech` | school-staging ingress (optional) |
| `school-blue.cybe.tech` | prod blue pool (optional — not yet set up) |
| `school-green.cybe.tech` | prod green pool (optional — not yet set up) |

> ⚠️ Earlier docs referenced `blue.school.cybe.tech` / `green.school.cybe.tech` /
> `staging.school.cybe.tech`. None of those exist on the cybe host today
> (verified April 2026) — they would require either a dedicated cert per
> subdomain, or a wildcard `*.school.cybe.tech` cert obtained via DNS-01.
> Prefer the flat pattern unless you need the nesting for a specific reason.

## Four steps (10 min total)

### 1. Kubernetes Ingress — add `host:` rule

Already done for `school-test.cybe.tech` in
`lab/test/ingress.yaml`. For new subdomains, use the same pattern:

```yaml
spec:
  ingressClassName: traefik
  rules:
    - host: <your-subdomain>.cybe.tech
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <svc-name>
                port:
                  number: <svc-port>
```

### 2. DNS — A record at SiteGround

Login to SiteGround → DNS Zone Editor for `cybe.tech`:

| Field | Value |
|-------|-------|
| Type | A |
| Name | `school-test` (becomes `school-test.cybe.tech`) |
| Points to | `<cybe server public IP>` (currently 202.165.34.138) |
| TTL | 3600 |

Wait ~2 min for propagation. Verify:

```bash
dig +short school-test.cybe.tech
# should return the cybe IP
```

### 3. Host nginx config on cybe server

SSH to cybe server. Create
`/etc/nginx/sites-available/school-test.cybe.tech`:

```nginx
# Proxies school-test.cybe.tech:8443 → Traefik NodePort 32118 → Ingress.
# TLS terminated here using the wildcard *.cybe.tech cert issued
# via DNS-01 (cert-manager renews every 60 days).
server {
    listen 8443 ssl http2;
    server_name school-test.cybe.tech;

    ssl_certificate     /etc/letsencrypt/live/cybe.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cybe.tech/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # Reasonable security headers — match what grafana/argo hosts use
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options           SAMEORIGIN         always;
    add_header X-Content-Type-Options    nosniff            always;

    # Rate limit — use the school_api zone from
    # /etc/nginx/conf.d/rate-limit-zones.conf
    limit_req zone=school_api burst=40 nodelay;

    location / {
        proxy_pass http://127.0.0.1:32118;

        # Host header tells Traefik which Ingress rule to match
        proxy_set_header Host              school-test.cybe.tech;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;

        # Next.js uses websockets for HMR (dev) and streaming SSR
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 60s;
        client_max_body_size 10m;
    }
}
```

Enable + reload:
```bash
sudo ln -s /etc/nginx/sites-available/school-test.cybe.tech \
           /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 4. Verify end-to-end

From anywhere on the internet:

```bash
# TLS handshake should succeed with *.cybe.tech cert
curl -sI https://school-test.cybe.tech:8443/ | head -3

# Should reach the frontend pod
curl -s https://school-test.cybe.tech:8443/ | head -20
```

Then in a browser:
```
https://school-test.cybe.tech:8443/admissions/register
```

Complete EOI form → success page → buttons visible (flag default OFF).
After Unleash bootstrap + flag flip, refresh → buttons hidden.

## Troubleshooting

### HTTP 404 with "page not found" (Traefik default)

Ingress `host:` didn't match. Cross-check:
1. `kubectl -n school-test get ingress school-test-ingress -o yaml | grep host:`
   → must be `school-test.cybe.tech`
2. nginx `proxy_set_header Host school-test.cybe.tech` present and correct
3. Browser actually sending that host (not `10.10.10.200`)

### HTTP 502 Bad Gateway

Traefik NodePort (32118) unreachable from nginx. Check:
```bash
curl -sI -H "Host: school-test.cybe.tech" http://127.0.0.1:32118/
# should return 200 or 301
```
If this fails too, Traefik is the problem, not nginx.

### TLS handshake failure

Cert path wrong. Check:
```bash
ls -la /etc/letsencrypt/live/cybe.tech/
# should see fullchain.pem + privkey.pem

openssl s_client -connect school-test.cybe.tech:8443 -servername school-test.cybe.tech < /dev/null 2>&1 | grep subject=
# should include *.cybe.tech
```

### Connection refused on port 8443

Firewall / ufw blocking. On cybe:
```bash
sudo ufw status | grep 8443
sudo ss -tlnp | grep 8443  # nginx should be listening
```

## Rolling this out for staging + dev

Same steps, substitute `test` → `staging` / `dev`:

- `lab/staging/ingress.yaml` + `lab/dev/ingress.yaml` need matching `host:` rules added
- DNS: `staging.school` and/or `dev.school` A records
- nginx: one config per subdomain (copy-paste + rename)

## Removing a subdomain

Reverse order: nginx disable → DNS delete → ingress host removal → cert
unchanged (wildcard covers all).
