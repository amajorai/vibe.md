---
name: vibe
description: "Ultimate vibe coding setup. Interview-driven, no-friction full-stack dev and deployment. Asks the user upfront which cloud provider (Hetzner/OVH/AWS) and deployment platform (Dokploy/Coolify) they want, then fully installs and configures the chosen stack: Better T Stack, GitHub CLI, Wrangler, and all relevant CLIs. Detects local vs. server environment automatically."
argument-hint: [project name or description]
---

# Vibe

You are setting up the ultimate vibe coding environment. Start with a short interview to lock in all decisions, then execute the setup in one clean pass.

**Project:** {{args}}


## Phase 0: Environment Detection

First, silently check the environment:

```bash
uname -a && cat /etc/os-release 2>/dev/null || sw_vers 2>/dev/null
curl -s --max-time 2 http://169.254.169.254/latest/meta-data/instance-id 2>/dev/null && echo "IS_AWS" || true
curl -s --max-time 2 http://169.254.0.1/metadata 2>/dev/null | grep -qi hetzner && echo "IS_HETZNER" || true
echo "USER=$USER HOME=$HOME"
```

Determine:
- **Local dev machine** (macOS, Windows, desktop Linux with GUI): → need to provision a VPS first
- **Already on a VPS / headless server**: → skip straight to setup

Confirm with the user: "I see you're on [local machine / a server]. Is that right?"


## Phase 1: Interview — Lock In All Choices

Ask all questions up front in a single message. Do not start any installation until answers are confirmed.


**Ask the user:**

> Let's configure your vibe stack. Answer these and I'll set everything up:
>
> **1. Where are you running this?**
> - [ ] Local dev machine (I need to provision a VPS)
> - [ ] Already on a server (skip provisioning)
>
> **2. Which cloud provider do you want?**
> - [ ] Hetzner Cloud — cheapest, EU-based (~€4/mo for CX22). Best default.
> - [ ] OVH — good value, global PoPs
> - [ ] AWS — most ecosystem, free tier (t3.micro)
> - [ ] I already have a server (skip provisioning)
>
> **3. Which deployment platform do you want?**
> - [ ] Dokploy — lightweight, Docker-native, self-hosted PaaS
> - [ ] Coolify — more features, great UI, self-hosted Heroku alternative
> - [ ] Both (Dokploy for apps, Coolify for databases/services)
>
> **4. What role should this server play?**
> - [ ] Root orchestrator — manages deployments AND can spin up more servers (install cloud CLIs)
> - [ ] Standalone app server — just runs my app
>
> **5. Deploy to edge as well?**
> - [ ] Yes — install Wrangler (Cloudflare Workers / Pages)
> - [ ] No

Once the user answers, proceed to the appropriate phases below.


## Phase 2A: Provision a VPS (local machine only)

Install the chosen cloud provider CLI, then spin up a server.

### Hetzner

```bash
# Install hcloud CLI
curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz | tar xz
sudo mv hcloud /usr/local/bin/ && hcloud version

# Authenticate (get token from cloud.hetzner.com → Security → API Tokens)
hcloud context create vibe

# Provision server
hcloud server create \
  --name vibe-root \
  --type cx22 \
  --image ubuntu-24.04 \
  --location nbg1 \
  --ssh-key ~/.ssh/id_rsa.pub

SERVER_IP=$(hcloud server ip vibe-root)
echo "Server ready at $SERVER_IP"
ssh root@$SERVER_IP
```

### OVH

```bash
# Install ovhcloud CLI
curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
ovhcloud login

# List available VPS plans and order one
ovhcloud vps list
# Then provision via OVH Control Panel or API — CLI provisioning varies by account type
```

### AWS

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install && aws --version

# Authenticate
aws configure
# Enter: Access Key ID, Secret Access Key, region (e.g. us-east-1), output format (json)

# Launch t3.small
aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --instance-type t3.small \
  --key-name <YOUR_KEY_PAIR> \
  --security-group-ids <YOUR_SG_ID> \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=vibe-root}]'

# Get IP
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=vibe-root" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
```

SSH into the server, then continue to Phase 2B.


## Phase 2B: Server Setup

You are now on the VPS. Run all steps in order.

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
# GitHub.com → HTTPS → paste a personal access token (classic, with repo + workflow scopes)
```

### Step 4: Install Deployment Platform

**Dokploy** (chosen or both):

```bash
# Install Dokploy (Docker-based, self-hosted PaaS)
curl -sSL https://dokploy.com/install.sh | sh
echo "Dokploy UI → http://$(curl -s ifconfig.me):3000"

# Install Dokploy CLI
bun add -g @dokploy/cli
dokploy --version

# Authenticate CLI (get API key from Dokploy UI → Settings → API)
dokploy auth -u http://localhost:3000 -t <YOUR_DOKPLOY_API_KEY>
# Or use env vars: DOKPLOY_URL and DOKPLOY_API_KEY
```

