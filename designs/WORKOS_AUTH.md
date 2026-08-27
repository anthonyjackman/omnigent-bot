# WorkOS integration for Omnigent

Research-only design doc. Evaluates Omnigent's authentication-provider surface
as it stands today, and lays out the minimal-change path to bind each
tenant's dedicated Omnigent instance to a WorkOS organization in Expedient's
single WorkOS environment (shared with the rest of the AI suite). Companion
to `designs/GLOSSARY.md`, `designs/HOST_REQUEST_FLOW.md`,
`designs/grok-bot-parity.md`.

Deployment model assumed: **one Omnigent server per client**, per-user hosts
under it, all clients authenticating against **one Expedient-owned WorkOS
instance** with **one WorkOS organization per client**.

---

## 1. What Omnigent ships today (the honest answer: vanilla, but good vanilla)

**There are no bundled "easy button" providers.** Grepping the entire repo:
zero WorkOS, Auth0, Okta, Keycloak, Cognito, Clerk, Entra, or
Google-specific auth code. Those names appear only as prose in deploy
READMEs. The one special-cased provider is **GitHub OAuth**
(`omnigent/server/oidc.py:118-123,334-356`) — and it exists only because
GitHub isn't OIDC. Everything else is a single generic OIDC implementation,
selected by env var. There is no "Sign in with X" button anywhere: in OIDC
mode the SPA never mounts a login page at all — an unauthenticated
`GET /v1/me` returns `login_url`, the client does a bare redirect to
`/auth/login`, and the server 302s to the IdP.

Three auth modes (`omnigent/server/auth.py:328` `resolve_auth_source()`):

| Mode | Selected when | Identity source |
|---|---|---|
| `header` | auth disabled, or pinned | Trusted proxy header (`X-Forwarded-Email`) |
| `accounts` | `OMNIGENT_AUTH_ENABLED=1`, no issuer | Built-in username/password, invite-only |
| `oidc` | `OMNIGENT_AUTH_ENABLED=1` + `OMNIGENT_OIDC_ISSUER` | Generic OIDC (the WorkOS path) |

### What OIDC mode actually implements (`omnigent/server/routes/auth.py`)

Present: discovery via `{issuer}/.well-known/openid-configuration` (fetched
once at startup, hard-fails if unreachable); authorization-code flow with
**PKCE S256**; confidential client only (`client_secret_post`); RS256/ES256
`id_token` verification against the JWKS (`aud` = client id, `iss` = exact
issuer string); CSRF `state` in a signed 5-minute cookie; sessions as an
HS256 JWT cookie (`__Host-ap_session`, default TTL 8 h, no sliding renewal);
domain allowlist admission; the **cli-login ticket flow** (`/auth/cli-login`
+ `/auth/cli-poll`) that the CLI, Electron, iOS, and Android shells all use,
which also hands the CLI a 30-day **login-grant refresh token**; and the
`/oauth/token` + `/oauth/revoke` refresh machinery.

Identity mapping: `user_id` = the **`email` claim, lowercased** — nothing
else. The IdP `sub` is never used; no fallback chain. `email_verified` must
be truthy unless `OMNIGENT_OIDC_SKIP_EMAIL_VERIFICATION=1`;
`OMNIGENT_OIDC_EMAIL_CLAIM` can rename the claim. **No other claim is
consumed** — no groups, roles, name, or org. `is_admin` comes solely from
the `admins:` config key / `<data_dir>/admins` file (additive, never
demotes); the code says outright this is OIDC mode's only admin signal.

Full env inventory (all config is env-only; there is no `auth:` YAML
section): `OMNIGENT_OIDC_ISSUER`, `_CLIENT_ID`, `_CLIENT_SECRET`,
`_REDIRECT_URI` (or `OMNIGENT_DOMAIN`), `_COOKIE_SECRET` (≥64 hex chars),
`_SCOPES` (default `openid email profile`), `_SESSION_TTL_HOURS` (8),
`_LOGOUT_REDIRECT_URI` (static), `_ALLOWED_DOMAINS` (+ `_PATH`),
`_ALLOW_INVITES`, `_SKIP_EMAIL_VERIFICATION`, `_EMAIL_CLAIM`; plus
`admins:` / `allowed_domains:` in the server config YAML.

### What is NOT there (the gap list)

1. **No extra authorization-request parameters.** The auth request is a
   fixed 7-key literal (`routes/auth.py:195-203`) — nothing can send
   WorkOS's `organization_id`, `login_hint`, `screen_hint`, etc.
2. **No claim-based admission.** Authorization is domain-suffix-only
   (`oidc_access.py:109-127`); an empty allowlist admits *any* user the IdP
   returns. No way to require `org_id == org_XXX`.
3. **No `nonce`** generated or checked (state/PKCE still cover CSRF/code
   injection; nonce is defense-in-depth).
