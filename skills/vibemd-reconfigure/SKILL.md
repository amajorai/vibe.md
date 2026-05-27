---
name: vibe-reconfigure
description: "Reconfigure an existing vibe server. Change server role (master ↔ orchestrator), swap or add a deployment platform (Dokploy/Coolify), or add/remove edge deployment (Wrangler). Safe to run on a live server — detects current state first and only applies the delta."
argument-hint: [optional flags]
---

# Reconfigure

You are reconfiguring an already-provisioned vibe server. Detect current state first, then ask what to change. Only apply the delta — do not reinstall things that are already correctly set up.

**Args:** {{args}}


## Phase 1: Detect Current State

Check the server config first — it's the fastest and most reliable source of truth:

```bash
cat /etc/vibemd/server.json 2>/dev/null || echo "NO_CONFIG"
```

**If config exists**, use it directly. No further detection needed. Show the user:

> **Current config (`/etc/vibemd/server.json`):**
> - Role: `[role]`
> - Platform: `[platform]`
> - Provider: `[provider]`
> - Edge (Wrangler): `[edge]`
> - AI coding CLIs: `[ai_clis]`
> - Workers: `[workers.length]`

**If no config**, fall back to runtime detection:

```bash
# Server role indicator — cloud CLI presence means orchestrator
which hcloud doctl aws 2>/dev/null && echo "CLOUD_CLI_FOUND" || echo "NO_CLOUD_CLI"

# Deployment platform
docker ps --format '{{.Names}}' 2>/dev/null | grep -iE 'dokploy|coolify' || echo "NO_PLATFORM_CONTAINERS"
which dokploy 2>/dev/null && echo "DOKPLOY_CLI" || true

# AI coding CLIs
which claude codex 2>/dev/null || echo "NO_AI_CLIS"

# Edge
which wrangler 2>/dev/null && echo "WRANGLER_INSTALLED" || echo "NO_WRANGLER"

# OS / distro
uname -a && cat /etc/os-release 2>/dev/null | head -5
```

Summarize what you find before asking questions.


## Phase 2: Interview

> **Important:** Use `AskUserQuestion` (one call per question) for all questions in this phase. Do NOT use markdown checkbox lists. Each question maps to one `AskUserQuestion` call with up to 4 options (a built-in "Other" option is always available). Only show options that represent a change from the current state.

**Question 1 — Server role:**

Use `AskUserQuestion` (single-select) with the question: "Server role (current: `[master / orchestrator]`) — what would you like to do?"

Options:
1. Keep as-is
2. Switch to master/standalone (removes cloud CLI; machine manages only itself)
3. Switch to orchestrator (installs cloud CLI to provision and connect worker servers)

**Question 2 — Deployment platform:**

Use `AskUserQuestion` (single-select) with the question: "Deployment platform (current: `[Dokploy / Coolify / both]`) — what would you like to do?"

Options:
1. Keep as-is
2. Switch to Dokploy (lightweight, Docker-native)
3. Switch to Coolify (more features, great UI)
4. Add the other platform alongside current one (not recommended — only for evaluating both with separate apps)

**Question 3 — Edge deployment (Wrangler):**

Use `AskUserQuestion` (single-select) with the question: "Edge deployment / Wrangler (current: `[installed / not installed]`) — what would you like to do?"

Options:
1. Keep as-is
2. Install Wrangler (Cloudflare Workers, Pages, DNS)
3. Remove Wrangler

**Question 4 — AI coding CLIs (part A):**

Use `AskUserQuestion` (single-select) with the question: "AI coding CLIs (current: `[Claude Code / Codex / both / none]`) — keep as-is or make a change?"

Options:
1. Keep as-is
2. Make a change (answer the next question)

If the user chooses "Make a change", ask a follow-up:

**Question 4 — AI coding CLIs (part B):**

Use `AskUserQuestion` (single-select) with the question: "Which AI coding CLI setup do you want?"

Options:
1. Install Claude Code only (`@anthropic-ai/claude-code`)
2. Install Codex CLI only (`@openai/codex`)
3. Install both Claude Code and Codex CLI
4. Remove all AI coding CLIs

Wait for all answers before making any changes.


## Phase 3: Apply Changes

Only apply what changed. Skip anything marked "Keep as-is."

### Role: Switch to Orchestrator

Install the cloud CLI matching the provider originally used to provision the server. See [../vibemd/references/orchestrator-setup.md](../vibemd/references/orchestrator-setup.md).

### Role: Switch to Master

Remove the cloud CLI:

```bash
# Hetzner
apt-get remove -y hcloud-cli 2>/dev/null || true
rm -f /usr/local/bin/hcloud

# DigitalOcean
rm -f /usr/local/bin/doctl

# AWS
pip3 uninstall -y awscli 2>/dev/null || rm -f /usr/local/bin/aws
```

Confirm removal: `which hcloud doctl aws 2>/dev/null || echo "Cloud CLI removed"`

### Platform: Switch or Add

Follow the platform installation steps in [../vibemd/references/server-setup.md](../vibemd/references/server-setup.md) (Step 4). If switching away from an existing platform, warn the user first:

> **Warning:** Switching platforms will not automatically migrate your existing apps, databases, or volumes. Make sure everything is backed up before proceeding.

Only proceed after explicit confirmation.

### Edge: Install Wrangler

```bash
bun add -g wrangler
wrangler --version
```

`wrangler login` opens a browser OAuth flow and will not work on a headless server. Use the API token approach instead (create a token at Cloudflare dashboard → My Profile → API Tokens, with Workers/Pages/DNS permissions):

```bash
export CLOUDFLARE_API_TOKEN=<your-cloudflare-api-token>
echo 'export CLOUDFLARE_API_TOKEN=<your-cloudflare-api-token>' >> ~/.bashrc
wrangler whoami   # verify auth
```

### Edge: Remove Wrangler

```bash
bun remove -g wrangler
```


## Phase 4: Update Server Config

After applying all changes, update `/etc/vibemd/server.json` to reflect the new state. If the file doesn't exist yet, create it.

```bash
# Read existing config or start from scratch
CONFIG=$(cat /etc/vibemd/server.json 2>/dev/null || echo '{"workers":[],"provisioned_at":"unknown"}')

mkdir -p /etc/vibemd
echo "$CONFIG" | jq \
  --arg role "<new-role>" \
  --arg platform "<new-platform>" \
  --arg edge "<true|false>" \
  --argjson ai_clis '["<claude-code|codex>"]' \
  '.role = $role | .platform = $platform | .edge = ($edge == "true") | .ai_clis = $ai_clis | .reconfigured_at = now | todate' \
  > /tmp/vibemd-server.json && mv /tmp/vibemd-server.json /etc/vibemd/server.json

echo "Updated /etc/vibemd/server.json:"
cat /etc/vibemd/server.json
```

Only update the fields that changed — leave `workers`, `provider`, `provisioned_at`, and other unchanged fields intact.


## Completion Checklist

- [ ] Current state read from config (or detected from runtime)
- [ ] User confirmed desired changes
- [ ] Server role updated (if changed)
- [ ] Deployment platform updated (if changed)
- [ ] Wrangler installed or removed (if changed)
- [ ] AI coding CLIs installed or removed (if changed)
- [ ] `/etc/vibemd/server.json` updated
- [ ] No unintended side effects (existing apps still running)
