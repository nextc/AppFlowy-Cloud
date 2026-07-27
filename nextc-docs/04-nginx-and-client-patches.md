# 04 — Nginx & Client Patches

The `Nginx` service is a reverse proxy **and** the place we patch the closed web SPA's behavior
without rebuilding it. Config is stored base64 in `NGINX_CONFIG_B64` (edit procedure in
[02](02-railway-deployment.md)). A committed copy of the current live config lives at
[`nginx.live.conf`](nginx.live.conf) (regenerate by decoding `NGINX_CONFIG_B64`).

## Routes

| Location | Upstream | Purpose |
|----------|----------|---------|
| `/gotrue/` | `__GOTRUE_HOST__:9999` | Auth API (strips `/gotrue` prefix) |
| `/ws` | `__CLOUD_HOST__:8000` | Collab websocket (upgrade headers, 86400s read timeout) |
| `/api` | `__CLOUD_HOST__:8000` | Cloud REST; sub-locations for `/api/workspace/{id}/publish` (256M), `/api/chat` (streaming), `/api/import` (2G) |
| `/minio-api/` | `__MINIO_HOST__:9000` | S3 API (strips prefix; unbuffered; large uploads) |
| `/ai/` | `__CLOUD_HOST__:8000` | AI endpoints |
| `/console` | `admin-frontend.railway.internal:3000` | Super-admin (forces `X-Forwarded-Proto https`) |
| `/minio/` + `= /minio` redirect | `minio.railway.internal:9001` | MinIO console (strips prefix, ws upgrade) |
| `= /billing/api/v1/subscriptions` | — (returns JSON) | **Billing spoof** (see below) |
| `^~ /billing/api/v1/active-subscription/` | — (returns JSON) | **Billing spoof** |
| `/` | `__WEB_HOST__:80` | Web SPA + client patches (billing/button/reload) |

Top-level `http {}` has `gzip on` + `gzip_types` for JS/CSS/HTML/JSON/wasm (needed because we disable
**upstream** gzip on `location /` — see button-hide below).

## Patch 1 — Billing-API spoof (unblocks all paid gates)

The OSS cloud enforces **no** seat cap server-side, but AppFlowy-Web gates features client-side on the
workspace's subscription plan. With no `/billing` endpoints, the web defaults to `free` → 2-member
invite cap. The invite modal's `getWorkspaceSubscriptions` calls **both** endpoints and filters one by
the other, so **both must be spoofed** (spoofing only `/subscriptions` left it stuck on Free):

```nginx
location = /billing/api/v1/subscriptions {
    default_type application/json;
    return 200 '{"code":0,"message":"","data":[{"plan":"team"},{"plan":"ai_max"}]}';
}
location ^~ /billing/api/v1/active-subscription/ {
    default_type application/json;
    return 200 '{"code":0,"message":"","data":["team","ai_max"]}';
}
```

These two are the **only** client-side subscription gates (verified across every `/billing/` call in
the web http_api): member limit (`InviteMember.tsx`) and >7MB file upload (~http_api line 1611).
`subscription-link` / `cancel-subscription` are upgrade/cancel actions, not gates. So this is the
single "unblock all paid features" fix. `SubscriptionPlan` enum: `free | pro | team | ai_max`.

> Caveat: `ai_max` unlocks the **AI UI**, but AI won't function without an OpenAI/Azure key on the
> `appflowy_ai` service. Hard-refresh the SPA after changing the spoof so it re-fetches.

## Patch 2 — Hide "Upgrade to Pro Plan" / "Get AI Max" buttons

Those menu items in `Workspaces.tsx` are hardcoded `{isOwner && ...}` — **not** plan-gated, so they
always show for owners. Hidden cosmetically by a `<script>` injected into `location /` via
`sub_filter` on `</head>`: a `MutationObserver` hides any `<button>` whose text === those labels.

This required **disabling upstream gzip** on `location /` (`proxy_set_header Accept-Encoding "";`) so
`sub_filter` can rewrite the (now-uncompressed) HTML — which is why our own nginx re-adds `gzip on` at
the `http {}` level so JS/CSS/etc. still reach the browser compressed.

## Patch 3 — `af_ws_retry` first-login auto-reload

Injected as a **second** `<script>` in the same `</head>` sub_filter replacement. Fixes the
first-login blank-sidebar race (see [03](03-onboarding-and-provisioning.md) and
[06](06-known-issues-and-fixes.md)).

Behavior:
- Hooks **both** `window.fetch` **and** `XMLHttpRequest` — AppFlowy-Web uses axios/XHR, so a
  fetch-only hook would miss the call.
- Watches responses whose pathname === `/api/workspace`.
- If `data` is an **empty array** (authenticated-but-unprovisioned → `200 {data:[]}`, verified in
  `src/api/workspace.rs:534` `list_workspace_handler` → `AppResponse::Ok().with_data`), it does a
  guarded `location.reload()`.
- **Bounded:** max 2 reloads via a `sessionStorage['af_ws_retry']` counter (delays 6s then 9s, to
  outlast one 5s poll). Sets the key to `'99'` — stop forever — once a non-empty list is seen.
- **Fails safe:** if the response shape ever differs, the retry simply never fires (no false reload)
  and the old manual-refresh workaround still applies. No reload loops.

All three scripts live in the same `</head>` sub_filter replacement (`sub_filter_once on`).

## Patch 4 — Owner-only delete-hider (role-aware)

Injected as a **third** `<script>` in the same `</head>` sub_filter replacement. Hides delete menu
items from non-Owner members; server enforces Owner-only deletion on REST endpoints; CRDT row/list
deletions (which have no REST endpoint) are blocked client-side by this UI guardrail. See
[06](06-known-issues-and-fixes.md).

Behavior:
- Reads the current user's **role** from `GET /api/workspace` response (field `role`, values
  `Owner`/`Member`/`Guest`) using the token in `localStorage.token`, matched to the active workspace
  ID (extracted from `/app/<wsId>` URL path).
- If role != `Owner`, a `MutationObserver` watches the DOM and hides clickable `<div>` elements
  inside popover menus whose text is one of: **Delete**, **Move to Trash**, **Delete permanently**,
  **Delete view**.
- **Fails open:** any error in the script (role fetch timeout, malformed response, localStorage miss)
  silently stops the observer but never breaks the app — deletion UI remains visible on error, and the
  server guard (enforce_role_strong) still prevents server deletes.
- Paired with server-side `enforce_role_strong(AFRole::Owner)` on 4 delete endpoints
  (`move_to_trash`, `delete_page_from_trash`, `delete_all_pages_from_trash`, `delete_collab`). CRDT
  deletes (kanban row/list/field) have no server equivalent and are guarded by this client-side hide.

## Editing checklist

1. Decode `NGINX_CONFIG_B64` to a working `.conf` (or start from committed [`nginx.live.conf`](nginx.live.conf))
   and edit it. Keep JS in `sub_filter` values **single-quoted only** — a stray double
   quote breaks the nginx double-quoted string.
2. Validate with `docker run nginx:1.27-alpine nginx -t` (substitute `__RESOLVER__` → `127.0.0.11`
   first). The `duplicate MIME type "text/html"` warning from `gzip_types` is harmless.
3. Re-encode → set `NGINX_CONFIG_B64` → `railway redeploy --service Nginx`.
4. Verify live: `curl -s https://b.mgvlabs.com/ | grep af_ws_retry` (and the billing spoof with a
   direct `curl` to those paths).
