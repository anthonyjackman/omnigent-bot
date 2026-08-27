# Omnigent contract glossary

Shared vocabulary for the program: replicating Grok Bot on our own
infrastructure (Nutanix AHV), open to any harness/model instead of being locked
to one vendor, while the same deployment keeps serving as a general agentic
coding tool. Companion to `designs/grok-bot-parity.md`.

Terms below are the *codebase's* meanings. Where our colloquial usage differs
(most importantly "server"), the glossary says so explicitly.

---

## 1. The four confusions to settle first

These are the places where casual language and codebase language diverge.
Getting these four right prevents most mistakes.

### 1.1 "Server" vs "host"

- A **server** is the central Omnigent control plane — the FastAPI web
  application (`omnigent server`, `omnigent/server/app.py`). It owns the
  database, auth, the web UI, policies, scheduling, and session records. **No
  agent code ever executes on the server.** There is one (logical) server per
  deployment; everyone shares it.
- A **host** is a machine where agents actually execute — it runs the
  `omnigent host` daemon, dials **out** to the server over a WebSocket tunnel,
  and spawns runners on demand. A laptop, an AHV VM, and a Modal container are
  all hosts.

> When we say "give every user their own server", in Omnigent vocabulary we
> mean **give every user their own host**. The phrase "registers it as a
> server with the central platform" is, precisely: *the machine registers as a
> host with the server.* Reserve the word "server" for the central platform,
> always.

### 1.2 "External host" vs "managed host"

The discriminator is **who provisions it**, not where it runs or who pays for
it. Both are rows in the same `hosts` table and identical to everything
downstream (same tunnel, same launch frames).

- **External host** — a human (or out-of-band automation) installed Omnigent
  on a machine and ran `omnigent host <server-url>` (`hosts.sandbox_provider
  IS NULL`). Our MVP AHV VMs are **external hosts** in Omnigent terms, even
  though from the org's point of view IT "manages" them.
- **Managed host** — the *server itself* provisioned a cloud sandbox via a
  sandbox provider (Modal, Daytona, E2B, Kubernetes, …), started
  `omnigent host` inside it, and armed it with a launch token
  (`omnigent/server/managed_hosts.py`). Today these are per-session and torn
  down on session delete.

> MVP: our AHV VMs are external hosts, provisioned out of band. Later: an AHV
> **sandbox provider** plugin makes them managed hosts, which is what unlocks
> server-driven sleep/wake and one-click "request a computer".

### 1.3 The two things both called "sandbox"

- **Cloud sandbox** (`omnigent/onboarding/sandboxes/`, `sandbox:` server
  config, `omnigent sandbox create`) — a disposable cloud VM/container that
  *hosts an agent*, i.e. the substrate under a managed host.
- **OS sandbox** (`omnigent/sandbox/`, `os_env.sandbox:` in agent YAML) —
  process-level isolation *inside* a runner: bubblewrap on Linux, Seatbelt on
  macOS, Job Objects on Windows, plus the L7 egress allow-list proxy and the
  secretless credential proxy.

In design docs, always write **cloud sandbox** or **OS sandbox**, never bare
"sandbox".

### 1.4 "Agent" vs "harness"

- An **agent** is a configuration: instructions + tools + policies + an
  executor choice, defined in YAML (single-file spec or a bundle directory).
  It's the *what*.
- A **harness** is the engine an agent runs on — Claude Code, Codex, Cursor,
  Pi, OpenCode, etc., wrapped by Omnigent in one of five integration modes.
  It's the *how*.

One agent can be re-pointed at another harness; one harness serves many
agents.

---

## 2. Glossary

### 2.1 Control plane (the shared platform)

| Term | Definition |
|---|---|
| **Server** | The central FastAPI app + web UI + database. Coordination only; never executes agent code. Deployed once per org (Render/Railway/K8s/Docker/…), or run loopback on a laptop for local mode. |
| **Deployment** | One installed server (possibly several replicas) + its database + the hosts registered to it. "The platform" in product language. |
| **Replica** | One server process. Host/runner tunnel registries are per-replica and in-memory; the DB is the cross-replica truth. |
| **Database** | Postgres (production) or SQLite (small/local). Sessions, users, hosts, permissions, policies, scheduled tasks. |
| **Artifact store** | Server-side blob storage (local disk, S3, or Databricks Volumes) for uploaded files and agent bundles. |
| **User / account** | An identity string (email-shaped) in one of three auth modes: `accounts` (built-in username/password, invite-only signup), `oidc` (SSO), `header` (trusted proxy). First user is admin. |
| **Admin** | A user with `is_admin` — manages members, server-wide policies, sharing mode. |
| **Workspace id** | Dormant multi-tenant partition key on every table (default `0`). NOT related to a session's "workspace" directory (see §2.2). Avoid the bare word "workspace" in design docs; say **tenant id** or **workspace directory**. |
| **Delegated token / device grant** | RFC 8628 device-authorization flow (`designs/DEVICE_AUTH.md`) letting an app (e.g. the Slack bot, or our provisioning automation) act as a user against a fail-closed path allowlist. |
| **Branding** | Server-supplied app name/logos; the SPA is white-label-able. |

