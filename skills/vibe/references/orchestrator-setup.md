# Cloud CLI for Orchestration (Step 8 — orchestrator role only)

**Only run this step if the user chose the orchestrator role** (the server should provision and manage other worker nodes). Skip it for a master/standalone server. The server itself runs headless Linux, so the Linux x86_64 binaries below are correct here (unlike Phase 3A, which runs on the local machine and may need a different OS/arch).

Install the CLI for **the single provider this orchestrator will manage** — run
only the block matching that provider. Do **not** run all four: `hcloud context
create`, `doctl auth init`, `ovhcloud login`, and `aws configure` are all
interactive and will hang/prompt if run blindly on a headless server, and you
only need one cloud's credentials here. Each block below avoids the interactive
prompt where possible.

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

## DigitalOcean

```bash
if ! command -v doctl >/dev/null 2>&1; then
  DOCTL_VERSION=$(curl -s https://api.github.com/repos/digitalocean/doctl/releases/latest \
    | grep '"tag_name"' | cut -d'"' -f4 | tr -d 'v')
  curl -fsSL "https://github.com/digitalocean/doctl/releases/latest/download/doctl-${DOCTL_VERSION}-linux-amd64.tar.gz" \
    | tar xz -C /usr/local/bin/
fi
# `doctl auth init` is interactive. Provide the token non-interactively instead.
# Get it from cloud.digitalocean.com → API → Tokens → Generate New Token (read+write).
export DIGITALOCEAN_ACCESS_TOKEN=<your-do-api-token>
echo 'export DIGITALOCEAN_ACCESS_TOKEN=<your-do-api-token>' >> ~/.bashrc
doctl auth init --access-token "$DIGITALOCEAN_ACCESS_TOKEN"
doctl compute droplet list   # verify auth works
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

Before spinning up a worker, fetch the live list of locations and sizes for your provider, then ask the user which to use. Run only the block matching your provider.

### Hetzner

```bash
# (One-time) register the orchestrator's public key so workers are reachable:
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" 2>/dev/null || true
hcloud ssh-key create --name orchestrator-key \
  --public-key-from-file ~/.ssh/id_ed25519.pub 2>/dev/null || true

# List available locations and server types, then ask the user to choose:
hcloud location list -o columns=name,city,country
hcloud server-type list -o columns=name,cores,memory,disk,cpu_type
# → Ask user: which location? which server type?

LOCATION=<chosen-location>
SERVER_TYPE=<chosen-server-type>

hcloud server create \
  --name vibe-worker-$(date +%s) \
  --type "$SERVER_TYPE" \
  --image ubuntu-24.04 \
  --location "$LOCATION" \
  --ssh-key orchestrator-key
```

### DigitalOcean

```bash
# (One-time) register the orchestrator's public key:
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" 2>/dev/null || true
doctl compute ssh-key import orchestrator-key \
  --public-key-file ~/.ssh/id_ed25519.pub 2>/dev/null || true
KEY_ID=$(doctl compute ssh-key list --format ID,Name --no-header | grep orchestrator-key | awk '{print $1}')

# List available regions and sizes, then ask the user to choose:
doctl compute region list --format Slug,Name,Available
doctl compute size list --format Slug,Memory,VCPUs,Disk,PriceMonthly | awk 'NR==1 || $NF+0 > 0'
# → Ask user: which region? which size?

REGION=<chosen-region>
SIZE=<chosen-size>

doctl compute droplet create vibe-worker-$(date +%s) \
  --region "$REGION" \
  --size "$SIZE" \
  --image ubuntu-24-04-x64 \
  --ssh-keys "$KEY_ID" \
  --wait

WORKER_IP=$(doctl compute droplet list --format Name,PublicIPv4 --no-header | grep vibe-worker | tail -1 | awk '{print $2}')
echo "Worker ready at $WORKER_IP"
```

### AWS

```bash
# List available regions, then ask the user to choose:
aws ec2 describe-regions \
  --query 'sort_by(Regions, &RegionName)[].RegionName' --output table
# → Ask user: which region?

export AWS_DEFAULT_REGION=<chosen-region>

# List instance types available in that region, then ask the user to choose:
aws ec2 describe-instance-type-offerings \
  --location-type region \
  --filters "Name=location,Values=$AWS_DEFAULT_REGION" \
  --query 'sort_by(InstanceTypeOfferings, &InstanceType)[?starts_with(InstanceType, `t3`) || starts_with(InstanceType, `m6i`)].InstanceType' \
  --output table
# → Ask user: which instance type?

INSTANCE_TYPE=<chosen-instance-type>

AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query 'Parameter.Value' --output text)

aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --key-name vibe-key \
  --security-group-ids <YOUR_SG_ID> \
  --count 1 \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=vibe-worker-$(date +%s)}]"
```

---

After the droplet/instance is running, register it in your deployment platform:

```bash
# Register in Dokploy: UI → Settings → Servers → Add Server → paste IP + SSH key
# Register in Coolify: UI → Servers → Add Server → paste IP
```

### Install AI Coding CLIs on worker (if "workers" or "both" was chosen in Phase 2)

*Skip if "orchestrator only" or "None / Skip" was chosen.*

SSH into the worker and install whichever CLIs were selected:

```bash
ssh root@<WORKER_IP> bash -s << 'EOF'
# Claude Code (if chosen)
bun add -g @anthropic-ai/claude-code
echo 'export ANTHROPIC_API_KEY=<your-anthropic-api-key>' >> ~/.bashrc

# Codex CLI (if chosen)
bun add -g @openai/codex
echo 'export OPENAI_API_KEY=<your-openai-api-key>' >> ~/.bashrc
EOF
```

> **A freshly created worker is a bare OS — it has no Bun, no deployment
> platform, nothing.** Creating the server above only provisions it; it does
> **not** run Phase 3B on it. To make a worker usable you have two options:
>
> 1. **Let the deployment platform manage it (most common):** just register the
>    worker's IP in Dokploy/Coolify (commands above). The platform installs its
>    own agent over SSH; you do **not** need to run vibemd on the worker.
> 2. **Set it up as a full vibemd server:** if you want Bun / GitHub CLI / the
>    platform installed directly on the worker, SSH into it and **run the `vibemd`
>    skill again on that machine**, choosing the **Master/standalone** role.
>
> Pick option 1 unless you have a specific reason to fully provision the worker.
