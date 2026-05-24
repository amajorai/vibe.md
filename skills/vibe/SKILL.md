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

### Hetzner

The Hetzner CLI ships per-platform binaries. Pick the one for your OS — the `*-linux-amd64*` asset is **only** for x86_64 Linux:

- macOS: `brew install hcloud`
- Linux (x86_64): `hcloud-linux-amd64.tar.gz` / Linux (arm64): `hcloud-linux-arm64.tar.gz`
- Windows: download `hcloud-windows-amd64.zip` from the [releases page](https://github.com/hetznercloud/cli/releases/latest), or `scoop install hcloud`

```bash
# Example for Linux x86_64 — substitute the asset matching your OS/arch:
curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz | tar xz
sudo mv hcloud /usr/local/bin/ && hcloud version

# Authenticate (get token from cloud.hetzner.com → Security → API Tokens)
hcloud context create vibe

# Make sure you have a local SSH key pair first. If you have neither
# ~/.ssh/id_ed25519.pub nor ~/.ssh/id_rsa.pub, generate one (no passphrase):
if [ ! -f ~/.ssh/id_ed25519.pub ] && [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
fi
# Pick whichever public key exists (prefer ed25519):
PUBKEY=$( [ -f ~/.ssh/id_ed25519.pub ] && echo ~/.ssh/id_ed25519.pub || echo ~/.ssh/id_rsa.pub )

# Register your public SSH key with Hetzner first (one-time). This uploads the
# key file and gives it a NAME we reference below. The `|| true` makes a re-run
# safe — Hetzner errors if a key with this name already exists.
hcloud ssh-key create --name vibe-key --public-key-from-file "$PUBKEY" 2>/dev/null || true

# Provision server. --ssh-key takes the NAME (or fingerprint) of a key already
# registered in Hetzner — NOT a file path.
hcloud server create \
  --name vibe-server \
  --type cx22 \
  --image ubuntu-24.04 \
  --location nbg1 \
  --ssh-key vibe-key

SERVER_IP=$(hcloud server ip vibe-server)
echo "Server ready at $SERVER_IP"

# Hetzner Ubuntu images log in as root:
ssh root@"$SERVER_IP"
```

### OVH

OVH does not support fully scripted VPS provisioning via a simple CLI flow. The CLI is useful for listing and inspecting, but **you must order/create the VPS in the OVH Control Panel** (or via the OVH API).

```bash
curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
ovhcloud login

# Inspect existing VPS instances:
ovhcloud vps list
```

To provision: go to the [OVH Control Panel](https://www.ovh.com/manager/) → order a VPS → choose Ubuntu, add your SSH key during checkout, and note the IP address it assigns. Because there is no provisioning command, this is a **manual step** — wait until the Control Panel shows the VPS as delivered/running and gives you an IP before continuing.

Then connect (note the SSH user OVH assigns differs by image — it is often `ubuntu` or `debian`, shown in the Control Panel; the root account is usually disabled for direct login):

```bash
SERVER_IP=<your-ovh-vps-ip>
ssh ubuntu@"$SERVER_IP"   # adjust the user to match what the Control Panel shows
```

Once that SSH connection succeeds you are on the server — proceed to the "Transition to the server" note below and then Phase 3B, exactly like the other providers.

### AWS

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# -o overwrites without an interactive "replace?" prompt on a re-run; the
# `|| ... --update` handles "AWS CLI already installed" so a re-run won't error.
unzip -o awscliv2.zip && { sudo ./aws/install || sudo ./aws/install --update; } && aws --version
# (on macOS use the .pkg installer from https://aws.amazon.com/cli/ instead)

aws configure
# Enter: Access Key ID, Secret Access Key, region (e.g. us-east-1), output format (json)

# IMPORTANT: AMI IDs are region-specific. The ID below is Ubuntu 22.04 LTS in
# us-east-1 only — it will NOT work in other regions. Look up the current AMI
# for your region first, e.g.:
#   aws ec2 describe-images --owners 099720109477 \
#     --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*' \
#               'Name=state,Values=available' \
#     --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text
# Or use an SSM public parameter (resolves per-region automatically):
#   --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id

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

# AWS Ubuntu AMIs log in as the 'ubuntu' user (Amazon Linux uses 'ec2-user'):
ssh ubuntu@"$SERVER_IP"
```

### Transition to the server (all providers)

After provisioning, you must move from your **local machine** onto the **remote server**. SSH in using the connect command shown for your provider above (`ssh root@"$SERVER_IP"` for Hetzner, `ssh ubuntu@...` for AWS/OVH).

Once that SSH command succeeds, your shell prompt changes (you are now on the server) and **every Phase 3B command runs there**, not on your local machine. The `$SERVER_IP` variable set during provisioning lives only in your local shell and is **not** carried over the SSH connection — that's fine, Phase 3B does not need it.

**If SSH fails**, do not continue to Phase 3B yet. Common causes and fixes:

- *Connection refused / timed out:* the server may still be booting. Wait 30–60s and retry. On AWS, confirm the security group allows inbound TCP 22 from your IP.
- *Permission denied (publickey):* you are using the wrong key or the wrong user. Try `ssh -i ~/.ssh/<your-key> <user>@"$SERVER_IP"`, and verify the user matches the provider (Hetzner `root`, AWS Ubuntu `ubuntu`, OVH usually `ubuntu`/`debian`).
- *No public key registered:* re-check that the key you uploaded to the provider matches your local `~/.ssh/*.pub`.
- *Host key changed warning:* if you reused an IP, run `ssh-keygen -R "$SERVER_IP"` then retry.

Resolve SSH before proceeding — all remaining setup happens inside that session.


## Phase 3B: Server Setup

Run all steps in order **on the server** (inside your SSH session — not on your local machine). If you provisioned from local in Phase 3A, you should already be connected via `ssh`. These commands assume you are the `root` user; if your provider logs you in as a non-root user (e.g. `ubuntu`), prefix the `apt-get`, `dd`, `tee`, and install commands with `sudo`.

### Step 1: System baseline

```bash
# DEBIAN_FRONTEND=noninteractive + the dpkg options keep apt from stopping on
# interactive prompts (e.g. "a new version of /etc/... is available"), which
# would otherwise hang the upgrade over SSH.
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" upgrade
apt-get install -y curl git unzip jq build-essential ca-certificates gnupg
```

### Step 2: Install Bun

```bash
curl -fsSL https://bun.sh/install | bash
# The installer appends to your shell rc file. Re-source whichever one applies,
# or simply start a new shell session to pick up Bun on your PATH:
source ~/.bashrc 2>/dev/null || source ~/.zshrc 2>/dev/null || true
# If `bun` is still not found, open a fresh login shell (e.g. log out and back
# in, or run `exec $SHELL -l`).
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

# On a headless server, `gh auth login` (interactive) tries to open a browser and
# is awkward over SSH. Authenticate non-interactively with a personal access
# token instead.
#
# Where to get GH_TOKEN: on github.com go to Settings → Developer settings →
# Personal access tokens → Tokens (classic) → Generate new token, and grant the
# `repo` + `workflow` scopes. Copy the token (it is shown only once) and paste it
# in place of the placeholder below — replace the whole <...> including brackets.
export GH_TOKEN=<your-personal-access-token>
echo "$GH_TOKEN" | gh auth login --with-token
gh auth status
```

### Step 4: Install Deployment Platform

Install based on the user's Phase 2 answer:

- Chose **Dokploy** → run the Dokploy block only.
- Chose **Coolify** → run the Coolify block only.
- Chose **Both** → run **both** blocks, in order (Dokploy first, then Coolify). They listen on different ports (Dokploy 3000, Coolify 8000) so they can coexist on one server — see the port-conflict note after the blocks.

**Dokploy** (if chosen, or if "Both"):

```bash
curl -sSL https://dokploy.com/install.sh | sh
echo "Dokploy UI → http://$(curl -s ifconfig.me):3000"

bun add -g @dokploy/cli
dokploy --version
```

Before authenticating the CLI, Dokploy must be fully started (it can take a
minute or two after install to come up) AND you must obtain an API key from the
web UI — which is only reachable once it is running. So:

1. Open the Dokploy UI (`http://<SERVER_IP>:3000`), complete first-run setup, then
   go to **Settings → API** and create a key.
2. Wait for the service to respond, then authenticate:

```bash
# Poll until the Dokploy UI is up (max ~2 min), then authenticate:
until curl -fsS --max-time 2 http://localhost:3000 >/dev/null 2>&1; do
  echo "Waiting for Dokploy to start..."; sleep 5
done
dokploy auth -u http://localhost:3000 -t <YOUR_DOKPLOY_API_KEY>
```

**Coolify** (if chosen, or if "Both"):

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
echo "Coolify UI → http://$(curl -s ifconfig.me):8000"

# The Coolify CLI is a Go program installed via `go install`, so Go must be
# present. Install it if missing:
if ! command -v go >/dev/null 2>&1; then
  curl -fsSL https://go.dev/dl/go1.22.5.linux-amd64.tar.gz -o /tmp/go.tar.gz
  rm -rf /usr/local/go && tar -C /usr/local -xzf /tmp/go.tar.gz
  echo 'export PATH="$PATH:/usr/local/go/bin:$HOME/go/bin"' >> ~/.bashrc
fi

# Put the Go toolchain on PATH for THIS shell first. When Go was just installed
# above, the .bashrc edit only affects FUTURE shells, so `go` is not yet callable
# here without this line — `go env`/`go install` below would fail otherwise.
export PATH="$PATH:/usr/local/go/bin"

# Now also put the Go install bin dir on PATH. Even when Go was already present,
# `go install` drops the `coolify` binary in $GOBIN (or $GOPATH/bin), which may
# not be on PATH yet, so the `coolify` command below would otherwise fail with
# "command not found". GOBIN is empty by default → fall back to $GOPATH/bin.
GO_BIN_DIR="$(go env GOBIN)"
[ -n "$GO_BIN_DIR" ] || GO_BIN_DIR="$(go env GOPATH)/bin"
export PATH="$PATH:$GO_BIN_DIR"

# Install Coolify CLI
go install github.com/coollabsio/coolify-cli/coolify@latest

# Like Dokploy, wait for the Coolify UI to start and create a token in the UI
# (Security → API Tokens) before authenticating:
until curl -fsS --max-time 2 http://localhost:8000 >/dev/null 2>&1; do
  echo "Waiting for Coolify to start..."; sleep 5
done
coolify context set-token self-hosted <YOUR_COOLIFY_TOKEN> \
  --url http://localhost:8000
```

> **Running Both on one server:** Dokploy (port 3000) and Coolify (port 8000)
> use different ports, so they do not collide there. Be aware they are **both
> full PaaS systems that manage Docker**, and each wants to own the reverse proxy
> on ports **80/443** (Dokploy uses Traefik, Coolify uses its own proxy). They
> cannot both bind 80/443 at the same time. If you run Both on one host, only one
> proxy can serve HTTP/HTTPS — plan to use Dokploy for apps and reach Coolify
> only via its `:8000` UI (or put them on separate servers). For most users a
> single platform is simpler; only pick Both if you specifically need it.

### Step 5: Open the firewall for the platform UIs

The deployment-platform web UIs (and your apps) are not reachable until their
ports are open. Open them at **both** layers that apply:

```bash
# Host firewall (UFW). Only run if UFW is active/installed; harmless if not.
if command -v ufw >/dev/null 2>&1; then
  ufw allow 22/tcp          # keep SSH open FIRST so you don't lock yourself out
  ufw allow 80/tcp
  ufw allow 443/tcp
  ufw allow 3000/tcp        # Dokploy UI  (skip if Dokploy not installed)
  ufw allow 8000/tcp        # Coolify UI  (skip if Coolify not installed)
  ufw --force enable
  ufw status
fi
```

You must **also** open these ports at the provider/cloud level, which a host
firewall cannot do:

- **Hetzner:** Cloud Console → Firewalls (or `hcloud firewall ...`), or none by default (all open).
- **AWS:** edit the instance's **Security Group** to allow inbound TCP 22, 80, 443, 3000, 8000.
- **OVH:** Control Panel → network/firewall settings for the VPS.

Restrict 3000/8000 to your own IP where possible — these admin UIs should not be
world-open in production. (The `hardening` skill offered at the end tightens this
further.)

### Step 6: Install Wrangler (if edge deployment chosen)

```bash
bun add -g wrangler
wrangler --version
```

`wrangler login` starts a **browser-based OAuth flow** and will not work on a
headless server without a GUI. Use one of these instead:

- **Recommended (token):** create a Cloudflare API token (Cloudflare dashboard →
  My Profile → API Tokens, with Workers/Pages/DNS permissions) and export it.
  Wrangler reads it automatically — no `wrangler login` needed:

  ```bash
  export CLOUDFLARE_API_TOKEN=<your-cloudflare-api-token>
  echo 'export CLOUDFLARE_API_TOKEN=<your-cloudflare-api-token>' >> ~/.bashrc
  wrangler whoami   # verify auth
  ```

- **Or copy local credentials:** run `wrangler login` on your local machine,
  then copy the credentials over (path varies by version, commonly
  `~/.config/.wrangler/` or `~/.wrangler/config/`):

  ```bash
  # from your LOCAL machine:
  # scp -r ~/.config/.wrangler root@"$SERVER_IP":~/.config/.wrangler
  ```

### Step 7: Cloud CLI for Orchestration (orchestrator role only)

**Only run this step if the user chose the orchestrator role** (the server should provision and manage other worker nodes). Skip it for a master/standalone server. The server itself runs headless Linux, so the Linux x86_64 binaries below are correct here (unlike Phase 3A, which runs on the local machine and may need a different OS/arch).

Install the CLI for **the single provider this orchestrator will manage** — run
only the block matching that provider. Do **not** run all three: `hcloud context
create`, `ovhcloud login`, and `aws configure` are all interactive and will
hang/prompt if run blindly on a headless server, and you only need one cloud's
credentials here. Each block below avoids the interactive prompt where possible.

**Hetzner:**

```bash
# (this server is Linux x86_64, so the linux-amd64 asset is correct here)
if ! command -v hcloud >/dev/null 2>&1; then
  curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz \
    | tar xz -C /usr/local/bin/
fi
# `hcloud context create` prompts for the token interactively, which hangs on a
# headless server. Provide it non-interactively instead. Get the token from
# cloud.hetzner.com → Security → API Tokens, then either:
#   (a) export it for hcloud to read directly (no context needed):
export HCLOUD_TOKEN=<your-hetzner-api-token>
echo 'export HCLOUD_TOKEN=<your-hetzner-api-token>' >> ~/.bashrc
hcloud server list   # verify auth works
#   (b) or pipe it into context create (uncomment if you prefer a named context):
# echo "<your-hetzner-api-token>" | hcloud context create orchestrator
```

**OVH:**

```bash
if ! command -v ovhcloud >/dev/null 2>&1; then
  curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
fi
ovhcloud login
# Note: OVH still cannot script VPS creation (see Phase 3A) — the CLI here is for
# listing/inspecting; new VPS orders go through the OVH Control Panel/API.
```

**AWS:**

```bash
# (./aws/install writes to /usr/local/bin and needs root)
if ! command -v aws >/dev/null 2>&1; then
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip -o awscliv2.zip && sudo ./aws/install
fi
aws configure
```

To register a new worker node with the deployment platform:

```bash
# Hetzner: register this orchestrator's public key with Hetzner (one-time), so
# new workers are reachable by key. Generate one first if needed:
#   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
hcloud ssh-key create --name orchestrator-key \
  --public-key-from-file ~/.ssh/id_ed25519.pub 2>/dev/null || true

# Spin up a new worker. --ssh-key takes the NAME of a key already registered in
# Hetzner (not a file path), so the orchestrator can SSH in to register it.
hcloud server create \
  --name vibe-worker-$(date +%s) \
  --type cx22 \
  --image ubuntu-24.04 \
  --location nbg1 \
  --ssh-key orchestrator-key

# Register in Dokploy: UI → Settings → Servers → Add Server → paste IP + SSH key
# Register in Coolify: UI → Servers → Add Server → paste IP
```

> **A freshly created worker is a bare OS — it has no Bun, no deployment
> platform, nothing.** Creating the server above only provisions it; it does
> **not** run Phase 3B on it. To make a worker usable you have two options:
>
> 1. **Let the deployment platform manage it (most common):** just register the
>    worker's IP in Dokploy/Coolify (commands above). The platform installs its
>    own agent over SSH; you do **not** need to run vibe on the worker.
> 2. **Set it up as a full vibe server:** if you want Bun / GitHub CLI / the
>    platform installed directly on the worker, SSH into it and **run the `vibe`
>    skill again on that machine**, choosing the **Master/standalone** role.
>
> Pick option 1 unless you have a specific reason to fully provision the worker.


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

The `ship` skill gives you a full-cycle development workflow: implement → verify → edge cases → tests → security review. Use it to build and ship features fast.

Check if it's already installed (`npx skills` is on-demand — no global install required):

```bash
npx --yes skills list 2>/dev/null | grep -qE '^ship$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

If not installed, ask the user: **"Would you like me to install the `ship` skill? It enables `/ship <feature>` for full-cycle development."**

If yes:
```bash
npx --yes skills add amajorai/ship.md -a claude-code -y
```

### 2. Install and run the `hardening` skill?

The `hardening` skill secures this server: SSH hardening, fail2ban, UFW firewall, unattended upgrades, AppArmor, and more.

Check if it's already installed:

```bash
npx --yes skills list 2>/dev/null | grep -qE '^hardening$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask the user: **"Would you like me to install and run the `hardening` skill to lock down this server?"**

If yes:
```bash
npx --yes skills add amajorai/skills/skills/hardening -a claude-code -y
```

A newly added skill is not picked up by the current Claude Code session. Tell the user exactly how to run it:

> The `hardening` skill is installed. To run it, **start a new Claude Code session** (or reload skills if your client supports it), then invoke it with `/hardening`. It will begin the server hardening interview.

Do not attempt to invoke `/hardening` yourself in this session — it was just installed and is not yet loaded.
