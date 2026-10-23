# 08 — History (decisions & pivots)

Chronological log reconstructed from all project conversations. Dates are when the work happened
(2026-07). This is the "why we ended up here" narrative; the "how it works now" is in docs 01–07.

## Phase 0 — Third-party Railway template, first contact (~2026-07-03)

- Started on a **third-party Railway template** of AppFlowy-Cloud. Couldn't access the "gotrue site" /
  log in; wanted CLI-driven debugging.
- Wanted a working **admin console** + login → discovered `admin_frontend` wasn't deployed; `/console`
  404. Deployed the 2026 Next.js "Super Admin" — then found it needs commercial-only endpoints OSS
  lacks.
- Fixed `/console` login loop, MinIO console subpath, and admin identity issues:
  `ops@mgvlabs.com` role must be `supabase_admin` (not `admin`).
- Set up the custom domain **`b.mgvlabs.com`** — hit the **port 8080** issue (nginx must `listen 8080`)
  and the `*.mgvlabs.com` wildcard conflict from the `mgvlabs_web` project.
- Consolidated the public domain into a single `Nginx.PUBLIC_HOST` reference var.

## Phase 1 — The seat cap → decision to go OSS (~2026-07-04/05)

- Hit the wall: **"Seat limit reached … Upgrade your license"** — couldn't add a 2nd member.
- Requirement crystallized: **13 team members without paying**.
- Verified the cap is **commercial-only** (not in AGPL source) → **decision: self-build the AGPL OSS
  cloud** ("option C"), still hosted on Railway "for simplicity of devops".
