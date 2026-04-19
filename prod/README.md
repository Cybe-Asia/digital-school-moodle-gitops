# `prod/` — Blue/Green production overlays

This directory is the **structural scaffold** for Phase 2 blue/green
production, per Architecture §4 (namespace mapping), §7 (DNS), and §8
(deployment model).

## Status

**Not deployed.** There is no `cybe-prod` cluster yet (Arch §2.1:
production must run on external hardware, not cybe-lab). These
manifests exist so that:

1. The gitops repo is doctrinally complete — paths match Arch §9.
2. When the prod cluster lands, Argo Applications can be pointed here
   without designing the layout from scratch.
3. Operators can rehearse the cutover runbook on lab hardware by
   temporarily registering these as Argo Apps against lab (caveats
   below).

## Layout

```
prod/
├── blue/         # school-prod-blue namespace stack
├── green/        # school-prod-green namespace stack (identical shape)
└── routing/      # Traefik IngressRoute that selects live color
```

## The two colors

`blue/` and `green/` are near-identical kustomize overlays. Each
deploys the full school service stack:

- 5 Deployments (admission, auth, notification, otp, frontend) at
  3 replicas each (prod replica count per Roadmap §3).
- Own Neo4j StatefulSet + PVCs (data + backups).
- Own SealedSecrets — **sealed against the prod cluster's controller
  key**, NOT the lab key. See `runbooks/prod-first-deploy.md`.
- Own Ingress exposing a color-specific hostname
  (`blue.school.cybe.tech`, `green.school.cybe.tech`) for smoke
  testing before cutover.
- Resource-limits tuned to prod profile (8 CPU / 16Gi per color).

## The routing layer

`routing/` holds a single Traefik IngressRoute with hostname
`school.cybe.tech` pointing at whichever service is currently live.
Cutover = change the backend target in this one file, submit PR,
Argo syncs Traefik config, live traffic flips.

No pod reschedule. No image pull. No downtime.

## Cutover procedure

See `runbooks/prod-blue-green-cutover.md`.

## Rehearsing on cybe-lab (optional)

The prod manifests reference `school-prod-blue` and `school-prod-green`
namespaces. If you create those on the lab cluster and register Argo
Apps pointing at `prod/blue` and `prod/green`, you can exercise the
cutover mechanics without waiting for prod hardware.

**Caveats for lab rehearsal:**
- Lab lacks the resource budget for 2 × 3-replica stacks + Neo4j ×2.
  Reduce replicas to 1 during rehearsal, or run only one color at a
  time.
- Seal secrets against the lab sealed-secrets controller, not prod's
  (different public keys). Re-seal when moving to prod.
- Domain `school.cybe.tech` doesn't resolve to the lab node. Override
  with /etc/hosts or use a test domain on lab.
