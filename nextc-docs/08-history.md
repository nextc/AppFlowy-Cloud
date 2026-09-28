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
- User: "change the admin password to `mgv-team-2026`" → done + verified.

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
- **Root cause (code reading, not profiled):** `Database.tsx` called `setRowMap` once per resolved row
  doc, so a board's parallel row loads produced one React state update per row — each triggering
  `useRowsByGroup`'s full regroup + `observeDeep` resubscribe on every row (`selector.ts`) and a
  re-render of every mounted card (`CardPrimitive.tsx` consumes `rowMap` via context). Columns aren't
  horizontally virtualized (~14 mounted), compounding the re-render cost.
- **Fix:** `nextc/AppFlowy-Web` branch `compat-cloud-0.9.64` commit `cf3817ff` batches queued row-doc
  loads into `rowMap` on a 100 ms flush timer instead of one `setRowMap` per row; `rowMapRef` still
  updates immediately for dedup, and user-created rows still update immediately. `tsc`/`eslint` clean;
  code review found no CRITICAL/HIGH (one accepted MEDIUM: cell population can lag title/meta by up to
  100 ms).
- **Deploy:** rebuilt the same `compat-cloud-0.9.64` tag (GitHub Actions run `36487579668`) → digest
  `sha256:35391c89d62e8c7812ddf4bc8d60879ce1b2615858e11a88083e664b601989fd`; Railway `AppFlowy-Web`
  redeployed as deployment `af70e0c4` (SUCCESS, 2026-09-29 04:42 +07:00), superseding the 2026-09-28
  build (digest `sha256:43782e958…`, deployment `8d045693`). Live bundle confirmed to contain
  `ROW_MAP_FLUSH_MS` / `rowMapFlushTimerRef`.
- **Not profiled before/after** — user chose deploy-and-check over measuring. Tester's live
  confirmation is **PENDING** as of this writing.
- Full write-up in [06](06-known-issues-and-fixes.md); deploy mechanics in
  [02](02-railway-deployment.md).

## Standing constraints carried throughout

- No broad Railway↔GitHub access (only `nextc/AppFlowy-Cloud`).
- Never print/commit secrets — Railway vars / reference vars.
- `@mingle.com`-only signup + email verification.
- Host on Railway.
- Version pairing by release date.
