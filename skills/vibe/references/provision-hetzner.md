# Hetzner Provisioning

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

**If SSH fails**, do not continue to Phase 3B yet. Common causes and fixes:

- *Connection refused / timed out:* the server may still be booting. Wait 30–60s and retry. On AWS, confirm the security group allows inbound TCP 22 from your IP.
- *Permission denied (publickey):* you are using the wrong key or the wrong user. Try `ssh -i ~/.ssh/<your-key> <user>@"$SERVER_IP"`, and verify the user matches the provider (Hetzner `root`, AWS Ubuntu `ubuntu`, OVH usually `ubuntu`/`debian`).
- *No public key registered:* re-check that the key you uploaded to the provider matches your local `~/.ssh/*.pub`.
- *Host key changed warning:* if you reused an IP, run `ssh-keygen -R "$SERVER_IP"` then retry.

Resolve SSH before proceeding — all remaining setup happens inside that session.

**Transition to the server:** After provisioning, SSH in using `ssh root@"$SERVER_IP"`. Once your shell prompt changes (you are now on the server), every Phase 3B command runs there. The `$SERVER_IP` variable lives only in your local shell and is not carried over — that's fine, Phase 3B does not need it.
