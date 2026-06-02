---
name: vibe-provision-worker
description: "Provision a new worker server from an orchestrator. Reads /etc/vibemd/server.json for provider, platform, and AI CLI preferences, spins up a new VPS, registers it with Dokploy or Coolify, optionally installs AI coding CLIs, and records the worker in server.json."
argument-hint: [optional flags]
---

# Provision Worker

You are running on an **orchestrator server** and provisioning a new worker node. Read the server config first — it tells you the provider, platform, and preferences so you don't have to ask again.

**Args:** {{args}}


## Privacy Rule — Redact Sensitive Values by Default

Do not print these in chat, even when a command you just ran returned them: worker/server IP addresses, API keys / tokens, SSH private keys or fingerprints, passwords, or cloud credentials. Keep IPs in shell variables (e.g. `$WORKER_IP`) and reference the variable in later commands instead of echoing the raw value. When reporting progress, redact by omission:

> Worker provisioned and SSH is up. *(The IP is not shown in chat — ask if you need it.)*

**Exceptions you may show:** an SSH **public** key (public by design), and the worker IP **at the registration step** where the user must paste it into the Dokploy/Coolify UI — show it once there, flagged as needed for that manual step. If the user explicitly asks for the IP/token, give it in that one response only.


## Phase 1: Read Server Config

> **Note:** All command blocks in this skill assume a bash environment. On a Windows host, run them via the Bash tool / Git Bash.

```bash
cat /etc/vibemd/server.json 2>/dev/null || echo "NO_CONFIG"
```

If `NO_CONFIG` — stop and tell the user:

> **No server config found at `/etc/vibemd/server.json`.**
> This skill must run on a server that was set up with `/vibe`. Either:
> - Run `/vibe` on this machine first, or
> - Manually provide: provider (hetzner/digitalocean/ovh/aws), platform (dokploy/coolify), and whether to install AI coding CLIs on workers.

If config exists, parse and display it:

> **Current server config:**
> - Role: `[role]`
> - Provider: `[provider]`
> - Platform: `[platform]`
> - AI CLIs on workers: `[ai_clis_on_workers]` → will install: `[ai_clis]`

Verify the role is `orchestrator`. If it is `master`, warn:

> **Warning:** This server's role is `master`, not `orchestrator`. The cloud CLI may not be installed. You can still proceed if the cloud CLI is present, but provisioning workers from a master is not the standard setup. Continue anyway?


## Phase 2: Interview

> **Important:** Use `AskUserQuestion` (one call per question) for all questions in this phase. Do NOT use markdown checkbox lists. Each question maps to one `AskUserQuestion` call with up to 4 options (a built-in "Other" option is always available).

**Question 1 — Worker label/name:**

Ask via `AskUserQuestion` (free-text / open-ended): "What label or name should this worker have? (e.g. `worker-1`, `worker-us-east`, `worker-db`)"

**Question 2 — Region and size:**

First, fetch live options based on the provider in config:

**Hetzner:**
```bash
hcloud location list -o columns=name,city,country
hcloud server-type list -o columns=name,cores,memory,disk,cpu_type
```

**DigitalOcean:**
```bash
doctl compute region list --format Slug,Name,Available
doctl compute size list --format Slug,Memory,VCPUs,Disk,PriceMonthly | head -20
```

**AWS:**
```bash
aws ec2 describe-regions --query 'sort_by(Regions,&RegionName)[].RegionName' --output table
```

Then ask region and size each as a separate `AskUserQuestion` call. Present the top options (up to 4 most common/relevant ones) and rely on the built-in "Other" for anything not listed.

**Question 3 — AI coding CLIs on this worker:**

Use `AskUserQuestion` (single-select) with the question: "Install AI coding CLIs on this worker? (Config default: `[ai_clis_on_workers]` → `[ai_clis]`)"

Options:
1. Use config default
2. Install on this worker
3. Skip for this worker


## Phase 3: Provision the Worker

### Create the VPS

Use the provider from config. Run only the matching block.

**Hetzner:**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" 2>/dev/null || true
hcloud ssh-key create --name orchestrator-key \
  --public-key-from-file ~/.ssh/id_ed25519.pub 2>/dev/null || true

hcloud server create \
  --name <WORKER_LABEL> \
  --type <CHOSEN_SIZE> \
  --image ubuntu-24.04 \
  --location <CHOSEN_REGION> \
  --ssh-key orchestrator-key

