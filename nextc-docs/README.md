# nextc-docs — Self-hosted AppFlowy-Cloud (MGVLabs)

Single source of truth for how **MGVLabs runs AppFlowy-Cloud on Railway** as a self-built AGPL/OSS
stack — built to escape the commercial **1-member seat cap** and host a ~13-person team on our own
domain, with `@mingle.com`-only registration and email verification.

These docs are written for a future engineer (or a future Claude session) picking this up cold.
Everything here was learned the hard way across several build/debug sessions in 2026-07; the
"Known issues & fixes" catalog is the highest-value part — every entry is a bug we already paid for.

> **Live mirror of this knowledge** also exists in Claude memory at
> `~/.claude/projects/-Users-lastair-code-oss-AppFlowy-Cloud/memory/` (`oss-conversion.md`,
> `railway-deployment.md`, `current-plan.md`). These `nextc-docs/` files are the durable,
> repo-committed version. Keep both in sync when something changes.

---

## The one-paragraph summary

The commercial `appflowyinc/appflowy_cloud` image enforces a 1-seat license cap that is **not** in the
AGPL source. So we build `appflowy_cloud` + `appflowy_worker` + `admin-frontend` from the OSS git tag
`0.9.64` (via a GitHub Action in the **`nextc/AppFlowy-Cloud`** fork → public GHCR images), and run
them on Railway alongside commercial `gotrue:0.9.64` and a self-built
**`ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64`** (originally `appflowy_web:0.1.19`, paired by
**release date**, not version number — see [01](01-architecture.md) for the full web version
history). Several things the commercial layer normally handles had to be rebuilt
ourselves: **email** (Railway blocks SMTP → GoTrue `send_email` hook → our `email-hook` service →
Brevo HTTP API), **user provisioning + team onboarding** (our `provisioner` service), **member
management UI** (our `admin-ui` service), **signup domain restriction** (a Postgres trigger), and
**subscription-gate bypass** (nginx spoofs the billing API). The app is live at
**https://b.mgvlabs.com**.

---

## Document index

| # | Doc | What's in it |
|---|-----|--------------|
| — | [README.md](README.md) | This page — overview, quick reference, troubleshooting index |
| 01 | [01-architecture.md](01-architecture.md) | Every service, the version-pairing rule, auth/provisioning data flow, the OSS image build pipeline |
| 02 | [02-railway-deployment.md](02-railway-deployment.md) | Railway project wiring, reference vars, `PUBLIC_HOST`/custom domain, the `NGINX_CONFIG_B64` mechanism, deploy commands |
| 03 | [03-onboarding-and-provisioning.md](03-onboarding-and-provisioning.md) | The `provisioner` service: on-demand (LISTEN/NOTIFY) + 5s poll, auto-join, invite auto-accept, JWT minting, first-login race fix |
| 04 | [04-nginx-and-client-patches.md](04-nginx-and-client-patches.md) | Full nginx routing, billing-API spoof, upgrade-button hiding, `af_ws_retry` reload script, edit procedure |
| 05 | [05-admin-email-and-access.md](05-admin-email-and-access.md) | `admin-ui` member management, `/console` admin-frontend, `email-hook`+Brevo, `@mingle.com` trigger, credentials map |
| 06 | [06-known-issues-and-fixes.md](06-known-issues-and-fixes.md) | **The root-cause catalog** — every bug + its verified fix |
| 07 | [07-operations-runbook.md](07-operations-runbook.md) | Task recipes: add/remove member, change password, redeploy nginx, rebuild images, switch domain, cleanup; classifier-gated actions |
| 08 | [08-history.md](08-history.md) | Chronological log of decisions and pivots |

---

## Quick reference

### Services (Railway project `appflowy`, workspace MGVLabs, env `production`)

| Service | Image / source | Role |
|---------|----------------|------|
| **Nginx** | `NGINX_CONFIG_B64` (our config) | Reverse proxy; serves `b.mgvlabs.com`; billing spoof + client patches |
| **AppFlowy-Web** | `ghcr.io/nextc/appflowy_web:compat-cloud-0.9.64` (self-built, since 2026-09-04) | SPA front end |
| **AppFlowy-Cloud** | `ghcr.io/nextc/appflowy-cloud:0.9.64` (OSS) | API + collab; **no seat cap** |
| **AppFlowy-Worker** | `ghcr.io/nextc/appflowy-worker:0.9.64` (OSS) | Background jobs |
| **GoTrue** | `appflowyinc/gotrue:0.9.64` | Auth |
| **admin-frontend** | `ghcr.io/nextc/admin-frontend:0.9.64` (OSS Rust) | `/console` super-admin |
| **pgvector** | `pgvector/pgvector:pg16` | Postgres (app + `auth` schema) |
| **Redis** | official | Cache / pubsub |
| **MinIO** | `pgsty/minio:RELEASE.2026-08-04T00-00-00Z` (since 2026-10-23; `minio/minio` no longer pullable) | S3 storage |
| **email-hook** | Node (our `scratchpad/email-hook/`) | GoTrue email → Brevo HTTP API |
| **provisioner** | Node (our `scratchpad/provisioner/`) | Auto-provision + auto-join + invite-accept |
| **admin-ui** | Node/express (our `scratchpad/admin-ui/`) | Password-protected member management |

