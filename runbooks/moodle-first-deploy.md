# Moodle first-deploy runbook

Applies to any new environment (lab/test, lab/dev, lab/staging, prod/blue, prod/green).

## What gets deployed

```
Namespace: school-<env>
  Deployment: moodle           (PHP 8.2 + Apache, image ghcr.io/cybe-asia/digital-school-moodle:latest)
  Deployment: moodle-mariadb   (MariaDB 10.11, 5Gi PVC)
  Deployment: moodle-redis     (Redis 7, 1Gi PVC)
  PVC: moodle-pvc              (10Gi, moodledata)
  Secret: moodle-mariadb-secret (sealed)
  ConfigMaps: moodle-config, moodle-apache-vhost, moodle-php-ini
  Ingress: moodle-ingress      (host: school-<env>-moodle.cybe.tech)
```

Managed by the base module `base/moodle` + per-env overlay files.

## Post-deploy steps (one-time per environment)

After ArgoCD syncs for the first time:

1. **Wait for all pods Ready**

   ```bash
   NS=school-test  # or whichever env
   kubectl -n $NS wait --for=condition=ready pod -l app=moodle-mariadb --timeout=180s
   kubectl -n $NS wait --for=condition=ready pod -l app=moodle-redis --timeout=60s
   kubectl -n $NS wait --for=condition=ready pod -l app=moodle --timeout=300s
   ```

2. **Run the Moodle CLI installer**

   Creates Moodle's DB schema and the initial admin account. Safe to retry if
   MariaDB's socket isn't quite ready on first attempt (about 1/5 of deploys
   hit a race where the user account isn't propagated yet; just re-run).

   ```bash
   kubectl -n $NS exec deploy/moodle -- \
     php /var/www/html/admin/cli/install_database.php \
     --agree-license \
     --adminuser=admin \
     --adminpass='<choose a strong password>' \
     --adminemail='admin@cybe.tech' \
     --fullname="CYBE Moodle (${NS})" \
     --shortname="CYBE-${NS^^}"
   ```

   Should end with `Installation completed successfully.`

3. **Verify HTTP response**

   ```bash
   HOST=school-${NS#school-}-moodle.cybe.tech
   curl -sS -o /dev/null -w '%{http_code}\n' \
     --resolve ${HOST}:80:10.10.10.200 \
     http://${HOST}/login/
   # expect: 200
   ```

4. **Add public DNS**

   Add an A record for `school-<env>-moodle.cybe.tech` pointing to the
   cluster's external IP (same as other `school-<env>.cybe.tech` entries).

## Prod cutover notes (blue <-> green)

Moodle data is per-color (separate MariaDB per namespace). When cutting
over prod, do this BEFORE swinging the live-moodle ExternalName:

1. Disable quiz availability on the old-active Moodle (Site admin → Site
   administration → Plugins → Activity modules → Quiz → Disable).
2. Wait for in-flight attempts to save (~5 minutes).
3. mariadb-dump from old-active MariaDB.
4. Restore into new-active MariaDB.
5. rsync /var/moodledata between the two pods (plugins, uploaded
   question images, file-upload question attempts).
6. Update `live-moodle` ExternalName to point at new active.
7. Re-enable quiz availability.

See `runbooks/prod-cutover.md` for the coordinated app + Moodle checklist.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| 500 with 'Reverse proxy enabled' | $CFG->reverseproxy set but Traefik not forwarding headers | Remove `reverseproxy` from config.php (already done in base) |
| CLI install 'database overloaded' | MariaDB user not yet created | Re-run installer (race on first boot) |
| `max_input_vars must be at least 5000` | PHP default is 1000 | Ensure `moodle-php-ini` ConfigMap is mounted (already in base) |
| 502 Bad Gateway | Pod restarting after image pull | Wait 30s; check image pull secret if persistent |
