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
the `admin-ui.ADMIN_PASSWORD` Railway variable. Remove also blocks auto-rejoin (via `af_admin_excluded`); add re-allows.

## Rotate secrets (done 2026-10-23 after the Railway security incident)

Generate values locally and pass them straight to `railway variables --set … --skip-deploys`, never print them.
The JWT secret and MinIO keys live in one place (GoTrue / MinIO); every consumer is a `${{…}}` reference.

- **JWT:** set `GoTrue.GOTRUE_JWT_SECRET` → redeploy GoTrue, AppFlowy-Cloud, AppFlowy-Worker, provisioner, admin-ui together.
  Signed-in clients recover through their refresh token (observed 2026-10-23).
- **Email hook:** one random base64 value `X` → `email-hook.HOOK_SECRET=X` and
  `GoTrue.GOTRUE_HOOK_SEND_EMAIL_SECRETS=v1,whsec_X` → redeploy both.
- **MinIO:** set `MinIO.MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` → redeploy MinIO, then Cloud + Worker.
  Check inside the container: `mc alias set n http://127.0.0.1:9000 "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD"`.
- **ops@mgvlabs.com password:** neither GoTrue nor Cloud reads `GOTRUE_ADMIN_PASSWORD`; change the user
  itself with `PUT /gotrue/admin/users/{id}` (service-role JWT), then update the env var as a record.
  `GET /admin/users` returns 0 users with a service-role token (audience filter), so get the id another way.
- **Redis:** set `Redis.REDIS_PASSWORD`; `REDIS_URL` is built from it, and Cloud/Worker/admin-frontend reference
  `${{Redis.REDIS_URL}}`. Redeploy Redis first, then the three clients (data persists in `/data`).
- **Brevo:** create the key in Brevo, set `email-hook.BREVO_API_KEY`, redeploy email-hook, then revoke the old key.

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

1. In `nextc/AppFlowy-Cloud`, commit the change and push a tag for it (e.g. `0.9.64-<change-name>`).
2. Run the GitHub Action `build-oss.yml` (`workflow_dispatch`, input `ref` = the tag) — either from
   the dashboard, or via CLI:
   ```bash
   gh workflow run build-oss.yml -R nextc/AppFlowy-Cloud -f ref=<tag>
   ```
   Runners are amd64; builds all three images (`appflowy-cloud`, `appflowy-worker`,
   `admin-frontend`) in parallel, roughly 5–16 minutes depending on image, and pushes to
   `ghcr.io/nextc/{appflowy-cloud,appflowy-worker,admin-frontend}:<tag>` (public GHCR).
3. Point the service(s) that actually need the new tag at it — you don't have to deploy all three
   images every time (e.g. the 2026-09-30 owner-move change only redeployed `AppFlowy-Cloud`):
   ```bash
   railway service source connect --image ghcr.io/nextc/appflowy-cloud:<tag> --service AppFlowy-Cloud
   railway redeploy --service AppFlowy-Cloud --yes
   ```
   Expect a brief client reconnect on the redeploy.
   **Remember the version-pairing rule** ([01](01-architecture.md)) — bump gotrue/web to the
   **same-release-date** tags, not matching numbers. Don't bump one image casually.
   **Ordering rule:** when a change touches both server enforcement and client UI for the same
   feature (e.g. a new role-gate), **deploy Cloud before Web** — the web change should assume the
   server guard is already live, not the other way around.

## Deploy a custom web image (advanced)

**Avoid building custom web images when possible.** The default is to use a stock `appflowyinc/appflowy_web` image (e.g. `0.9.93`). Only build when a web-side code change is unavoidable (custom delete-hider, auth hook, etc.).

### Why to avoid it

- `appflowyinc/appflowy_web` has **no clean git tag** for released versions (the Docker image exists; the source tag does not) — you must guess the tag or build from a branch like `release/0.9.93` or `main`.
- Building from `main` is **far ahead** of any deployed cloud (`~0.16+` vs `0.9.64`). Version-skew will break: web connects but can't render the folder tree or deserialize collab formats the older cloud doesn't support.
- The stock image requires 3 env vars (`AF_BASE_URL`, `AF_GOTRUE_URL`, `AF_WS_V2_URL`); custom builds may not validate them at boot.

### If you must build

1. Create a fork (e.g. `nextc/AppFlowy-Web`) with a `build-oss-web.yml` workflow that mirrors `build-oss.yml` (checkout `inputs.ref`, build amd64, push to GHCR).
2. **Git-tag the source branch** (or check it out by tag/release-date) to match the known-good pairing with your cloud. E.g. for cloud `0.9.64` (2025-08-04), web was released ~2025-08-18; tag or branch from that date, not `main`.
3. Dispatch the workflow with `ref=<your-tag>` → `ghcr.io/nextc/appflowy_web:<your-tag>`.
4. **Test the version-pairing** in a throwaway Railway project first — ensure sidebar/folder renders against your cloud version.
5. If deploying a client-side UI change (e.g., delete-hider, button hiding), **consider nginx `sub_filter` injection instead** (see [04](04-nginx-and-client-patches.md)) — it avoids the rebuild + version-skew risk entirely.

### Rollback

Point the web service back to `appflowyinc/appflowy_web:0.9.93` (or whichever stock version was working) and redeploy.

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
