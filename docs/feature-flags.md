# Feature flags — Unleash integration guide

Self-hosted Unleash at `https://unleash.cybe.tech:8443/`. Use flags
to decouple deploy from release — ship code to prod hidden behind a
flag, flip when business is ready.

## Why feature flags?

| Without flags | With flags |
|---|---|
| Deploy = release. Broken feature = rollback. | Deploy once, release via toggle. Broken = flip off. |
| Feature in prod = feature visible to users. | Feature in prod = hidden until you flip the switch. |
| A/B testing = two branches, two deploys. | A/B testing = one deploy, two flag variants. |
| Canary = blue/green dance. | Canary = flag at 5% user rollout, ramp over hours. |

This is how Netflix, Shopify, Airbnb, and every modern SaaS shipping
10+ times/day operates.

## Initial setup (one-time admin)

1. Seal + commit the two SealedSecrets per
   `platform/unleash/secrets.README.md`
2. Wait for ArgoCD to sync `unleash-stack` Application → Healthy
3. DNS: add `unleash.cybe.tech` A record → cybe server IP
4. Host nginx: copy the template below to
   `/etc/nginx/sites-available/unleash.cybe.tech`, symlink to
   `sites-enabled`, `nginx -t && systemctl reload nginx`
5. Browser: https://unleash.cybe.tech:8443/ → login `admin` +
   password from 1Password

### nginx template for the cybe host

```nginx
# /etc/nginx/sites-available/unleash.cybe.tech
server {
    listen 8443 ssl http2;
    server_name unleash.cybe.tech;

    ssl_certificate /etc/letsencrypt/live/cybe.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cybe.tech/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Pass to Traefik NodePort (which then routes via Ingress)
    location / {
        proxy_pass http://127.0.0.1:32118;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;

        # Unleash admin UI uses websockets for live updates
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Client SDKs may hold connection during backoff
        proxy_read_timeout 60s;
    }
}
```

## Creating a flag (via UI)

1. Login → **Projects** → `default` → **Create toggle**
2. Name: `bulk-upload-v2` (kebab-case; this string appears in code)
3. Type: `release` (other types: `experiment`, `kill-switch`, `operational`)
4. Description: what this flag controls
5. Create. Flag is created **disabled everywhere** by default.
6. Add environments: click the flag → Environments tab → enable for
   `development`, `production`, etc.
7. Set strategies: gradual rollout (0% → 100%), userIds, applicationName,
   flexibleRollout with stickiness, etc.

## Creating a client SDK token

Each service needs a **client token** (NOT an admin token).

1. Login → **Admin** → **API access** → **New token**
2. Type: `Client`
3. Environment: select (e.g. `production` for prod, `development` for
   lab envs)
4. Projects: `*` (or restrict to specific)
5. Copy the token — it's shown ONCE.
6. Add as SealedSecret to the service's overlay — e.g. seal to
   `unleash-client-token` and reference from Deployment env.

## Service integration

### Rust services (admission, auth, notification, otp)

Add to `Cargo.toml`:

```toml
[dependencies]
unleash-api-client = "0.11"
tokio = { version = "1", features = ["full"] }
```

Initialize once at startup:

```rust
use unleash_api_client::client::ClientBuilder;

#[tokio::main]
async fn main() {
    let unleash_url = std::env::var("UNLEASH_URL")
        .unwrap_or_else(|_| "http://unleash.unleash.svc.cluster.local:4242".into());
    let unleash_token = std::env::var("UNLEASH_CLIENT_TOKEN")
        .expect("UNLEASH_CLIENT_TOKEN required");
    let app_name = std::env::var("UNLEASH_APP_NAME")
        .unwrap_or_else(|_| "admission-service".into());
    let env_name = std::env::var("UNLEASH_ENVIRONMENT")
        .unwrap_or_else(|_| "development".into());

    let client = ClientBuilder::default()
        .interval(10_000)  // poll every 10s
        .into_client::<YourFeatureEnum, reqwest::Client>(
            &unleash_url,
            &app_name,
            &env_name,
            Some(unleash_token),
        )
        .expect("unleash client");

    // Start background poller
    tokio::spawn({
        let client = client.clone();
        async move { client.poll_for_updates().await; }
    });

    // Later, in a handler:
    if client.is_enabled(YourFeatureEnum::BulkUploadV2, None, false) {
        new_bulk_upload(req).await
    } else {
        old_bulk_upload(req).await
    }
}
```

Define your feature enum once:

```rust
use enum_map::Enum;
use strum_macros::{EnumIter, EnumString, Display};

#[derive(Debug, Copy, Clone, EnumIter, EnumString, Display, Enum)]
#[strum(serialize_all = "kebab-case")]
pub enum AdmissionFeatures {
    BulkUploadV2,
    NewValidationRules,
    // add as you ship new flagged features
}
```

### Next.js frontend

```bash
npm install unleash-proxy-client
```

```tsx
// src/app/providers.tsx (or equivalent)
import { UnleashClient } from 'unleash-proxy-client';

export const unleash = new UnleashClient({
  url: process.env.NEXT_PUBLIC_UNLEASH_PROXY_URL!,  // see note below
  clientKey: process.env.NEXT_PUBLIC_UNLEASH_FRONTEND_TOKEN!,
  appName: 'digital-school-frontend',
  environment: process.env.NEXT_PUBLIC_APP_ENV ?? 'development',
  refreshInterval: 15,
});

unleash.start();

// In components:
import { useFlag } from '@unleash/nextjs';
const showNewUI = useFlag('new-dashboard-ui');
```

**Note for frontend**: browser-exposed code cannot use a backend client
token (leaks secret). Either:
1. Run an **Unleash proxy** (edge SDK) in the cluster that exposes
   safe subset of flags. Recommended.
2. Create a **frontend token** in Unleash (limited scope) and accept
   it's visible in network tab.

For Phase 1, frontend tokens are fine. Revisit with proxy when
user-count / flag count grows.

## Environment matrix

Map your K8s namespaces to Unleash environments in the UI:

| Unleash env   | K8s namespace(s)           | Purpose                    |
|---------------|----------------------------|----------------------------|
| `development` | school-dev                 | Always-on, dev experiments |
| `test`        | school-test                | QA flags, integration      |
| `staging`     | school-staging             | UAT, pre-prod validation   |
| `production`  | school-prod-blue/green     | Live users                 |

In Unleash UI: Admin → Environments → Create the above 4 (Unleash
only ships with `development` and `production` by default).

Each environment has independent flag state — e.g. `bulk-upload-v2`
can be ON in `staging`, OFF in `production`.

## Typical release flow with flags

### Before (branch-gated)
```
day 1:  merge staging/bulk-upload → test at staging
day 5:  merge to main → cascade to prod (feature live for everyone)
```

### After (flag-gated)
```
day 1:  merge bulk-upload code to main (flag OFF in prod everywhere)
        → cascades to prod, shipped but invisible
day 2:  flip flag ON for internal team only (userId strategy)
        → internal users try it
day 3:  flip flag to 5% gradualRollout
        → 5% of real users see it; monitor error rate in Grafana
day 4:  ramp to 25%, 50%, 100% over a few hours
day 5:  remove flag from code (next release cleans up conditional)
```

**Zero rollbacks needed**. Bug at 5%? Flip back to 0% in 1 click.

## Flag hygiene rules

1. **Every flag must have an owner** (Unleash tag: `owner:<name>`).
2. **Every flag must have an expected remove-by date** (tag:
   `remove-by:2026-05-01`).
3. **Quarterly: review all flags**. Delete any that are fully rolled
   out and older than 90 days — dead conditionals rot code fast.
4. **Don't nest flags inside flags** more than 1 level.
5. **Kill-switch flags** (type: `kill-switch`) should exist for every
   risky subsystem (payment, auth, email sending). Default to ON.
   Turn off in an incident.

## Observability

Unleash pod exposes Prometheus metrics at `:4242/internal-backstage/prometheus`.
Can be scraped by the existing kube-prometheus-stack. Useful metrics:
- `unleash_counter_metrics_api_response_time` (SDK call latency)
- `unleash_counter_client_register` (SDK connection count)

Create a Grafana dashboard tracking flag-check QPS per service —
if a service suddenly calls a flag 100× more, something's wrong.

## Rollback

If Unleash itself goes down:
- **Client SDKs serve stale cache** — last known flag state, no
  disruption to end-users.
- If cache is also cold (new pod starting), SDKs serve the **hard-
  coded default** passed to `is_enabled(flag, default)` in code.
- **Always pass a safe default** (usually `false` — hide the feature).

This is why feature flags are **strictly safer** than blue/green
rollbacks: you can fail open (feature off) without any deploy.

## Further reading

- https://docs.getunleash.io/reference/activation-strategies
- https://martinfowler.com/articles/feature-toggles.html (Fowler's
  original post — mandatory reading before creating your first flag)
