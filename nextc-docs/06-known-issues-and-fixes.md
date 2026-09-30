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

## Custom web image deployed to prod: compat-cloud-0.9.64 (2026-09-04)

- **What shipped:** AppFlowy-Web Railway service swapped from stock `appflowyinc/appflowy_web:0.9.93`
  to self-built `ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64`, via
  `railway service source connect --image ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64 --service
  AppFlowy-Web` + `railway redeploy --service AppFlowy-Web --from-source -y` (prod deploys are
  classifier-gated, so this was user-run). **Rollback:** point the service back to
  `appflowyinc/appflowy_web:0.9.93`.
- **Why a custom build at all — the compat branch:** see the entry above. `nextc/AppFlowy-Web` branch
  `compat-cloud-0.9.64` starts at `d0704de2` (the last commit before the folder-API break `c8f44099`)
  and carries three cherry-picks / new commits:
  - `05592a79` — GHCR build workflow (`.github/workflows/build-oss-web.yml`, `workflow_dispatch` from
    `main` with a `ref` input, tags `ghcr.io/nextc/appflowy_web:<ref>`, publicly pullable — mirrors
    the cloud repo's `build-oss.yml`).
  - `11914cf4` — cherry-pick of Owner-only delete gating (`a3a5d0b4` on `main`, 2026-07-27; 6 conflicts
    hand-resolved keeping this branch's Feb-2026 code structure while preserving the `canDelete` gates
    across all 9 UI sites and the CRDT dispatch guards, per [08 §Phase 8](08-history.md)). Its
    `actionPermissionGates.test.tsx` test file was **dropped** from the pick — it references APIs that
    don't exist on this older base — and still exists only on `main`.
  - `ee87105f` — new feature commit, see below.
  - **Later addition (2026-09-28):** `8d1819ca` — "fix: stop pre-filling new rows from active view
    filters". See "New card auto-assigned a member on Enter" below for the bug it fixes and the
    redeploy details.
- **New feature: shareable card URLs.** Opening a board card's detail modal writes `?r-modal=<rowId>`
  into the URL via `history.pushState` (so browser Back closes the card); closing removes it via
  `history.replaceState` (so Back doesn't reopen it). `Database.tsx` gained an `onModalRowIdChange`
  prop plus an effect that reacts to URL-driven changes of `modalRowId` (back/forward navigation,
  cold-loading a shared link). Cross-database modals (relation rows opened with an explicit `viewId`)
  are intentionally **not** URL-synced — their link can't be rebuilt from the current page's URL.
  `DatabaseView.tsx` reads `window.location.search` directly (not the `useSearchParams` snapshot) when
  writing the param, because the modal's expand button pushes `?r=` and clears the modal in the same
  tick — a stale snapshot would race it. Expand hands over to the pre-existing `?r=<rowId>` full-page
  deep link, dropping `r-modal`. **Verified live on prod:** card click → URL param appears, Escape
  removes it, cold-loading the link opens the board with the modal open and hydrated, expand → `?r=`.
- **New feature: hidden fields expanded in card detail.** `DatabaseRowProperties.tsx` now inits
  `isFilterHidden` with `useState(false)` in code — this is what retires nginx Patch 5 (see
  [04](04-nginx-and-client-patches.md)). The `/static2/` module-graph path and the HTML `sub_filter`
  rewrites Patch 5 also relies on still serve the new bundle correctly and needed no changes.
- **Open caveats (as of this deploy):**
  1. The write path (editing/moving cards) was **not** explicitly verified on this build — a manual
     edit test on prod is still pending. *Partial positive evidence 2026-09-04:* a database-view
     rename (REST page rename + Yjs `view.set(name)` via `useUpdateDatabaseView`) persisted
     server-side across a fresh reload (see the "BoardTemplate tab" entry below). Card/cell edits
     remain untested.
  2. This build's `vite.config.ts` loads React 18.2.0 / ReactDOM 18.2.0 from `cdn.jsdelivr.net` at
     runtime (build-time CDN externals) — a jsdelivr outage would break the app entirely.
  3. ~~In dev testing, a first-ever load with a cold IndexedDB briefly rendered rows with empty cells
     until row docs finished syncing (a reload fixed it).~~ **Hit prod at scale and FIXED same day —
     see the follow-up entry below ("Boards show 'Untitled' cards for minutes on first visit").**
  4. ~~Cosmetic-only: the BoardTemplate board's tab label renders as the underlying view's name
     ("Kid Achievement") on this build.~~ **Data fixed 2026-09-04** — root cause + the rename-modal
     no-op bug it exposed are written up below ("BoardTemplate tab showed 'Kid Achievement'").
- **Verification run:** `tsc` + `eslint` clean; manual in-browser check against production data via a
  dev proxy (deep-link cold load, Escape/backdrop close, Back/Forward, expand, board render).

---

## Boards show "Untitled" cards for minutes on first visit (compat build) — FIXED (2026-09-04)

- **Symptom:** right after the compat-cloud-0.9.64 deploy, every board rendered all cards as
  "Untitled" with empty statuses on the first visit from a given browser; content trickled in slowly
  (minutes), and a reload after hydration showed everything instantly (IndexedDB now warm).
- **Root cause:** this web era bulk-loads row content via `POST
  /api/workspace/{w}/database/{db}/blob/diff`, which cloud 0.9.64 does not serve (404 — route absent
  from `src/api/workspace.rs`). With no seed source, every DatabaseRow collab hydrated one-by-one
  over WebSocket. Meanwhile the page-view fetch the app already makes (`GET
  /api/workspace/{w}/page-view/{viewId}`) returns **all row doc states in one response**
  (`row_data`, raw Yjs doc_state per row — the server decodes `EncodedCollab` before sending, see
  `get_page_collab_data_for_database` in `src/biz/workspace/page_view.rs`) — and the compat build's
  `view-loader` logged the row count and then discarded it (it expected blob/diff to supply rows).
- **Fix (web branch commit `9b5c4d20`, redeployed same day):** `fetchAndApply` in
  `src/application/view-loader/index.ts` now seeds the row-doc cache from `row_data` via a new
  `seedRowDocCacheFromPageData()` in `src/application/database-blob/index.ts` (encoder v1; applies
  directly to already-open row docs, otherwise fills the same seed cache the blob prefetch feeds).
  Same image tag rebuilt via `build-oss-web.yml`; deploy = `railway redeploy --service AppFlowy-Web
  --from-source -y` only (tag unchanged).
- **Verification:** dev with fully wiped IndexedDB against prod — TripFold (content-heavy board)
  hydrated completely in under 5 seconds (previously minutes), `[Database] page-data seed cache
  prepared` → `row seed hit` debug logs confirm the path; prod bundle `index-CCp6jI2L.js` confirmed
  serving with the fix string present; prod TripFold renders fully hydrated.
- **Note:** rows are seeded per page-view fetch (skipped when the database doc is already cached in
  IndexedDB) — the long-tail of never-opened rows on an already-cached database still hydrates via
  WS, which is acceptable since visible rows land in IndexedDB on first hydration.

---

## BoardTemplate tab showed "Kid Achievement" (compat build) — data FIXED, rename bug OPEN (2026-09-04)

- **Symptom:** the BoardTemplate page (view `8354ba8a-6779-4596-a64a-85186b6f7ade`, Tips Sharing
  space) showed "BoardTemplate" in the sidebar/breadcrumb/title but "Kid Achievement" as its single
  database view tab. Renaming the tab to "BoardTemplate" via right-click → Rename appeared to do
  nothing.
- **Root cause (two names, and the tab prefers the hidden one):** a database page has **two name
  records** — the folder/outline name (what the sidebar shows) and the view's own `name` inside the
  database Yjs collab. The compat build renders the tab label Yjs-first:
  `DatabaseTabItem.tsx:66` — `const name = yjsName || override || defaultName;` (upstream code at
  the compat base `d0704de2`, not a fork change). This board's folder name was "BoardTemplate" but
  its inner Yjs name was still "Kid Achievement" — the inner name is stamped at creation and page
  renames don't touch it. Provenance of the stale name *(unverified)*: likely a duplicate of the
  round-3-migrated "Kid Achievement" board — BoardTemplate is absent from the migration manifest and
  the migration scripts never write database-view names.
- **Why renaming didn't fix it — silent no-op bug (still OPEN in the fork):** `RenameModal.tsx`
  guards `if (newValue === view.name) return;`, and `view.name` there comes from `DatabaseTabs`'
  `renameView`, which prefers the **outline/folder** name. So the modal pre-fills "BoardTemplate",
  saving "BoardTemplate" trips the "unchanged" guard, and the Yjs write in `useUpdateDatabaseView`
  (`dispatch.ts` — REST page rename, then `view.set(YjsDatabaseKey.name, …)`) never runs. No error,
  no toast. Any board whose two names diverge cannot be renamed to its folder name in one step.
  **Code fix (not yet made):** make `renameView` in `DatabaseTabs.tsx` resolve the current name the
  same way `DatabaseTabItem` renders it (Yjs-first); needs a web rebuild + redeploy.
- **Data fix applied (prod, 2026-09-04):** renamed the tab twice in the web UI — to "TempName"
  (different string, so the guard passes and both the folder name and the Yjs view name get
  written), then back to "BoardTemplate". Verified after a fresh page reload: tab, title,
  breadcrumb, and sidebar all show "BoardTemplate" — i.e. the write persisted server-side (partial
  positive evidence for the open write-path caveat above; card edits still untested).
- **Workaround for any future occurrence:** double-rename (temp name → intended name) from the tab's
  right-click menu.

---

## New web build shows "connecting forever"

- **Symptom:** after deploying a new AppFlowy-Web build, the SPA loads but "Connecting..." spinner never completes; network tab shows WebSocket never attempts to connect.
- **Root cause:** pre-existing misconfigured env var `APPFLOWY_WS_BASE_URL=wss://b.mgvlabs.com/ws` (MISSING `/v2` suffix). Old web `0.9.93` read the correct fallback `AF_WS_V2_URL` when `APPFLOWY_WS_BASE_URL` was empty; a newer web build (~0.16+) prefers the `APPFLOWY_*` vars and surfaces the bad value. The entrypoint only falls back to `AF_WS_V2_URL` when the new-style var is absent.
- **Fix:** set `APPFLOWY_WS_BASE_URL=wss://b.mgvlabs.com/ws/v2` on the AppFlowy-Web service env vars, then redeploy. The env var is injected into `index.html` as `window.__APP_CONFIG__` at container start (the stock `appflowyinc/appflowy_web:0.9.93` Dockerfile entrypoint `./docker-entrypoint.sh` does this).
- **Verification:** DevTools→Network→WS tab shows websocket connection attempts to `/ws/v2`; app connects + loads sidebar.

---

## New web build: sidebar/folder won't load (version-skew) — root cause pinned down + durable fix found (2026-09-04)

- **Symptom:** a fresh web image boots (200), connects to cloud (no errors in cloud logs), and the authorization endpoint fires, but the sidebar and folder tree never render — the page stays blank or shows only top chrome. Also seen as **every individual view 404ing with "No access to this page"** once past the sidebar.
- **Root cause (confirmed 2026-09-04):** `nextc/AppFlowy-Web:main` carries upstream commit `c8f44099`
  "Use new folder api (#257)" (2026-02-18), which switched view loading to
  `GET /api/workspace/{workspace_id}/view/{view_id}?depth=` + `/view/{view_id}/navigation`. Cloud
  `0.9.64` (git tag, same source the OSS cloud image is built from) does **not** serve either route —
  confirmed by reading `src/api/workspace.rs` in this repo: the only `depth` param that exists is on
  the unrelated `get_page_view_handler` (~line 2481), not a per-view `/view/{id}?depth=` endpoint, and
  there is no `navigation` route at all. Every view request 404s. Building web from `main` (or any
  descendant of `c8f44099`) against cloud `0.9.64` is **unconditionally broken**, not just
  version-skewed in a soft sense — earlier entries in this doc treated this as generic "version
  mismatch"; this is the specific commit and API surface responsible.
- **Why it happened:** testing a custom web rebuild to deploy the delete-guardrail (see entry below). Building from `main` is the default (`git clone …`), not from a release tag. No tagged releases exist for web `0.9.93` on GitHub (the Docker image exists; the source tag does not).
- **Old fix / rollback (still valid):** reconnect the web service to `appflowyinc/appflowy_web:0.9.93` (the stock image, not a custom build) and redeploy.
- **New durable fix (2026-09-04):** build custom web images from a **compat branch pinned before `c8f44099`** instead of `main`. `nextc/AppFlowy-Web` branch `compat-cloud-0.9.64` is based at
  `d0704de2` ("chore: support resend email (#256)", 2026-02-11 — the direct first parent of
  `c8f44099`), so it has everything up to but not including the folder-API break. This unblocks custom
  web rebuilds again (delete-guardrail cherry-pick, new features) while staying deployable against
  cloud `0.9.64`. Full deployment details in the entry below and in
  [08](08-history.md).
- **Lesson:** never build a custom web image from `main` when the deployed cloud is several releases behind — check what commit actually broke compatibility (don't just assume "too new") and branch from immediately before it. The version-pairing rule (by release date, not number) applies to rebuilds too.

---

---

## Board drag lands a card in a phantom "No Status" column (padded select-option) — FIXED (2026-09-28)

- **Symptom:** on the "CoreDating - Mingle2" board, dragging a card from column "M2A-P49" to "In
  Progress" made it jump into a "No Status" column that had never been visible before.
- **Root cause:** the Status option on that board had both **id and name** `"In Progress "`
  (trailing space — legacy artifact from the Kan.bn migration). The web build deployed 2026-09-04
  (`compat-cloud-0.9.64`) groups cards in `src/application/database-yjs/group.ts` (~lines 190–194):
  the cell value is split on `,`, `.trim()`-ed, then matched against the option's **id or name**.
  Trimmed `"In Progress"` matched neither, so the card landed in the No-Status group (group id ==
  field id) — that column only renders once it has cards, which is why it "appeared from nowhere".
  The `.trim()` was added upstream in commit `1042e51cc` (2025-12-11), **after** the previously
  deployed `appflowyinc/appflowy_web:0.9.93` image (Aug 2025) — so this is a regression introduced by
  the 2026-09-04 web swap (see "Custom web image deployed to prod" above), not new bad data.
- **Scope:** scanned all 19 AppFlowy boards for padded select-option ids — only **CoreDating -
  Mingle2** (database `832b8a9a-6d98-4287-81fd-e02b7c8849aa`, field `PHAgs5`) and **CoreDating -
  JSH** (database `8cab7d38-5b68-4905-9a3d-4d65917710a9`, field `MqkY5U`) had them.
- **Fix (data only, no row writes):** trimmed the option **name** to `"In Progress"` while keeping
  the padded **id** `"In Progress "` (which is also the board's group id) — the trimmed lookup now
  matches by name and still resolves back to the existing padded id/group, so no row writes were
  needed. Script: `mgvlabs_kb/appflowy-migration/fix_padded_option.mjs` (database-collab
  `web-update`, `--dry` run first, executed with `keepalive.mjs` attached to hold the workspace actor
  open — see the cold-actor snapshot bug above). Collab read-back confirmed the trimmed name on both
  boards.
- **Verified:** Mingle2 database collab `af_collab.updated_at = 2026-09-28 08:28:56 UTC` (postdates
  both writes below; blob length 0 because the collab is S3-backed — see the storage-fact entry
  below, this is expected, not data loss). JSH database collab: 5338-byte inline blob containing the
  trimmed name, `updated_at 08:20:17 UTC`.
- **Lesson:** never let select-option ids or names carry leading/trailing whitespace — a newer web
  client can start `.trim()`-ing cell values at any point and silently orphan the row into "No
  Status". Same root defect as "Select-option names with leading/trailing whitespace can never be set
  via REST" above, different symptom (there: cell writer trims on write; here: web client trims on
  read/group).

---

## New card auto-assigned a member on Enter — active view filter leaking into new rows — FIXED (2026-09-28)

- **Symptom:** creating a card and pressing Enter auto-assigned the member "Tung Tran" — with no UI
  action taken to select them.
- **Root cause:** the Mingle2 board view (`19ee2ad7-8dbf-4a0b-b9f7-a0dee1956754`) had a saved filter
  (id `HW68kN`) on the Members field (`kk6znA`), condition 2, content `"Tung Tran"`. Upstream web's
  `useNewRowDispatch` copies every active filter's value into a newly created row's cells so the row
  stays visible on the filtered view. Filters are stored in the **database collab**, which is shared
  by everyone — so the whole team was also only ever seeing Tung Tran's cards on that view. It was
  the only saved filter across all 19 boards; who added it is *(unverified — needs check)*.
- **Fix (data):** removed the filter via `mgvlabs_kb/appflowy-migration/fix_padded_option.mjs
  --remove-filter`; collab read-back shows the view's filter list is now empty.
- **Fix (code):** `nextc/AppFlowy-Web` branch `compat-cloud-0.9.64` commit `8d1819ca` — "fix: stop
  pre-filling new rows from active view filters". Both copies of `useNewRowDispatch`
  (`src/application/database-yjs/dispatch.ts` and `dispatch/row.ts`) no longer write filter values
  into a new row's cells; the `shouldOpenRowModal` behavior (calendar date-field filter,
  CreatedTime/LastEditedTime filters) is unchanged. **Tradeoff:** on a filtered view, a newly created
  row that doesn't match the filter is now hidden from that view until edited to match it. `tsc`: no
  errors in the touched files; `eslint`: 0 errors.
- **Deploy:** pushed to `nextc/AppFlowy-Web`; GitHub Actions `build-oss-web.yml` run `36397783913`
  succeeded (2m26s) → `ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64` now at digest
  `sha256:43782e958d901f66ea4860f79bd3518edd0db73879411f200cd6143ae6f8fa59`. Railway `AppFlowy-Web`
  redeployed as deployment `8d045693` (SUCCESS, 2026-09-28 15:34 +07:00). Live bundle
  `/static2/js/index-dwoNjbo_.js` verified to contain the new no-prefill code. Full deploy mechanics
  in [02](02-railway-deployment.md).
- **Rollback:**
  `ghcr.io/nextc/appflowy_web@sha256:30d363df087b545845157b06a2fb910d878556bf24dfccf21dcd9cdd43f3bcc4`
  (previous build, commit `9b5c4d20`, Railway deployment `54624144`).
- **Verification pending:** rendered-board confirmation by the tester (hard reload) — not yet done as
  of this writing.

---

## Board first-load scroll jank — FIXED (2026-09-29)

- **Symptom (tester):** on first load of a board, horizontal scrolling stutters/lags while cards are
  still loading; smooth once all cards are loaded.
- **First fix attempt — commit `cf3817ff`, deployed but did NOT fix the symptom (retained as a minor,
  non-harmful change):** code-reading-only diagnosis found AppFlowy-Web
  `src/components/database/Database.tsx` called `setRowMap({...prev,[rowId]:doc})` once per resolved
  row doc (`populateRowFromCache`/`ensureRow`), producing ~N React updates for a board's parallel row
  loads. `cf3817ff` — "perf: batch row-doc loads into rowMap to stop board scroll jank on first
  load" — queues row docs (`enqueueRowDoc`) and flushes them into `rowMap` once per 100 ms
  (`ROW_MAP_FLUSH_MS`); `rowMapRef` still updates immediately for dedup; user-created rows
  (`createNewRow`) still update immediately. `tsc`/`eslint` clean; code review found no CRITICAL/HIGH
  (one accepted MEDIUM: `useRow` reads from `rowMap` state so a card's cells can populate up to 100 ms
  after its title/meta). Deployed as digest
  `sha256:35391c89d62e8c7812ddf4bc8d60879ce1b2615858e11a88083e664b601989fd`, Railway `AppFlowy-Web`
  deployment `af70e0c4` (SUCCESS, 2026-09-29 04:42 +07:00). **This was not profiled before deploying —
  the tester then reported the board still laggy on first load** ("seems like cards being rendered"):
  the fix targeted a real but minor inefficiency, not the dominant cost.
- **Correct root cause — found by profiling the live site, not by code reading:** measured on prod in
  Chrome with the Long Animation Frame API (`PerformanceObserver` type `long-animation-frame`,
  buffered) during a cold load of CoreDating - Mingle2. Method: deleted all 138 IndexedDB databases
  for b.mgvlabs.com (the app keeps one `y-indexeddb` database per row, named
  `<dbId>_rows_<rowId>`), then reloaded and scrolled horizontally (the login token lives in
  `localStorage` and survives the IndexedDB wipe). Result: 98 long frames totaling 11,991 ms; 11,418 ms
  of that was attributed to 272 `DOMWebSocket.onmessage` handler invocations (~135–147 ms each, spread
  over ~8 s); React scheduler work itself was only ~503 ms.
  - `react-use-websocket` 4.13.0 applies every message via `flushSync(() => setLastMessage(message))`
    (`node_modules/react-use-websocket/dist/lib/use-websocket.js:143`), so the resulting render runs
    **synchronously inside `onmessage`**.
  - `useAppflowyWebSocket` returns a new object per message; `AppSyncLayer` included that `webSocket`
    (and `broadcastChannel`) in the `SyncInternalContext` memo deps, and `useSync` set a new
    `lastUpdatedCollab` on every collab message.
  - Net effect: `SyncInternalContext`'s value changed per message → `useAllContextData`'s memo
    (depended on the whole `syncData`) rebuilt the app-wide `AppContext` → every `useAppContext`
    consumer (sidebar, board, every card) re-rendered synchronously on every WebSocket message.
- **Actual fix — commit `e6390209`:** `nextc/AppFlowy-Web` branch `compat-cloud-0.9.64` commit
  `e6390209` — "perf: stop re-rendering the whole app on every WebSocket message" (4 files):
  - Removed the unused `webSocket`/`broadcastChannel` fields from `SyncInternalContextType` and its
    value (no consumers relied on them).
  - `useSync` now sets `lastUpdatedCollab` only for `Types.Folder` messages (the only reader,
    `AppBusinessLayer.tsx` ~220, was already gated on `Folder`).
  - `useAllContextData`'s memo now depends on `syncData?.eventEmitter` only, not the whole `syncData`
    object.
  - Code review (sonnet `code-reviewer`): no CRITICAL/HIGH; confirmed `SyncInternalContext`'s value
    stays referentially stable across non-Folder messages (`registerSyncContext`/`flushAllSync`/
    `syncAllToServer`/`awarenessMap` are all stable references). `tsc` no non-test errors; `eslint` 0
    errors.
- **Deploy:** GitHub Actions `build-oss-web.yml` run `36490175194` succeeded →
  `ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64` digest
  `sha256:15e6ccc8b531bea5b9894885b7de30532716412b40014b5a8856aa8d4485aa58`. Railway `AppFlowy-Web`
  deployment `7958bda5` (SUCCESS, 2026-09-29 05:06 +07:00). Live entry bundle `index-DYksSXv-.js`.
  Full deploy mechanics in [02](02-railway-deployment.md).
- **Verification (same profiling method, live prod after this deploy):** 5 long frames, 401 ms total,
  longest 123 ms; the WebSocket handler no longer appears in the attribution; the remaining ~375 ms is
  React scheduler work during the initial board render.

  | Metric | Before (cf3817ff live) | After (e6390209 live) |
  |---|---|---|
  | Long frames | 98 | 5 |
  | Total long-frame time | 11,991 ms | 401 ms |
  | Time in WS `onmessage` handler | 11,418 ms | 0 ms |
  | Longest single frame | 147 ms | 123 ms |

- **Rollback:**
  `ghcr.io/nextc/appflowy_web@sha256:35391c89d62e8c7812ddf4bc8d60879ce1b2615858e11a88083e664b601989fd`
  (commit `cf3817ff`, deployment `af70e0c4`) — now **removed/superseded**.
- **Lesson:** profile before fixing perf. The first fix (`cf3817ff`) was chosen from code reading
  alone and targeted roughly 0.5 s of the ~12 s of jank; the Long Animation Frame API on the live site
  identified the real cost (synchronous re-renders triggered by every WebSocket message) in minutes —
  the prod bundle kept readable names, which made attribution straightforward.
- **Side note:** an "Untitled" card assigned to Tung Tran exists in Mingle2's Released column — likely
  the tester's card from the auto-assign bug *(unverified who created it)*.

---

## CRITICAL: card edits (title/description) not saved, lost on hard reload — FIXED (2026-09-29)

- **Symptom (tester):** updating a card's description/title no longer saved; a hard reload (Cmd+R)
  made the edit disappear.
- **Live diagnosis (prod, web deployment `7958bda5`, bundle `index-DYksSXv-.js`):** on card "test"
  (row `44941408-03d1-46c3-96f9-1dc4a08cad3e`, CoreDating - Mingle2), editing the title changed it
  locally, but no WebSocket update for the row collab (type 4) was ever sent — the server's collab
  read still returned `'test'`. Walking the page's React database context: all 135/135 row docs in
  `rowMap` had `guid` still in `rowKey` form `"<dbId>_rows_<rowId>"` with a single `'update'` listener
  (`y-indexeddb` only) — i.e. never registered for sync (the app's `createRow` in
  `useViewOperations.ts` sets `doc.guid = rowId` and calls `registerSyncContext`). Calling the
  context's `createRow(rowKey)` directly returned a **different** Y.Doc instance (registered,
  `guid = rowId`) than the one already in `rowMap`.
- **Root cause:** a race in AppFlowy-Web `src/application/services/js-services/cache/index.ts`
  `getOrCreateRowDocEntry` — it checked the `rowDocs` map, `await`ed `openCollabDBWithProvider`
  (IndexedDB open), then set the entry. A mounting card's `bindRowSync → registerRowSync → createRow`
  and the board's `populateRowFromCache → createRowFast` request the same row concurrently, before the
  awaited open resolves → two Y.Docs created for one row; sync got registered on one, the board/modal
  edited the other → edits persisted only to local IndexedDB, never reached the server. The race is in
  upstream code; the two same-day perf deploys (`cf3817ff`, `e6390209`) most likely changed load
  timing so it hit every card *(causal timing link unverified)*.
- **Fix:** AppFlowy-Web `compat-cloud-0.9.64` commit `5fad2172` — "fix: share in-flight row doc opens
  so card edits sync to the server" — adds a `pendingRowDocEntries` map so concurrent callers for the
  same row share the in-flight open instead of racing to create separate docs (ORDER-tagged comment in
  the diff). `tsc` clean for non-test code, `eslint` clean. The existing `cache.test.ts` does not cover
  row docs — no tests changed or added.
- **Deploy:** GitHub Actions `build-oss-web.yml` run `36496870314` succeeded. Railway `AppFlowy-Web`
  deployment `51d67a26` (SUCCESS, 2026-09-29 06:16 +07:00). Live bundle `index-BtS1lMX7.js`. Image
  digest `sha256:d867498ee094a9d0e7df2d79b6501986bff62ee64358141fe550954bbe275b37` (read from the GHCR
  registry manifest). Full deploy mechanics in [02](02-railway-deployment.md).
- **Post-deploy verification (prod):** 135/135 row docs now registered for sync (`guid = rowId`, 2
  `update` listeners). The edit typed during the bug window (`'test123'`, held only in local
  IndexedDB) synced to the server automatically on reload; a fresh edit (`'test123' → 'test'`) reached
  the server and survived a hard reload.
- **Recovery note:** edits made while the bug was live are sitting in that browser's IndexedDB (both
  duplicate docs persisted to the same per-row IndexedDB database) and self-recover by reopening the
  board in the same browser on the fixed build — unless the browser's site data was cleared in
  between.
