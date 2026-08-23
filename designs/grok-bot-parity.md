# Grok Bot parity analysis: what Omnigent needs to become an open-source Grok Bot

Research-only analysis (no implementation). Maps xAI's Grok Bot feature set onto
the Omnigent codebase and lists every feature we would need to add — with a
concrete implementation path for each — to run an open-source equivalent.

**Operating assumptions** (given for this analysis):

1. Every user gets their own dedicated external (cloud) host; the desktop app
   connects to it through a shared Omnigent server.
2. Local hosts may be disabled entirely, so all execution happens on each
   user's personal cloud server.
3. The multi-harness layer is the model strategy: a user's own Claude or
   ChatGPT subscription, an API key, or any gateway — never locked to one
   vendor the way Grok Bot is locked to Grok 4.6.

---

## 1. What Grok Bot actually is

Launched August 11, 2026 by xAI (SpaceXAI), built by Cursor (codename "Sand")
on Cursor's infrastructure. Desktop app (macOS, Windows, Linux) + iOS; no web
app; Android "coming". Access bundled into SuperGrok Heavy ($300/mo), Cursor
Ultra ($200/mo), Cursor Teams Premium ($120/seat/mo), with a shared weekly
usage pool plus on-demand overage.

The product model, distilled:

| # | Grok Bot concept | What it does |
|---|---|---|
| F1 | **Named bots (roster)** | Persistent named agents, each with a role/"job description", its own conversation history and memory. Created from a home screen; synced across devices. |
| F2 | **Persistent cloud computer** | An account-scoped, always-on Linux VM with a browser, filesystem (Finder-like manager), and terminal. Files, browser sessions/logins, preferences, and memory persist. Bots keep working with the laptop closed. |
| F3 | **Computer use** | Bots drive the browser like a human (navigate/click/type), so *any* web tool works without an API — plus terminal commands and file ops. |
| F4 | **Live view + takeover** | Watch the bot's screen in a 1:1 chat; the bot hands the user control for passwords, passkeys, 2FA, CAPTCHAs, and payment confirmations, then takes the computer back. |
| F5 | **Connectors ("Plugins")** | ~31 built-in integrations (Gmail, Google Calendar/Drive, OneDrive, Outlook, Teams, SharePoint, Salesforce, Notion, GitHub, Linear, HubSpot, …) plus **bring-your-own MCP server**; an in-app plugin/skill discovery surface. |
| F6 | **Skills** | Reusable instruction sets — "the how" of a task. A community marketplace ecosystem is forming around them. |
| F7 | **Routines** | "The when": recurring runs on a schedule (hourly/daily/custom) or on an event (e.g. an email arrives), executing in the cloud while the user is away. |
| F8 | **Teach a task** | Record a live browser demonstration (≤10 min); the bot observes and drafts a reusable skill from it. |
| F9 | **Group chats / bot-to-bot** | Bots message each other, share context in threads and group chats, delegate, and even create other bots; inter-agent history is stored. |
| F10 | **Approvals** | Auto-Review rules per action class (Require Approval / Always Allow), proposed operation + inputs rendered for review, approve-once/deny, a permission-escalation posture (read → write → execute). Bots finish jobs end-to-end and only surface for approvals. |
| F11 | **Notifications** | Push notifications when a bot has a result, question, or approval request; per-bot settings; approve from the phone. |
| F12 | **Voice + attachments** | Dictate requests; attach files; bots produce documents, decks, spreadsheets. |
| F13 | **Memory** | Per-bot stable preferences, role context, and summaries of prior work; learns the user's preferences over time. |

Known weaknesses of the early beta (our differentiation openings): no sandbox
("test" runs do real work), no audit view yet, no per-product spend cap, all
bots share one computer *and its logins*, no web app, model-locked, $120–300/mo.

## 2. Where Omnigent already is

Omnigent's architecture is unusually well shaped for this product:

- **Server / host / runner split with outbound-only tunnels.** A managed cloud
  sandbox host and a laptop host are *the same object* downstream — same
  `hosts` row, same tunnel, same launch frames (`omnigent/server/managed_hosts.py`).
  Ten sandbox providers (Modal, Daytona, Blaxel, E2B, Islo, CoreWeave,
  OpenShell, Boxlite, Kubernetes, Lakebox). Durable host identity across
  sandbox relaunches; wake-on-message (`resume_managed_host`) already wired
  into the send path.
