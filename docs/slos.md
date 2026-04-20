# Service Level Objectives — school-prod

Framework + starter rules. All numeric targets are **placeholders**
until you have ≥2 weeks of real prod traffic to calibrate against.
Revise them, don't treat them as gospel.

## Why SLOs

An SLO is a specific measurable promise about user-visible behaviour.
It turns "the site should be fast and not break" into "99.5% of API
requests return in <500ms over rolling 28 days." You can then:

- Alert when **error budget** (time allowed to be broken) is burning
  too fast, not when arbitrary thresholds are crossed
- Decide release cadence based on remaining budget
- Talk to stakeholders in terms they trust ("99.5% availability")

## What to measure

### SLI — Service Level Indicator (the measurement)

The raw numeric signal you track. Two classic ones:

| SLI | Definition | PromQL |
|---|---|---|
| **Availability** | fraction of requests that don't return 5xx | `success_rate = 1 - rate(5xx) / rate(total)` |
| **Latency** | fraction of requests served under a target time | `fast_rate = rate(hist_bucket{le="0.5"}) / rate(hist_count)` |

### SLO — Service Level Objective (the target)

What fraction of time you want the SLI to be "good." Written as:

> **99.5%** of HTTP requests succeed (non-5xx) over rolling 28 days

The complement (0.5%) is your **error budget**: ~3.5 hours per 28 days
you're "allowed" to be broken. Burn through it and you should slow
releases down.

## Proposed SLOs for school-prod

**⚠️ These are placeholders.** Revisit after you have traffic data.

| # | Name | Target | Measurement window | Rationale |
|---|---|---|---|---|
| 1 | Auth API availability | 99.5% | 28 days | Critical — login must work |
| 2 | Auth API latency (p95) | <500ms | 28 days | User-visible signin latency |
| 3 | Admission API availability | 99.0% | 28 days | Lead capture can tolerate brief outages |
| 4 | Admission API latency (p95) | <1000ms | 28 days | Less latency-sensitive than login |
| 5 | Frontend availability | 99.5% | 28 days | Homepage outage = catastrophic brand hit |
| 6 | Neo4j availability | 99.5% | 28 days | Every backend call depends on it |
| 7 | Backup freshness | daily run success | 7 days | L3-S2: backup must run every day |

99.5% = 3h36m downtime allowed per 28d.
99.0% = 7h12m allowed.

These are conservative for a fresh prod launch — tighten as you
actually hit the targets consistently.

## Prerequisites before the SLOs are real

This file describes INTENT. Several things must be wired first:

### 1. Service-side metric instrumentation

Services need to expose request metrics in Prometheus format. Today
only Kubernetes-level metrics exist (pod CPU/memory). App-level
request counters + latency histograms are **not instrumented**.

Implement via OpenTelemetry SDK in each service OR via a sidecar
(e.g., Envoy proxy emitting `http_*` metrics). Estimated: 1–2 days per
service.

### 2. ServiceMonitor manifests per service

Once services emit metrics, add a `ServiceMonitor` CR so Prometheus
scrapes them:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: auth-service
  namespace: school-prod-blue
spec:
  selector:
    matchLabels:
      app: auth-service
  endpoints:
    - port: metrics
      interval: 30s
