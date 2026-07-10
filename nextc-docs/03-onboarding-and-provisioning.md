# 03 — Onboarding & Provisioning

Everything about getting a `@mingle.com` teammate from "signed up" to "working in the shared
workspace" — fully hands-off. The engine is the **`provisioner`** service (Node, source in scratchpad
`provisioner/`, deployed in the `appflowy` project).

## Why it exists

AppFlowy-Cloud only creates a user's `af_user` + default workspace via
`GET /api/user/verify/{access_token}` → `verify_token()`, which the web calls **only** from its
`/auth/callback` route. Corporate email **link-scanners burn the single-use magic-link token** before
the human clicks, so that browser step frequently never provisions. The provisioner does it
server-side, no browser and no email dependency.

## What the provisioner does each sweep

1. **`sweep()` — provision** confirmed `@mingle.com` users that have no `af_user`:
   `SELECT auth.users WHERE email_confirmed_at IS NOT NULL AND no af_user AND email LIKE '%@mingle.com'
   AND email <> ops@mgvlabs.com`, and for each **mints a gotrue-valid user JWT** and calls
   `GET {cloud}/api/user/verify/{jwt}` → creates `af_user` + default workspace.
2. **`ensureSharedInvites()` — auto-join** every provisioned `@mingle.com` user to the shared
   workspace (`SHARED_WORKSPACE_ID=dc8cd0fa-5d51-45c7-9811-a3d81499c9a0`): mint the **workspace
   owner's** JWT and `POST /api/workspace/{shared}/invite` with
   `[{email, role:"Member", skip_email_send:true}]` (skip_email_send avoids the blocked SMTP → clean
   200). Skips owner, existing members, already-invited, and anyone in `af_admin_excluded`.
3. **`acceptPendingInvites()` — auto-accept** pending invites (workspace-invite emails are also
   SMTP-blocked, but the invite row is committed): `POST /api/workspace/accept-invite/{id}` with the
   invitee's minted JWT. This updates the access-control cache (a raw DB `status=1` UPDATE would fire
   the `add_to_af_workspace_member` trigger but miss the cache).

Net effect: a teammate signs in (magic link) → provisioner provisions + auto-joins them to the shared
workspace, now within ~1s (see on-demand below).

## Two triggers: on-demand (primary) + poll (fallback)

### On-demand — Postgres LISTEN/NOTIFY (added 2026-07-07)

The provisioner installs (idempotent, on startup, via its **own** pg connection — not
classifier-gated) a trigger on `auth.users`:

```sql
CREATE OR REPLACE FUNCTION af_notify_provision() RETURNS trigger AS $$
  BEGIN PERFORM pg_notify('af_provision', NEW.id::text); RETURN NEW; END; $$ LANGUAGE plpgsql;

CREATE TRIGGER af_provision_trg
  AFTER INSERT OR UPDATE OF email_confirmed_at ON auth.users
  FOR EACH ROW WHEN (NEW.email_confirmed_at IS NOT NULL)
  EXECUTE FUNCTION af_notify_provision();
```

It holds a dedicated `pg.Client` running `LISTEN af_provision` (reconnects on drop) and, on
notification, runs a **debounced** `runSweep()` (~0.5s). So provisioning fires the instant a user is
created or confirmed instead of waiting for the poll.

**Why LISTEN/NOTIFY and not a GoTrue auth hook** (e.g. Custom Access Token hook): the auth-hook runs
*inside* the login path, so a slow/failed provision call would add latency to — or break — every
login. NOTIFY is fully decoupled; the worst case is a missed notification, which the poll catches.

Startup markers to look for: `on-demand trigger installed` + `listening for af_provision notifications`.
(Live notification delivery was **not observed in-session** — the direct `railway ssh psql` test-fire
is classifier-gated. Verify with the one-liner in [07](07-operations-runbook.md), or the next real
signup.)

### Poll — fallback