- **26 harnesses** across SDK, subprocess, ACP, and native-TUI tracks (Claude
  Code, Codex, Cursor, Copilot, Pi, OpenCode, Hermes, Kimi, Qwen, Goose, Kiro,
  Antigravity, Devin, and — notably — xAI's own `grok agent stdio` as an ACP
  row). Credential kinds cover API keys, **Claude Pro/Max and ChatGPT
  subscriptions**, OpenAI-compatible gateways, local models, Databricks,
  Bedrock. Per-user BYO credentials *already work* when each user has their
  own host, which is exactly the assumed deployment.
- **Multi-user auth** (accounts, OIDC, header/proxy), per-session ACL sharing,
  presence, co-driving, forking; RFC 8628 device-grant delegated auth.
- **A real governance layer Grok Bot lacks**: ~24 built-in policies (approval
  gates, blast-radius, spend caps incl. per-user daily budgets, PII, CEL),
  bwrap/seatbelt sandboxing with an L7 egress allow-list and a **secretless
  credential proxy**, policy enforcement centralized even over MCP calls.
- **Product surface**: web SPA served by the server; thin Electron/iOS/Android
  shells around it; approvals inbox; RRULE "Automations" with agent-callable
  scheduling tools; Slack integration with in-Slack approvals; server-side
  streaming dictation; file manager/viewer/editors; terminals; sub-agent graph;
  multi-agent orchestration proven by the bundled `polly` example.

## 3. Capability match-up

| Grok Bot feature | Omnigent today | Verdict |
|---|---|---|
| F1 Named bots | Agent specs + sessions exist; no persistent "bot" identity with its own standing thread/memory | **Product-layer gap** |
| F2 Persistent cloud computer | Managed hosts are per-*session*, torn down on delete; no idle reaper; workspace persistence only on Islo/K8s | **Core gap, narrow build** |
| F3 Computer use | `browser_*` tools exist but execute only in the Electron desktop pane; no headless/server browser | **Biggest technical gap** |
| F4 Live view + takeover | Electron browser pane + terminal takeover exist; nothing for a cloud-side browser | Follows F3 |
| F5 Connectors | Slack only; MCP support strong but per-agent-bundle, no OAuth, no user-level registry, no catalog UI | **Large gap, MCP-shaped** |
| F6 Skills | Claude-compatible SKILL.md skills, three sources, `/` menu | Mostly there; no authoring UI or registry |
| F7 Routines | RRULE Automations (≥hourly, must recur, needs online host, no events) | **Medium gap**: one-shot, events, wake-on-fire |
| F8 Teach a task | Nothing | Late-phase build on F3 |
| F9 Group chats | `sys_session_send`/inbox/subagents are strong; no user-facing multi-bot group chat | Product-layer gap |
| F10 Approvals | Policy ASK + approval cards + approve-by-link + Slack approvals + "don't ask again" | **Near parity already** |
| F11 Push notifications | None (local-only, app must be open) | **Hard requirement, standard build** |
| F12 Voice + files | Dictation (Web Speech + server sherpa-onnx path), attachments, file manager | Mostly there |
| F13 Memory | Hindsight SaaS extra only; prefs are localStorage | **Gap**: built-in memory backend |

## 4. The feature list, with implementation paths

Ordered as workstreams. Effort figures are rough engineer-weeks (ew) for a
first shippable cut.

### W1 — Dedicated per-user cloud computer (foundation) · ~6–10 ew

The host-architecture seams make this a narrow generalization of what exists:

1. **Per-user host grain.** Today `launch_managed_host` mints a fresh host per
   *session* and `terminate_managed_host` fires on session delete. Add a
   `resolve_or_launch_dedicated_host(user_id)` that reuses the existing
   launch/resume machinery, keyed on a reserved host name per user (the
   `UniqueConstraint(workspace_id, user_id, name)` already supports it), and
   exempt dedicated hosts from session-delete teardown.
