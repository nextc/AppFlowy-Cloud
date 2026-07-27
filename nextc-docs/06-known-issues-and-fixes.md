# 06 — Known Issues & Fixes (the root-cause catalog)

Every entry here is a bug we already diagnosed and fixed. Read the symptom, trust the root cause, apply
the fix. **When something breaks, check provisioning before chasing version-skew** — that mistake cost
real time.

---

## Seat cap — "Seat limit reached … Upgrade your license"

- **Where:** the `/console` super-admin when adding a member; the whole reason for this project.
- **Root cause:** commercial `appflowyinc/appflowy_cloud` enforces a 1 Member/Owner seat cap that is
  **not in the AGPL source** (verified: no cap check in `invite_workspace_members`, no seat strings,
  no commercial feature).
- **Fix:** run `appflowy_cloud` + `worker` built from OSS git tag `0.9.64`
  (`ghcr.io/nextc/*:0.9.64`). No cap. Validated: 15 invites, 0 seat errors. See
  [01](01-architecture.md).

---

## RecordNotFound — blank sidebar / "Disconnected" on **every** login (no self-heal)

- **Symptom:** after login the sidebar is stuck on skeletons, "Disconnected from cloud", can't create
  a workspace ("no record found"). Refresh does **not** help. Cloud logs:
  `Record not found: Can't find the user profile for <uuid>` on `/api/user/profile`,
  `/api/user/workspace`, `POST /api/workspace`.
- **Root cause:** the gotrue user authenticates fine, but **AppFlowy-Cloud never created the
  `af_user`**. Cloud's only provisioning path is `GET /api/user/verify/{access_token}` →
  `verify_token()`, which the web (v0.1.x, magic-link-only login) calls **only** from its
  `/auth/callback` route (`signInWithUrl` reads `#access_token` from the URL fragment). **GoTrue had
  no `GOTRUE_URI_ALLOW_LIST`**, so it only honored redirects equal to `SITE_URL` and bounced
  `/auth/callback` → site root. The user landed on `/#access_token=...` (not `/auth/callback`), so
  `signInWithUrl`/`verifyToken` never ran → no `af_user` → every `/api/*` 404s.
- **Why it "worked locally":** gotrue always allows `localhost` redirects — so a fresh docker-compose
  deploy on **any** public domain hits the same wall. It is **not** a template artifact, and it was
  **not** collab version-skew (the long-running red herring).
- **Fix (both):**
  1. `GOTRUE_URI_ALLOW_LIST=https://b.mgvlabs.com/**` on GoTrue (so it honors the `/auth/callback`
     redirect).
  2. `email-hook` `buildLink` forces `redirect_to=${site}/auth/callback` (the web's own
     `Redirect_to` header is unreliable — nginx underscore-handling can drop it).
- **Verified:** magiclink → gotrue 303 → `/auth/callback#access_token` → `/api/user/verify`
  `is_new:true` → profile + workspace + folder all 200.
- **Durable fix on top:** the `provisioner` service (below / [03](03-onboarding-and-provisioning.md))
  removes the browser dependency entirely, because link-scanners burn the token anyway.

---

## First-login race — blank sidebar on **first** login, works after a manual refresh

- **Symptom:** distinct from RecordNotFound — it **self-heals on refresh**. Provisioner logs show
  `provisioned <user> (is_new=true)` from the **poll**, i.e. `af_user` was created *after* the user
  logged in.
- **Root cause:** the browser `/auth/callback`→verify provisioning is burned by corporate
  link-scanners, so `af_user` is created only by the provisioner. A brand-new user's app calls
  `GET /api/workspace` **before** the sweep runs → `200 {data:[]}` (verified in
  `src/api/workspace.rs:534` `list_workspace_handler` → `AppResponse::Ok().with_data`) → web renders a
  blank sidebar off the empty list and never re-fetches.
- **Fix (three layers):**
  0. **On-demand provisioning** — Postgres `LISTEN/NOTIFY` trigger on `auth.users` → provisioner
     provisions within ~1s of signup/confirm.
  1. **Poll** `POLL_INTERVAL_MS` 30000 → 5000.
  2. **Client auto-reload** `af_ws_retry` (nginx-injected) — reloads once (bounded to 2) when an
     authenticated user's workspace list comes back empty.
- Detail in [03](03-onboarding-and-provisioning.md) and [04](04-nginx-and-client-patches.md).

---

## Member limit — "Member limit reached, please upgrade" (invite modal)

