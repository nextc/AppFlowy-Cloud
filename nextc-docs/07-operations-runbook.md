# 07 — Operations Runbook

Task recipes. Commands assume the Railway CLI is logged in as `ops@mgvlabs.com` and the project is
`appflowy` (ID `788e3d6d-a6d2-48ee-bca4-ebe4aec01ef3`). Secrets are **never** printed — use Railway
vars / reference vars.

> **Classifier-gated** (hand to the user with a leading `! `): prod-DB writes/deletes via
> `railway ssh psql`, `railway domain` mutations, GitHub `gh`/`git push`, `railway delete`, reading
> the Railway token. Our own services' idempotent DDL (provisioner/admin-ui creating a table/trigger
> via their own pg connection) is **not** gated.

## Onboard a teammate

Nothing to do — it's automatic. The teammate: go to **https://b.mgvlabs.com** → enter their
`@mingle.com` email → **click the emailed magic link** (this web build has **no code-entry field**;
typing an OTP code does NOT provision). Within ~1s the provisioner provisions them and auto-joins them
to the shared workspace. If they see a blank sidebar on the very first load, it now auto-reloads; a
manual refresh is the fallback.

## Manage members (add / remove / change role)

Use the member-management UI: **https://admin-ui-production-8ea6.up.railway.app**, password
`mgv-team-2026`. Remove also blocks auto-rejoin (via `af_admin_excluded`); add re-allows.

## Change the member-management UI password

```bash
railway variables --service admin-ui --set "ADMIN_PASSWORD=<new>" --skip-deploys
railway redeploy --service admin-ui -y
# verify: login with new pw → 200, old pw → 401
```

## Edit nginx (routing / billing spoof / client patches)

Full procedure in [02](02-railway-deployment.md#the-nginx-config-mechanism). Short form:
edit `scratchpad/nginx_new4.conf` → `docker … nginx -t` (substitute `__RESOLVER__`→127.0.0.11) →
`base64 | tr -d '\n'` → `railway variables --service Nginx --set NGINX_CONFIG_B64=…` →
`railway redeploy --service Nginx -y`. Keep `nginx_backup.conf` for rollback. Verify live:
`curl -s https://b.mgvlabs.com/ | grep af_ws_retry`.

## Rebuild the OSS images (new version)

1. In `nextc/AppFlowy-Cloud`, ensure the target git tag exists.
2. Run the GitHub Action `build-oss.yml` (`workflow_dispatch`, input `ref` = the tag). Runners are
   amd64; images push to `ghcr.io/nextc/{appflowy-cloud,appflowy-worker,admin-frontend}:<ref>`.
3. Point services at the new tag:
   ```bash
   railway service source connect --image ghcr.io/nextc/appflowy-cloud:<ref> --service AppFlowy-Cloud
   railway redeploy --service AppFlowy-Cloud --from-source
   ```
   **Remember the version-pairing rule** ([01](01-architecture.md)) — bump gotrue/web to the
   **same-release-date** tags, not matching numbers. Don't bump one image casually.

## Switch the whole app to a new domain

1. `railway variables --service Nginx --set "PUBLIC_HOST=<new-host>" --skip-deploys`
2. Ensure the nginx server block still `listen`s on **both 80 and 8080** (Railway custom domains →
   8080).
3. Redeploy the 5 domain-consuming services (GoTrue, AppFlowy-Cloud, AppFlowy-Web, admin-frontend,
   MinIO) + Nginx.
4. Attach the domain in the Railway dashboard (CLI `railway domain` mutations are `Unauthorized`).
5. Add the new origin to `GOTRUE_URI_ALLOW_LIST=https://<new-host>/**`.

## Verify on-demand provisioning end-to-end (LISTEN/NOTIFY)

Startup logs should show `on-demand trigger installed` + `listening for af_provision notifications`.
To fire a harmless test notification (credential stays inside Railway) — **run it yourself**, it's
classifier-gated:

```bash
! railway ssh --service pgvector 'psql -c "SELECT pg_notify('\''af_provision'\'','\''selftest'\'');"'; railway logs --service provisioner | grep -i "notify selftest"
```

Seeing `[provisioner] notify selftest` confirms trigger→NOTIFY→listener→sweep. Otherwise the next real
signup exercises it.

## Recover from a DB wipe

Deploy order: GoTrue (recreate `auth`) → AppFlowy-Cloud (fresh migrate) → Worker → Web →
admin-frontend. Then:
- Re-apply the `@mingle.com` trigger (`scratchpad/deploy/mingle-trigger.sql`) — **wiped with the DB**.
- Confirm GoTrue email-hook env (`GOTRUE_HOOK_SEND_EMAIL_*`, `AUTOCONFIRM=false`) and
  `GOTRUE_URI_ALLOW_LIST=https://b.mgvlabs.com/**`.
- The provisioner re-installs its `af_provision` trigger + `af_admin_excluded` table automatically on
  startup.

## Clean up test users (classifier-gated → user runs)

The provisioner/admin-ui pg queries are **not** gated, but a raw `railway ssh psql` DELETE is. Hand
the user a `!`-prefixed `railway ssh --service pgvector "psql … DELETE …"`. Removing from the shared
workspace via admin-ui is enough to cosmetically resolve most cases (blocks re-join via
`af_admin_excluded`) without a full account delete.

## Health checks

- `curl -s -o /dev/null -w '%{http_code}' https://b.mgvlabs.com/` → 200 (web)
- `.../gotrue/health` → 200 (GoTrue has no homepage; `/gotrue/` 404 is normal)
- `.../api` → 401 (auth-gated, routing OK)
- `railway logs --service provisioner | tail` → sweep/notify activity, `interval 5000ms`

## Rollback an image

`railway service source connect --image appflowyinc/<component>:latest --service <Service>` then
`railway redeploy --from-source`. (Caution: `latest` reintroduces the commercial seat cap on cloud.)
