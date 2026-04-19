# Traefik → `ingress-system` Migration

Moves Traefik from k3s's default `kube-system` namespace into the
dedicated `ingress-system` namespace declared in Roadmap §2.1 L1-S1.

## Prerequisites

- Interactive SSH to every k3s server node (need sudo for
  `/etc/rancher/k3s/config.yaml`).
- Planned maintenance window of 5–10 minutes. During the window,
  ingress traffic to school-*, moodle, and any other namespace using
  Traefik will be disrupted.
- `helm` on operator laptop.
- Grafana / Prometheus + CODEOWNERS already in place (both are
  consumers of Traefik ingress; you want to validate them after the
  move).
- This runbook's companion PR (`feature/ingress-system-prep`) already
  merged so that:
    - `ingress-system` namespace exists in git (declarative)
    - NetworkPolicy `allow-from-traefik` permits both namespaces

## Why move

- Namespace registry (L1-S1) completion.
- Cleaner RBAC scoping: Traefik gets a dedicated namespace separate
  from k3s system components.
- Easier ArgoCD ownership — today Traefik is managed by k3s's
  embedded HelmChart controller; after the move, Argo + Helm chart
  owns it fully (removes k3s HelmChart CRD drift).

## Plan

```
current:                       target:
 kube-system                    kube-system
   └─ traefik (k3s-managed)       └─ (coredns, kube-proxy, etc.)
                                ingress-system
                                  └─ traefik (Helm-managed via Argo)
```

## Procedure

### 1. Disable k3s's embedded Traefik

On **every** k3s server node:

```sh
sudo vi /etc/rancher/k3s/config.yaml
# Ensure (create file if absent):
disable:
  - traefik
```

This tells k3s to stop managing Traefik on next restart. It will not
delete the existing Deployment — that's our job.

### 2. Restart k3s

Rolling, one node at a time:

```sh
sudo systemctl restart k3s        # on server node
# wait ~30s for the node to re-join the control plane
kubectl get nodes                 # confirm all nodes Ready
```

k3s will not reinstall Traefik. The existing Traefik pod keeps serving
traffic because the Deployment manifest in kube-system still exists.
We haven't touched it yet.

### 3. Install fresh Traefik in ingress-system

```sh
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  -n ingress-system --create-namespace \
  --set service.type=LoadBalancer \
  --set providers.kubernetesIngress.publishedService.enabled=true \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=false \
  --set deployment.replicas=2 \
  --set podLabels.app\\.kubernetes\\.io/name=traefik
```

Important labels: `app.kubernetes.io/name=traefik` is what our
NetworkPolicy matches. The chart sets this automatically but verify:

```sh
kubectl get pods -n ingress-system --show-labels | grep traefik
```

### 4. Point ingress class to the new Traefik

Each Ingress in school-*/lab-*/prod-* currently references
`ingressClassName: traefik`. The k3s-installed Traefik owns that
IngressClass today. After step 3, the new Traefik ALSO claims it
(`ingressClass.enabled=true`), but the old one will have been removed
in the next step.

Check which IngressClass exists:

```sh
kubectl get ingressclass
```

If there are two with the same name, there's a race. Resolve by
setting `ingressClass.enabled=false` during initial Helm install and
transferring ownership AFTER the old Traefik is gone (step 5). Simpler
to just do step 5 quickly.

### 5. Remove old Traefik from kube-system

```sh
kubectl -n kube-system delete deployment traefik
kubectl -n kube-system delete service traefik
kubectl -n kube-system delete serviceaccount traefik
kubectl -n kube-system delete helmchart traefik
kubectl -n kube-system delete helmchart traefik-crd
```

k3s no longer reinstalls (step 1). External traffic is now flowing
through the new Traefik.

### 6. Smoke test

```sh
# From a school-dev pod
kubectl exec -n school-dev deploy/auth-service -- \
  curl -s --max-time 5 -o /dev/null -w "%{http_code}\n" http://localhost/

# From outside the cluster
curl -s -o /dev/null -w "%{http_code}\n" http://<node-ip>/api/v1/auth-service/
```

Expected: both 2xx/4xx (any HTTP response; connection refused would
mean Traefik is not routing).

Also check ArgoCD UI: all *-services applications should remain
Synced + Healthy. Grafana should remain reachable at
grafana.lab.cybe.tech (currently routed via the moved Traefik).

### 7. Tighten the NetworkPolicy (optional but recommended)

Once Traefik is definitively in ingress-system, shrink the allow rule
in `base/governance/network-policy.yaml` — remove the kube-system
branch. This is a separate PR. See the file for the dual-source
structure currently there.

## Rollback

If step 3 or 4 goes wrong:

```sh
# Reinstate k3s's embedded Traefik
sudo sed -i '/disable:/,/- traefik/d' /etc/rancher/k3s/config.yaml
sudo systemctl restart k3s
# k3s will reinstall Traefik in kube-system within a minute
```

If step 5 goes wrong (new Traefik not routing):

```sh
# Reapply the k3s-managed Traefik manually
kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
```

## Post-migration TODO

- Transfer Traefik ownership to ArgoCD — add an Argo Application that
  tracks the Helm chart so upgrades are GitOps-managed. Currently
  this runbook uses `helm install` direct.
- Rotate `kube-prometheus-stack-grafana` Ingress annotations if any
  referenced the old Traefik namespace explicitly (they don't today).
- Move the VolumeClaim for Traefik's LE cert (once cert-manager lands)
  into ingress-system too.