- **Symptom:** the "Send invites" button disabled after 1–2 members; a client-side gate, **not** a
  server cap (the OSS server has no cap).
- **Root cause:** `InviteMember.tsx`'s `getWorkspaceSubscriptions` calls **both**
  `/billing/api/v1/subscriptions` **and** `/billing/api/v1/active-subscription/{id}` and filters one
  by the other. OSS has no `/billing` → web defaults to `free` → limit 2.
- **Fix:** nginx spoofs **both** endpoints to report `team`+`ai_max`. Spoofing only one leaves it
  stuck on Free. These two are the only client-side subscription gates. See
  [04](04-nginx-and-client-patches.md).

---

## Invite appears to fail / teammate can't be invited

- **Symptom:** inviter sees an invite "error" in the UI; teammate never gets an email.
- **Root cause:** the cloud sends workspace-invite emails over SMTP (`APPFLOWY_MAILER_SMTP_*` →
  Brevo :2525) → **Railway blocks SMTP** → "connection timeout", and the invite API returns an error
  to the UI. But the `af_workspace_invitation` row **is** committed (status=0 pending).
- **Fix:** the `provisioner` auto-accepts pending invites each sweep (and auto-joins everyone to the
  shared workspace anyway). The inviter's UI "error" is **harmless** — the invitee is added within
  ~1s. See [03](03-onboarding-and-provisioning.md).

---

## No email arrives (verification / magic-link / invite)

- **Root cause:** Railway blocks outbound SMTP (587/2525 time out; 443 open).
- **Fix:** GoTrue `send_email` hook → our `email-hook` service → Brevo **HTTP API**. Brevo needs
  Authorised-IPs **off** + account **activated** + a **v3** API key. See
  [05](05-admin-email-and-access.md).

---

## Custom domain 502 — "Application failed to respond"

- **Root cause:** Railway routes **custom** domains to target port **8080**, but nginx only listened
  on 80 (the auto `.up.railway.app` domain works on 80 via magic networking, masking it).
- **Fix:** nginx server block listens on **both** `80` and `8080`. `railway domain` mutations are
  CLI-`Unauthorized` (dashboard/user only); the dual-listen is the CLI-doable workaround. Also:
  `board.mgvlabs.com` was blocked by a `*.mgvlabs.com` wildcard on the separate `mgvlabs_web` project.
  See [02](02-railway-deployment.md).

---

## `/console` user list empty / `403 not_admin`

- **Root cause:** `ops@mgvlabs.com` had GoTrue role `admin`; GoTrue's `/admin/*` API needs
  `supabase_admin` (or `service_role`).
- **Fix:** `UPDATE auth.users SET role='supabase_admin' WHERE email='ops@mgvlabs.com'`, then re-login
  for a fresh JWT. See [05](05-admin-email-and-access.md).

---

## "Upgrade to Pro Plan" / "Get AI Max" buttons showing for owners

- **Root cause:** hardcoded `{isOwner && ...}` in `Workspaces.tsx` — not plan-gated.
- **Fix:** nginx `sub_filter` injects a `MutationObserver` script that hides buttons with those exact
  labels. See [04](04-nginx-and-client-patches.md).

---

## JWT `missing field phone` (401 on Authorization endpoints)

- **Root cause:** endpoints using the `Authorization` extractor decode the JWT locally into
  `GoTrueJWTClaims`, which requires non-optional `email`, `phone`, `app_metadata`, `user_metadata`,
  `role`. A minimal `{sub,email}` JWT works for `/api/user/verify` but 401s here.
- **Fix:** minted JWTs include all five claims. See [03](03-onboarding-and-provisioning.md).

---

## "No access to this page" after a DB wipe

- **Root cause:** the browser cached the last workspace URL (`/app/<old-id>`) which no longer exists.
- **Fix:** incognito or clear site storage.

---

## Database grid rows not syncing to other clients / orphaned DatabaseRow collabs

- **Symptom:** a database (grid) shows rows on the laptop that created them, but other laptops (and any
  client reading fresh from server) see fewer or stale rows. Reload does not help on other machines.
- **Root cause:** an AppFlowy grid is multiple collab objects. The row list lives in the **Database**
  collab's `row_orders` (partition_key=1); each row's cell data is a separate **DatabaseRow** collab
  (partition_key=4). Adding a row persisted the DatabaseRow collab but the Database-collab update that
  registers the row in `row_orders` never reached Postgres. The server builds the row list from
  `row_orders` (src/biz/workspace/page_view.rs `get_page_collab_data_for_database` ~line 2076:
  `db_body.views.get_row_orders` → batch_get_collab for only those ids), so rows absent from row_orders
  are unreachable by every other client even though their data sits intact in `af_collab`.
