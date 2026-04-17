# Neo4j Restore Drill (L9-S1)

Validates that backups produced by the daily `neo4j-backup` CronJob
(`base/neo4j/backup-cronjob.yaml`) can be restored into a clean Neo4j
and that the restored database matches the source.

Run this drill at least once per environment per quarter, or after any
change to the backup pipeline.

## Prerequisites

- Kubectl access to the cluster
- Target environment has a running `neo4j-0` pod with at least one
  backup file in `/backups/neo4j-*.cypher`
- Target namespace quota has ~1 CPU / 1Gi headroom for the scratch pod

## Procedure

Replace `${NS}` with `school-dev`, `school-test`, or `school-staging`.

### 1. Capture source state

```sh
NS=school-dev
kubectl exec -n $NS neo4j-0 -- cypher-shell \
  -a bolt://localhost:7687 -u neo4j -p "$(kubectl get secret -n $NS neo4j-secret -o jsonpath='{.data.NEO4J_AUTH}' | base64 -d | cut -d/ -f2)" \
  --format plain "MATCH (n) RETURN count(n) AS nodes;"
kubectl exec -n $NS neo4j-0 -- ls -lh /backups/
```

Record the node count and pick the backup file to restore (usually the
latest).

### 2. Copy the backup file locally

```sh
BACKUP=neo4j-YYYYMMDD-HHMMSS.cypher
kubectl exec -n $NS neo4j-0 -- cat /backups/$BACKUP > /tmp/restore-test.cypher
```

### 3. Launch the scratch pod

Apply `runbooks/neo4j-restore-drill-pod.yaml` (adjust `namespace` if
needed):

```sh
kubectl apply -f runbooks/neo4j-restore-drill-pod.yaml
```

### 4. Wait for bolt port to accept connections

```sh
for i in $(seq 1 24); do
  if kubectl exec -n $NS neo4j-restore-drill -- cypher-shell \
       -a bolt://localhost:7687 -u neo4j -p drilltest123 "RETURN 1;" >/dev/null 2>&1; then
    echo "ready after ${i} attempts"; break
  fi
  sleep 5
done
```

### 5. Copy the backup into the scratch pod and restore

```sh
kubectl cp /tmp/restore-test.cypher $NS/neo4j-restore-drill:/tmp/restore.cypher
kubectl exec -n $NS neo4j-restore-drill -- sh -c \
  "cypher-shell -a bolt://localhost:7687 -u neo4j -p drilltest123 --file /tmp/restore.cypher"
```

### 6. Verify

```sh
kubectl exec -n $NS neo4j-restore-drill -- cypher-shell \
  -a bolt://localhost:7687 -u neo4j -p drilltest123 --format plain \
  "MATCH (n) RETURN count(n) AS total, collect(DISTINCT labels(n)[0]) AS labels;"
kubectl exec -n $NS neo4j-restore-drill -- cypher-shell \
  -a bolt://localhost:7687 -u neo4j -p drilltest123 --format plain \
  "SHOW CONSTRAINTS YIELD name, type;"
```

The node count and constraint list **must match** what you recorded in
step 1.

### 7. Tear down

```sh
kubectl delete pod neo4j-restore-drill -n $NS
```

## Last drill evidence

| Field | Value |
|---|---|
| Date (UTC) | 2026-04-17 |
| Environment | school-dev |
| Operator | devops |
| Backup file | `neo4j-20260417-171500.cypher` |
| Source nodes | 3 |
| Restored nodes | 3 |
| Source constraints | 4 UNIQUENESS |
| Restored constraints | 4 UNIQUENESS |
| Restore duration | ~5s (3-node dataset) |
| Result | **PASS** |

## Troubleshooting

- **`Unable to connect to localhost:7687`** — Neo4j not ready yet.
  Bolt takes 30–60s after pod start. Re-run the wait loop.
- **`Unrecognized setting. No declared setting with name: PORT...`** —
  Kubernetes injected legacy service env vars into the pod. The drill
  pod spec sets `enableServiceLinks: false` and
  `NEO4J_server_config_strict__validation_enabled=false` to
  short-circuit this.
- **`apoc.export.cypher.all ... No such file or directory`** — APOC
  treating export path as relative to `/var/lib/neo4j/import/`. Confirm
  `NEO4J_apoc_import_file_use__neo4j__config=false` is set on the
  source Neo4j pod.

## Known limitations

- Drill runs inside the same namespace as the source, consuming its
  quota briefly. If staging is resource-tight, run the drill during
  off-hours.
- Restored DB is in an empty scratch pod with no PVC — data does not
  persist past the drill. This is intentional: the drill validates the
  **backup file**, not the target infrastructure.
- Large DBs (> ~100K nodes) may require increasing the scratch pod's
  heap size (`NEO4J_dbms_memory_heap_max__size`) and timeout budgets.
