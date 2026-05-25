# Cloud CLI for Orchestration (Step 7 — orchestrator role only)

**Only run this step if the user chose the orchestrator role** (the server should provision and manage other worker nodes). Skip it for a master/standalone server. The server itself runs headless Linux, so the Linux x86_64 binaries below are correct here (unlike Phase 3A, which runs on the local machine and may need a different OS/arch).

Install the CLI for **the single provider this orchestrator will manage** — run
only the block matching that provider. Do **not** run all three: `hcloud context
create`, `ovhcloud login`, and `aws configure` are all interactive and will
hang/prompt if run blindly on a headless server, and you only need one cloud's
credentials here. Each block below avoids the interactive prompt where possible.

## Hetzner

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

## OVH

```bash
if ! command -v ovhcloud >/dev/null 2>&1; then
  curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
fi
ovhcloud login
# Note: OVH still cannot script VPS creation (see Phase 3A) — the CLI here is for
# listing/inspecting; new VPS orders go through the OVH Control Panel/API.
```

## AWS

```bash
# (./aws/install writes to /usr/local/bin and needs root)
if ! command -v aws >/dev/null 2>&1; then
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip -o awscliv2.zip && sudo ./aws/install
fi
aws configure
```

## Registering a new worker node

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