### 2.2 Compute plane (where agents run)

| Term | Definition |
|---|---|
| **Host** | A machine running the `omnigent host` daemon, registered to a server over the outbound-only **host tunnel** (`WS /v1/hosts/{id}/tunnel`, control frames only). Owned by exactly one user (`hosts.user_id`); a second user's attempt to register the same host id is refused (409). Liveness = heartbeat within 90 s. |
| **External host** | A host provisioned out of band (§1.2). `omnigent login <url>` then `omnigent host <url> --background`. |
| **Managed host** | A host the server provisioned inside a cloud sandbox (§1.2). Durable `host_id`, disposable substrate; can be relaunched or (on capable providers) resumed. |
| **Local mode / local server** | `omnigent host ""` or bare `omnigent start`: a loopback server + this machine as its host, for single-user laptop use. This is the mode we would disable in the hosted product. |
| **Sandbox provider** | A pluggable driver (`omnigent.sandbox_providers` entry point, `docs/extending/sandbox_providers.md`) that can create/terminate — and on some providers resume — cloud sandboxes. Ten exist (Modal, Daytona, Blaxel, E2B, Islo, CoreWeave, OpenShell, Boxlite, Kubernetes, Lakebox). **An AHV provider would be the eleventh.** |
| **Host image** | The container/VM image a managed host boots from (`ghcr.io/omnigent-ai/omnigent-host`), with Omnigent + harness CLIs prebaked. Our AHV VM template plays this role. |
| **Launch token** | Per-launch secret (`X-Omnigent-Host-Token`, stored only as a digest) that lets a managed host authenticate its tunnel without any user credential entering the sandbox. |
| **Runner** | The per-session worker *process* on a host. Owns the harness, the tool loop, terminals, and the workspace directory. Reached from the server through the **runner tunnel** (`WS /v1/runners/{id}/tunnel` — full HTTP-over-WebSocket). Forked from a preloaded **zygote** for fast start. |
| **Workspace (directory)** | The filesystem directory on a host that a session operates in (`omnigent_conversation_metadata.workspace`). A session bound to a host always has one. |
| **Worktree** | A git worktree a host can create so parallel sessions work on isolated checkouts of one repo. |
| **Terminal** | A tmux-backed PTY inside a runner — the agent's own pane plus user-launched ones; attachable from the browser (read-only for non-owners). |
| **OS sandbox** | Process isolation around tools/terminals inside a runner (§1.3): bwrap/Seatbelt/Job Objects, deny-by-default env, egress rules. |
| **Egress proxy / credential proxy** | The L7 allow-list MITM for all HTTP(S) leaving an OS sandbox; the credential proxy injects real secrets on the way out so they never exist inside the sandbox. |

### 2.3 Agent plane (what runs)

