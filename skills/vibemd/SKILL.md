---
name: vibe
description: "Server setup skill. Detects local vs. remote environment. If local, provisions a VPS on the chosen cloud provider. If already on a server, asks whether this is an orchestrator (manages more servers) or a standalone master (everything on one machine, like Coolify). Installs Bun, GitHub CLI, Dokploy/Coolify, and optional Wrangler, then opens the firewall for the platform UIs. Ends by offering to install the ship and hardening skills."
argument-hint: [optional flags]
---

# Vibe

You are setting up a production server environment. Interview first, execute in one clean pass.

**Args:** {{args}}


## Privacy Rule — Redact Sensitive Values by Default

Never print the following in chat, even if you just ran a command that returned them:

- Server IP addresses or hostnames
- API keys, personal access tokens, or bearer tokens
- SSH private keys or fingerprints
- Passwords or passphrases
- Cloud provider credentials or account IDs
- Any value the user passed as a secret placeholder (e.g. `<YOUR_DOKPLOY_API_KEY>`)

When one of these values would naturally appear in your response, replace it with a placeholder and offer to share on request. Examples:

> Your server has been provisioned. *(Ask me for the IP if you need it.)*

> GitHub CLI authenticated successfully. *(The token is not shown in chat — ask if you need it repeated.)*

> Dokploy is up and the CLI is authenticated. *(API key kept out of chat.)*

If the user explicitly asks — "show me the IP", "what's the token?", "give me the full command with the real values" — then output the real value in that one response only. Do not repeat it in follow-up messages unless asked again.


## Phase 0: Auto-Update

*Skip unless `{{args}}` contains `--update`, or `SKILLS_AUTO_UPDATE: true` is set in your project CLAUDE.md.*

This phase is best-effort and must never block the user. If the command below fails (CLI not installed, no network, node/npx not on PATH), continue silently to Phase 1.

```bash
npx --yes skills update vibe -y 2>/dev/null || true
```

If — and only if — the output indicates the skill was actually updated, stop here and tell the user: **"This skill was just updated. Re-run your command to use the new version."** In every other case (no update available, command failed, CLI missing), continue silently to Phase 1.

## Phase 1: Environment Detection

### Check for existing vibemd config first

```bash
cat /etc/vibemd/server.json 2>/dev/null || echo "NO_CONFIG"
```

**If config exists** — this server is already set up. Show the current state and stop. Do not proceed to Phase 2.

> **This server is already configured.**
>
> **Current setup (`/etc/vibemd/server.json`):**
> - Role: `[role]`
> - Platform: `[platform]`
> - Provider: `[provider]`
> - Edge (Wrangler): `[edge]`
> - AI coding CLIs: `[ai_clis]`
> - Workers provisioned: `[workers.length]`
> - Set up: `[provisioned_at]`
>
> **What would you like to do?**
> - **Add worker servers** → run `/vibe-provision-worker` *(orchestrator role only)*
> - **Change role, platform, or installed tools** → run `/vibe-reconfigure`
> - **Harden this server** (SSH, fail2ban, UFW) → run `/hardening`
> - **Start fresh** — re-run full setup on this machine (destructive — will overwrite config)

If the user chooses "Start fresh", continue to the environment detection below. Otherwise stop here.

### Detect environment (new setup only)

```bash
uname -a && cat /etc/os-release 2>/dev/null || sw_vers 2>/dev/null
curl -s --max-time 2 http://169.254.169.254/latest/meta-data/instance-id 2>/dev/null && echo "IS_AWS" || true
curl -s --max-time 2 http://169.254.0.1/metadata 2>/dev/null | grep -qi hetzner && echo "IS_HETZNER" || true
curl -s --max-time 2 http://169.254.169.254/metadata/v1/id 2>/dev/null && echo "IS_DIGITALOCEAN" || true
echo "USER=$USER HOME=$HOME"
```

Determine:
- **Local dev machine** (macOS, Windows, desktop Linux with GUI): need to provision a VPS first
- **Already on a VPS / headless server**: skip straight to server setup

Tell the user: "I see you're on [local machine / a server]. Is that right?"


## Phase 2: Interview