2. **Provisioning trigger outside session-create** — on first login /
   `/v1/me`, or an explicit `POST /v1/hosts` (which doesn't exist today), so
   the computer exists before the first chat.
3. **Idle sleep + wake.** The wake half (`resume_managed_host`, single-flight,
   re-arms tokens, surfaces "asleep" in the UI) exists; build the sleep half —
   a server-side reaper keyed on `runner_last_seen`/last turn. Watch the wake
   latency budget (today's 120 s online timeout is a product problem for an
   every-morning wake).
4. **Persistent workspace.** Only Islo (pause/resume + volume) and Kubernetes
   (operator PVCs) survive a stop today; K8s `$HOME` is an `emptyDir` and its
   `resume()` is delete-and-recreate. Standardize on one or two providers with
   a **per-user persistent volume** (Kubernetes + per-user PVC is the
   best-shaped OSS default; Islo the turnkey one). Modal's 24 h cap rules it
   out for "your computer in the cloud".
5. **Disable local hosts.** No switch exists. Add a server-config key
   (`hosts: {mode: dedicated|any|external_only}`) advertised via `GET
   /v1/info`, enforced at the tunnel-registration route, `GET /v1/hosts`,
   `POST /v1/hosts/{id}/runners`, and the session-create validator, and read
   by the SPA picker + the desktop `host-control` IPC.
6. **Host lifecycle admin**: delete/revoke/rotate APIs (today `delete_host`
   has exactly one caller), per-user host quotas, and fixing the
   provision-leak on server shutdown.

### W2 — BYO models per user (multi-harness, not Grok-locked) · ~2–5 ew

Under the one-host-per-user assumption this **largely already works**: each
user's host owns its `~/.omnigent/config.yaml`, OS keychain/secrets file, and
vendor CLI logins, and the harness layer already supports Claude/ChatGPT
subscriptions, API keys, and gateways. What's left:

1. **Web-first credential onboarding.** The UI credential write path
   (`POST /v1/hosts/{id}/harnesses/{h}/credential`) covers only
   anthropic/openai `key`/`gateway`/`adopt`. Extend it to subscriptions and
   the remaining providers, and drive the vendor **device-code OAuth flows**
   (`claude setup-token`, `codex login`) from the browser — both are
   device-flow shaped, and `CLAUDE_CODE_OAUTH_TOKEN` / `CODEX_ACCESS_TOKEN`
   are already in `HARNESS_CREDENTIAL_ENV_VARS`, sidestepping keychain files.
2. **First-class xAI family** (optional but cheap): today Grok is an ACP CLI
   row with vendor OAuth and `XAI_API_KEY` is catalog-only; xAI's API is
   OpenAI-compatible, so adding it as a first-class key/gateway family is
   small.
3. **Default assistant agent.** Ship a bundled "assistant" agent spec (the way
   `polly` ships for coding) whose harness/model resolve from whatever the
   user connected — so the product works identically on a Claude sub, a
   ChatGPT sub, or an OpenRouter key.

Skip (not needed under per-user hosts): the shared-host per-user credential
lane (per-user `OMNIGENT_CONFIG_HOME`, launch-frame credential payloads) —
that's the 3–6 week alternative only if hosts are ever shared.

### W3 — Named bot roster · ~4–6 ew

A mostly product-layer construct over existing primitives:

1. **`Bot` entity** = (name, avatar, role/job description → agent-spec
   instructions, harness/model binding, memory namespace, notification prefs,
   standing 1:1 thread). Back it with the existing `agents` table + a new
   per-user bots table; a bot's "chat" is a long-lived session (or session
   series with auto-continuation) bound to the user's dedicated host.
2. **Home surface**: a bot roster screen ("+ New Agent"), replacing the
   session-centric landing when the deployment is in assistant mode (feature
   flag). Sessions/projects UI remains underneath for power users.
3. **Bot-scoped context**: pin the bot's role instructions into the prompt via
   the existing spec pipeline; per-bot memory bank (W6); per-bot notification
   settings (W5).

### W4 — Cloud computer use: headless browser + live view + takeover · ~8–14 ew

The biggest technical gap. Today `browser_navigate/snapshot/click/type/
screenshot` are schema-only tools whose execution is claimed by the Electron
renderer's `WebContentsView`; with no desktop app attached they time out.
Grok Bot's defining capability is a browser that lives *on the cloud computer*.

Path:

1. **Headless executor on the host.** Implement the existing `browser_*`
   contract in the runner against **Playwright/Chromium** running on the
   user's host (the tool schemas are already Playwright-MCP-shaped:
   accessibility-tree snapshots with `[ref=N]` handles). Keep the Electron
   pane as one of two executors: renderer claims it if present, host executes
   headlessly otherwise — the claim/CAS protocol already exists.
2. **Persistent browser profile** on the host volume (cookies/logins survive,
   per-user by construction — an isolation win over Grok Bot's shared-logins
   design).
3. **Live view**: stream the headless browser into the existing
   `BrowserPane` via CDP `Page.screencast` frames (or WebRTC later) over the
   established runner-tunnel WS channels; render read-only in web/desktop/iOS.
4. **Takeover**: forward user input events from the pane to CDP
   (`Input.dispatch*`) while pausing the agent — the same posture as the
   existing shared-terminal takeover. This delivers Grok Bot's
   password/2FA/CAPTCHA/payment handoff.
