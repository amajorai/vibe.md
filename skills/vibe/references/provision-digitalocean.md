# DigitalOcean Provisioning

Install `doctl`:

- macOS: `brew install doctl`
- Linux: download from [GitHub releases](https://github.com/digitalocean/doctl/releases/latest) — grab the `doctl-*-linux-amd64.tar.gz` asset
- Windows: `scoop install doctl` or download from the releases page

```bash
# Linux install example (auto-detects latest version):
DOCTL_VERSION=$(curl -s https://api.github.com/repos/digitalocean/doctl/releases/latest \
  | grep '"tag_name"' | cut -d'"' -f4 | tr -d 'v')
curl -fsSL "https://github.com/digitalocean/doctl/releases/latest/download/doctl-${DOCTL_VERSION}-linux-amd64.tar.gz" | tar xz
sudo mv doctl /usr/local/bin/ && doctl version

# Authenticate (get API token from cloud.digitalocean.com → API → Tokens → Generate New Token)
doctl auth init
```

### Step 1: List available regions

Run this and show the full output to the user:

```bash
doctl compute region list --format Slug,Name,Available
```

Ask the user: **"Which region would you like? (enter the Slug, e.g. `nyc3`, `sfo3`, `lon1`, `fra1`, `sgp1`)"**

### Step 2: List available droplet sizes

Run this and show the output to the user:

```bash
REGION=<chosen-region>
doctl compute size list --format Slug,Memory,VCPUs,Disk,PriceMonthly \
  | awk 'NR==1 || $NF+0 > 0'
```

For quick reference, common general-purpose sizes:

| Slug | vCPU | RAM | Disk | $/mo |
|------|------|-----|------|------|
| `s-1vcpu-1gb` | 1 | 1 GB | 25 GB | $6 |
| `s-1vcpu-2gb` | 1 | 2 GB | 50 GB | $12 |
| `s-2vcpu-4gb` | 2 | 4 GB | 80 GB | $24 |
| `s-4vcpu-8gb` | 4 | 8 GB | 160 GB | $48 |
| `c-2` | 2 | 4 GB | 50 GB | $36 (CPU-optimized) |
| `m-2vcpu-16gb` | 2 | 16 GB | 50 GB | $84 (memory-optimized) |

Ask the user: **"Which droplet size would you like? (enter the Slug, e.g. `s-2vcpu-4gb` is a solid general-purpose default)"**

### Step 3: Provision

Use `REGION` and `SIZE` from the user's answers:

```bash
REGION=<chosen-region>
SIZE=<chosen-size>

# Make sure you have a local SSH key pair first:
if [ ! -f ~/.ssh/id_ed25519.pub ] && [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
fi
PUBKEY=$( [ -f ~/.ssh/id_ed25519.pub ] && echo ~/.ssh/id_ed25519.pub || echo ~/.ssh/id_rsa.pub )

# Upload SSH key to DigitalOcean (idempotent — re-import is safe):
doctl compute ssh-key import vibe-key --public-key-file "$PUBKEY" 2>/dev/null || true
KEY_ID=$(doctl compute ssh-key list --format ID,Name --no-header | grep vibe-key | awk '{print $1}')

doctl compute droplet create vibe-server \
  --region "$REGION" \
  --size "$SIZE" \
  --image ubuntu-24-04-x64 \
  --ssh-keys "$KEY_ID" \
  --wait

SERVER_IP=$(doctl compute droplet get vibe-server --format PublicIPv4 --no-header)
# IP captured in $SERVER_IP — used by the ssh command below, not echoed in chat.

# DigitalOcean Ubuntu images log in as root:
ssh root@"$SERVER_IP"
```

**If SSH fails**, do not continue to Phase 3B yet. Common causes and fixes:

- *Connection refused / timed out:* the droplet may still be booting — `--wait` usually handles this, but wait 30–60s more and retry. Check the DigitalOcean control panel to confirm the droplet shows "active."
- *Permission denied (publickey):* wrong key or wrong user. Try `ssh -i ~/.ssh/<your-key> root@"$SERVER_IP"`.
- *No public key registered:* verify the key was imported — `doctl compute ssh-key list`.
- *Host key changed warning:* if you reused an IP, run `ssh-keygen -R "$SERVER_IP"` then retry.

Resolve SSH before proceeding — all remaining setup happens inside that session.

**Transition to the server:** After provisioning, SSH in using `ssh root@"$SERVER_IP"`. Once your shell prompt changes (you are now on the server), every Phase 3B command runs there. The `$SERVER_IP` variable lives only in your local shell and is not carried over — that's fine, Phase 3B does not need it.