WORKER_IP=$(hcloud server describe <WORKER_LABEL> -o format='{{.PublicNet.IPv4.IP}}')
# IP captured in $WORKER_IP for later steps — not echoed (see Privacy Rule).
```

**DigitalOcean:**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" 2>/dev/null || true
doctl compute ssh-key import orchestrator-key \
  --public-key-file ~/.ssh/id_ed25519.pub 2>/dev/null || true
KEY_ID=$(doctl compute ssh-key list --format ID,Name --no-header | grep orchestrator-key | awk '{print $1}')

doctl compute droplet create <WORKER_LABEL> \
  --region <CHOSEN_REGION> \
  --size <CHOSEN_SIZE> \
  --image ubuntu-24-04-x64 \
  --ssh-keys "$KEY_ID" \
  --wait

WORKER_IP=$(doctl compute droplet list --format Name,PublicIPv4 --no-header | grep <WORKER_LABEL> | awk '{print $2}')
# IP captured in $WORKER_IP for later steps — not echoed (see Privacy Rule).
```

**AWS:**
```bash
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query 'Parameter.Value' --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type <CHOSEN_SIZE> \
  --key-name vibe-key \
  --count 1 \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=<WORKER_LABEL>}]" \
  --query 'Instances[0].InstanceId' --output text)

# Wait for running state
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
WORKER_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
# IP captured in $WORKER_IP for later steps — not echoed (see Privacy Rule).
```

### Wait for SSH to be ready

```bash
echo "Waiting for SSH on the new worker..."   # IP kept in $WORKER_IP, not printed
until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 root@"$WORKER_IP" echo "ready" 2>/dev/null; do
  sleep 5
done
echo "SSH ready."
```

### Bootstrap the worker (Bun + platform agent)

```bash
ssh -o StrictHostKeyChecking=no root@"$WORKER_IP" bash -s << 'BOOTSTRAP'
export DEBIAN_FRONTEND=noninteractive
apt-get update -q
apt-get install -y -q curl git
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc 2>/dev/null || true
BOOTSTRAP
```

### Register with the deployment platform

This is the one step where the user must paste the worker IP into the platform UI, so showing it **once here** is allowed (see the Privacy Rule). The SSH public key is public and also needed for registration.

```bash
# Dokploy (if platform is dokploy or both):
# UI → Settings → Servers → Add Server → paste IP and your SSH public key
echo "Dokploy: open the Dokploy UI on port 3000 → Settings → Servers → Add Server"
echo "  Worker IP (needed for this manual step): $WORKER_IP"
echo "  SSH public key (paste into the UI): $(cat ~/.ssh/id_ed25519.pub)"

# Coolify (if platform is coolify or both):
# UI → Servers → Add → New Server → paste IP
echo "Coolify: open the Coolify UI on port 8000 → Servers → Add → New Server"
echo "  Worker IP (needed for this manual step): $WORKER_IP"
```

Pause and ask the user to complete the UI registration step before continuing. After they confirm, do not repeat the IP in later messages.

### Install AI coding CLIs on worker (if chosen)

*Skip if the user chose "Skip for this worker" in Phase 2.*

Install the binaries on the worker (auth is configured separately, below):

```bash
# Read which CLIs to install from config / user choice
ssh root@"$WORKER_IP" bash -s << 'AICLI'
source ~/.bashrc 2>/dev/null || true
# Claude Code (if in ai_clis)
bun add -g @anthropic-ai/claude-code
# Codex CLI (if in ai_clis)
bun add -g @openai/codex
AICLI
```

**Authentication — each CLI offers two equal paths** (full commands in [../vibe/references/server-setup.md](../vibe/references/server-setup.md) Step 7):

- **Subscription login:** Claude → put a `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`, run where a browser exists) in the worker's `~/.bashrc`. Codex → `codex login --device-auth` on the worker, or copy `~/.codex/auth.json` from a machine where you ran `codex login`.
- **API key:** add `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` to the worker's `~/.bashrc`.

Tokens and keys are secrets — set them on the worker (SSH in or via your secrets manager); do not print them in chat.


## Phase 4: Update Server Config

Append the new worker to the `workers` array in `/etc/vibemd/server.json`:

```bash
PROVISIONED_AT=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
CONFIG=$(cat /etc/vibemd/server.json)

# Use jq to append the worker entry
echo "$CONFIG" | jq \
  --arg name "<WORKER_LABEL>" \
  --arg ip "$WORKER_IP" \
  --arg provider "$(echo "$CONFIG" | jq -r .provider)" \
  --arg region "<CHOSEN_REGION>" \
  --arg size "<CHOSEN_SIZE>" \
  --arg at "$PROVISIONED_AT" \
  '.workers += [{name: $name, ip: $ip, provider: $provider, region: $region, size: $size, provisioned_at: $at}]' \
  > /tmp/vibemd-server.json && mv /tmp/vibemd-server.json /etc/vibemd/server.json

echo "Updated /etc/vibemd/server.json:"
cat /etc/vibemd/server.json
```


## Completion Checklist

- [ ] Server config read and validated
- [ ] Worker label, region, and size confirmed
- [ ] VPS created and SSH accessible
- [ ] Worker bootstrapped (Bun installed)
- [ ] Worker registered in Dokploy / Coolify UI
- [ ] AI coding CLIs installed on worker (if chosen)
- [ ] `/etc/vibemd/server.json` updated with new worker entry