Use AskUserQuestion for each question below — one call at a time, waiting for the answer before asking the next. Do not use markdown checkbox lists. Do not start any installation until all questions are answered.

### If local machine:

Ask these questions in order:

**Q1 — Cloud provider**

```
AskUserQuestion(
  question: "You're on a local machine. I'll provision a cloud server for you, then set it up. Which cloud provider do you want to use? (Region and server size will be chosen in the next step — I'll fetch the live options from the provider's CLI after you authenticate.)",
  options: [
    "Hetzner Cloud — cheapest, EU-based (~€4/mo for CX22). Best default.",
    "DigitalOcean — simple pricing, great UX, global regions",
    "OVH — good value, global PoPs",
    "AWS — most ecosystem, free tier (t3.micro)"
  ]
  // "Other" (built-in) → user already has a server; ask for the IP and skip provisioning
)
```

**Q2 — Server role**

```
AskUserQuestion(
  question: "What role should the new server play? (Not sure? Start with Master. This can be changed at any time by running /vibe-reconfigure on your server.)",
  options: [
    "Master/standalone — everything runs on this one machine (like a single-machine Coolify setup). No need to manage other servers.",
    "Orchestrator — this is the control plane. It manages deployments AND can provision and connect more worker servers as needed (installs the cloud CLI on the server in Step 7)."
  ]
)
```

**Q3 — Deployment platform**

```
AskUserQuestion(
  question: "Which deployment platform? (Choosing 'Both' is not recommended — it adds unnecessary complexity. Your full stack should all live on one platform. Only consider it if you're evaluating both platforms with completely separate apps.)",
  options: [
    "Dokploy — lightweight, Docker-native, self-hosted PaaS",
    "Coolify — more features, great UI, self-hosted Heroku alternative",
    "Both (Dokploy for apps, Coolify for databases/services)"
  ]
)
```

**Q4 — Edge deployment**

```
AskUserQuestion(
  question: "Deploy to edge as well?",
  options: [
    "Yes — install Wrangler (Cloudflare Workers, Pages, DNS)",
    "No"
  ]
)
```

**Q5 — AI coding CLIs**

```
AskUserQuestion(
  question: "Which AI coding CLI(s) should be installed?",
  options: [
    "None / Skip",
    "Claude Code (@anthropic-ai/claude-code) — needs ANTHROPIC_API_KEY",
    "Codex CLI (@openai/codex) — needs OPENAI_API_KEY",
    "Both"
  ]
)
```

**Q6 — AI CLI install location (ask only if Orchestrator was chosen in Q2 AND an AI CLI was chosen in Q5)**

```
AskUserQuestion(
  question: "Where should the AI coding CLI(s) be installed?",
  options: [
    "On this orchestrator server only",
    "On worker servers when provisioned",
    "Both orchestrator and workers"
  ]
)
```

---

### If already on a server:

Ask these questions in order:

**Q1 — Server role**

```
AskUserQuestion(
  question: "You're already on a server. What role does it play? (Not sure? Start with Master. This can be changed at any time by running /vibe-reconfigure on your server.)",
  options: [
    "Master/standalone — everything runs here. One machine to rule them all (like a single-machine Coolify setup). No need to manage other servers.",
    "Orchestrator — this is the control plane. It manages deployments AND can provision and connect more worker servers as needed."
  ]
)
```

**Q2 — Deployment platform**

```
AskUserQuestion(
  question: "Which deployment platform? (Choosing 'Both' is not recommended — it adds unnecessary complexity. Your full stack should all live on one platform. Only consider it if you're evaluating both platforms with completely separate apps.)",
  options: [
    "Dokploy — lightweight, Docker-native, self-hosted PaaS",
    "Coolify — more features, great UI, self-hosted Heroku alternative",
    "Both (Dokploy for apps, Coolify for databases/services)"
  ]
)
```

**Q3 — Edge deployment**

```
AskUserQuestion(
  question: "Deploy to edge as well?",
  options: [
    "Yes — install Wrangler (Cloudflare Workers, Pages, DNS)",
    "No"
  ]
)
```

**Q4 — AI coding CLIs**

