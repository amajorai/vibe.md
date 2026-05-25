---
name: vibe
description: "Server setup skill. Detects local vs. remote environment. If local, provisions a VPS on the chosen cloud provider. If already on a server, asks whether this is an orchestrator (manages more servers) or a standalone master (everything on one machine, like Coolify). Installs Bun, GitHub CLI, Dokploy/Coolify, and optional Wrangler, then opens the firewall for the platform UIs. Ends by offering to install the ship and hardening skills."
argument-hint: [optional flags]
---

# Vibe

You are setting up a production server environment. Interview first, execute in one clean pass.

**Args:** {{args}}


## Phase 0: Auto-Update

*Skip unless `{{args}}` contains `--update`, or `SKILLS_AUTO_UPDATE: true` is set in your project CLAUDE.md.*

This phase is best-effort and must never block the user. If the command below fails (CLI not installed, no network, node/npx not on PATH), continue silently to Phase 1.

```bash
npx --yes skills update vibe -y 2>/dev/null || true
```

If — and only if — the output indicates the skill was actually updated, stop here and tell the user: **"This skill was just updated. Re-run your command to use the new version."** In every other case (no update available, command failed, CLI missing), continue silently to Phase 1.

## Phase 1: Environment Detection

Silently detect the environment:

```bash
uname -a && cat /etc/os-release 2>/dev/null || sw_vers 2>/dev/null
curl -s --max-time 2 http://169.254.169.254/latest/meta-data/instance-id 2>/dev/null && echo "IS_AWS" || true
curl -s --max-time 2 http://169.254.0.1/metadata 2>/dev/null | grep -qi hetzner && echo "IS_HETZNER" || true
echo "USER=$USER HOME=$HOME"
```

Determine:
- **Local dev machine** (macOS, Windows, desktop Linux with GUI): need to provision a VPS first
- **Already on a VPS / headless server**: skip straight to server setup

Tell the user: "I see you're on [local machine / a server]. Is that right?"


## Phase 2: Interview

Ask all questions in a single message. Do not start any installation until answers are confirmed.

### If local machine:

> You're on a local machine. I'll provision a cloud server for you, then set it up.
>
> **1. Which cloud provider?**
> - [ ] Hetzner Cloud — cheapest, EU-based (~€4/mo for CX22). Best default.
> - [ ] OVH — good value, global PoPs
> - [ ] AWS — most ecosystem, free tier (t3.micro)
> - [ ] I already have a server (give me the IP and I'll skip provisioning)
>
> **2. What role should the new server play?**
> - [ ] **Master/standalone** — everything runs on this one machine (like a single-machine Coolify setup). No need to manage other servers.
> - [ ] **Orchestrator** — this is the control plane. It manages deployments AND can provision and connect more worker servers as needed (installs the cloud CLI on the server in Step 7).
>
> **3. Which deployment platform?**
> - [ ] Dokploy — lightweight, Docker-native, self-hosted PaaS
> - [ ] Coolify — more features, great UI, self-hosted Heroku alternative
> - [ ] Both (Dokploy for apps, Coolify for databases/services)
>
> **4. Deploy to edge as well?**
> - [ ] Yes — install Wrangler (Cloudflare Workers, Pages, DNS)
> - [ ] No

### If already on a server:

> You're already on a server. How should it be configured?
>
> **1. What role does this server play?**
> - [ ] **Master/standalone** — everything runs here. One machine to rule them all (like a single-machine Coolify setup). No need to manage other servers.
> - [ ] **Orchestrator** — this is the control plane. It manages deployments AND can provision and connect more worker servers as needed.
>
> **2. Which deployment platform?**
> - [ ] Dokploy — lightweight, Docker-native, self-hosted PaaS
> - [ ] Coolify — more features, great UI, self-hosted Heroku alternative
> - [ ] Both (Dokploy for apps, Coolify for databases/services)
>
> **3. Deploy to edge as well?**
> - [ ] Yes — install Wrangler (Cloudflare Workers, Pages, DNS)
> - [ ] No

Once the user answers, proceed to the appropriate phases.


## Phase 3A: Provision a VPS (local machine only)

Install the chosen cloud provider CLI and spin up a server. These commands run on **your local machine** (macOS, Windows, or Linux).

- **Hetzner:** see [references/provision-hetzner.md](references/provision-hetzner.md)
- **OVH:** see [references/provision-ovh.md](references/provision-ovh.md)
- **AWS:** see [references/provision-aws.md](references/provision-aws.md)

After provisioning and SSHing in, your shell prompt changes to the server. Every Phase 3B command runs there. If you already have a server IP, skip straight to Phase 3B.


## Phase 3B: Server Setup

Run all steps in order on the server. Full step-by-step: see [references/server-setup.md](references/server-setup.md).

- Steps 1–6 apply to all roles (system baseline, Bun, GitHub CLI, deployment platform, firewall, Wrangler).
- Step 7 (cloud CLI for orchestration) applies **only if the user chose the orchestrator role**: see [references/orchestrator-setup.md](references/orchestrator-setup.md).


## Completion Checklist

- [ ] Environment detected and confirmed
- [ ] VPS provisioned and SSH'd into (if starting from local)
- [ ] System packages updated
- [ ] Bun installed
- [ ] GitHub CLI installed and authenticated (if the workflow uses GitHub — skip otherwise)
- [ ] Deployment platform installed (Dokploy / Coolify / both) + CLI authenticated
- [ ] Firewall opened for the platform UI ports (host UFW + provider/security group)
- [ ] Wrangler installed and authenticated via API token (if edge)
- [ ] Cloud CLI on server for orchestration (only if orchestrator role)


## Final Step: Optional Skills

The server is ready. Two more things I can set up for you:

### 1. Install the `ship` skill?

Full-cycle dev workflow: implement → verify → edge cases → tests → security review.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^ship$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

If not installed, ask: **"Would you like me to install the `ship` skill? It enables `/ship <feature>` for full-cycle development."**

If yes: `npx --yes skills add amajorai/ship.md -a claude-code -y`

### 2. Install and run the `hardening` skill?

Secures the server: SSH hardening, fail2ban, UFW, unattended upgrades, AppArmor, and more.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^hardening$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install and run the `hardening` skill to lock down this server?"**

If yes: `npx --yes skills add amajorai/skills/skills/hardening -a claude-code -y`

A newly added skill is not loaded in the current session. Tell the user: **"Start a new Claude Code session, then run `/hardening`."** Do not invoke it yourself.