### URLs

- App: **https://b.mgvlabs.com**
- Super-admin console: `https://b.mgvlabs.com/console`
- Member management (ours): **https://admin-ui-production-8ea6.up.railway.app**

### Where credentials live (never hard-code — all in Railway vars / reference vars)

| Secret | Location |
|--------|----------|
| Postgres | `pgvector` service (`DATABASE_URL` / `DATABASE_URL_PRIVATE`) |
| GoTrue JWT secret | `GoTrue.GOTRUE_JWT_SECRET` (referenced by Cloud, Worker, provisioner, admin-ui) |
| MinIO / S3 keys | `MinIO.MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` (referenced by Cloud/Worker `APPFLOWY_S3_*`) |
| Email-hook signing secret | `email-hook.HOOK_SECRET` = the part after `whsec_` in `GoTrue.GOTRUE_HOOK_SEND_EMAIL_SECRETS` |
| Super-admin login | `ops@mgvlabs.com` + `GoTrue.GOTRUE_ADMIN_PASSWORD` |
| Member-management UI password | `admin-ui.ADMIN_PASSWORD` (read it in Railway; never write it here) |
| Brevo API key | `email-hook.BREVO_API_KEY` (sealed: unreadable via CLI/API) |
| Redis | `Redis.REDIS_PASSWORD` (Cloud/Worker/admin-frontend reference `${{Redis.REDIS_URL}}`) |
| Public domain | `Nginx.PUBLIC_HOST` = `b.mgvlabs.com` (single source; referenced everywhere) |

### Troubleshooting index (symptom → doc)

| Symptom | See |
|---------|-----|
| Blank sidebar / "Disconnected" on **every** login, no self-heal | [06 §RecordNotFound](06-known-issues-and-fixes.md) — provisioning/allow-list |
| Blank sidebar on **first** login, works after refresh | [06 §First-login race](06-known-issues-and-fixes.md) — provisioner latency + `af_ws_retry` |
| "Seat limit reached … upgrade your license" (console) | [06 §Seat cap](06-known-issues-and-fixes.md) — must run OSS build |
| "Member limit reached, please upgrade" (invite modal) | [06 §Member limit](06-known-issues-and-fixes.md) — nginx billing spoof |
| Teammate can't be invited / invite shows an error | [03](03-onboarding-and-provisioning.md) — SMTP blocked; provisioner auto-accepts |
| No email arrives (verify/magic-link/invite) | [05 §email-hook](05-admin-email-and-access.md) — Railway blocks SMTP |
| Custom domain 502 "Application failed to respond" | [06 §Domain 8080](06-known-issues-and-fixes.md) — nginx must `listen 8080` |
| `/console` user list empty / `403 not_admin` | [06 §Admin role](06-known-issues-and-fixes.md) — role must be `supabase_admin` |
| "Upgrade to Pro Plan" / "Get AI Max" buttons showing | [04 §Button hide](04-nginx-and-client-patches.md) |
| Where do I manage members? | [05 §admin-ui](05-admin-email-and-access.md) — web has no member UI in 0.1.19 |
| Database rows visible on one laptop but not others | [06 §Orphaned DatabaseRow](06-known-issues-and-fixes.md) — collab sync failure |
| Edits never live-sync between browsers | [06 §No realtime sync](06-known-issues-and-fixes.md) — web build must be ≥0.9.93 |
| WebSocket keeps dropping / reconnecting every ~30s | [06 §WebSocket 1003](06-known-issues-and-fixes.md) — cloud version-skew; needs cherry-pick |

### Standing constraints (do not violate)

- **Never grant Railway broad GitHub access** — only `nextc/AppFlowy-Cloud`. This is why OSS images
  are built via GitHub Actions → public GHCR, and Railway pulls them as plain public images.
- **Never print or commit secrets** — use Railway vars / reference vars.
- **`@mingle.com`-only signup** enforced by the `enforce_signup_domain` Postgres trigger.
- **Version pairing is by release date**, not version number — don't "upgrade" a single image casually.
- Prod-DB writes/deletes via `railway ssh psql` and GitHub/domain mutations are **auto-mode
  classifier-gated** — those get handed to the user via `! <cmd>`.