| Term | Definition |
|---|---|
| **Agent (spec)** | User-authored YAML defining instructions, executor (harness + model + auth), tools, policies, OS access. Two formats: single-file spec (`docs/AGENT_YAML_SPEC.md`) and bundle directory (`omnigent/spec/AGENTSPEC.md` — config + AGENTS.md + skills/ + tools/ + sub-agents). |
| **Template vs session agent** | `agents` table rows are `template` (registered server-wide, unique name) or `session` (a per-conversation copy that can drift). **Agents have no per-user owner** — a template agent is visible to every user of the deployment. |
| **Harness** | The wrapped engine (§1.4). 26 ids across five **integration modes**: SDK in-process (claude-sdk, openai-agents, cursor, copilot, antigravity), CLI subprocess (codex, pi, kimi, hermes), ACP subprocess (acp:\*, goose, qwen, devin, grok), native TUI (claude-native, codex-native, …), native server (opencode-native). |
| **Native harness** | A resident vendor TUI running in a runner-owned tmux pane, mirrored into Omnigent; the vendor CLI keeps its own UX and Omnigent injects turns, policy hooks, and an MCP relay. |
| **Executor** | The Python class implementing one harness's lifecycle inside the runner (`omnigent/inner/*_executor.py`). |
| **Provider / credential kind** | How a model is paid for and reached, in `~/.omnigent/config.yaml` on the **host**: `key` (vendor API key), `subscription` (logged-in `claude`/`codex` CLI — Claude Pro/Max, ChatGPT), `gateway` (any OpenAI/Anthropic-compatible proxy), `local` (Ollama), `databricks`, `cli-config`, `bedrock`. Credentials are per-host, which is why per-user hosts give per-user billing for free. |
| **Model family** | The credential namespace a harness draws from: `anthropic`, `openai`, `gemini` (per-family defaults coexist). |
| **Builtin tools / `sys_*`** | Omnigent's first-party tools bridged into every harness: `sys_os_*` (file/shell), `sys_terminal_*`, `sys_session_*` (spawn/fan-out), async inbox, timers, scheduled tasks, `web_search`/`web_fetch`, `upload_file`/`download_file`, `browser_*`, memory, policy tools. |
| **MCP server / MCP relay** | Model Context Protocol tool servers an agent declares (http or stdio); for native harnesses Omnigent injects a single relay MCP server into the vendor CLI so Omnigent tools and policies apply there too. |
| **Skill** | A `SKILL.md` instruction package (Claude Code-compatible) discovered from agent bundles, repo `.claude/skills/`, or home dirs; surfaced in the composer's `/` menu. |
| **Sub-agent** | A child agent another agent can dispatch (`type: agent` in the spec, or dynamic `spawn`) — possibly on a different harness/model. The basis of multi-agent orchestration (see `examples/polly`). |

### 2.4 Interaction plane (how work happens, who sees it)

| Term | Definition |
|---|---|
| **Session** | One conversation with one agent, bound (usually) to a host + workspace directory. The unit of access control, forking, and UI. "Conversation" in the DB. |
| **Turn** | One user message → agent work → response cycle, executed by the session's runner. |
| **Elicitation** | Any mid-turn request for human input: policy approvals, native permission prompts, structured questions. Surfaced as approval cards in the UI, the inbox, approve-by-link pages, and Slack. |
| **Policy** | A declarative guard evaluated at six phases (request/response/tool_call/tool_result/llm_request/llm_response) returning ALLOW/DENY/ASK, at three scopes: session, agent spec, server-wide default. Spend caps, blast-radius, approval gates live here. |
| **Permission levels** | Per-session grants: READ < EDIT (co-drive) < MANAGE < OWNER, plus a `__public__` anyone-with-the-link grant. Governed by the server-wide **sharing mode**. |
| **Project** | A user's folder grouping sessions in the sidebar, with default host/workspace/agent prefill config. |
| **Fork** | Copy a session from a chosen point, optionally onto a different agent, harness, or host. **Switch agent / switch host** mutate a session in place. |
| **Automation (scheduled task)** | A stored RRULE schedule that creates a session and dispatches a prompt when it fires; needs a live host at fire time. "Automations" in the product UI, `scheduled_task` in code. |
| **Inbox** | The cross-session surface for pending approvals and unseen comments (`/inbox`); also the agent-side `sys_read_inbox` lane where sub-agent results land. |
| **Presence / co-driving** | Live avatars of viewers in a session; EDIT-level users can send messages that run on the owner's host. |

### 2.5 Access matrix (who can touch what)

| Object | Owner | Others |
|---|---|---|
| Host | The registering user only. Listed, launched on, and browsed only by its owner. **Hosts cannot be shared across users.** | Nothing — not even visible. |
| Session | Creator (OWNER). | Per-user grants READ/EDIT/MANAGE; optional public link; scheduler-created sessions grant the task owner. |
| Agent (template) | Nobody in particular — server-wide. | Every user can run it; admins curate. |
| Agent (session copy) | Follows the session's grants. | Same. |
| Policy | Session-scope: session participants; agent-scope: whoever edits the spec; default scope: admins. | — |
| Credentials (model keys/subscriptions) | Live on the host (`~/.omnigent`, keychain) — so, its owner. Never stored on the server. | Inaccessible. |
| Files/artifacts | Scoped to their session. | Via session grants. |

Consequence worth stating: **user B can use user A's compute only through a
session A shared** (B co-drives A's session on A's host). There is no
"shared host" object. A "shared server" for a team means shared *control
plane*, with compute still per-user.