```
AskUserQuestion(
  question: "Which AI coding CLI(s) should be installed?",
  options: [
    "None / Skip",
    "Claude Code (@anthropic-ai/claude-code) — needs ANTHROPIC_API_KEY",
    "Codex CLI (@openai/codex) — needs OPENAI_API_KEY",
    "Both"
  ]
)
```

**Q5 — AI CLI install location (ask only if Orchestrator was chosen in Q1 AND an AI CLI was chosen in Q4)**

```
AskUserQuestion(
  question: "Where should the AI coding CLI(s) be installed?",
  options: [
    "On this orchestrator server only",
    "On worker servers when provisioned",
    "Both orchestrator and workers"
  ]
)
```

---

Once all questions are answered, proceed to the appropriate phases.


## Phase 3A: Provision a VPS (local machine only)

Install the chosen cloud provider CLI and spin up a server. These commands run on **your local machine** (macOS, Windows, or Linux).

- **Hetzner:** see [references/provision-hetzner.md](references/provision-hetzner.md)
- **DigitalOcean:** see [references/provision-digitalocean.md](references/provision-digitalocean.md)
- **OVH:** see [references/provision-ovh.md](references/provision-ovh.md)
- **AWS:** see [references/provision-aws.md](references/provision-aws.md)

After provisioning and SSHing in, your shell prompt changes to the server. Every Phase 3B command runs there. If you already have a server IP, skip straight to Phase 3B.


## Phase 3B: Server Setup

Run all steps in order on the server. Full step-by-step: see [references/server-setup.md](references/server-setup.md).

- Steps 1–6 apply to all roles (system baseline, Bun, GitHub CLI, deployment platform, firewall, Wrangler).
- Step 7 (AI coding CLIs) applies **only if Claude Code / Codex CLI was chosen** in Phase 2. For orchestrators, only install here if "orchestrator only" or "both" was chosen — worker installation is handled in orchestrator-setup.md.
- Step 8 (cloud CLI for orchestration) applies **only if the user chose the orchestrator role**: see [references/orchestrator-setup.md](references/orchestrator-setup.md).

Step 4 of server-setup.md installs the platform CLI on the server itself (so you can script deployments from the server). Phase 3C below installs the same CLI on your **local machine** so you can manage the server from your dev environment.


## Phase 3C: Install Platform CLI Locally (local machine only)

*Skip if you started directly on a server (Phase 1 detected a VPS).*

Install the CLI for whichever platform(s) were chosen in Phase 2, on your **local machine**. This lets you deploy, inspect, and manage your server without SSH-ing in.

### Dokploy CLI (if chosen)

```bash
# macOS / Linux / Windows (requires Node/Bun)
bun add -g @dokploy/cli      # preferred if Bun is installed
# or:
npm install -g @dokploy/cli

dokploy --version
```

