# 05 — Admin, Email & Access

## Credentials map (all in Railway vars — never hard-coded)

| What | Login / value | Where the secret lives |
|------|---------------|------------------------|
| Super-admin console `/console` | `ops@mgvlabs.com` | `GoTrue.GOTRUE_ADMIN_PASSWORD` |
| Member-management UI (`admin-ui`) | password only | `admin-ui.ADMIN_PASSWORD` (read it in Railway; never write it here) |
| MinIO root (no public console since 2026-10-23; use `railway ssh --service MinIO` + `mc`) | `MinIO.MINIO_ROOT_USER` | `MinIO.MINIO_ROOT_PASSWORD` (Cloud/Worker `APPFLOWY_S3_*` reference these) |
| Postgres | — | `pgvector` service |
| GoTrue JWT secret | — | `GoTrue.GOTRUE_JWT_SECRET` (Cloud, Worker, provisioner, admin-ui reference it) |
| Brevo API key | — | `email-hook.BREVO_API_KEY` |

## `admin-ui` — member management (our service)

**Why it exists:** AppFlowy-Web **v0.1.19 has no wired member-management UI**. `app/settings/Settings.tsx`
(the Members/Workspace/Account tabs, reads `?setting=<ITEM>`) is **not rendered anywhere** (0 refs in
app.hooks / AppRouter / AppPage / SideBar) — there's no Settings button and `?setting=MEMBERS` does
nothing. Only **inviting** is in the UI (workspace switcher → "Invite members"; `InviteMember.tsx` is
invite-form-only, no member list). Workspace rename/delete *is* available (switcher → hover → "…").

So we built **`admin-ui`** (Node + express + pg, source scratchpad `admin-ui/`, deployed in the
`appflowy` project) — a password-protected page at
**https://admin-ui-production-8ea6.up.railway.app**.

- **Lists / adds / removes / role-changes** members of `SHARED_WORKSPACE_ID` by minting the workspace
  **owner's** JWT and calling the cloud member API (see [03](03-onboarding-and-provisioning.md) for
  the API + JWT details). Add = invite `skip_email_send` + accept.
- **Owner is protected** from removal/demotion.
- **Remove** → cloud `DELETE` + INSERT into `af_admin_excluded` (so the provisioner won't re-add).
  **Add** → DELETE from `af_admin_excluded` (re-allow).
- Session = HMAC of `ADMIN_PASSWORD`; login uses `timingSafeEqual` on the HMAC.
- Env (reference vars): `DATABASE_URL`, `GOTRUE_JWT_SECRET`, `CLOUD_URL`, `SHARED_WORKSPACE_ID`,
  `ADMIN_PASSWORD`.

To change the password:
```bash
railway variables --service admin-ui --set "ADMIN_PASSWORD=<new>" --skip-deploys
railway redeploy --service admin-ui -y
```

## `/console` — super-admin (admin-frontend)

Now the **OSS Rust `admin_frontend`** image (`ghcr.io/nextc/admin-frontend:0.9.64`), reachable at
`/console`, `basePath=/console`. It reads `ADMIN_FRONTEND_REDIS_URL` / `_GOTRUE_URL` /
`_APPFLOWY_CLOUD_URL` / `_PATH_PREFIX=/console`, listens on :3000.

> **History:** we first deployed the 2026 Next.js "AppFlowy Super Admin" image, but it needs
> commercial-only endpoints (`/api/admin/users`, `/api/user/gotrue-profile`) that OSS lacks → 404s.
> The OSS Rust admin at the paired `0.9.64` is the compatible one.

The `/console` nginx block forces `X-Forwarded-Proto https` (Railway terminates TLS; nginx sees http)
so Super Admin sets Secure session cookies and avoids Mixed-Content on browser calls.

### Admin identity notes
- `ops@mgvlabs.com` is a GoTrue **system admin** (`app_metadata.is_system_admin: true`).
  AppFlowy-Cloud intentionally **skips workspace creation** for it, so it **cannot** use the `/app`
  workspace UI ("unable to load workspace") — only `/console`.
- Its GoTrue role must be **`supabase_admin`**, not `admin`. AppFlowy-Cloud accepts `admin` (via
  `is_system_admin`) but GoTrue's own `/admin/*` API rejects it (`403 not_admin`, empty user list).
  Fix: `UPDATE auth.users SET role='supabase_admin' WHERE email='ops@mgvlabs.com'` then re-login for a
  fresh JWT. (Matches `admin_frontend/src/web_app.rs:524` `role == "supabase_admin"`.)
- GoTrue is an API with no homepage — `/gotrue/` 404 is normal; check `/gotrue/health`.

## Email — GoTrue `send_email` hook → `email-hook` → Brevo (REQUIRED)

**Railway blocks outbound SMTP** (ports 587/2525 time out; 443 open) — so Brevo SMTP can't work, and
neither can GoTrue's built-in SMTP mailer nor the cloud's workspace-invite SMTP. Fix: GoTrue's
**`send_email` hook** (the `AppFlowy-IO/auth` fork supports it) → our **`email-hook`** Node service →
**Brevo HTTP API**.

Config on **GoTrue**:
- `GOTRUE_HOOK_SEND_EMAIL_ENABLED=true`
- `GOTRUE_HOOK_SEND_EMAIL_URI` = the webhook's **public HTTPS** URL (gotrue rejects `http` for
  non-localhost, so the hook has its own Railway public domain: `email-hook-production.up.railway.app`)
- `GOTRUE_HOOK_SEND_EMAIL_SECRETS=v1,whsec_<b64>` matching the webhook's `HOOK_SECRET`
- `GOTRUE_MAILER_AUTOCONFIRM=false` (verification required)

`email-hook` specifics (source scratchpad `email-hook/`):
- `buildLink` **forces `redirect_to=${site}/auth/callback`** (`AUTH_CALLBACK_PATH` overridable). The
  incoming `data.redirect_to` is unreliable — the web sends it via a `Redirect_to` header that nginx
  underscore-handling can drop. This is half of the RecordNotFound fix (see
  [06](06-known-issues-and-fixes.md)).
- Logs `action` + recipient only — **never the token** (security).

Brevo prerequisites: **disable "Authorised IPs"** (Railway egress rotates), the account must be
**activated** (add a company profile), and use a **v3 API key** (`xkeysib-...`) in `BREVO_API_KEY`.

## `@mingle.com`-only registration

OSS has no signup whitelist (commercial-only), so it's a Postgres trigger **`enforce_signup_domain`**
on `auth.users` (rejects non-`@mingle.com`, except `ops@mgvlabs.com`) + `GOTRUE_DISABLE_SIGNUP=false`
+ `GOTRUE_MAILER_AUTOCONFIRM=false`. **Fragile to GoTrue auth re-migrations — re-add if it disappears.**
SQL staged at `scratchpad/deploy/mingle-trigger.sql`.
