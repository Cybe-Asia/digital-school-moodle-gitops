# Blue/Green Production Cutover

Procedure for promoting a release candidate from `lab/staging` to live
production traffic.

Doc refs: Architecture §8 (blue/green deployment model), Roadmap §6
(Phase 2 scope).

## Preconditions

- `cybe-prod` cluster exists and has ArgoCD, Traefik, sealed-secrets,
  and the school `platform-system` / `monitoring` / `ingress-system`
  namespaces.
- Both `school-prod-blue` and `school-prod-green` are deployed from
  `prod/blue` and `prod/green`. Both are Synced + Healthy in Argo.
- `prod/routing` is deployed. The live hostname `school.cybe.tech`
  currently resolves to **blue**.
- Release candidate has passed `lab/staging` smoke tests.
- Backups of both colors' Neo4j are fresh (<24h old).

## The cutover

### 1. Identify live vs. idle color

```
kubectl -n school-prod-routing get svc live-auth-service -o yaml | grep externalName
# auth-service.school-prod-blue.svc.cluster.local   -> blue is live
#   OR
# auth-service.school-prod-green.svc.cluster.local  -> green is live
```

For this runbook assume **blue is live**, we're deploying to **green**.

### 2. Deploy the candidate to green

Open a PR on `digital-school-gitops` that updates
`prod/green/kustomization.yaml`'s `images:` block to the candidate's
`sha-<gitsha>` tags (same digests as the last validated lab/staging
build — per Roadmap L8, "no rebuild between staging and prod").

Merge the PR. Argo syncs green. Watch:

```
argocd app wait school-prod-green-services --health
```

### 3. Smoke test green in isolation

```
curl -s https://green.school.cybe.tech/api/v1/auth-service/health | jq
curl -s https://green.school.cybe.tech/ | grep -q 'Digital School'
```

Green should be fully functional on its color-specific hostname
**before** it receives live traffic. If any check fails, STOP — debug
green with no customer impact.

### 4. Cutover live traffic

Open a PR that edits `prod/routing/live-ingress.yaml`. In every
ExternalName Service, change:

```yaml
externalName: <svc>.school-prod-blue.svc.cluster.local
# to
externalName: <svc>.school-prod-green.svc.cluster.local
```

This PR requires two reviewers per CODEOWNERS. After merge, Argo
syncs Traefik (~30s). `school.cybe.tech` now resolves to green.

### 5. Monitor

Grafana board `school-live-traffic` (set up during Phase 2 bring-up):
- request rate, error rate, p95 latency per service
- error budget burn

Alert channels: whatever your team uses (Slack/PagerDuty).

Watch for 30 minutes minimum. Expected behaviour:
- Error rate: unchanged from blue baseline
- Latency: within ±10% of blue baseline
- Pod restarts: none
- Neo4j connection errors: zero

### 6. Rollback (if needed)

```
git revert <cutover-PR-merge-SHA> -m 1
git push origin main
```

Argo syncs Traefik back. `school.cybe.tech` resolves to blue again.
Recovery time <60s.

Green stays deployed — do NOT delete it. It's your cross-check for the
post-mortem.

### 7. Teardown (only after 24h bake-in)

Once green has been stable for 24h AND the team is confident:
- Open PR dropping blue's Deployment replicas to 0 (keep Neo4j + PVC)
  so blue becomes the idle side for the NEXT release.
- Image digests stay pinned in `prod/blue/` until the next cutover PR
  flips blue and green semantically.

## Never do

- Delete blue (or green) immediately after cutover. Rollback capability
  requires both sides to be deployable within seconds.
- Deploy a release candidate directly to the live color. Always deploy
  to the idle color first, cut over via routing.
- Rebuild images between staging and prod. Same digest only (Roadmap
  L8 AC: "Staging image digest equals test digest").
- Skip the color-specific smoke test. `curl school.cybe.tech` hits
  live users; `curl green.school.cybe.tech` hits no one.

## Related runbooks

- `runbooks/prod-first-deploy.md` — initial bring-up of the prod cluster
- `runbooks/neo4j-restore-drill.md` — validate backup before any
  cutover
