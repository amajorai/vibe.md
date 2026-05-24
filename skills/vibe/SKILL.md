---
name: vibe
description: "Server setup skill. Detects local vs. remote environment. If local, provisions a VPS on the chosen cloud provider. If already on a server, asks whether this is an orchestrator (manages more servers) or a standalone master (everything on one machine, like Coolify). Installs Bun, GitHub CLI, Dokploy/Coolify, and optional Wrangler. Ends by offering to install the ship and hardening skills."
argument-hint: [optional flags]
---

# Vibe

You are setting up a production server environment. Interview first, execute in one clean pass.

**Args:** {{args}}


## Phase 0: Auto-Update

*Skip if `{{args}}` contains `--no-update`, or if `SKILLS_AUTO_UPDATE: false` is set in your project CLAUDE.md.*

```bash
npx skills update vibe -y
```

If the skill was updated, stop here and tell the user: **"This skill was just updated. Re-run your command to use the new version."** Otherwise continue silently.

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
> **2. Which deployment platform?**
> - [ ] Dokploy — lightweight, Docker-native, self-hosted PaaS
> - [ ] Coolify — more features, great UI, self-hosted Heroku alternative
> - [ ] Both (Dokploy for apps, Coolify for databases/services)
>
> **3. Deploy to edge as well?**
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

Install the chosen cloud provider CLI and spin up a server.

### Hetzner

```bash
curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz | tar xz
sudo mv hcloud /usr/local/bin/ && hcloud version

# Authenticate (get token from cloud.hetzner.com → Security → API Tokens)
hcloud context create vibe

# Provision server
hcloud server create \
  --name vibe-server \
  --type cx22 \
  --image ubuntu-24.04 \
  --location nbg1 \
  --ssh-key ~/.ssh/id_rsa.pub

SERVER_IP=$(hcloud server ip vibe-server)
echo "Server ready at $SERVER_IP"
ssh root@$SERVER_IP
```

### OVH

```bash
curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
ovhcloud login

# List available VPS plans and order one
ovhcloud vps list
# Provision via OVH Control Panel or API
```

### AWS

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install && aws --version

aws configure
# Enter: Access Key ID, Secret Access Key, region (e.g. us-east-1), output format (json)

aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --instance-type t3.small \
  --key-name <YOUR_KEY_PAIR> \
  --security-group-ids <YOUR_SG_ID> \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=vibe-server}]'

SERVER_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=vibe-server" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
echo "Server ready at $SERVER_IP"
ssh ubuntu@$SERVER_IP
```

SSH into the server, then continue to Phase 3B.


## Phase 3B: Server Setup

Run all steps in order on the server.

### Step 1: System baseline

```bash
apt-get update && apt-get upgrade -y
apt-get install -y curl git unzip jq build-essential ca-certificates gnupg
```

### Step 2: Install Bun

```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun --version
```

### Step 3: Install GitHub CLI

```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
  https://cli.github.com/packages stable main" \
  | tee /etc/apt/sources.list.d/github-cli.list

apt-get update && apt-get install -y gh
gh auth login
# GitHub.com → HTTPS → paste a personal access token (classic, repo + workflow scopes)
```

### Step 4: Install Deployment Platform

**Dokploy** (if chosen):

```bash
curl -sSL https://dokploy.com/install.sh | sh
echo "Dokploy UI → http://$(curl -s ifconfig.me):3000"

bun add -g @dokploy/cli
dokploy --version

# Authenticate CLI (get API key from Dokploy UI → Settings → API)
dokploy auth -u http://localhost:3000 -t <YOUR_DOKPLOY_API_KEY>
```

**Coolify** (if chosen):

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
echo "Coolify UI → http://$(curl -s ifconfig.me):8000"

# Install Coolify CLI
go install github.com/coollabsio/coolify-cli/coolify@latest

# Authenticate (get token from Coolify UI → Security → API Tokens)
coolify context set-token self-hosted <YOUR_COOLIFY_TOKEN> \
  --url http://localhost:8000
```

### Step 5: Install Wrangler (if edge deployment chosen)

```bash
bun add -g wrangler
wrangler --version
wrangler login
```

### Step 6: Cloud CLI for Orchestration (orchestrator role only)

Install the same provider CLI on this server so it can provision more nodes:

```bash
# Hetzner
curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz \
  | tar xz -C /usr/local/bin/
hcloud context create orchestrator
# Paste Hetzner API token

# OVH
curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
ovhcloud login

# AWS
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && ./aws/install
aws configure
```

To register a new worker node with the deployment platform:

```bash
# Hetzner: spin up a new worker
hcloud server create \
  --name vibe-worker-$(date +%s) \
  --type cx22 \
  --image ubuntu-24.04 \
  --location nbg1

# Register in Dokploy: UI → Settings → Servers → Add Server → paste IP + SSH key
# Register in Coolify: UI → Servers → Add Server → paste IP
```


## Completion Checklist

- [ ] Environment detected and confirmed
- [ ] VPS provisioned and SSH'd into (if starting from local)
- [ ] System packages updated
- [ ] Bun installed
- [ ] GitHub CLI installed and authenticated
- [ ] Deployment platform installed (Dokploy / Coolify / both) + CLI authenticated
- [ ] Wrangler installed and logged in (if edge)
- [ ] Cloud CLI on server for orchestration (if orchestrator role)


## Final Step: Optional Skills

The server is ready. Two more things I can set up for you:

### 1. Install the `ship` skill?

The `ship` skill gives you a full-cycle development workflow: implement → verify → edge cases → tests → security review. Use it to build and ship features fast.

Check if it's already installed:

```bash
npx skills list 2>/dev/null | grep -q "^ship" && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

If not installed, ask the user: **"Would you like me to install the `ship` skill? It enables `/ship <feature>` for full-cycle development."**

If yes:
```bash
npx skills add amajorai/ship.md
```

### 2. Install and run the `hardening` skill?

The `hardening` skill secures this server: SSH hardening, fail2ban, UFW firewall, unattended upgrades, AppArmor, and more.

Check if it's already installed:

```bash
npx skills list 2>/dev/null | grep -q "^hardening" && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask the user: **"Would you like me to install and run the `hardening` skill to lock down this server?"**

If yes:
```bash
npx skills add amajorai/skills/skills/hardening
```

Then invoke `/hardening` to begin the server hardening interview.
