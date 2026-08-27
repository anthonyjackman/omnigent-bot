# Request-a-Host: device-grant provisioning flow

Design note (research only). How the existing delegated-auth machinery lets an
external provisioning system spin up a per-user AHV host with **zero forked
Omnigent code**, and where the genuinely additive integration points are.
Companion to `designs/GLOSSARY.md` and `designs/grok-bot-parity.md`.

Constraint honored throughout: **add beside Omnigent, don't modify it.** The
org-specific system (approvals, AHV automation) lives in its own repo, the way
`integrations/slack/` does; anything that must live inside Omnigent goes
through existing extension seams or a small upstreamable PR.

---

## 1. The device grant flow, precisely

Omnigent implements the OAuth 2.0 **Device Authorization Grant (RFC 8628)** —
`designs/DEVICE_AUTH.md`, `omnigent/server/routes/device_auth.py`. It exists
so an application (the Slack bot today; our provisioning portal tomorrow) can
act *as a user* against the server without ever seeing the user's password or
IdP credentials.

Actors: the **client** (our portal), the **server** (Omnigent), the **user**
(in a browser where they're already logged into Omnigent).

1. **Authorize.** The client calls `POST /oauth/device/authorize` with its
   `client_id` (a public display string, e.g. `"host-provisioner"`). The
   server returns a `device_code` (client-side secret), a short `user_code`
   (confusable-free alphabet — no 0/O, 1/I/L), and a verification URI.
   The pair lives **10 minutes** (`_DEVICE_CODE_TTL_SECONDS = 600`).
2. **Consent.** The user opens the verification URI — a consent page served
   by the Omnigent server, requiring their existing logged-in browser session
   — enters/confirms the `user_code`, and approves "host-provisioner wants to
   act on your behalf".
3. **Poll.** Meanwhile the client polls `POST /oauth/token`
   (`grant_type=urn:ietf:params:oauth:grant-type:device_code`, ≥ 5 s
   interval). Pending → `authorization_pending`; after approval → an
   **access token + refresh token**, with the approving user as `sub`.
4. **Use.** The access token is a bearer JWT presented as
   `Authorization: Bearer <jwt>`. `auth.py` resolves it to the user on every
   request, with a **live revocation check** on the grant.

Token properties that matter for an unattended host:

| Property | Value | Source |
|---|---|---|
| Access-token TTL | **1 hour** — client refreshes silently | `_ACCESS_TOKEN_TTL_SECONDS = 3600` |
| Refresh token | Rotates on every refresh; **reuse detection revokes the grant** (theft signal), with a retry-tolerance window so a lost response doesn't brick an unattended host | `_handle_refresh_grant` |
| Absolute grant lifetime | **30 days**, then refresh is refused and the user must re-consent. Operator-extendable via `OMNIGENT_GRANT_MAX_LIFETIME_DAYS` — the docstring explicitly blesses this for "deployments running unattended hosts" | `_GRANT_MAX_LIFETIME_SECONDS` |
| Scope | `"sessions"` → requests are checked against a **fail-closed path allowlist**: `/health`, `/v1/agents`, `/v1/hosts`, `/v1/sessions`, `/v1/runners`, `/oauth/token`, `/oauth/revoke` (prefix match, so `/v1/hosts/{id}/tunnel` passes) | `_DELEGATED_ALLOWED_PREFIXES` |
| Revocation | `POST /oauth/revoke`, plus per-request `grant_id` revocation lookup — a revoked grant dies on the next request/refresh (an already-open WebSocket persists until its next reconnect/ping cycle) | `auth.py:_check_cookie` |
| Feature gate | **Default off.** `OMNIGENT_DEVICE_GRANT_ENABLED=1` mounts the consent routes; optional `OMNIGENT_DEVICE_CLIENT_SECRET` restricts who may even *start* a grant (our portal would hold it) | `app.py` |

Two grant flavors share this machinery — don't conflate them:

- **Device grant** (above): third-party client, `scope="sessions"`,
  path-allowlisted. What the portal uses in `accounts` auth mode.
- **Login grant**: minted by the *interactive* login flows (`/auth/login`,
  `/auth/cli-login`) so the user's own CLI walks away with refresh material.
  No scope → **full user authority**, still revocable by grant id. This is
  what `omnigent login` already stores in `~/.omnigent/auth_tokens.json`.

### Auth-mode matrix (decides which flow the portal drives)

| Server auth mode | Flow available to the portal | Token the VM ends up holding |
|---|---|---|
| `accounts` (built-in) | Device grant (consent page mounts here, and only here) | Scoped delegated token — least authority, **recommended** |
| `oidc` (SSO) | Consent routes never mount. Use the **cli-login ticket flow**: portal starts a ticket at `/auth/cli-login`, sends the user to sign in at the IdP, polls `/auth/cli-poll`, receives a session JWT + refresh token (a login grant) | Unscoped login grant — full user authority on the VM |
| `header` (trusted proxy) | Neither — the server mints no tokens in this mode | Not supported |

The `/oauth/token` refresh/revoke half mounts in both `accounts` and `oidc`
modes, so refresh works for either flavor.

---

## 2. The Request-a-Host flow, end to end

Everything org-specific lives in the **portal** (approvals + AHV automation),
a separate service we own.

1. **Request.** The user clicks "Request a computer" (see §3 for where that
   button lives) → lands on the portal, authenticated by org SSO.
2. **Consent now.** The portal immediately runs the grant flow (§1) — the
   device-code window is only 10 minutes, so consent happens at request time,
   not at approval time. The portal now holds a refreshable grant for that
   user and keeps it server-side (encrypted), exactly as the Slack
   integration does (`integrations/slack/src/omnigent_slack/{oauth,tokens}.py`
   is the reference client to copy).
3. **Approve.** The approval workflow runs on its own clock (manager
   sign-off, quotas). Any latency under the 30-day grant lifetime is fine —
   the portal refreshes the grant while it waits.
4. **Provision.** On approval the portal drives Prism Central: clone the
   golden VM image (Linux + Omnigent + harness CLIs, systemd unit for
   `omnigent host` disabled), boot, then via cloud-init/SSH:
   - write `~/.omnigent/auth_tokens.json` (0600) with
     `{token, user_id, expires_at, refresh_token}` keyed by the server URL —
     the file format `omnigent/cli_auth.py` documents;
   - enable the systemd unit running
     `omnigent host https://omnigent.<org>` .
5. **Register.** The daemon dials `WS /v1/hosts/{host_id}/tunnel` with the
   bearer. The tunnel path is on the delegated allowlist; the token's `sub`
   is the requesting user, so the host row binds to them automatically —
   ownership needs no extra configuration. The host appears in *their* host
   picker and nobody else's.
6. **Run.** Sessions launch runners on the host; runner traffic
   (`/v1/runners/…`, `/v1/sessions/…`) is inside the allowlist. The daemon's
   background refresh keeps the hourly bearer current across rotations
   (`omnigent/host/connect.py` retains a refreshable auth context).
7. **Lifecycle.**
   - *Re-consent*: at the grant lifetime (30 d default) refresh stops and
     the host drops offline. Choose one: raise
     `OMNIGENT_GRANT_MAX_LIFETIME_DAYS`; have the portal nudge the user to
     re-consent monthly and rewrite the token file; or (OIDC deployments)
     use login grants.
   - *Offboarding*: portal calls `POST /oauth/revoke` (and powers the VM
     down). The revocation check kills API access immediately; the tunnel
     dies at its next reconnect.
   - *Idle sleep* (later, out of band): AHV powers the VM down; Omnigent
     shows the host offline and reconnects when it returns. Wake-on-message
     requires the AHV sandbox provider (parity plan W1) — explicitly not MVP.

---

## 3. Keeping it additive (the anti-fork strategy)

Ranked from zero-touch to small-upstream-PR:

1. **Zero Omnigent changes (true MVP).** The button lives in the portal, not
   Omnigent; users reach it by URL (announce it in the host picker's
   "connect a host" copy via ops docs). The whole flow above works today with
   two env vars flipped (`OMNIGENT_DEVICE_GRANT_ENABLED`, optionally
   `OMNIGENT_DEVICE_CLIENT_SECRET`) — configuration, not code.
2. **Additive server routes, no fork.** `create_app()` accepts
   `extra_routers`; deployments already ship custom entrypoints (the
   Databricks app's `src/app.py` is the precedent). A ~20-line wrapper
   entrypoint can mount `POST /v1/hosts/requests` (proxying to the portal)
   without touching upstream files. The `debug_router_modules` config key
   does the same purely from YAML — it works, but is documented as a
   diagnostics seam ("production config leaves this unset"), so treat it as
   MVP-only.
3. **Small generic PRs upstream** — the real long-term anti-fork move is to
   contribute seams, keeping org logic in our repo:
   - an operator-configurable "Request a host" link (e.g.
     `hosts: {request_url: …}` in server config, surfaced via `GET /v1/info`
     and rendered as a button in the host picker / NewChatDialog);
   - a non-interactive token install: `omnigent login <url> --with-refresh-token`
     (today the portal must write `auth_tokens.json` directly — an internal
     format, so pin the Omnigent version baked into the golden image until
     this exists);
   - `/v1/me` on the delegated allowlist (or 401-tolerant probes): several
     CLI paths probe `GET /v1/me`, which a scoped token cannot reach; the
     daemon itself makes no REST calls, but verify the `omnigent host`
     startup path end-to-end with a scoped token and upstream whichever fix
     is cleanest;
   - later, the **AHV sandbox provider** via the `omnigent.sandbox_providers`
     entry point — already a first-class plugin surface designed for
     out-of-tree providers, i.e. in-band provisioning with no fork either.
4. **Package the org system like `integrations/slack/`**: separate repo or
   `integrations/` package, own release cadence, reusing the Slack
   integration's oauth/token-store code shape. Omnigent upgrades then never
   collide with our code.

## 4. Open items to verify during implementation

- Run `omnigent host` against a real server with a *scoped* delegated token
  (not a login grant) and confirm no startup path 401s on a non-allowlisted
  endpoint (`/v1/me` probes are the suspects).
- Decide the auth mode early (accounts vs OIDC) — it changes the portal's
  flow and the authority of the token stored on VMs (§1 matrix).
- Confirm `auth_tokens.json` schema against the pinned Omnigent version in
  the golden image; add an integration test that boots the image against a
  staging server.
- Grant-expiry observability: the portal should track grant `expires_at` and
  alert before hosts silently drop offline.