**Coolify** (chosen or both):

```bash
# Install Coolify (self-hosted Heroku alternative)
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
echo "Coolify UI → http://$(curl -s ifconfig.me):8000"

# Install Coolify CLI
# Linux/macOS via Go:
go install github.com/coollabsio/coolify-cli/coolify@latest
# Or macOS via Homebrew:
# brew install coollabsio/coolify-cli/coolify-cli

# Authenticate CLI (get token from Coolify UI → Security → API Tokens)
coolify context set-token self-hosted <YOUR_COOLIFY_TOKEN> \
  --url http://localhost:8000
```

### Step 5: Install Wrangler (if edge deployment chosen)

```bash
bun add -g wrangler
wrangler --version
wrangler login
# Opens browser — log in with your Cloudflare account
```

### Step 6: Cloud CLIs for Orchestration (root orchestrator only)

Install the same provider CLI chosen in Phase 2A on this server so it can spin up more nodes:

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


## Phase 3: Scaffold the Project

```bash
bun create better-t-stack@latest {{args}}
cd {{args}}

# CLI will prompt:
# - Frontend: Next.js or TanStack Router
# - Backend: tRPC or Hono
# - Database: SQLite (libsql), PostgreSQL, or MySQL
# - Auth: Better Auth or none
# - Add-ons: Tailwind, Shadcn, PWA, Biome, etc.

bun install
```

Create the GitHub repo:

```bash
gh repo create {{args}} --public --source=. --remote=origin --push
```


## Phase 4: Spec-Driven Development

Every feature starts as a GitHub issue. This is the core loop.

### Create a spec issue

```bash
gh issue create \
  --title "feat: <feature name>" \
  --body "## Goal
<what this achieves for the user>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] <criterion 3>

## Out of scope
- <what we are NOT doing>"
```

### Implement

Use `/ship <issue description or URL>` — runs the full implement → verify → simplify → security review cycle.

For multiple issues in parallel:
```bash
gh issue list --label "ready" --json number,title,body
# Feed to /batch for parallel worktree execution
```

### PR + auto-deploy

```bash
gh pr create --title "feat: <name>" --body "Closes #<issue-number>"
# Merge → deployment platform auto-deploys via webhook
```


## Phase 5: Deploy

### Via Dokploy CLI

```bash
# List projects
dokploy project all

# Deploy an existing application
dokploy application deploy --applicationId <APP_ID>

# Create a new application linked to your repo
dokploy application create \
  --projectId <PROJECT_ID> \
  --name {{args}} \
  --buildType nixpacks

# Check status
dokploy application one --applicationId <APP_ID>
```

### Via Coolify CLI

```bash
# List applications
coolify list applications

# Trigger a deployment
coolify deploy application <APP_UUID>

# Check logs
coolify logs application <APP_UUID>
```

### Via Wrangler (edge)

```bash
# Workers
wrangler deploy

# Pages
wrangler pages deploy ./dist --project-name={{args}}

# Secrets
wrangler secret put DATABASE_URL
```


## Phase 6: Scale — Spin Up More Servers (orchestrator only)

```bash
# Hetzner: new worker node
hcloud server create \
  --name vibe-worker-$(date +%s) \
  --type cx22 \
  --image ubuntu-24.04 \
  --location nbg1

# Register in Dokploy: UI → Settings → Servers → Add Server → paste IP + SSH key
# Register in Coolify: UI → Servers → Add Server → paste IP
```


## Daily Workflow Cheat Sheet

```
gh issue create --title "feat: X"      # spec it
/ship <issue description>               # build it  
gh pr create                            # review it
# merge → auto-deploy fires             # ship it
wrangler deploy                         # or push to edge
dokploy application deploy --applicationId <ID>   # or trigger manually
coolify deploy application <UUID>       # or via Coolify
```


## Completion Checklist

- [ ] Environment detected and confirmed
- [ ] Provider chosen and CLI authenticated
- [ ] VPS provisioned (if starting from local)
- [ ] Bun + GitHub CLI installed on server
- [ ] Deployment platform installed (Dokploy / Coolify / both) + CLI authenticated
- [ ] Wrangler installed and logged in (if edge)
- [ ] Cloud CLI on server for orchestration (if orchestrator mode)
- [ ] Better T Stack project scaffolded
- [ ] GitHub repo created and linked
- [ ] Deployment webhook configured (auto-deploy on merge to main)
- [ ] First spec issue created
- [ ] End-to-end deploy tested
