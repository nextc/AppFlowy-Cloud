# 01 — Architecture

## Goal

Self-hosted AppFlowy on Railway with **no seat limit** (13+ members), a **working web app** at
`b.mgvlabs.com`, **`@mingle.com`-only registration**, and **email verification** — without paying for
the commercial license.

## Why OSS, not the commercial image

The prebuilt `appflowyinc/appflowy_cloud` image enforces a commercial **1 Member/Owner seat cap**
("Seat limit reached / Upgrade your license"). This is a **closed-source add-on that is NOT in the
AGPL repo** — verified: `invite_workspace_members` in `src/biz/workspace/ops.rs` has no cap check,
there are no seat-limit strings, and no commercial cargo feature. The cap lives in commercial-only DB
tables (`af_self_host_commercial_license`) and code.

**Consequence:** build `appflowy_cloud` (and `worker`, `admin-frontend`) from AGPL source and the cap
is gone. Validated end-to-end 2026-07-05: an OSS build (tag `0.9.64`) invited 15 members with 0 seat
errors.

## The version-pairing rule (critical, non-obvious)

The three moving images have **independent version lines** and are paired by **same-day release
date, NOT matching version numbers**:

| Component | What we run | Line / note |
|-----------|-------------|-------------|
| `appflowy_cloud` | OSS build from git tag **`0.9.64`** | public git tags are `0.9.x`; commercial image line is `0.16.x` (~11 months ahead) |
| `appflowy_worker` | OSS build from git tag **`0.9.64`** | same repo, same tag |
| `gotrue` | `appflowyinc/gotrue:0.9.64` | in the 0.9.x era the gotrue image is tagged with the cloud release number; its Dockerfile clones `auth@0.8.0` |
| `appflowy_web` | `ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64` (self-built, since 2026-09-04) | see version history below |
| `admin_frontend` | OSS build from tag `0.9.64` (Rust) | optional; the `/console` portal |

All five in the `0.9.64` set were pushed on **2025-07-04**. Collab-protocol fixes `#1495` (ping
timeout), `#1464` (old-client init_sync loop), `#1465` (CollabParams) are all **inside** 0.9.64.

**`appflowy_web`'s version history (unlike the other four, it has moved twice since first deploy):**

1. **`appflowyinc/appflowy_web:0.1.19`** (initial deploy, 2026-07-06) — web was on the `0.1.x` line
   then, paired by release date with cloud `0.9.64`. It later jumps to `0.9.93` numbering
   (2025-08-18) — **this is why web `0.13.3`/`0.16.x` — later stock or `main`-built images — never
   worked with 0.9.64 cloud**: they're numbered far ahead but the pairing that matters is by release
   date and API surface, not the number.
2. **`appflowyinc/appflowy_web:0.9.93`** (2026-07-10) — upgraded for websocket/realtime support (web
   `0.1.19` had zero websocket code). See [06 §No realtime sync](06-known-issues-and-fixes.md).
3. **`ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64`** (2026-09-04, current) — self-built from a
   `nextc/AppFlowy-Web` branch pinned to the last commit (`d0704de2`) before upstream broke
   compatibility with cloud `0.9.64`'s folder API (commit `c8f44099`, see
   [06](06-known-issues-and-fixes.md)). Carries the Owner-only delete gating and new features
   (shareable card URLs, hidden-fields-expanded-by-default) that stock `0.9.93` doesn't have. Full
   detail in [06](06-known-issues-and-fixes.md) and [08 §Phase 9](08-history.md). **Rollback:**
   reconnect `appflowyinc/appflowy_web:0.9.93`.

**Why tag `0.9.64` and not `main`:** `main` HEAD = `0.9.64 + ~76 untagged commits` — no same-day web
image exists for it, so its collab-protocol pairing is unknowable. An earlier prod that ran OSS cloud
from `main` was broken. `0.9.64` is the newest *released* OSS with a verifiable same-day web/gotrue.

> **Historical note:** a lot of early debugging blamed "collab version-skew" (cloud 2026-05 vs web
> 2026-06). The consistent `0.9.64` stack is correct and worth keeping, but it was **not** the actual
> fix — the real bug was user provisioning (see [06](06-known-issues-and-fixes.md)). Don't re-chase
> version-skew first; check provisioning first.

## The OSS image build pipeline

Because we **refuse to grant Railway GitHub access to all repos** (privacy/security; only
`nextc/AppFlowy-Cloud` is allowed, and Railway's UI couldn't even see a single-repo grant), OSS
images are built entirely on GitHub and pulled by Railway as **plain public images**:

```
git tag 0.9.64  (in nextc/AppFlowy-Cloud fork)
        │
        ▼  GitHub Action  .github/workflows/build-oss.yml   (workflow_dispatch, input `ref`, default 0.9.64)
        │  matrix: cloud / worker / admin ; runners are amd64 (Mac is arm64 — can't build amd64 locally without slow emulation)
        ▼
ghcr.io/nextc/appflowy-cloud:0.9.64
ghcr.io/nextc/appflowy-worker:0.9.64      (all PUBLIC — no GitHub auth needed to pull)
ghcr.io/nextc/admin-frontend:0.9.64
        │
        ▼  railway service source connect --image ...
Railway services pull the public images
```

Build facts: `Dockerfile` (root) builds the `appflowy_cloud` bin via cargo-chef with `SQLX_OFFLINE=true`
and the committed `.sqlx` offline data. Needs `workflow` token scope + Actions enabled on the fork.
Rollback for any service = point its Source back to `appflowyinc/*:latest`.

## Auth + provisioning data flow (the heart of it)

```
User → b.mgvlabs.com (AppFlowy-Web SPA)
     → enters @mingle.com email
     → GoTrue sends magic-link email
          via send_email hook → email-hook service → Brevo HTTP API   (SMTP is blocked on Railway)
     → user clicks magic link → GoTrue verifies → issues access_token
     → browser SHOULD hit /auth/callback → GET /api/user/verify/{token}
          → AppFlowy-Cloud verify_token() creates af_user + default workspace
```

**The fragility:** corporate email link-scanners (Outlook SafeLinks, etc.) prefetch and **burn the
single-use magic-link token** before the human clicks, so the browser `/auth/callback` → `verify`
step often never provisions. That's why the **`provisioner`** service exists — it provisions
server-side, independent of the browser. See [03](03-onboarding-and-provisioning.md).

Key invariant: **AppFlowy-Cloud's ONLY provisioning path is `GET /api/user/verify/{access_token}` →
`verify_token()`.** There is no middleware auto-provision. If `af_user` doesn't exist, every
`/api/user/*` and `/api/workspace/*` returns `RecordNotFound` and the sidebar is blank.

## What the commercial layer did that we had to rebuild

| Commercial feature | Our replacement |
|--------------------|-----------------|
| Email sending (SMTP) | `email-hook` service → Brevo HTTP API ([05](05-admin-email-and-access.md)) |
| Signup domain whitelist | `enforce_signup_domain` Postgres trigger ([05](05-admin-email-and-access.md)) |
| Reliable user provisioning | `provisioner` service ([03](03-onboarding-and-provisioning.md)) |
| Team onboarding / shared workspace | `provisioner` auto-join ([03](03-onboarding-and-provisioning.md)) |
| Member management UI | `admin-ui` service ([05](05-admin-email-and-access.md)) |
| Subscription / billing | nginx billing-API spoof ([04](04-nginx-and-client-patches.md)) |