- **Detection:** dump `af_collab` blobs (hex) for partition_key 1 and 4 for the workspace; a
  DatabaseRow oid whose UUID string does not appear in any Database collab blob is an orphan. **CAUTION:**
  a deleted row also looks orphaned — verify identity by decoded cell text before re-attaching; do not
  resurrect deleted rows.
- **Fix:** build a minimal Yjs update that pushes the missing `{id,height:60}` entries into each view's
  `row_orders` YArray (both inline and non-inline views), and POST it to
  `/api/workspace/v1/{ws}/collab/{oid}/web-update` with `collab_type=1`, signed as the workspace owner.
  Verified idempotent and surgical (fields/metas/filters untouched) locally before sending. Build list
  repair: `64c0fdbf` went len 2481→2822, both views 3→6 rows, 0 orphans confirmed by decoding the blob
  straight from Postgres.

---

## No realtime sync at all: web build 0.1.19 has no websocket client

- **Symptom:** edits never live-propagate between browsers; only a reload shows another client's changes.
- **Root cause:** `appflowyinc/appflowy_web:0.1.19` (paired same-day with cloud `0.9.64`) has **NO
  websocket client** — grep of the served bundle and all chunks shows zero `WebSocket`, zero WASM, zero
  workers. It syncs only via HTTP POST `.../web-update`, with a dead `wsURL` constant
  `wss://test.appflowy.cloud/ws/v1`. Realtime web collab did not exist until web `0.9.93` (2025-08-18),
  which speaks `/ws/v2` and drops `web-update` entirely.
- **Fix:** upgrade AppFlowy-Web image to `appflowyinc/appflowy_web:0.9.93` and set three runtime env
  vars the new image **requires** (it exits at boot if any is missing): `AF_BASE_URL`, `AF_GOTRUE_URL`,
  `AF_WS_V2_URL` (all via `${{Nginx.PUBLIC_HOST}}`; `AF_WS_V2_URL=wss://<host>/ws/v2`). The image injects
  `window.__APP_CONFIG__` into `index.html` at container start via `/docker-entrypoint.sh`. API-compatible
  with cloud `0.9.64` (all `/api/workspace/*` subroutes and billing spoof paths present). Nginx already
  proxies `/ws` with Upgrade headers and `proxy_read_timeout 86400s`, so no routing change needed.
  **Rollback:** reconnect image `0.1.19` (env vars ignored by old image).
- **Verified:** served bundle `index-DGuCBBDk.js` has new `WebSocket` and `/ws/v2`, `__APP_CONFIG__`
  injected, `af_ws_retry` nginx sub_filter survived.

---

## WebSocket drops every ~25–30s with close 1003 "only binary content is supported"

- **Symptom:** after upgrading web to a `/ws/v2` build, the socket connects (101) but closes ~25–30s
  later and reconnect-loops (DevTools shows repeated 101s lasting ~26–29s). Server log shows the same
  client `Join()`ing every ~25–30s.
- **Root cause:** version skew. Every `/ws/v2` web build (`0.9.93`, `0.9.99`, latest) sends a TEXT
  heartbeat frame (`"echo"`, react-use-websocket heartbeat interval 25s). Cloud tag `0.9.64` closes on
  **ANY** text frame: `services/appflowy-collaborate/src/ws2/actors/session.rs` ~line 222
  `ws::Message::Text(_) => ctx.close(... CloseCode::Unsupported "only binary content is supported")`.
  Upstream fixed this in commit `a948c8b3` (#1542, 2025-08-04): `ws::Message::Text(data) => ctx.text(data)`
  (echo it back). Cloud `0.9.64` predates the fix; the web build requires it → unwinnable without patching
  cloud. Nginx (`/ws` `proxy_read_timeout 86400s`) is **not** the cause. Client heartbeat timeout is 60s,
  not 30 — the 1003 close is the actual trigger.
- **Fix:** cherry-pick commit `a948c8b3` onto a branch off tag `0.9.64`, rebuild the OSS cloud image,
  redeploy. Concretely: fork `nextc/AppFlowy-Cloud` branch `0.9.64-ws-text-echo` (`git cherry-pick
  a948c8b3`), ran the fork's `build-oss.yml` workflow with input `ref=0.9.64-ws-text-echo` (dispatch from
  `main` since the workflow file only exists on `main`; checkout step uses `inputs.ref`) → published
  `ghcr.io/nextc/appflowy-cloud:0.9.64-ws-text-echo`. Only the `appflowy-cloud` image matters (the patch
  is in the collaborate crate compiled into it); `worker` and `admin-frontend` unchanged. Deployed via
  Railway service source `connect --image` and redeploy. **Rollback:** reconnect `ghcr.io/nextc/appflowy-cloud:0.9.64`.