---

## 3. Dedicated and shared computers on our infrastructure, in plain terms

### The mental model

Three layers, three words:

1. **The platform** (Omnigent *server*): one central web app for the whole
   org. Everyone logs into the same URL from the desktop app, browser, or
   phone.
2. **Your computer** (an Omnigent *host*): a VM on Nutanix AHV that belongs to
   one user. Their agents live and work there: their files, their logins,
   their model credentials, their long-running jobs.
3. **Your agents** (agent specs on harnesses): what actually does the work,
   executing on *your computer*, coordinated and governed *by the platform*.

This is exactly Grok Bot's shape — app + personal cloud computer + bots —
except the control plane is ours, the computer is on our AHV cluster, and the
model/harness underneath is whatever the user connects.

### Dedicated computer, MVP (out of band — no Omnigent changes required)

1. Ops (or a self-service portal we build beside Omnigent) clones an AHV VM
   from a golden image: Linux + Omnigent + harness CLIs preinstalled (the
   moral equivalent of the `omnigent-host` container image).
2. On that VM, *as the user's Omnigent identity*:
   `omnigent login https://omnigent.<org>` (browser/SSO once), then
   `omnigent host https://omnigent.<org> --background`.
3. The VM dials out and registers as an **external host owned by that user**.
   No inbound firewall rules — the tunnel is outbound-only.
4. The user opens the desktop app pointed at the central server, picks their
   computer in the host picker, and works. Registration survives reboots
   (host identity persists in `~/.omnigent/config.yaml`; run the daemon under
   systemd).

The only genuinely fiddly step is 2 — getting *the user's* login onto the VM
so the host binds to the right owner. Options, easiest first: the user runs
the login over SSH/console once; cloud-init drops a script and the user
completes the browser auth; or the provisioning portal uses the existing
**device grant** flow (`OMNIGENT_DEVICE_GRANT_ENABLED`) to mint a delegated
token on the user's behalf — that flow was built for exactly this
(the Slack integration is its first consumer).

### Shared computers

Because a host is single-owner, "shared" decomposes into three supported
shapes — pick per use case:

- **Shared control plane** (always true): one server, all users, invites/SSO,
  admin policies. This is "the shared Omnigent".
- **Shared session**: A shares a session (EDIT) with B; B co-drives on A's
  computer. Right for pairing and review, not for "team compute".
- **Shared big VM**: one large AHV VM, one OS account per user, each running
  their own `omnigent host` under their own login. Omnigent sees N hosts;
  users see "their computer". Right for cheap multi-tenancy on one box, with
  OS-level isolation between users.
- *(Later, if a true team-owned host is wanted, that's new work — a host
  owned by a group — and should be a deliberate design, not assumed.)*

### What "later" looks like (in-band, and where sleep lives)

When we outgrow out-of-band: write an **AHV sandbox provider** (a
`SandboxLifecycle` plugin — create/terminate/resume against Prism Central
APIs) and adopt the parity plan's W1 ("dedicated per-user cloud computer").
Then the server itself provisions a user's computer at first login, the
`hosts: {mode: dedicated}` switch turns off local/laptop hosts, and
sleep/wake becomes real: the platform's existing `resume_managed_host` path
wakes a stopped VM when a message or automation fires.

Until then, idle sleep can indeed live entirely outside the platform: AHV-side
idle detection powers VMs down; Omnigent simply shows the host offline and
sessions reconnect when it returns. The one thing out-of-band sleep cannot do
is *wake on message* — that requires the provider integration, which is why
it's sequenced "later".

---

## 4. Why this glossary exists (the goal, named)

We are building one deployment that is simultaneously:

1. **An open Grok Bot** — persistent personal cloud computers + named
   always-on agents + connectors/routines — on **our** AHV infrastructure,
   with **any** harness and the user's own model subscription or key
   (`designs/grok-bot-parity.md`).
2. **A general agentic coding tool** — the thing Omnigent already is
   (sessions, worktrees, terminals, multi-harness orchestration), for the
   same users on the same computers.
3. **An org platform** — shared control plane, SSO, policies, spend caps,
   audit — where compute is per-user by construction.

The three parts must share one vocabulary because they share one object
model: the *server* is the platform, a *host* is a user's computer, an
*agent on a harness* is the worker, and a *session* is the unit of access.
Every design doc, ticket, and conversation in this program should use the
terms exactly as defined here.