4. **No RP-initiated logout** — logout clears the cookie and redirects to a
   static URL; no `end_session_endpoint`/`id_token_hint` (the id_token is
   discarded after the callback).
5. **IdP refresh tokens ignored**; JWKS re-fetched once per login (no
   caching client).
6. **No SSO branding surface** — `/v1/info` doesn't even expose the auth
   mode; no login page, no provider label.
7. **CLI-login tickets are per-process memory** — multi-replica servers need
   sticky sessions (our per-tenant instances will run 1 replica; non-issue
   for MVP).
8. **Email is the primary key.** A WorkOS email change = a new Omnigent
   identity (`omnigent/server/identity_migration.py` exists to remap).

### Extension seams that matter

`create_app(auth_provider=…)` accepts a custom provider, but multiple
`isinstance(auth_provider, UnifiedAuthProvider)` gates mean a from-scratch
provider loses the auth routes, device-grant store, and `/oauth/*` — the
viable out-of-tree pattern is **subclassing `UnifiedAuthProvider`** with
`_source = "oidc"`. There is no auth plugin registry. Given how small the
real gaps are, patching upstream generically beats maintaining a subclass.

---

## 2. The WorkOS side

Relevant constructs in one Expedient **WorkOS environment**:

- **AuthKit** is the hosted auth layer over the shared user pool, and it is
  a **standard OIDC provider**: discovery lives at
  `https://api.workos.com/user_management/{client_id}/.well-known/openid-configuration`
  (or under a custom auth domain, e.g. `auth.expedient.<tld>`). Standard
  authorization-code + PKCE, confidential clients, RS256 id_tokens with the
  profile/email claims when those scopes are requested.
- **Organizations** — one per client, exactly as planned. Per-organization
  authentication policies (SSO required, MFA, domain rules) live in WorkOS.
- **`organization_id` authorization parameter** — WorkOS's documented
  mechanism for a "single-workspace" app: passed on the authorization
  request, it pins the login to one org and skips the org picker. This is
  the natural binding for a dedicated per-client instance — and it is
  precisely gap #1 above.
- **`org_id` / `role` / `permissions` claims** — WorkOS access tokens carry
  them; org context is available to the relying app. Enforcing
  `org_id == <client's org>` server-side is gap #2.
- **WorkOS Connect ("OAuth Applications")** — lets the Expedient AuthKit
  user pool act as an OAuth/OIDC *provider* to registered applications, each
  with its **own client_id + up to 5 non-expiring secrets** and OIDC
  discovery metadata. The selected organization surfaces as the `org_id`
  claim in issued tokens.

### Two binding models (the "application or whatever the correct construct" question)

**Model A — each instance is a direct AuthKit client.** All Omnigent
instances share the environment's client_id; each registers its own redirect
URI (`https://omnigent.<client>.expedient.<tld>/auth/callback`); org
dedication comes from sending `organization_id` on the auth request plus
validating the `org_id` claim. Fastest; no new WorkOS objects; but all
instances share one credential and the org pinning depends on the two small
Omnigent patches.

**Model B — one WorkOS Connect OAuth application per Omnigent instance
(recommended).** Each client's instance gets its own client_id/secret and
its own OIDC discovery endpoint, all backed by the same Expedient user pool
and orgs — the same shape the rest of the AI suite's products would use.
Per-tenant credential isolation and revocation fall out for free; a
compromised instance never holds a credential that spans clients.

Console items to verify for Model B (the team has WorkOS access; these are
configuration questions, not blockers): that a Connect app's **id_token
carries `email` + `email_verified`** (Omnigent keys on the email claim —
`OMNIGENT_OIDC_EMAIL_CLAIM`/`_SKIP_EMAIL_VERIFICATION` are the fallbacks),
the **exact `iss` string** its discovery document declares (Omnigent
compares exactly, no trailing-slash normalization), and whether an app can
be **restricted to a single organization** in the dashboard (if yes, gap #1
becomes optional for Model B and org enforcement is WorkOS-side).

---

## 3. Config-only pilot (zero code changes)

Works today against AuthKit, per instance:

```dotenv
OMNIGENT_AUTH_ENABLED=1
OMNIGENT_OIDC_ISSUER=<issuer from the discovery document — verify exact string>
OMNIGENT_OIDC_CLIENT_ID=client_...
OMNIGENT_OIDC_CLIENT_SECRET=sk_...
OMNIGENT_OIDC_REDIRECT_URI=https://omnigent.<client>.expedient.<tld>/auth/callback
OMNIGENT_OIDC_COOKIE_SECRET=<openssl rand -hex 32>
OMNIGENT_OIDC_ALLOWED_DOMAINS=<client's email domains>   # empty = admit ANY WorkOS user
```
```yaml
# <data_dir>/config.yaml
admins: [<client admin emails>]        # the only admin mechanism under OIDC
allowed_domains: [<client domains>]
```