- **Verified:** two-session ws diagnostic — a passive session stays open answering PINGs; the echo
  session's TEXT `"echo"` at t+25s now gets a TEXT `"echo"` back and the socket stays open (before the
  patch it closed with 1003 within 0.4s).

---

## LATENT UPSTREAM BUG: Cold workspace actor never snapshots → silent data loss

**Still present even on current AppFlowy-Cloud main** — worth reporting to AppFlowy-IO. The workspace
actor snapshots **only** if a websocket session is attached; cold HTTP writes are never persisted.

- `services/appflowy-collaborate/src/ws2/actors/workspace.rs` `Handler<Snapshot>` reschedules the next
  snapshot **only** in the `Ok(_)` arm; the `Err` arm just logs (~line 613–635). **AND** `snapshot_workspace`
  (`collab_manager.rs` ~315) silently returns `Ok` if `get_workspace_updates(..up_to)` is empty — which
  happens when `up_to == last_message_id == Rid::default()==0`, the case for a Workspace actor
  **created by** an HTTP write with no attached ws session (`Handler<Terminate>` stops it at 60s because
  `sessions_by_client_id.is_empty()`).
- **Net effect:** on a websocket-less deployment, an HTTP `web-update` that births the actor is never
  persisted (dies at 60s, snapshot no-ops at `up_to=0`), while an update arriving while a ws session is
  attached advances `last_message_id` and **does** persist.
- **Why it matters:** fixing the websocket (entries 2 + 3 above) also keeps the actor alive so
  snapshots actually run — one of the hidden wins from migrating to `/ws/v2` builds.

---

## Large file uploads die with broken pipe (single-PUT ≥ ~10–20 MB)

- **Symptom:** `PUT /api/file_storage/{ws}/v1/blob/{parent_dir}` fails with client-side broken pipe /
  connection reset for bodies ≥ ~10–20 MB (observed 2026-07-18 during the kanban migration: every
  video failed, a 8.4 MB PNG succeeded, a 12.9 MB PNG failed). No size check exists in the cloud's
  storage layer — the connection is cut at the nginx/Railway edge (body-size limit).
- **Fix:** use the multipart API — `POST /{ws}/create_upload` (client-chosen `file_id`, e.g.
  content-hash + extension) → `PUT /{ws}/upload_part/{parent_dir}/{file_id}/{upload_id}/{n}` with
  ≤ 8 MB parts (S3 minimum 5 MB per non-final part) → `PUT /{ws}/complete_upload` with the e_tags.
  Retrieval URL is identical to the single-PUT path (`/v1/blob/{parent_dir}/{file_id}`).
  Working implementation: `mgvlabs_kb/appflowy-migration/repair_uploads.py`.

## Board shows every group under "Hidden Groups" and renders no columns

- **Symptom:** a database view switched to Board layout renders an empty canvas — all groups sit in the
  **Hidden Groups** rail with blank names and `0` counts, even though the rows exist and carry valid
  select values, and `hide_ungrouped_column`/`collapse_hidden_groups` are both `false` with every group
  entry `visible: true`.
- **Root cause:** the group-column entries inside `views[*].groups[0].groups` were written as **`Y.Map`
  shared types**. The deployed web build (`0.9.93`) reads each entry with a plain-object accessor, so
  `visible` resolves to `undefined` → `!column.visible` is true → **every** group is classified hidden
  (`useGetBoardHiddenGroup`). Native AppFlowy boards store these as **plain Any maps**
  (`{"id":"JgyfLX","visible":true}`) inside the `Y.Array` — confirmed by diffing a server-created board
  against a hand-built one; the entry container type was the *only* difference.
- **Fix:** rewrite the inner array with plain objects (`arr.push(ids.map(id => ({id, visible: true})))`)
  and POST as a `web-update` (`collab_type=1`). Script: `mgvlabs_kb/appflowy-migration/fix_groups.mjs`.