- **Detection recipe:** in the page, walk the React fiber from a board/modal element up to the context
  value carrying `rowMap`; any row doc whose `guid` is not the bare `rowId` is not registered for
  sync.
- **Collateral (own mistake during diagnosis, reverted):** an accidental click on "+ New property" in
  the card modal created an empty "Text" field (id `HrsnJo`, 2026-09-28 22:55:44 UTC) on Mingle2;
  removed via a database-collab web-update (fields map + both views' `field_orders`/`field_settings`);
  server now shows the original 7 fields.
- **Lesson:** after any change that alters load timing, verify the write path (edit → server
  read-back → hard reload), not just rendering/perf — the compat build's open write-path caveat
  (entry above, "Custom web image deployed to prod") had flagged this exact gap as untested back on
  2026-09-04.

---

## `af_collab.blob` is 0 bytes for large collabs — expected (S3-backed), not data loss

- **What it looks like:** querying `af_collab` for a large database/document collab shows `blob`
  length `0`, which reads like the write never landed.
- **Actually correct behavior:** `services/appflowy-collaborate/src/config.rs:209`
  `APPFLOWY_COLLAB_S3_THRESHOLD` defaults to `8000` bytes — any `encoded_collab_v1` larger than that
  is stored in **S3** (key `collabs/{workspace_id}/{object_id}/`, `disk_cache.rs` ~lines 121, 723)
  instead of inline in Postgres, and `af_collab.blob` is left empty (length 0) **by design**.
- **How to check persistence instead:** use `af_collab.updated_at` postdating your write as the
  persistence signal, not blob length/presence. (Postgres database name is `railway` — e.g.
  `railway ssh --service pgvector "psql -U postgres -d railway ..."`.)
- **Example (2026-09-28, the fix above):** Mingle2 database collab — blob length 0,
  `updated_at 2026-09-28 08:28:56 UTC` (after the write) — correctly S3-backed, not data loss. JSH
  database collab — 5338-byte inline blob (under the 8000-byte threshold) containing the trimmed
  name, `updated_at 08:20:17 UTC`.

---

## Owner-only move guardrail — Members can no longer move pages between spaces (2026-09-30)

- **Motivation:** extends the 2026-07-25 delete guardrail (above) — the workspace Owner asked that
  only Owners be able to **move** pages between spaces/parents, not just delete. Before this change
  any Member could move any page: verified 2026-09-30 by calling `POST
  /api/workspace/{ws}/page-view/{view}/move` as a test Member (`thanh+1@mingle.com`) → `200 {code:0}`.
- **Current Owners:** `thanh@mingle.com` and `bao.pham@mingle.com` (verified from
  `af_workspace_member`/`af_roles` 2026-09-30). Every other member, including `thanh+1@mingle.com`, is
  `Member`.
- **Server:** AppFlowy-Cloud commit `44219d40` — "feat: gate move-page endpoint on Owner role" — adds
  `enforce_role_strong(&uid, &workspace_uuid, AFRole::Owner)` to `move_page_handler`
  (`src/api/workspace.rs`), mirroring the existing delete-guardrail pattern. Tag
  `0.9.64-owner-move` (= `44219d40`); the only code diff vs tag `0.9.64-delete-guardrail` is this
  5-line change (everything else in the diff is `nextc-docs`).
- **Web:** `nextc/AppFlowy-Web` branch `compat-cloud-0.9.64` commit `bbd5ea7c` — "feat: restrict
  moving pages to workspace Owners":
  - New `useCanMoveContent()` hook (Owner-only) in `AuthInternalContext.ts`.
  - `usePageOperations.movePage` refuses non-Owners client-side and shows a toast: *"Only workspace
    owners can move pages."*
  - "Move to" is hidden from `MoreActionsContent` (the page header "…" menu) for non-Owners.
  - The `ViewModal` space picker renders as a plain, non-interactive label for non-Owners instead of a
    dropdown.
  - `tsc` clean (non-test), `eslint` 0 errors.
- **UX fact:** "Move to" only exists in the page header "…" menu and the page view modal on this
  build — the sidebar "…" menu has no "Move to" entry, and this web build has no sidebar
  drag-and-drop for pages.
- **Build/deploy (Cloud):** GitHub Actions `build-oss.yml` on `nextc/AppFlowy-Cloud`, run
  `36684420913` (success; `appflowy-cloud` 15m6s, `appflowy-worker` 15m29s, `admin-frontend` 4m46s) →
  `ghcr.io/nextc/appflowy-cloud:0.9.64-owner-move` digest
  `sha256:ee1519394a158546e34f8c3c2ac0132cbc22167a1ca4e3844005591e9150da89`. Railway
  `AppFlowy-Cloud` switched via `railway service source connect --image
  ghcr.io/nextc/appflowy-cloud:0.9.64-owner-move --service AppFlowy-Cloud` + `railway redeploy`;
  deployment `f8dd443b` SUCCESS (2026-09-30 14:50 +07:00). Only the `appflowy-cloud` service image
  was switched — the `worker`/`admin-frontend` images from the same run were built but not deployed.
- **Deploy (Web):** two deployments the same day — `516e8320` (14:12 +07:00, commit `458f8e4a`, the
  checklist-blur fix below, bundle `index-DLqcuVjZ.js`) then `3281e4a7` (14:51 +07:00, commit
  `bbd5ea7c`, bundle `index-DSRaTnPM.js`). Web image digest for `bbd5ea7c` *(unverified — not
  recorded)*. Full deploy mechanics in [02](02-railway-deployment.md).
- **Verification (live, 2026-09-30):** Member `thanh+1` move → HTTP 200 body `{code:1012, message:
  "User does not have permissions to execute this action"}`, page unmoved. Owner move to QC and back
  → `code:0`; test page "Note for bundle/package ID" (view `92415f2c…`) restored to its original
  position in Tips Sharing. Live web entry bundle `index-DSRaTnPM.js` contains the string "Only
  workspace owners can move pages." **Not checked:** the Member-side UI hiding was not visually
  confirmed (no Member browser session available) — tag as *(unverified — visually)*.
- **Related report investigated, not a bug:** a tester reported "can't move board," which turned out
  to mean moving a **page** between sidebar spaces (not cards). Owner moves worked in testing; under
  the new spec, Member moves are blocked **by design**. `bao.pham` is an Owner — if he still can't
  move pages, Cloud logs around the same window showed repeated `ExpiredSignature` WS auth failures
  from some client every 2–4 minutes (whose session is *(unverified)*) — the actionable next step is
  to ask him to re-login.
- **Rollbacks:**
  - Cloud → `ghcr.io/nextc/appflowy-cloud:0.9.64-delete-guardrail`
    `sha256:b4373228b6da40e03143c0e6e6b3d6869e42a26c6ba61ffd0baf4e943769d9aa` (previous deployment
    `7b3d4b0a`) — Members can move pages again.
  - Web → `ghcr.io/nextc/appflowy_web@sha256:3bc96f915acf944a874462bbd0ffdc7b933c95f27220e3938659df5b8cd61a38`
    (commit `458f8e4a`, deployment `516e8320`). **Caveat:** this rollback target lacks the Owner-move
    UI gating (no toast, "Move to" visible to everyone) — the server (if not also rolled back) would
    still refuse the write, so a Member would see the move silently fail.

## Checklist sub-task with a typed description silently discarded on blur — FIXED (2026-09-30)

- **Symptom (tester):** "can't create a sub-task with a description" — typing a description into the
  new-sub-task input then clicking elsewhere made the typed text vanish; nothing was created.
- **Root cause:** `NewTask.tsx`'s `onBlur` only called `setShowCreateInput(false)` (hid the input) —
  it never submitted the typed text. Pressing **Enter** or clicking the **+** button did work
  (verified live: created "sub1" via Enter and "sub2" via the + button on test card
  `44941408…`, both persisted server-side) — only the blur path silently discarded input.
- **Fix:** web commit `458f8e4a` — "fix: create checklist task on blur instead of discarding typed
  text" — `onBlur` now calls `onCreateTask` when the input has text. **Escape still discards** the
  input (unchanged, intentional).
- **Deploy:** Railway `AppFlowy-Web` deployment `516e8320` (2026-09-30 14:12 +07:00, GitHub Actions
  run `36682159134`, bundle `index-DLqcuVjZ.js`). Superseded same day by deployment `3281e4a7` (the
  Owner-move deploy above) — both fixes are live in the current build.
- **Verification:** the create-on-Enter and create-on-+ paths were verified live pre-fix; the
  onBlur-submits-text behavior itself was **not** re-tested live after deploy *(unverified — needs a
  live check: type a description, click away, confirm the sub-task is created)*.

## Data fix: duplicate card in Mingle2 board view row_orders (2026-09-30)

- **Symptom:** on the Mingle2 board, card "Messages — Add an Photo library permission popup…" (row
  `e1673d42`) rendered **twice**.
- **Root cause:** the board view's `row_orders` (view `19ee2ad7…`) listed `e1673d42` at both index 4
  and its normal position — 137 entries vs 136 in the paired grid view (`e0ac14cf`). Likely cause:
  concurrent moves of the same card (a Yjs array move is delete+insert, and two concurrent inserts can
  both survive) *(likely, unverified)*.
- **Fix:** removed the duplicate (index-4) entry from `row_orders` via a database-collab
  `web-update`. Both views now report 136 rows with no duplicates.

## Open issue: brief post-load window where cards read "Untitled" and silently drop edits

- **Observed:** after a warm load, cards render "Untitled" with empty statuses for roughly 2–3
  seconds (up to ~14 s on a cold load) before row content finishes hydrating. During that window,
  card moves and cell edits made against those still-hydrating rows are **silently dropped**:
  `useUpdateCellDispatch` returns early with a `Log.warn` when the row isn't yet in `rowMap`, and
  `useMoveCardDispatch` throws "Unable to reorder card."
- **Distinct from the fixed "Untitled for minutes" bug** (see "Boards show 'Untitled' cards for
  minutes on first visit" above, fixed 2026-09-04) — that entry was full-board hydration taking
  minutes on a cold IndexedDB; this is a much shorter per-load window that is still present after that
  fix and is **not currently fixed**.

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