5. **Optional full-desktop tier** later: Xvfb + a VNC bridge (x11vnc/KasmVNC +
   noVNC) for non-browser GUI apps, following the Anthropic computer-use
   reference container layout.

Open-source fills: **Playwright** (+ `microsoft/playwright-mcp` as a
reference/starting point), **browser-use** (MIT) if we want a higher-level
browser-agent loop rather than raw tools, Anthropic's computer-use reference
implementation, KasmVNC/noVNC for the desktop tier. Note harnesses can also
bring their own (Claude Code + Playwright MCP) — but a first-party executor is
what makes it uniform across harnesses and policy-governed.

### W5 — Push notifications · ~3–5 ew

Absent entirely (no APNs/FCM/Web Push; today's notifications are client-local
while the app is open) and non-negotiable for "bot works while you sleep":

1. Server-side notification service + device-token registry, fed by the
   events that already exist (turn end, elicitation/approval request, produced
   file, scheduled-run completion).
2. **Web Push** (VAPID) for browser + Electron; **APNs** for the existing iOS
   shell (README lists it as the known gap); **FCM** for Android.
3. Deep-link taps into the session/approval (the `omnigent://` scheme and
   approve-by-link page already exist). Per-bot notification prefs live on the
   Bot entity.
4. Self-hosters without Apple/Google credentials: optional **ntfy** backend.

### W6 — Built-in memory · ~3–5 ew

1. A first-party memory backend implementing the existing
   `hindsight_retain/recall/reflect` tool contract locally (SQLite FTS +
   optional pgvector embeddings), so agent-facing behavior is unchanged and
   the SaaS extra becomes one of two backends. Per-bot `bank_id` is already
   the resolution default.
2. Rolling "summary of prior work" per bot: compaction summaries already
   exist per session; persist them into the bot's bank at session end.
3. Move user preferences server-side (they're localStorage today) so they
   sync across the user's devices.

Open-source fills if we'd rather adopt than build: **mem0** (Apache-2.0) or
**Letta/MemGPT** as the memory engine behind the same three tools.

### W7 — Connectors and a plugin surface · ~6–10 ew (framework) + per-connector

MCP-first, matching Grok Bot's own architecture (its connectors are hosted
MCP + BYO MCP):

1. **User-level connector registry.** Today MCP servers are per-agent-bundle
   (plus session-scoped CRUD). Add per-user connector config stored
   server-side, materialized into every bot session on the user's host.
2. **MCP OAuth 2.1** support (missing today — static headers only): the
   authorization-code flow, token storage per user (encrypted, or on the
   user's host to keep the server secretless), refresh. This unlocks the
   entire hosted-MCP ecosystem (GitHub, Notion, Linear, Sentry, …) with no
   per-connector code.
3. **Curated catalog UI** ("Plugins"): a vetted list of MCP servers — official
   GitHub/Notion/Linear/Slack servers, Google Workspace MCP (community
   servers cover Gmail/Calendar/Drive), Microsoft Graph MCP for
   Outlook/Teams/SharePoint/OneDrive, Salesforce/HubSpot community servers —
   with one-click connect. Stdio servers install onto the user's host (and
   should get the sandbox treatment; stdio MCP is un-sandboxed today, an
   acknowledged hole).
4. **BYO MCP** already works; keep it, add it to the catalog UI.
5. **Channels**: the existing Slack integration is the template; for
   WhatsApp/Telegram/etc., bridge through **OpenClaw** (already integrable via
   ACP and documented in `docs/openclaw.md`) rather than building channel
   adapters.

Open-source fills: the MCP server ecosystem itself; **Nango** (ELv2) or
**Arcade** if we want a managed-OAuth broker rather than implementing MCP
OAuth; Composio is the commercial benchmark to track.

### W8 — Routines upgrade (schedules + events) · ~4–6 ew

The RRULE Automations base is solid (DB-backed scheduler, per-run cost caps,
agent-callable `sys_scheduled_task_*`, run history, UI). Gaps vs Grok Bot:

1. **One-shot and sub-hourly runs** (today: must fire ≥2×, ≥hourly).
2. **Wake-on-fire**: firing requires an online host today; wire the scheduler
   through `resume_managed_host` so a routine wakes the user's sleeping
   computer (the resume path exists — this is plumbing).
3. **Event triggers**: add a trigger type beyond RRULE — (a) an authenticated
   inbound **webhook endpoint** per task, and (b) **connector events** built
   on it (e.g. Gmail watch/Pub/Sub push, GitHub webhooks, or a short-interval
   poll fallback executed as a cheap host-side check).