```

### 3. Recording rules for the SLIs

Pre-computed time-series so queries are fast. Example in the template
below — commit as a `PrometheusRule` alongside the existing alert
rules.

### 4. Burn-rate alerts

Alert on *rate of budget consumption*, not threshold crosses. Multi-
window multi-burn-rate pattern (Google SRE workbook Ch. 5):

- **Fast burn** — 2% of 28d budget burned in 1h → page immediately
- **Slow burn** — 5% of 28d budget burned in 6h → ticket, investigate

## Template PrometheusRule (commit when services emit metrics)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: school-slo
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    - name: school.slo.recording
      interval: 30s
      rules:
        # SLI: Auth API success rate (5m window)
        - record: school:auth:success_rate_5m
          expr: |
            sum(rate(http_requests_total{namespace="school-prod-blue", service="auth-service", status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{namespace="school-prod-blue", service="auth-service"}[5m]))

        # SLI: Auth API p95 latency (5m window)
        - record: school:auth:latency_p95_5m
          expr: |
            histogram_quantile(0.95,
              sum by (le) (rate(http_request_duration_seconds_bucket{namespace="school-prod-blue", service="auth-service"}[5m]))
            )

        # Error budget burned over 28d (1.0 = full budget used)
        - record: school:auth:budget_burned_28d
          expr: |
            1 - (
              sum(rate(http_requests_total{namespace="school-prod-blue", service="auth-service", status!~"5.."}[28d]))
              /
              sum(rate(http_requests_total{namespace="school-prod-blue", service="auth-service"}[28d]))
            ) / (1 - 0.995)

    - name: school.slo.alerts
      rules:
        - alert: SchoolAuthFastBurn
          expr: |
            (1 - school:auth:success_rate_5m) > (14.4 * (1 - 0.995))
            and
            (1 - school:auth:success_rate_1h) > (14.4 * (1 - 0.995))
          for: 2m
          labels:
            severity: critical
            slo: auth_availability
          annotations:
            summary: "Auth API fast-burn: 2% of 28d budget consumed in 1h"
            runbook: https://github.com/Cybe-Asia/digital-school-gitops/blob/main/docs/slos.md

        - alert: SchoolAuthSlowBurn
          expr: |
            (1 - school:auth:success_rate_6h) > (6 * (1 - 0.995))
            and
            (1 - school:auth:success_rate_1d) > (6 * (1 - 0.995))
          for: 15m
          labels:
            severity: warning
            slo: auth_availability
          annotations:
            summary: "Auth API slow-burn: 5% of 28d budget consumed in 6h"
            runbook: https://github.com/Cybe-Asia/digital-school-gitops/blob/main/docs/slos.md
```

(The multipliers `14.4` and `6` come from the SRE workbook's
recommended multi-window multi-burn-rate tables — balances false-
positive / false-negative rates.)

## Grafana dashboard (future)

A dedicated SLO dashboard should show:
- **Current period**: how many % of budget used, time-remaining
- **Burn rate**: current rate vs the budget-exhaustion rate
- **Historical**: monthly trend over ~6 months

Can be copied from grafana.com/dashboards or built bespoke.

## Error budget policy (the human side)

SLOs without a policy are just numbers. Proposed policy:

| Budget consumed | What the team does |
|---|---|
| < 50% | Business as usual, ship features |
| 50–80% | Slow releases, prioritise reliability work |
| 80–100% | Feature freeze, all hands on reliability until refresh |
| > 100% | Post-mortem required, adjust SLO or fix root cause |

## Sign-off & review

SLOs without stakeholder buy-in are worthless. When ready to go "real":

1. Draft concrete SLO + budget policy (this doc)
2. Stakeholder review — product owner agrees the numbers are right
3. Ship the PrometheusRules
4. Wire alerts to a real channel (Slack/PagerDuty)
5. Quarterly review — are the SLOs measuring what we care about?

## Timeline suggestion

- **Week 1 prod live**: collect metrics, no SLOs yet
- **Week 2–3**: instrument services with OpenTelemetry or Envoy sidecars
- **Week 4**: write initial SLOs based on observed baseline (75th percentile of current numbers = reasonable target)
- **Month 2**: review + tighten or relax based on real traffic
- **Month 3+**: burn-rate alerts wired to notification channels

## What NOT to do

- **Don't copy Google's 99.99%** unless you have the infrastructure to
  back it up. 99.5% is an excellent starting point.
- **Don't pick SLOs that aren't user-visible.** "CPU < 80%" is not an
  SLO — no user cares about CPU, they care about whether their request
  returns quickly.
- **Don't alert on single thresholds** ("CPU > 90% for 1 min" is noisy
  and usually fine). Alert on budget burn rate.
- **Don't forget to review SLOs quarterly.** Traffic patterns change;
  SLOs must too.
