# 02 — Railway Deployment

Railway workspace **MGVLabs**, project **`appflowy`** (ID `788e3d6d-a6d2-48ee-bca4-ebe4aec01ef3`),
env **production**. CLI account: `ops@mgvlabs.com`.

## The public domain lives in exactly one place

The **`Nginx`** service holds **`PUBLIC_HOST`** (host only, no scheme) = **`b.mgvlabs.com`**.
Every other service references it as a Railway **reference variable** `${{Nginx.PUBLIC_HOST}}`, so the
domain is defined once. Composed forms in use: `https://${{Nginx.PUBLIC_HOST}}`,
`${{Nginx.PUBLIC_HOST}}/gotrue`, `wss://${{Nginx.PUBLIC_HOST}}/ws`.

Vars that reference it:
- **GoTrue:** `GOTRUE_SITE_URL`, `API_EXTERNAL_URL`, `GOTRUE_MAILER_EXTERNAL_HOSTS`
- **AppFlowy-Cloud:** `APPFLOWY_CLOUD_BASE_URL`, `APPFLOWY_WEB_URL`, `APPFLOWY_BASE_URL`
- **AppFlowy-Web:** `APPFLOWY_BASE_URL`, `APPFLOWY_GOTRUE_BASE_URL`, `APPFLOWY_WS_BASE_URL`
- **admin-frontend:** `APPFLOWY_BASE_URL`, `APPFLOWY_ALLOWED_ORIGINS`
- **MinIO:** `MINIO_BROWSER_REDIRECT_URL`

> **To move the whole app to a new domain:** change `PUBLIC_HOST` once, then redeploy those 5 services.

### Reference-var gotchas
- `railway variables --set` **does** store references, but `railway variables --json/--kv`
  **resolves** them on display — so to prove a value is a live reference, change `PUBLIC_HOST` and
  observe, don't trust `--json`.
- `RAILWAY_PUBLIC_DOMAIN` / `RAILWAY_SERVICE_NGINX_URL` are auto-injected but **locked to the
  `.up.railway.app` domain** — they never reflect a custom domain. That's why we keep our own
  `PUBLIC_HOST`.
- Use `--skip-deploys` when setting vars to avoid deploy churn, then `railway redeploy` explicitly.

## Custom domain

`b.mgvlabs.com` is a custom domain on the `Nginx` service.

- **Railway routes custom domains to target port `8080`**, but the Nginx `PORT=80` and its config
  originally only had `listen 80` → a custom domain got `x-railway-fallback: true` 502
  ("Application failed to respond"). **Fix: the nginx server block listens on both `80` and `8080`.**
  (The auto `.up.railway.app` domain works on 80 via Railway magic networking; only the custom domain
  needs 8080.)
- `railway domain` **mutations** return `Unauthorized` for the CLI token (only reads work) — attach a
  domain or change its target port in the dashboard, or hand the command to the user. Making nginx
  listen on 8080 is the CLI-doable workaround for the port issue.
- `board.mgvlabs.com` was **blocked** by a wildcard `*.mgvlabs.com` attached to the separate
  `mgvlabs_web` project — Railway ties hostname ownership to a project, so free it there first. The
  CLI returns a misleading `Unauthorized` for a domain claimed elsewhere.

## The nginx config mechanism (NOT this repo's `nginx/nginx.conf`)

The deployed `Nginx` service holds its config **base64-encoded in `NGINX_CONFIG_B64`**, with
placeholders substituted at container start from Nginx-service env vars:
`__GOTRUE_HOST__`, `__CLOUD_HOST__`, `__MINIO_HOST__`, `__WEB_HOST__`, `__RESOLVER__`.

**To edit routing** (full detail + current routes in [04](04-nginx-and-client-patches.md)):

```bash
# 1. decode current value to a file (keep a backup!)
#    working copies live in scratchpad: nginx_new4.conf (current), nginx_backup.conf (rollback)

# 2. edit the .conf

# 3. validate — substitute placeholders first (esp. __RESOLVER__ → 127.0.0.11, else nginx -t fails on the literal)
sed -e 's/__RESOLVER__/127.0.0.11/g' -e 's/__GOTRUE_HOST__/gotrue.railway.internal/g' \
    -e 's/__CLOUD_HOST__/appflowy-cloud.railway.internal/g' -e 's/__WEB_HOST__/appflowy-web.railway.internal/g' \
    -e 's/__MINIO_HOST__/minio.railway.internal/g' nginx_new4.conf > /tmp/nginx_test.conf
docker run --rm -v /tmp/nginx_test.conf:/etc/nginx/nginx.conf:ro nginx:1.27-alpine nginx -t

# 4. re-encode and set
B64=$(base64 < nginx_new4.conf | tr -d '\n')
railway variables --service Nginx --set "NGINX_CONFIG_B64=$B64" --skip-deploys

# 5. redeploy (briefly affects the whole site — keep the backup for rollback)
railway redeploy --service Nginx -y
```