4. **Skill-linked routines**: let a routine reference a skill + bot (the
   Grok Bot "skill is the how, routine is the when" model) — thin metadata on
   top of the existing prompt field.
5. Reliability niceties: missed-fire replay policy, retries, and a per-routine
   pause switch (partially present).

### W9 — Group chats and bot-to-bot collaboration · ~4–8 ew

The hard part (fan-out, inbox wake-ups, cross-vendor sub-agents, subagent
panel/graph) exists; what's missing is the user-facing surface:

1. A **group session type**: multiple bot participants + the user;
   `@mention` routing decides which bot takes the turn; bots address each
   other through the existing `sys_session_send`/inbox lane with transcripts
   folded into the group thread.
2. **Delegation affordances**: "pass ownership" = re-binding the active
   responder; a bot creating another bot = `sys_session_create` gated by the
   existing `spawn_bounds`/purpose-guard policies (already built).
3. Live computer view per working bot inside the group (Grok Bot lacks this —
   cheap differentiation once W4 lands).

### W10 — Skills authoring, sharing, and (later) teach-a-task · ~3–5 ew + ~6–10 ew

1. **Authoring UI**: create/edit SKILL.md from the web app (the Monaco/TipTap
   editors and the `onboarding-buddy` agent that scaffolds specs already
   exist — expose them); per-bot skill assignment.
2. **Distribution**: skills are the one extension type with no registry story
   (harnesses, sandbox providers, and policies all have one). Add a simple
   `GET /v1/skills` + import-from-git/URL, and lean on Claude-Code
   compatibility (we already parse `~/.claude/skills` and plugin
   marketplaces) to inherit the existing community skill ecosystem.
3. **Teach a task** (after W4): record a takeover/demonstration segment as
   CDP input events + periodic snapshots, then have the agent draft a
   SKILL.md from the recording for user review. This is Grok Bot's most
   novel feature; everything it needs (live view, takeover, skills) is
   produced by earlier workstreams.

### W11 — Assistant-mode product polish · ~4–8 ew

- **Onboarding**: account → computer provisions (W1) → connect a model (W2) →
  name your first bot (W3) → grant notifications (W5). All existing dialogs
  are coding-centric today.
- **Files**: the manager/viewer stack is strong; add drag-drop upload into the
  Files panel, folder/zip download, and fix iOS blob downloads (#969).
- **Mobile**: Android release path (signing + store), iOS APNs (W5),
  Electron auto-update CI (currently manual).
- **Deliverables**: document/deck/spreadsheet generation via harness skills
  (docx/pptx/xlsx skills already exist in the ecosystem) rather than new code.
- **Usage/limits**: per-user budgets exist (`user_daily_cost`, cost policies);
  add operator-facing quota pools if the deployment meters compute.
- Optional: TTS (explicit non-goal today) — only if voice replies matter.

## 5. What we already do better (keep and market)

- **Real sandboxing + egress control + secretless credentials** — Grok Bot has
  no sandbox and its bots share one login surface. Per-user hosts + bwrap/
  seatbelt + the credential proxy is a materially safer design.
- **Policy engine** (spend caps, blast radius, approval rules at server/agent/
  session scope) vs. Grok Bot's "audit view coming soon".
- **Any model, any harness** — including using Grok itself via the existing
  `grok` ACP row, next to Claude/ChatGPT subscriptions.
- **A real web app**, session sharing/co-driving, forking, and self-hosting.

## 6. Suggested phasing

| Phase | Workstreams | Outcome |
|---|---|---|
| 0 — Foundation | W1, W2, W5 | Every user has a persistent, wake-able cloud computer reachable from desktop/mobile, on their own model credentials, with push notifications. |
| 1 — Assistant core | W3, W6, W8, W7 (registry + OAuth + first 5 connectors) | Named bots with memory, routines that run while you sleep, Gmail/Calendar/GitHub/Notion/Slack-class connectors. |
| 2 — Computer use | W4, W7 (catalog buildout), W9 | Bots that drive a browser on their computer with live view + takeover; group chats. |
| 3 — Differentiation | W10 (incl. teach-a-task), W11, audit surface | Skill authoring/sharing, demonstration learning, polished assistant-mode apps. |

Total rough order of magnitude: ~45–80 engineer-weeks to Grok Bot parity, of
which the genuinely novel engineering is W4 (headless computer use with live
view/takeover) and W8.3 (event triggers); most of the rest is deliberate
productization of seams the codebase already has.