Authenticate against your new server (run **after** the Dokploy UI is up and you've created an API key in Settings → API):

```bash
dokploy auth -u http://<SERVER_IP>:3000 -t <YOUR_DOKPLOY_API_KEY>
# verify:
dokploy project all
```

### Coolify CLI (if chosen)

**macOS / Linux:**

```bash
# Option A — curl installer (Linux/macOS)
curl -fsSL https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.sh | bash

# Option B — Homebrew (macOS/Linux)
brew install coollabsio/coolify-cli/coolify-cli

coolify --version
```

**Windows (PowerShell):**

```powershell
irm https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.ps1 | iex
coolify --version
```

Authenticate against your new server (run **after** the Coolify UI is up and you've created an API token in Security → API Tokens):

```bash
coolify context add -d my-server http://<SERVER_IP>:8000 <YOUR_COOLIFY_TOKEN>
# switch to it and verify:
coolify context use my-server
coolify server list
```

### If "Both" was chosen

Run both blocks above in order. You'll have `dokploy` pointing at port 3000 and `coolify` pointing at port 8000 on the same server.


## Completion Checklist

- [ ] Environment detected and confirmed
- [ ] VPS provisioned and SSH'd into (if starting from local)
- [ ] System packages updated
- [ ] Bun installed
- [ ] GitHub CLI installed and authenticated (if the workflow uses GitHub — skip otherwise)
- [ ] Deployment platform installed on server (Dokploy / Coolify / both) + server-side CLI authenticated
- [ ] Firewall opened for the platform UI ports (host UFW + provider/security group)
- [ ] Wrangler installed and authenticated via API token (if edge)
- [ ] AI coding CLIs installed and API keys set (Claude Code / Codex — if chosen)
- [ ] AI coding CLIs installed on workers (if orchestrator + workers chosen)
- [ ] Cloud CLI on server for orchestration (only if orchestrator role)
- [ ] Platform CLI installed locally and authenticated against the server (Dokploy / Coolify / both)
- [ ] `/etc/vibemd/server.json` written


## Write Server Config

Write the source-of-truth config to the server. Every future skill reads this so it always knows the current setup without having to detect it.

```bash
mkdir -p /etc/vibemd

cat > /etc/vibemd/server.json << EOF
{
  "role": "<master|orchestrator>",
  "platform": "<dokploy|coolify|both>",
  "provider": "<hetzner|digitalocean|ovh|aws|unknown>",
  "edge": <true|false>,
  "ai_clis": [<"claude-code"|"codex">],
  "ai_clis_on_workers": <true|false>,
  "vibe_version": "1.0.0",
  "provisioned_at": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "workers": []
}
EOF

echo "Server config written:"
cat /etc/vibemd/server.json
```

Replace all `<...>` placeholders with the actual values chosen during Phase 2 before running.


## Final Step: Optional Skills

The server is ready. Offer the skills below **one at a time, in order**. Wait for the user's answer before moving to the next. Always proceed to the next offer regardless of whether the user accepted or rejected the previous one — do not skip any step.

### Detecting the CLI environment

Before offering any skill installs, detect which AI CLI is running this session:

```bash
echo "CODEX=${CODEX:-false}"
echo "CODEX_SANDBOX=${CODEX_SANDBOX:-}"
```

- If `CODEX=true` or `CODEX_SANDBOX` is set → **Codex mode**: newly installed skills are reloaded automatically; you can invoke them immediately after install.
- Otherwise → **Claude Code mode**: the user must manually run `/reload-plugins` in their session before the skill becomes available.

### 1. Install the `ship` skill?

Full-cycle dev workflow: implement → verify → edge cases → tests → security review.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^ship$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

If not installed, ask: **"Would you like me to install the `ship` skill? It enables `/ship <feature>` for full-cycle development."**

If yes: `npx --yes skills add amajorai/ship.md -a claude-code -y`

- **Codex mode:** invoke `/ship` immediately.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session, then run `/ship`."** Do not invoke it yourself.

### 2. Install and run the `hardening` skill?

Secures the server: SSH hardening, fail2ban, UFW, unattended upgrades, AppArmor, and more.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^hardening$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install and run the `hardening` skill to lock down this server?"**

If yes: `npx --yes skills add amajorai/skills/skills/hardening -a claude-code -y`

- **Codex mode:** invoke `/hardening` immediately. Once it completes (or if the user declined), continue to offer #3.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session, then run `/hardening`."** Do not invoke it yourself. Then immediately continue to offer #3.

### 3. Install the `party` skill?

GitHub Projects kanban board that builds itself. Issues move through Backlog → Ready → In Progress → In Review → Done automatically. The agent refines requirements, spawns parallel build subagents, tracks PRs, and updates the board — you just watch.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^party$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install the `party` skill? It turns your GitHub Issues into a self-driving kanban board — run `/party` to start."**

If yes: `npx --yes skills add amajorai/party.md -a claude-code -y`

- **Codex mode:** invoke `/party --setup` immediately.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session, then run `/party --setup`."** Do not invoke it yourself.

### 4. Install the `hunt` skill?

Systematic bug-hunting workflow. Instruments the codebase with targeted logging, reads the logs to confirm root cause, makes a surgical fix, and verifies clean. Use it when something is broken and you don't know why.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^hunt$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install the `hunt` skill? It enables `/hunt <bug description>` for systematic debugging."**

If yes: `npx --yes skills add amajorai/fix.md -a claude-code -y`

- **Codex mode:** tell the user hunt is ready — invoke when needed.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session so `/fix` becomes available."** Do not invoke it yourself.