- **General lesson:** when hand-authoring collab structures, **diff against a natively-created object**
  — Yjs shared types (`Y.Map`/`Y.Array`) and plain `Any` values are both valid Yjs but are *not*
  interchangeable to clients that read one shape.

## Select-option names with leading/trailing whitespace can never be set via REST

- **Symptom:** a row's SingleSelect/MultiSelect cell silently stays empty after `POST
  .../database/{db}/row`, even though the value exactly matches an existing option name. The card then
  has no status and lands nowhere on a grouped board.
- **Root cause:** the cell writer (`cell_from_json_value`, collab-database `select_type_option.rs`)
  runs each incoming value through **`str::trim`** before matching option names — so an option literally
  named `"In Progress "` (trailing space) is unreachable: the trimmed input never equals the untrimmed
  option name. No error is raised; the cell is just left empty.
- **Fix:** write the option **id** straight into the row collab
  (`cells[field_id] = {field_type, data: "<option_id>", …}`, `collab_type=4` web-update). Script:
  `mgvlabs_kb/appflowy-migration/set_select_cell.mjs`. Better long-term: trim option names at creation.
- **Note:** after such a repair, `GET .../row/detail` can keep returning the stale empty value for a
  while (server-side row cache) even though the collab on disk and the web UI are both correct —
  verify against the rendered board or a cold re-read, not an immediate REST read.
- **Found:** 1 card of 517 during the Kan.bn migration; caught only by reconciling *rendered* column
  counts against the source board, which is why that reconciliation is worth doing.

## Owner-only delete guardrail — Members can edit but not delete

- **Motivation:** workspace admin (Owner) wanted to restrict content deletion to Owners only; Members can edit pages/rows but not delete them. AppFlowy has only 3 fixed roles (Owner/Member/Guest) with no custom/manager tier without Rust enum changes.
- **Architecture finding:** deleting kanban rows/lists is CRDT-only (no REST endpoint) — the server cannot gate it, only clients can. Server can gate page/collab deletes (REST endpoints). This is a **guardrail** (client UI + server REST), not tamper-proof.
- **Implementation (2026-07-25):**
  - **AppFlowy-Cloud**: `src/api/workspace.rs` — 4 endpoints changed to `enforce_role_strong(AFRole::Owner)`: move-to-trash, delete-page-from-trash, delete-all-pages-from-trash, delete_collab.
  - **Web client** (`appflowyinc/appflowy_web:0.9.93`): delete menu items are **FULLY HIDDEN** for non-Owners in 9 sites: RowMenu, CardToolbar, DatabaseRowModal, OptionMenu (select-option/tag), ViewActions (database view tab), ColumnMenu (board group/column), PropertyMenu (grid field), app/header/MoreActionsContent.tsx (page delete), and fullcalendar EventPopoverContent. Gated via new `useCanDeleteContent()` hook + dispatch guards.
  - **Desktop client** (AppFlowy Flutter fork): delete menu items fully hidden using existing `role.canDelete` getter. Board card delete keyboard shortcut is gated in `board_shortcut_container.dart` `_removeHandler` (returns unhandled for non-Owners). Grid has no delete-row keyboard shortcut.
- **Verification:** `cargo check` clean (Docker); TypeScript `pnpm type-check` / `tsc` clean. Dart format parses; `flutter analyze` requires full codegen toolchain not available locally.
- **Implementation level:** guardrail (bloc-event handlers themselves are not role-guarded, so deletion logic is gated at dispatch layer, not tamper-proof).
- **Deployment attempt (2026-07-27/28): ROLLED BACK, then replaced with nginx guardrail.**
  - **Initial plan:** Cloud `0.9.64-delete-guardrail` via CI (success). Web built from new fork `nextc/AppFlowy-Web` via new `build-oss-web.yml` (amd64, public GHCR).
  - **Failure:** new web image **broke version-pairing** — built from `AppFlowy-Web:main` (~0.16.x, far ahead of cloud 0.9.64). Connected + joined workspace (cloud logs clean) but couldn't render the folder tree against the older cloud; sidebar/folder wouldn't load.
  - **Rollback:** reconnected web to `appflowyinc/appflowy_web:0.9.93` (stock image, no custom build).
  - **Final solution (live 2026-07-28):** **nginx-injected role-aware delete-hider** (Patch 4 in [04](04-nginx-and-client-patches.md)) on the stock `0.9.93` image. Same delete-item hiding as the custom build, but delivered as a `sub_filter` script instead of baked into a web rebuild. Server cloud guard (`0.9.64-delete-guardrail`) deployed and active.
  - **Coverage:** server REST deletes (page/collab) blocked by cloud `enforce_role_strong`; kanban CRDT deletes (rows/lists/fields) blocked by nginx-injected UI guardrail on stock 0.9.93. **Lesson:** building custom web images from main breaks version-pairing; prefer nginx `sub_filter` for web-side changes on stock images.
  - **Rollback (if needed):** cloud→`0.9.64-ws-text-echo`, web→`appflowyinc/appflowy_web:0.9.93`, nginx (Patch 4 already in stock config).
  - **Desktop:** NOT deployed (separate binary distribution).

---

## New web build shows "connecting forever"

- **Symptom:** after deploying a new AppFlowy-Web build, the SPA loads but "Connecting..." spinner never completes; network tab shows WebSocket never attempts to connect.
- **Root cause:** pre-existing misconfigured env var `APPFLOWY_WS_BASE_URL=wss://b.mgvlabs.com/ws` (MISSING `/v2` suffix). Old web `0.9.93` read the correct fallback `AF_WS_V2_URL` when `APPFLOWY_WS_BASE_URL` was empty; a newer web build (~0.16+) prefers the `APPFLOWY_*` vars and surfaces the bad value. The entrypoint only falls back to `AF_WS_V2_URL` when the new-style var is absent.
- **Fix:** set `APPFLOWY_WS_BASE_URL=wss://b.mgvlabs.com/ws/v2` on the AppFlowy-Web service env vars, then redeploy. The env var is injected into `index.html` as `window.__APP_CONFIG__` at container start (the stock `appflowyinc/appflowy_web:0.9.93` Dockerfile entrypoint `./docker-entrypoint.sh` does this).
- **Verification:** DevTools→Network→WS tab shows websocket connection attempts to `/ws/v2`; app connects + loads sidebar.