`POLL_INTERVAL_MS=5000` (was 30000). Even if a NOTIFY is missed, `af_user` exists within seconds.
Sweeps are **serialized** (`sweeping`/`pending` flags) so a notify-sweep and the poll can't overlap,
and a notification arriving mid-sweep re-runs it once (never misses a just-confirmed user).

## The first-login race, and the client-side third layer

Even with fast provisioning, a brand-new user's app can call `GET /api/workspace` **before**
provisioning finishes → gets `200 {data:[]}` → web renders a blank sidebar off the empty list and
**never re-fetches**. Refresh works once provisioning has completed. Fix layers:

0. **On-demand provision** (above) — sub-second.
1. **5s poll** — fallback.
2. **Client auto-reload** (`af_ws_retry`, injected by nginx) — if an authenticated user's workspace
   list comes back empty, reload once (bounded to 2, then stop). Detail in
   [04](04-nginx-and-client-patches.md).

Belt-and-suspenders: a new teammate should land on a populated sidebar without touching refresh.

## JWT minting — the critical detail

The provisioner (and admin-ui) mint gotrue-valid JWTs: **HS256 signed with `GOTRUE_JWT_SECRET`**.

- For `GET /api/user/verify/{token}` (token in the path), a **minimal** `{sub, email}` JWT works —
  that path calls gotrue `/user` to resolve the user.
- For any endpoint using the **`Authorization` header extractor** (accept-invite, invite, member API),
  the token is decoded **locally** into `GoTrueJWTClaims` (`libs/gotrue-entity/src/gotrue_jwt.rs`),
  which requires **non-optional** `email`, `phone`, `app_metadata`, `user_metadata`, `role`. A minimal
  JWT 401s with `missing field phone`. **The minted JWT must include all five.** Canonical claims:
  ```json
  { "sub": "<user_id>", "email": "<email>", "phone": "", "aud": "authenticated",
    "role": "authenticated", "app_metadata": {}, "user_metadata": {}, "iat": <now>, "exp": <now+300> }
  ```

Build gotchas (all avoided by self-minting): `generate_link type=magiclink` upserts → duplicate-key
500 on existing users; `type=recovery` → 404 for passwordless (magic-link) users. **Self-minting the
JWT directly is simplest and works.**

## Member API surface (0.9.64)

| Action | Call |
|--------|------|
| Invite | `POST /api/workspace/{id}/invite` body `[{email, role:"Member", skip_email_send:true}]` |
| Accept | `POST /api/workspace/accept-invite/{invite_id}` |
| Remove | `DELETE /api/workspace/{id}/member` body `["email"]` |
| Change role | `PUT /api/workspace/{id}/member` body `{email, role}` |

`AFRole` serializes by name: `"Owner"` / `"Member"` / `"Guest"` (role_id 1/2/3).

## Auto-join vs. manual-remove conflict → `af_admin_excluded`

If an admin removes a member, the next auto-join sweep would re-add them. Resolved with a shared table
`af_admin_excluded (email text primary key, added_at)`:
- `admin-ui` **INSERTs** on remove → `ensureSharedInvites()` skips excluded emails.
- `admin-ui` **DELETEs** on add → re-allows.
- Both services `CREATE TABLE IF NOT EXISTS af_admin_excluded` on startup.

Note: remove takes a member out of the shared workspace and blocks re-join, but their gotrue account +
personal workspace remain (a full account delete is the classifier-gated DB delete).

## Provisioner env (Railway reference vars)

`DATABASE_URL=${{pgvector.DATABASE_URL_PRIVATE}}`, `GOTRUE_JWT_SECRET=${{GoTrue.GOTRUE_JWT_SECRET}}`,
`GOTRUE_URL` / `CLOUD_URL` (internal `.railway.internal`), `SITE_URL=https://b.mgvlabs.com`,
`ADMIN_EMAIL=ops@mgvlabs.com`, `SHARED_WORKSPACE_ID=dc8cd0fa-5d51-45c7-9811-a3d81499c9a0`,
`POLL_INTERVAL_MS=5000`. Unset `SHARED_WORKSPACE_ID` to revert to invite-based (no auto-join).
