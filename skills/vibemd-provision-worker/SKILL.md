---
name: vibe-provision-worker
description: "Provision a new worker server from an orchestrator. Reads /etc/vibemd/server.json for provider, platform, and AI CLI preferences, spins up a new VPS, registers it with Dokploy or Coolify, optionally installs AI coding CLIs, and records the worker in server.json."
argument-hint: [optional flags]
---

# Provision Worker

You are running on an **orchestrator server** and provisioning a new worker node. Read the server config first — it tells you the provider, platform, and preferences so you don't have to ask again.

**Args:** {{args}}


## Phase 1: Read Server Config

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
echo "Worker IP: $WORKER_IP"
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
echo "Worker IP: $WORKER_IP"
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
echo "Worker IP: $WORKER_IP"
```

### Wait for SSH to be ready

```bash
echo "Waiting for SSH on $WORKER_IP..."
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

```bash
# Dokploy (if platform is dokploy or both):
# UI → Settings → Servers → Add Server → paste IP and your SSH public key
echo "Dokploy: open http://$(curl -s ifconfig.me):3000 → Settings → Servers → Add Server"
echo "  IP: $WORKER_IP"
echo "  SSH key: $(cat ~/.ssh/id_ed25519.pub)"

# Coolify (if platform is coolify or both):
# UI → Servers → Add → New Server → paste IP
echo "Coolify: open http://$(curl -s ifconfig.me):8000 → Servers → Add → New Server"
echo "  IP: $WORKER_IP"
```

Pause and ask the user to complete the UI registration step before continuing.

### Install AI coding CLIs on worker (if chosen)

*Skip if the user chose "Skip for this worker" in Phase 2.*

```bash
# Read which CLIs to install from config / user choice
ssh root@"$WORKER_IP" bash -s << 'AICLI'
source ~/.bashrc 2>/dev/null || true

# Claude Code (if in ai_clis)
bun add -g @anthropic-ai/claude-code
echo 'export ANTHROPIC_API_KEY=<your-anthropic-api-key>' >> ~/.bashrc

# Codex CLI (if in ai_clis)
bun add -g @openai/codex
echo 'export OPENAI_API_KEY=<your-openai-api-key>' >> ~/.bashrc
AICLI
```

Remind the user: **"API keys were added as placeholders in `~/.bashrc` on the worker. SSH in and replace `<your-*-api-key>` with real values."**


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