- Hard constraint from the user: **do NOT grant Railway broad GitHub access** — only
  `nextc/AppFlowy-Cloud`; Railway's UI couldn't do a single-repo grant → "shady". **Resolution:** build
  OSS images via **GitHub Actions → public GHCR**, Railway pulls them as plain public images. (User's
  arm64 Mac can't build amd64 without slow emulation → GitHub amd64 runners.)
- Validated OSS build (tag `0.9.64`) on a throwaway project: **15 invites, 0 seat errors.**
- Added requirements: **`@mingle.com`-only registration** + **email verification**.
- Discovered **Railway blocks SMTP** → built the **`email-hook`** service (GoTrue `send_email` hook →
  Brevo HTTP API). Enforced the domain whitelist with a Postgres trigger.

## Phase 2 — Convert prod in place (2026-07-06)

- Chose to **fix the existing `appflowy` project in place** (its domain/email-hook/nginx/trigger were
  proven-good; only image versions were skewed): repoint image tags to the consistent **`0.9.64`** set,
  **wipe the DB** (no real data), redeploy.
- Established the **version-pairing rule** (by release date, not number): cloud/worker/admin OSS
  `0.9.64` + gotrue `0.9.64` + web `0.1.19`.
- Deployed the consistent stack. First acceptance test **still failed** (blank sidebar, "no record
  found").

## Phase 3 — The real root cause (2026-07-06)

- Spent time on a **collab version-skew theory** — the user pushed back twice: *"if this docker in
  local of this github works, we should deploy it to railway instead of trying to solve the broken
  template."*
- Read the cloud logs → **RecordNotFound**: the `af_user` was never created. Root cause: **GoTrue
  lacked `GOTRUE_URI_ALLOW_LIST`**, so `/auth/callback` (the only provisioning trigger) was bounced →
  no provisioning. Fixed with the allow-list + the email-hook `redirect_to` force. Verified end-to-end.
- **Lesson recorded:** version-skew was a red herring; check provisioning first.
- User confirmed: **"worked now."**

## Phase 4 — Team onboarding at scale (2026-07-07)

- Teammate `bao.pham@mingle.com` hit the same blank-sidebar problem → the browser `/auth/callback`
  provisioning is unreliable because **email link-scanners burn the magic-link token**.
- Built the **`provisioner`** service — server-side sweep that provisions confirmed `@mingle.com`
  users by minting a gotrue-valid JWT and calling `/api/user/verify`.
- "Can't invite my teammate to workspace" + shared-doc access → **SMTP-blocked invites**. Extended the
  provisioner to **auto-accept pending invites**. Learned the strict `GoTrueJWTClaims` requirement
  (all five claims).
- "Member limit reached, please upgrade" → **client-side billing gate**. Spoofed **both** billing
  endpoints in nginx (team + ai_max). "still disabled" → needed the second endpoint too.
- User asked for one fix for all subscription-blocked features → confirmed the two billing endpoints
  are the only gates; this is the single unblock.
- "hide those upgrade buttons" → nginx `sub_filter` MutationObserver script.
- "where can I manage members?" → web v0.1.19 has **no member UI** → built the **`admin-ui`** service
  (password-protected). User chose **"Auto-join everyone (fully automatic)"** for onboarding →
  provisioner `ensureSharedInvites()` + shared workspace. Resolved auto-join-vs-remove with
  `af_admin_excluded`.
- User: "change the admin-ui password" → done + verified (value lives only in Railway; rotated 2026-10-23 after the Railway security incident).

## Phase 5 — First-login polish + on-demand provisioning (2026-07-07)

- `thanh+1@mingle.com`: blank sidebar on **first** login, **fixed by refresh** → the async-provisioning
  race (app calls `/api/workspace` before the poll provisions → `200 {data:[]}`).
- Fix: **shorten poll 30s→5s** + inject a **client auto-reload** (`af_ws_retry`, hooks fetch + XHR,
  bounded, fails safe).
- User: "can Provision on demand … when account is created?" → added **Postgres LISTEN/NOTIFY** — a
  trigger on `auth.users` NOTIFYs the provisioner, which provisions within ~1s. Chosen over a GoTrue
  auth-hook because NOTIFY can't slow/break login. Poll + client reload retained as fallbacks.
- User: "document everything … into a folder named nextc-docs" → **this documentation set.**

## Phase 6 — Realtime sync diagnosis & collab fixes (2026-07-09/10)

- Database grid **rows missing on other clients** → diagnosed as orphaned DatabaseRow collabs. The
  Database collab's `row_orders` was not being updated when rows were added; row data existed but was
  unreachable. Built a surgical Yjs repair to re-register rows in `row_orders` without touching
  fields/metas/filters.
- **Web build `0.1.19` has zero websocket code** — it only syncs via HTTP `web-update`. No realtime.
  Traced to version pairing: web `0.1.19` predates websocket support (added in `0.9.93`, 2025-08-18).
  Upgraded web image to `0.9.93` + required env vars (`AF_BASE_URL`, `AF_GOTRUE_URL`, `AF_WS_V2_URL`).
- Web upgrade revealed a **cloud version-skew bug**: `0.9.93` sends TEXT heartbeat frames, but cloud
  `0.9.64` closes on ANY text frame with 1003. Upstream fixed in commit `a948c8b3` (#1542). Cherry-picked
  the fix onto `0.9.64`, built + deployed `0.9.64-ws-text-echo` image.
- **Identified a latent upstream bug** in workspace actor snapshots: cold HTTP writes (no ws session
  attached) are never persisted because snapshot only reschedules on success, and `up_to=0` (no messages
  yet) triggers a silent no-op. The websocket upgrade (keeping the actor alive) also fixes the snapshot
  logic in practice.
- Updated the known-issues catalog with all three bug diagnoses + the latent finding.

## Phase 7 — Kan.bn kanban → AppFlowy data migration (2026-07-18)

- Migrated **all 17 live Kan.bn boards** (kb.mgvlabs.com) into the shared workspace as Grid pages
  under a new **Kanban Boards** space, each formatted like the existing "Build List" grid (Name
  primary / Status single-select from the board's lists / Done / Card Link URL / Attachments media /
  Labels + Members + Due where used; card description + checklists + comments as row documents).
- Fully API-driven with the owner's self-minted JWT: `POST page-view` (Grid), one Yjs `web-update`
  per board (drop the 3 seeded rows, default "Type" field → "Status" with select options), then REST
  `fields`/`row` inserts. `/ws/v2` keepalive held open throughout — required by the cold-actor
  snapshot bug (see [06](06-known-issues-and-fixes.md)).
- Attachments: single-PUT blob upload dies at the edge for ≥ ~10–20 MB bodies (broken pipe) → used
  the multipart upload API (8 MB parts) in an idempotent reconcile pass that compares source files
  vs row Media cells and patches DatabaseRow collabs (`collab_type=4`, root `data`→`data`→`cells`).
- Verified end-state: **517/517 cards, 230/230 attachments** across all boards, byte-exact blob
  round-trips, counts stable after full actor cool-down. Tooling + per-board view/db id map archived
  at `mgvlabs_kb/appflowy-migration/` (outside this repo).
- Operational notes: Kan.bn API returns 401s on request bursts (~90 s lockout; pace ~1 req/s);
  Railway CLI project link is directory-scoped (user `!` commands need `cd` into this repo first).
- **2026-07-20 follow-up:** all 17 pages converted Grid → **Board (kanban) layout** via Yjs
  `web-update` on both collabs — DB view `layout` 0→1 + `layout_settings["1"]` + a `groups`
  GroupSetting on the Status field, folder view `layout` 1→2 (the two layout enums differ:
  DatabaseLayout Board=1 vs folder ViewLayout Board=2). Verified cold-persistent. The 3 boards
  visible in the Kan.bn workspace listing but 404 on fetch (Core Project, Startup Project, Test —
  trashed upstream) remain unmigratable via API.
- **Board-render bug found by looking at the actual screen** (same day): the first flip wrote group
  columns as `Y.Map`s, which the deployed web build can't read → all groups fell into "Hidden Groups"
  and boards rendered blank. Diffing against a natively-created board showed entries must be **plain
  objects**; fixed across all 17 boards (`fix_groups.mjs`) and verified visually + cold. Full write-up
  in [06](06-known-issues-and-fixes.md).
- **Final acceptance:** every board's *rendered* kanban columns were extracted from the live DOM and
  reconciled against the source board — **17/17 boards, 517/517 cards in the correct columns**. That
  pass caught one further defect: a card whose Kan.bn list was named `"In Progress "` (trailing space)
  had an empty Status, because the REST cell writer trims input before matching option names. Repaired
  by writing the option id into the row collab (`set_select_cell.mjs`); see [06](06-known-issues-and-fixes.md).
- **Migration is ongoing, not one-off** — the team still works in the old Kanban. Tracked in
  `mgvlabs_kb/appflowy-migration/MIGRATION-LOG.md` (per-board card/attachment counts, view/db ids,
  round-2 recipe), with `check_new_cards.py` giving a read-only drift report (new / moved / gone
  cards, new boards) that treats live AppFlowy as truth via each row's Kan.bn `Card Link` id.
  Measured 2 days after round 1: **5 new + 15 moved cards**.
- **Round 2 (2026-07-20, "new cards only")** — `add_new_cards.py --apply` added **+6 cards / +9
  attachments** (Mingle2 1, QuestDay 2, TripFold 3) onto the existing databases without touching
  existing rows, so AppFlowy-side Status edits survive. End state **523 cards / 239 attachments**;
  `check_new_cards.py` reports `new=0, gone=0` on all 17 boards (card *sets* match Kan.bn exactly).
  The 15 moved cards were deliberately left at their AppFlowy Status per the chosen policy.
  Recurring trap confirmed again: **REST `row/detail` serves stale rows for minutes after a collab
  write** (and the browser caches too) — verify against the rendered board or a raw collab read.

## Phase 8 — Role-based access control: delete guardrail (2026-07-25/28)

- User requirement: **Owner-only delete** — Members can edit but not delete documents or kanban lists.
- Constraint: AppFlowy has only 3 roles (Owner/Member/Guest), no custom tier without Rust enum + migration + casbin changes.
- Architecture: CRDT deletes (rows/lists) are client-only; REST deletes (pages/collabs) can be server-gated. This is a guardrail (UI + REST), not tamper-proof.
- Implementation: Cloud `4 endpoints` → `enforce_role_strong(AFRole::Owner)`, web delete items FULLY HIDDEN in 9 sites (RowMenu, CardToolbar, DatabaseRowModal, OptionMenu, ViewActions, ColumnMenu, PropertyMenu, MoreActionsContent, EventPopoverContent), Flutter delete items fully hidden + board card delete keyboard shortcut gated in `board_shortcut_container.dart`. Verified builds (`cargo check` Docker / `tsc` clean; Flutter only `dart format`, no `flutter analyze`). Guardrail level: bloc-event handlers not role-guarded.

### 2026-07-27/28 — Deployment saga

- **Cloud image:** `ghcr.io/nextc/appflowy-cloud:0.9.64-delete-guardrail` built via GitHub CI (`build-oss.yml` on `nextc/AppFlowy-Cloud`). Railway `AppFlowy-Cloud` service reconnected → redeploy **SUCCESS**. Cloud REST deletes now blocked for non-Owners.
- **Web image attempt (FAILED):** New fork `nextc/AppFlowy-Web` + `build-oss-web.yml`. Built `appflowy_web:0.9.93-delete-guardrail` from `AppFlowy-Web:main` (default branch). Deployed to Railway `AppFlowy-Web` service.
  - **Problem:** web loads (200), connects to cloud (no errors), but sidebar/folder never render. Root cause: **building from `main` is ~0.16+ (several major versions ahead of cloud `0.9.64`)**. Version-skew on collab format / folder serialization breaks deserialization. No code issue; version mismatch.
  - **Lesson:** never build custom web images from `main` when deployed cloud is multiple releases behind. The version-pairing rule applies to rebuilds too. No clean git tag exists for web `0.9.93` (Docker image exists; source tag doesn't). Rebuilding is dangerous.
  - **ROLLBACK:** web service reconnected to `appflowyinc/appflowy_web:0.9.93` (stock image). Redeploy **SUCCESS**. Web functions normally.
- **Final solution (2026-07-28):** Delivered the web-side delete-hider via **nginx `sub_filter` injection** (Patch 4, [04](04-nginx-and-client-patches.md)) instead of a custom web build. Same delete UI hide (role-aware MutationObserver), zero version-skew risk, no custom image needed. Updated nginx config → NGINX_CONFIG_B64 → redeploy. **All green:** web 200, sidebar loads, delete items hidden for non-Owners, server guards active.
- **Final coverage (live 2026-07-28):** 
  - **Server:** cloud `0.9.64-delete-guardrail` enforces `AFRole::Owner` on REST deletes (move-to-trash, delete-page, delete-all, delete-collab).
  - **CRDT deletes (kanban rows/lists/fields):** no server endpoint exists; guarded by nginx-injected UI hider on stock web `0.9.93`.
  - **Workspace `dc8cd0fa`:** Members: `bao.pham@mingle.com`, `thanh+1@mingle.com` (both can edit, not delete). Owner: `thanh@mingle.com` (full perms).
  - **Desktop:** NOT deployed (separate binary distribution; would need custom Flutter build + real `flutter analyze` in full codegen toolchain).
- **Outcome:** Guardrail is live and functional. Members can edit but cannot delete. Server guards prevent REST deletion; nginx guardrail prevents UI-triggered CRDT deletion. See [06](06-known-issues-and-fixes.md) for the full deployment log and rollback steps.

## Phase 9 — Compat web fork + shareable card URLs deploy (2026-09-04)

- **Problem:** `nextc/AppFlowy-Web:main` was undeployable against cloud `0.9.64` — upstream commit
  `c8f44099` "Use new folder api (#257)" (2026-02-18) moved view loading onto
  `GET /view/{id}?depth=` + `/view/{id}/navigation`, neither of which cloud `0.9.64` serves, so every
  view 404s ("No access to this page"). This is the concrete root cause behind the older, more
  generic "sidebar/folder won't load (version-skew)" finding from Phase 8 — see
  [06](06-known-issues-and-fixes.md).
- **Resolution:** branched `compat-cloud-0.9.64` off `d0704de2` (2026-02-11, the immediate parent of
  `c8f44099`) — everything up to but not including the folder-API break. Cherry-picked the GHCR build
  workflow (`05592a79`) and the Owner-only delete gating from `main` (`11914cf4`, cherry-pick of
  `a3a5d0b4`, 6 conflicts hand-resolved; its `main`-only test file `actionPermissionGates.test.tsx`
  references APIs absent on this base and was dropped from the pick).
- **New feature (`ee87105f`):** shareable card URLs — opening a board card writes `?r-modal=<rowId>`
  into the URL (push on open so Back closes it, replace on close so Back doesn't reopen it);
  `Database.tsx`/`DatabaseView.tsx` react to URL-driven changes for back/forward and cold-loaded
  shared links; cross-database relation-row modals are intentionally not URL-synced. Also flipped the
  row-detail hidden-fields default to expanded in code (`DatabaseRowProperties.tsx`
  `useState(isFilterHidden=false)`), which **retires** nginx Patch 5 (its sub_filter is now inert —
  see [04](04-nginx-and-client-patches.md)).
- **Deploy:** built via GitHub Action `build-oss-web.yml` (workflow_dispatch, `ref=compat-cloud-0.9.64`)
  → `ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64` (public GHCR, same pattern as the cloud images).
  Railway `AppFlowy-Web` service reconnected via `railway service source connect --image ...` +
  `railway redeploy --from-source -y` (user-run; classifier-gated). **Rollback:** reconnect
  `appflowyinc/appflowy_web:0.9.93`.
- **Verified:** `tsc` + `eslint` clean; manual in-browser check against prod data via dev proxy —
  card-click URL param, Escape/backdrop close, Back/Forward, cold deep-link load, expand → `?r=`,
  board render.
- **Open caveats carried forward:** write path (editing/moving cards) not yet explicitly verified on
  this build; React 18.2.0/ReactDOM loaded from `cdn.jsdelivr.net` at runtime (build-time CDN
  externals — a jsdelivr outage breaks the app); BoardTemplate's tab label cosmetically shows the
  underlying view's name. Full detail in [06](06-known-issues-and-fixes.md).
- **Same-day follow-up fix:** the cold-IndexedDB empty-cells quirk hit prod at scale — every first
  visit showed boards full of "Untitled" cards for minutes (cloud 0.9.64 lacks this era's
  `blob/diff` bulk-row endpoint, so rows hydrated one-by-one over WS). Fixed in web branch commit
  `9b5c4d20`: the view loader now seeds row docs from the page-view fetch's `row_data` payload it
  had been discarding. Image rebuilt on the same tag, redeployed; cold-cache board load verified in
  seconds. Full detail in [06](06-known-issues-and-fixes.md).

## Phase 10 — Two prod bugs on the compat web build: phantom "No Status" column + filter-leaking new rows (2026-09-28)

- **Bug 1 — board drag lands cards in a phantom "No Status" column:** CoreDating - Mingle2's Status
  option `"In Progress "` (trailing space, legacy Kan.bn artifact) stopped matching once the
  2026-09-04 compat web build's grouping logic started `.trim()`-ing cell values (upstream commit
  `1042e51cc`, 2025-12-11) — cards fell into an unlabeled No-Status group that only renders once it
  has cards. Scanned all 19 boards; only Mingle2 and CoreDating - JSH had padded option ids. Fixed by
  trimming the option **name** only (kept the padded id, which doubles as the group id) via a
  database-collab web-update — no row writes needed. Verified via collab read-back and
  `af_collab.updated_at`.
- **Bug 2 — new cards auto-assigned member "Tung Tran":** the Mingle2 board view had a saved Members
  filter (`content: "Tung Tran"`) that upstream `useNewRowDispatch` was copying into every newly
  created row so the row would stay visible on the filtered view — since filters live in the shared
  database collab, this also meant the whole team's filtered view only ever showed Tung Tran's cards.
  Fixed in two parts: removed the stray filter (data fix), and shipped `nextc/AppFlowy-Web` branch
  `compat-cloud-0.9.64` commit `8d1819ca` so `useNewRowDispatch` no longer pre-fills new rows from
  active filters (tradeoff: a new row that doesn't match a filter is now hidden on that view until
  edited to match it). Rebuilt + redeployed the same `compat-cloud-0.9.64` tag (GitHub Actions run
  `36397783913`, Railway deployment `8d045693`); rollback is the previous digest, commit `9b5c4d20`,
  deployment `54624144`.
- **Storage-layer clarification uncovered along the way:** a `0`-byte `af_collab.blob` for a large
  collab is **expected** — `APPFLOWY_COLLAB_S3_THRESHOLD` (default 8000 bytes) routes bigger collabs
  to S3 instead of Postgres; use `updated_at`, not blob presence, as the persistence signal.
- Full write-ups in [06](06-known-issues-and-fixes.md); image/deploy detail in
  [02](02-railway-deployment.md).
- **Still open:** rendered-board confirmation by the tester (hard reload) for Bug 2 — not done as of
  this writing.

## Phase 11 — Board first-load scroll jank fix (2026-09-29)

- **Problem:** tester reported horizontal scrolling stutters/lags on a board's first load while cards
  are still loading; smooths out once loading finishes.
- **First attempt — commit `cf3817ff`, deployed but did NOT fix the symptom:** code-reading-only
  diagnosis found `Database.tsx` called `setRowMap` once per resolved row doc, so a board's parallel
  row loads produced one React state update per row. `cf3817ff` batches queued row-doc loads into
  `rowMap` on a 100 ms flush timer instead of one `setRowMap` per row; `rowMapRef` still updates
  immediately for dedup, user-created rows still update immediately. `tsc`/`eslint` clean; code review
  found no CRITICAL/HIGH. Deployed as digest `sha256:35391c89d…`, Railway deployment `af70e0c4`
  (SUCCESS, 2026-09-29 04:42 +07:00) — **not profiled before deploying**. The tester then reported the
  board was still laggy on first load; this fix addressed a real but minor inefficiency, not the
  dominant cost, and is retained as a harmless change.
- **Root cause, found by profiling the live site:** the Long Animation Frame API
  (`PerformanceObserver` type `long-animation-frame`) on a cold load of CoreDating - Mingle2 (all 138
  IndexedDB row-doc databases for b.mgvlabs.com deleted, then reload + scroll) showed 98 long frames /
  11,991 ms total, with 11,418 ms attributed to 272 `DOMWebSocket.onmessage` invocations — not to React
  render work (~503 ms). `react-use-websocket` 4.13.0 applies each message via
  `flushSync(() => setLastMessage(message))`, running the resulting render synchronously inside
  `onmessage`; `AppSyncLayer`'s `SyncInternalContext` memo included the per-message `webSocket` object
  and `useSync` bumped `lastUpdatedCollab` on every collab message, so the context value changed on
  every WebSocket message and rebuilt the app-wide `AppContext` — re-rendering the sidebar, board, and
  every card synchronously per message.
- **Actual fix — commit `e6390209`** ("perf: stop re-rendering the whole app on every WebSocket
  message"): drops unused `webSocket`/`broadcastChannel` from `SyncInternalContextType`; `useSync` only
  sets `lastUpdatedCollab` for `Types.Folder` messages; `useAllContextData`'s memo now depends on
  `syncData?.eventEmitter` only. Code review: no CRITICAL/HIGH. `tsc`/`eslint` clean. Deployed as
  digest `sha256:15e6ccc8b…`, Railway deployment `7958bda5` (SUCCESS, 2026-09-29 05:06 +07:00), live
  bundle `index-DYksSXv-.js`.
- **Verified fixed:** same profiling method on live prod after this deploy — 5 long frames / 401 ms
  total (longest 123 ms), WebSocket handler no longer in the attribution, remaining time is React's
  initial board render.
- **Lesson:** profile before fixing perf — the first fix was chosen from code reading alone and
  targeted ~0.5 s of the ~12 s of jank; the Long Animation Frame API on the live site (readable names
  in the prod bundle) found the real cost in minutes.
- Full write-up in [06](06-known-issues-and-fixes.md); deploy mechanics in
  [02](02-railway-deployment.md).

## Phase 12 — CRITICAL card-edit-not-saved bug found and fixed same day as the scroll-jank fix (2026-09-29)

- **Problem:** shortly after the scroll-jank fix (Phase 11) deployed, the tester reported card
  title/description edits weren't saving — a hard reload made the edit disappear.
- **Live diagnosis on prod:** editing a card's title changed it locally but sent no WebSocket update
  for the row collab; the server's collab read still returned the old value. Walking the page's React
  database context showed all 135/135 row docs in `rowMap` were registered under the wrong `guid`
  (`rowKey` form, not the bare `rowId`) with only a local IndexedDB listener — never wired for sync.
- **Root cause:** a race in AppFlowy-Web's `getOrCreateRowDocEntry` (`cache/index.ts`) — it checked
  for an existing row doc, awaited an IndexedDB open, then registered the entry; two callers
  requesting the same row concurrently (a card mount's `bindRowSync` vs. the board's
  `populateRowFromCache`) each created a separate Y.Doc before the await resolved, so sync got
  registered on one doc while the board/modal edited the other. The race is upstream code; the two
  same-day perf deploys (`cf3817ff`, `e6390209`) most likely changed load timing enough to hit every
  card, though that causal link is unverified.
- **Fix:** `compat-cloud-0.9.64` commit `5fad2172` adds a `pendingRowDocEntries` map so concurrent
  callers share the in-flight doc open instead of racing to create duplicates. `tsc`/`eslint` clean
  (no row-doc test coverage existed to update).
- **Deploy:** GitHub Actions `build-oss-web.yml` run `36496870314` → same tag, Railway deployment
  `51d67a26` (SUCCESS, 2026-09-29 06:16 +07:00), live bundle `index-BtS1lMX7.js`. **The prior
  deployment (`7958bda5`, commit `e6390209`) still has this bug and must not be used as a rollback
  target.**
- **Verified:** post-deploy, all 135/135 row docs registered correctly; an edit made during the bug
  window (held only in local IndexedDB) synced to the server on reload; a fresh edit reached the
  server and survived a hard reload. Edits made by users during the bug window self-recover on
  reopening the board in the same browser, unless site data was cleared.
- **Collateral:** an accidental extra field ("Text", id `HrsnJo`) created during diagnosis on Mingle2
  was removed via a database-collab web-update.
- **Lesson:** a perf-motivated web change (Phase 11) needs its write path re-verified, not just its
  render/perf profile — the same gap the compat-build deploy's open write-path caveat flagged back in
  Phase 9.
- Full write-up in [06](06-known-issues-and-fixes.md); deploy mechanics in
  [02](02-railway-deployment.md).

## Phase 13 — Owner-only page-move gating, checklist-blur fix, data fixes (2026-09-30)

- **Extended the 2026-07-25 delete guardrail (Phase 8) to page moves:** the workspace Owner asked
  that only Owners be able to move pages between spaces/parents, matching the existing "Members can
  edit but not delete" model. Verified beforehand that any Member could move a page (`POST
  .../page-view/{view}/move` as Member `thanh+1` → `200`). Confirmed current Owners are
  `thanh@mingle.com` and `bao.pham@mingle.com` from `af_workspace_member`/`af_roles`; all other
  members are `Member`.
  - **Cloud** commit `44219d40` adds `enforce_role_strong(AFRole::Owner)` to `move_page_handler`,
    mirroring the delete-guardrail pattern (5-line diff over `0.9.64-delete-guardrail`). Built as tag
    `0.9.64-owner-move`, deployed to `AppFlowy-Cloud` (Railway deployment `f8dd443b`, SUCCESS,
    2026-09-30 14:50 +07:00) — deployed **before** the web change per the new Cloud-before-Web
    ordering rule (see [07](07-operations-runbook.md)).
  - **Web** commit `bbd5ea7c` on `compat-cloud-0.9.64` adds `useCanMoveContent()`, refuses the move
    client-side with a toast for non-Owners, hides "Move to" in the page header menu, and turns the
    space picker into a read-only label for non-Owners. Deployed as Railway deployment `3281e4a7`
    (SUCCESS, 2026-09-30 14:51 +07:00).
  - Verified live: Member move → `code:1012` "User does not have permissions…", unmoved; Owner move
    worked and the test page was restored to its original position.
  - A tester report of "can't move board" turned out to be page-move, now blocked for Members by
    design; investigating a similar report from Owner `bao.pham` surfaced repeated `ExpiredSignature`
    WS auth failures in the Cloud logs (whose session is unverified) — advised re-login.
  - Full write-up, digests, and rollback caveats in [06](06-known-issues-and-fixes.md) and
    [02](02-railway-deployment.md).
- **Checklist sub-task description discarded on blur — fixed:** `NewTask.tsx`'s `onBlur` only hid the
  input instead of submitting it, silently losing typed sub-task descriptions (Enter and the `+`
  button both worked correctly). Fixed in web commit `458f8e4a` — onBlur now creates the task if text
  was typed; Escape still discards. Deployed as Railway deployment `516e8320` (SUCCESS, 2026-09-30
  14:12 +07:00), ahead of the move-gating deploy above. See [06](06-known-issues-and-fixes.md).
- **Data fix:** the Mingle2 board view had card `e1673d42` listed twice in `row_orders` (rendered
  twice on the board) — likely from concurrent card moves (unverified). Removed the duplicate entry
  via a database-collab web-update; both the board and grid views now agree at 136 rows.
- **New open issue logged (not fixed):** a short post-load window (~2–3 s warm, up to ~14 s cold)
  where cards still show "Untitled" and moves/cell-edits against those rows are silently dropped —
  distinct from the (already-fixed) full-board "Untitled for minutes" cold-IndexedDB bug. See
  [06](06-known-issues-and-fixes.md).

## Phase 14 — Card-edit-during-load fix, board-in-space move fix (2026-09-30)

- **Closed the Phase 13 open issue** (brief post-load window where card edits/moves against
  still-hydrating rows were silently dropped): measured on prod that `rowMap` stayed empty (0/136) for
  ~4 s after board render. Web commit `5e66186b` — "fix: apply card edits made while the board is
  still loading instead of dropping them" — adds `useResolveRowDoc`/`useWithRowDoc`, which wait for the
  row doc (`ensureRow`) and its data (`ROW_DATA_READY_TIMEOUT_MS = 15000`) before applying, instead of
  dropping immediately; wired into move/cell-edit/date-cell/row-meta/duplicate-row paths. Field-wide
  ops (clear/duplicate/switch property) still skip unloaded rows — known limitation, unchanged.
  Deployed as Railway `AppFlowy-Web` deployment `9a70fb02` (GitHub Actions run `36703140092`, SUCCESS,
  2026-09-30 ~17:35 +07:00), live bundle `index-WcDiN_Fe.js`. An actual edit timed to land *during* the
  loading window was not re-verified live post-deploy (unverified — needs a manual timed check).
- **Found and fixed a second Owner-move-day regression, unrelated to the Owner-only gate:** "Move to"
  was disabled for every board sitting directly in a space (Core Dating Board, Startup Project Board —
  19 boards total). Root cause: `isLinkedDatabaseViewUnderDocument()` in
  `src/application/view-utils.ts` misclassified space-parented boards (spaces are `Document`-layout
  views too) as linked views embedded in a document. Fixed in web commit `c6edc1bd` — "fix: allow
  moving databases that sit directly in a space" — plus a new unit test (79/79 passing in that file).
  Deployed as Railway `AppFlowy-Web` deployment `86985d2d` (GitHub Actions run `36720064526`, SUCCESS,
  2026-09-30 20:16:31 +07:00). Verified live as Owner `thanh@mingle.com` on CoreDating - Mingle2:
  "Move to" enabled, target picker lists all 5 spaces.
  - **Lesson:** the earlier same-day Owner-move verification (Phase 13) only exercised a Doc page
    (Tips Sharing), which is why this board-specific regression was missed — test board pages too.
- Full write-up, digests, and rollback caveats in [06](06-known-issues-and-fixes.md); deploy mechanics
  in [02](02-railway-deployment.md).

## Phase 15 — Secret rotation after the Railway security incident (2026-10-23)

- Rotated: GoTrue JWT secret (now referenced by Cloud, Worker, provisioner, admin-ui), the email-hook
  signing secret (both sides), MinIO root credentials = S3 keys (Cloud/Worker now reference MinIO),
  admin-ui password, and the `ops@mgvlabs.com` password (via GoTrue admin API).
- `minio/minio:latest` was no longer pullable → MinIO moved to `pgsty/minio:RELEASE.2026-08-04T00-00-00Z`.
- Removed the public `/minio/` console route from nginx (`nginx.live.conf` updated).
- Removed the plaintext admin-ui password from these docs (it was pushed to the public fork).
- Edge-log review Oct 19 → Oct 23: only team traffic plus generic scanners; no hits on GoTrue admin,
  `/console` or the MinIO console from outside.

## Standing constraints carried throughout

- No broad Railway↔GitHub access (only `nextc/AppFlowy-Cloud`).
- Never print/commit secrets — Railway vars / reference vars.
- `@mingle.com`-only signup + email verification.
- Host on Railway.
- Version pairing by release date.