## Deploying a Node service (email-hook / provisioner / admin-ui)

These are our own Node services, deployed from source in the scratchpad via `railway up`.

> **Stray-project trap:** `railway up` from an **unlinked** directory creates a brand-new stray
> Railway project. **Always link first:**
> ```bash
> cd <service-source-dir>
> railway link -p 788e3d6d-a6d2-48ee-bca4-ebe4aec01ef3 -s <service-name>
> railway up --detach
> ```

## Switching an image source

```bash
railway service source connect --image ghcr.io/nextc/appflowy-cloud:0.9.64 --service AppFlowy-Cloud
railway redeploy --service AppFlowy-Cloud --from-source
```

The image tag can't be read back via the CLI (needs an API token, classifier-gated); `service source
connect` is silent on success, so a clean redeploy / fresh migration is the proxy confirmation.

Same mechanism for the web service (used 2026-09-04 to switch to a self-built compat image — see
[06](06-known-issues-and-fixes.md)):

```bash
railway service source connect --image ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64 --service AppFlowy-Web
railway redeploy --service AppFlowy-Web --from-source -y
```

Rollback for AppFlowy-Web: reconnect `appflowyinc/appflowy_web:0.9.93` (the stock image).

**2026-09-28 redeploy (same tag, new digest):** the `compat-cloud-0.9.64` tag was rebuilt again
(commit `8d1819ca` — new-row filter-prefill fix, see [06](06-known-issues-and-fixes.md)) via the same
`build-oss-web.yml` workflow (run `36397783913`) and pushed to the **same** tag; `railway redeploy
--service AppFlowy-Web --from-source -y` picked it up as Railway deployment `8d045693` (SUCCESS,
2026-09-28 15:34 +07:00). New digest:
`sha256:43782e958d901f66ea4860f79bd3518edd0db73879411f200cd6143ae6f8fa59`. Because the tag is reused,
a plain redeploy does not by itself prove which build is live — check the served bundle or the
digest. **Rollback to the pre-09-28 build specifically** (not stock 0.9.93):
`ghcr.io/nextc/appflowy_web@sha256:30d363df087b545845157b06a2fb910d878556bf24dfccf21dcd9cdd43f3bcc4`
(commit `9b5c4d20`, Railway deployment `54624144`).

**2026-09-29 redeploy #1 (same tag, new digest) — did NOT fix the board scroll-jank symptom:** the
`compat-cloud-0.9.64` tag was rebuilt again (commit `cf3817ff` — a board first-load scroll-jank fix
attempt, see [06](06-known-issues-and-fixes.md)) via the same `build-oss-web.yml` workflow (run
`36487579668`) and pushed to the **same** tag; `railway redeploy --service AppFlowy-Web --from-source
-y` picked it up as Railway deployment `af70e0c4` (SUCCESS, 2026-09-29 04:42 +07:00). New digest:
`sha256:35391c89d62e8c7812ddf4bc8d60879ce1b2615858e11a88083e664b601989fd`. This **supersedes and
removes** the 2026-09-28 build. The tester reported the board was still laggy on first load after this
deploy — `cf3817ff` addressed a real but minor inefficiency, not the dominant cause (see
[06](06-known-issues-and-fixes.md)).

**2026-09-29 redeploy #2 (same tag, new digest) — the actual scroll-jank fix:** profiling with the
Long Animation Frame API identified the true cost as synchronous whole-app re-renders on every
WebSocket message (see [06](06-known-issues-and-fixes.md)). The `compat-cloud-0.9.64` tag was rebuilt
again (commit `e6390209` — "perf: stop re-rendering the whole app on every WebSocket message") via the
same `build-oss-web.yml` workflow (run `36490175194`) and pushed to the **same** tag; `railway
redeploy --service AppFlowy-Web --from-source -y` picked it up as Railway deployment `7958bda5`
(SUCCESS, 2026-09-29 05:06 +07:00). New digest:
`sha256:15e6ccc8b531bea5b9894885b7de30532716412b40014b5a8856aa8d4485aa58`. Live entry bundle
`index-DYksSXv-.js`. This **supersedes and removes** redeploy #1 above. **Rollback to the pre-09-29
build specifically:**
`ghcr.io/nextc/appflowy_web@sha256:43782e958d901f66ea4860f79bd3518edd0db73879411f200cd6143ae6f8fa59`
(commit `8d1819ca`, Railway deployment `8d045693`).

**2026-09-29 redeploy #3 (same tag, new digest) — fixes CRITICAL card-edit-not-saved bug:** live
diagnosis on prod found card title/description edits were syncing only to local IndexedDB and never
reaching the server, so they vanished on hard reload (see [06](06-known-issues-and-fixes.md)). The
`compat-cloud-0.9.64` tag was rebuilt again (commit `5fad2172` — "fix: share in-flight row doc opens
so card edits sync to the server") via the same `build-oss-web.yml` workflow (run `36496870314`) and
pushed to the **same** tag; `railway redeploy --service AppFlowy-Web --from-source -y` picked it up as
Railway deployment `51d67a26` (SUCCESS, 2026-09-29 06:16 +07:00). Live bundle `index-BtS1lMX7.js`.
New digest: `sha256:d867498ee094a9d0e7df2d79b6501986bff62ee64358141fe550954bbe275b37` (read from the
GHCR registry manifest; no newer build has been pushed to the tag since). This **supersedes and
removes** redeploy #2 above.
**Rollback caveat — CRITICAL:** the redeploy #2 image (digest
`sha256:15e6ccc8b531bea5b9894885b7de30532716412b40014b5a8856aa8d4485aa58`, commit `e6390209`, Railway
deployment `7958bda5`) **still has the card-edit-not-saved bug** — do **not** use it as a rollback
target. Every earlier compat build also contains the racy code: `cache/index.ts` is identical from
`9b5c4d20` through `e6390209` (verified with `git diff`), including `8d1819ca` (digest
`sha256:43782e958d901f66ea4860f79bd3518edd0db73879411f200cd6143ae6f8fa59`). The race depends on timing,
so older builds may hit it less often, but none of them is a safe rollback for edit persistence.

**2026-09-30 — Owner-only page-move gating (Cloud + Web):** two service switches, deployed Cloud
first since the web change depends on the server enforcing it.

- **Cloud:** built from `nextc/AppFlowy-Cloud` commit `44219d40` via `build-oss.yml` run
  `36684420913` → `ghcr.io/nextc/appflowy-cloud:0.9.64-owner-move` digest
  `sha256:ee1519394a158546e34f8c3c2ac0132cbc22167a1ca4e3844005591e9150da89`.
  ```bash
  railway service source connect --image ghcr.io/nextc/appflowy-cloud:0.9.64-owner-move --service AppFlowy-Cloud
  railway redeploy --service AppFlowy-Cloud
  ```
  Railway deployment `f8dd443b` (SUCCESS, 2026-09-30 14:50 +07:00). Only the `appflowy-cloud` image
  was switched (the `worker`/`admin-frontend` images from the same run were built but not deployed).
  **Rollback:** `ghcr.io/nextc/appflowy-cloud:0.9.64-delete-guardrail`
  `sha256:b4373228b6da40e03143c0e6e6b3d6869e42a26c6ba61ffd0baf4e943769d9aa` (previous deployment
  `7b3d4b0a`).
- **Web:** same tag `compat-cloud-0.9.64`, two redeploys the same day: `516e8320` (14:12 +07:00,
  commit `458f8e4a` — checklist onBlur fix, GitHub Actions run `36682159134`, bundle
  `index-DLqcuVjZ.js`), then `3281e4a7` (14:51 +07:00, commit `bbd5ea7c` — Owner-only move gating,
  GitHub Actions run `36684546174`, bundle `index-DSRaTnPM.js`). Image digest for `bbd5ea7c`
  *(unverified — not recorded)*. **Rollback to the pre-2026-09-30 build:**
  `ghcr.io/nextc/appflowy_web@sha256:3bc96f915acf944a874462bbd0ffdc7b933c95f27220e3938659df5b8cd61a38`
  (commit `458f8e4a`, deployment `516e8320`) — carries the checklist fix but **not** the move-gating
  UI (see [06](06-known-issues-and-fixes.md) for the caveat).

Full bug/verification detail in [06](06-known-issues-and-fixes.md); build/deploy recipe in
[07](07-operations-runbook.md).

## Deploy-order after a DB wipe

GoTrue (recreate `auth` schema) → AppFlowy-Cloud (fresh migrate) → Worker → Web → admin-frontend.
Then re-apply the `@mingle.com` trigger (it's wiped with the DB) and confirm the email-hook env.

## Classifier-gated actions (hand to the user via `! <cmd>`)

The Claude auto-mode safety classifier blocks these — expect to give the user the command to run:
- Prod-DB **writes/deletes** via `railway ssh --service pgvector psql` (unless user-directed for that
  exact target). Idempotent DDL run by our own services (e.g. the provisioner creating a table/trigger
  via its own pg connection) is **not** gated.
- `railway domain` mutations, GitHub `gh repo fork` / `git push`, `railway delete`.
- Reading the Railway API token or materializing credentials to stdout.
