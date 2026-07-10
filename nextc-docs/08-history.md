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

## Standing constraints carried throughout

- No broad Railway↔GitHub access (only `nextc/AppFlowy-Cloud`).
- Never print/commit secrets — Railway vars / reference vars.
- `@mingle.com`-only signup + email verification.
- Host on Railway.
- Version pairing by release date.
