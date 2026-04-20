# Unleash operations runbook

## Daily: nothing

Unleash is low-ops. ArgoCD keeps it Synced+Healthy. No daily action.

## Weekly: flag hygiene

Once a week, scan flags:

```bash
# Login to Unleash → Flags page → filter by "archived: false"
# For each flag, ask: is it still gating live code?
#   If yes + fully rolled out (100% for > 14 days) → schedule removal
#   If no → archive it
```

## On PagerDuty: "Unleash API down"

### Triage

```bash
kubectl -n unleash get pods
kubectl -n unleash logs deploy/unleash --tail=100
kubectl -n unleash logs deploy/unleash-postgres --tail=50
```

### Common causes

| Symptom                                  | Cause                              | Fix                                                           |
|------------------------------------------|------------------------------------|---------------------------------------------------------------|
| Unleash pod CrashLoop `ECONNREFUSED`     | Postgres down or slow              | Check postgres pod; restart it                                |
| Unleash pod Healthy but UI 500           | App-level DB pool exhausted        | `kubectl rollout restart deploy/unleash -n unleash`           |
| Postgres PVC full                        | Too many events retained           | Bump PVC size; run `DELETE FROM events WHERE created_at < …`  |
| Admin token rejected                     | Token revoked / DB wiped           | Re-seal `unleash-admin-credentials`; rotate                   |
| All pods OK, still unreachable from app  | NetworkPolicy misconfigured        | Check `unleash-server-ingress` NP includes app's namespace    |

### Impact assessment

Check the SDK client logs on app side. If you see:
```
Unleash: failed to fetch features, using cached
```
**Users are NOT affected** — SDK is serving stale flags. This is
the designed behavior. You have hours/days to fix.

If you see:
```
Unleash: no cache available, using defaults
```
**Users see hardcoded defaults** — usually feature OFF. Verify
default behavior is safe for each critical flag.

### Restart order

If both Unleash and Postgres are broken:

```bash
# 1. Restart postgres first
kubectl -n unleash rollout restart deploy/unleash-postgres
kubectl -n unleash rollout status deploy/unleash-postgres

# 2. Then Unleash
kubectl -n unleash rollout restart deploy/unleash
kubectl -n unleash rollout status deploy/unleash
```

## Backup + restore

### Daily backup (TODO — add cronjob)

Not yet configured. Follow pattern from `base/neo4j/backup-cronjob.yaml`:

```yaml
# platform/unleash/backup-cronjob.yaml (to be added)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: unleash-postgres-backup
  namespace: unleash
spec:
  schedule: "0 3 * * *"  # 3am daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16.3-alpine
              command:
                - sh
                - -c
                - |
                  pg_dump -h unleash-postgres -U "$PGUSER" unleash \
                    | gzip > /backup/unleash-$(date +%Y%m%d).sql.gz
                  # 7-day retention
                  find /backup -name "unleash-*.sql.gz" -mtime +7 -delete
              env:
                - name: PGUSER
                  valueFrom:
                    secretKeyRef:
                      name: unleash-postgres-credentials
                      key: username
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: unleash-postgres-credentials
                      key: password
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: unleash-backup
          restartPolicy: OnFailure
```

### Manual backup (while above is pending)

```bash
kubectl -n unleash exec deploy/unleash-postgres -- \
  pg_dump -U unleash unleash \
  | gzip > unleash-$(date +%Y%m%d).sql.gz
```

### Restore

```bash
# Copy backup file to pod
kubectl -n unleash cp ./unleash-20260420.sql.gz unleash-postgres-xxx:/tmp/

# Restore
kubectl -n unleash exec unleash-postgres-xxx -- sh -c \
  "gunzip -c /tmp/unleash-20260420.sql.gz | psql -U unleash unleash"

# Restart Unleash server to pick up restored state
kubectl -n unleash rollout restart deploy/unleash
```

## Scaling

Current: 2 Unleash replicas, 1 Postgres. Good for ~100 services /
1000 flag-checks/sec.

To scale up (if Unleash CPU > 70% sustained):

1. **Horizontal**: `kubectl scale deploy/unleash --replicas=3`
   (but postgres becomes bottleneck — see below)
2. **Postgres**: move from single-pod Deployment to StatefulSet
   with replication (Zalando postgres-operator / CNPG). Effort ~1 day.
3. **Caching proxy**: install Unleash Edge proxy, put it in front of
   Unleash — client SDKs hit the proxy, proxy caches. Offloads Unleash.

At 1000 services / 10k flag-checks/sec, you want the proxy.

## Upgrading Unleash version

1. Bump image tag in `platform/unleash/unleash-server.yaml` (e.g.
   `unleashorg/unleash-server:6.4.0`)
2. Read the release notes for any DB migration warnings
3. Commit to main → ArgoCD auto-sync → rolling update (zero downtime
   because `maxUnavailable: 0`)
4. Unleash runs postgres migrations on boot automatically
5. Watch `kubectl -n unleash logs deploy/unleash -f` during rollout
   for migration errors