HTTPS is required (the `__Host-` cookie prefix). In this pilot the only
org guard is the domain allowlist — acceptable for a first tenant, not for
GA (a WorkOS user from another org with a matching email domain would be
admitted; multi-domain clients are awkward; contractors on foreign domains
fail closed).

## 4. The work plan (minimal, additive, upstream-first)

### Phase 1 — two small generic-OIDC PRs upstream (~1–2 weeks incl. tests)

Both land in two files (`omnigent/server/oidc.py::OIDCConfig.from_env`,
`omnigent/server/routes/auth.py`), and neither mentions WorkOS — they are
generic improvements consistent with the codebase's posture (GitHub is the
only branded branch, and only because it isn't OIDC):

1. **`OMNIGENT_OIDC_EXTRA_AUTH_PARAMS`** (`k=v,k=v`) merged into the
   authorization request (`routes/auth.py:195-203`, ~20 LOC). Carries
   `organization_id=org_XXX` for Model A; also unlocks `login_hint`,
   `prompt`, Auth0 `connection`, etc. for other users of Omnigent.
2. **Claim-based admission**: e.g. `OMNIGENT_OIDC_REQUIRED_CLAIMS`
   (`org_id=org_XXX`), enforced in `OidcAdmissionPolicy`
   (`oidc_access.py:69-127`); requires `_resolve_oidc_email`
   (`routes/auth.py:800-914`) to return the verified claims dict instead of
   just the email (~40 LOC, one signature change). This is the real tenant
   boundary — the instance rejects any login whose token isn't from the
   client's org, regardless of email domain.

Worth adding to the same PR series while we're there (small, uncontroversial):
`nonce` (~10 LOC), a cached `PyJWKClient` (hoist one line), and an
`auth_source`/SSO-label field in `GET /v1/info` so a branded "Continue with
Expedient ID" page becomes possible later (purely cosmetic — the redirect
flow already works headlessly).

### Phase 2 — optional hardening (as needed)

RP-initiated logout (needs the id_token retained and
`end_session_endpoint` read from discovery); a DB-backed cli-ticket store if
any instance ever runs >1 replica; **stay on email identity** — pair it with
a WorkOS-side policy that emails are stable, and use
`identity_migration.py` for the rare rename, rather than absorbing the
blast radius of switching the PK to `sub`.

### Explicitly rejected: header mode behind oauth2-proxy

Fronting instances with an OIDC proxy and running Omnigent in `header` mode
would be zero-code — but header mode never mounts `/auth/cli-login`,
`/auth/cli-poll`, `/oauth/token`, or the device-grant machinery
(`app.py:2728,2741-2852`). That kills `omnigent login`, the iOS/Android
shells' entire login path, and the Request-a-Host provisioning flow
(`designs/HOST_REQUEST_FLOW.md`), which depends on login grants. Native OIDC
mode is the only mode compatible with the rest of this program.

### Fit with the rest of the program

Per-tenant dedicated instances mean **no multi-tenant middleware is needed**
— each server stays single-workspace (`workspace_id=0`); the WorkOS org is
enforced at admission, not at the data layer. In OIDC mode the
Request-a-Host portal uses the **cli-login ticket flow** (issuing
full-authority login grants) rather than the accounts-only device-grant
consent page; the portal design already covers this
(`designs/HOST_REQUEST_FLOW.md` §1 matrix). The provisioning portal itself
can authenticate its own UI against the same WorkOS org — one identity
across the suite, as intended.

## 5. Per-tenant provisioning checklist (ops runbook skeleton)

For each new client: create the WorkOS **organization** (SSO/MFA policy per
client contract) → create the instance's **Connect OAuth application**
(Model B) or register its redirect URI (Model A) and mint credentials →
deploy the Omnigent instance with the §3 env block + org claim pin (Phase 1)
→ set `admins:` to the client's admin(s) → smoke-test the four login paths
(web redirect, `omnigent login` ticket flow, desktop, iOS/Android) → hand to
the Request-a-Host portal for host provisioning against the same instance.

## 6. Verification items before committing to the design

- Confirm from the WorkOS dashboard the exact issuer string and the id_token
  claim set (email, email_verified) for both a direct AuthKit client and a
  Connect application; pilot with `OMNIGENT_OIDC_SCOPES="openid email
  profile"` and adjust `_EMAIL_CLAIM` only if needed.
- Confirm whether Connect apps accept/require `organization_id` on the auth
  request or support dashboard-level org restriction (decides whether gap #1
  is needed for Model B at all).
- Exercise the cli-login ticket flow against AuthKit end to end (Safari/
  system-browser hops on iOS/Android; Electron navigates in-place — the
  popup allowlist only affects connector OAuth, not user login).
- Session-length UX: Omnigent's cookie is non-sliding (default 8 h); decide
  the per-client TTL (`OMNIGENT_OIDC_SESSION_TTL_HOURS`) since re-auth is a
  full IdP round trip.