---

## New web build: sidebar/folder won't load (version-skew)

- **Symptom:** a fresh web image boots (200), connects to cloud (no errors in cloud logs), and the authorization endpoint fires, but the sidebar and folder tree never render — the page stays blank or shows only top chrome.
- **Root cause:** building the web image from `AppFlowy-Web:main` is **far ahead of the deployed cloud**. Version mismatch on the collab format or the API surface means the web can join the workspace (auth passes) but can't deserialize or render the folder structure against the older cloud's data. Specific root: main web ~0.16+ uses a newer database collab schema / folder serialization that the cloud `0.9.64` → web `0.9.93` pair does not support.
- **Why it happened:** testing a custom web rebuild to deploy the delete-guardrail (see entry above). Building from `main` is the default (`git clone …`), not from a release tag. No tagged releases exist for web `0.9.93` on GitHub (the Docker image exists; the source tag does not).
- **Fix / rollback:** reconnect the web service to `appflowyinc/appflowy_web:0.9.93` (the stock image, not a custom build) and redeploy. If you must build a custom web image, git-tag it from the known-good release date, or patch `0.9.93` directly (avoiding main).
- **Lesson:** never build a custom web image from `main` when the deployed cloud is several releases behind. The version-pairing rule (by release date, not number) applies to rebuilds too. **Prefer nginx `sub_filter` injection for web-side changes** (e.g., client-side UI hides, script injection) to avoid the need for custom web builds and version-skew risk.

---

---

## Deploy / tooling gotchas (quick list)

- `railway up` from an **unlinked** dir creates a **stray project** — always `railway link -p … -s …`
  first.
- `railway variables --json/--kv` **resolves** reference vars — don't trust it to prove a value is a
  live reference; change the source and observe.
- `railway redeploy` on a `railway up` source service triggers a full rebuild each time.
- bitnami/minio won't deploy on Railway → use official `minio/minio` with a Dockerfile wrapper:
  `CMD ["minio","server","/data","--console-address",":9001"]`.
- GoTrue needs both `DATABASE_URL` **and** `GOTRUE_DATABASE_URL` + `API_EXTERNAL_URL`; cloud
  migrations require GoTrue's `auth` schema to exist **first** (deploy order matters).
- `generate_link type=magiclink` upserts → duplicate-key 500 on existing users; `type=recovery` → 404
  for passwordless users → **self-mint the JWT instead**.
