# Server Setup (Phase 3B)

Run all steps in order **on the server** (inside your SSH session — not on your local machine). If you provisioned from local in Phase 3A, you should already be connected via `ssh`. These commands assume you are the `root` user; if your provider logs you in as a non-root user (e.g. `ubuntu`), prefix the `apt-get`, `dd`, `tee`, and install commands with `sudo`.

## Step 1: System baseline

```bash
# DEBIAN_FRONTEND=noninteractive + the dpkg options keep apt from stopping on
# interactive prompts (e.g. "a new version of /etc/... is available"), which
# would otherwise hang the upgrade over SSH.
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" upgrade
apt-get install -y curl git unzip jq build-essential ca-certificates gnupg
```

## Step 2: Install Bun

```bash
curl -fsSL https://bun.sh/install | bash
# The installer appends to your shell rc file. Re-source whichever one applies,
# or simply start a new shell session to pick up Bun on your PATH:
source ~/.bashrc 2>/dev/null || source ~/.zshrc 2>/dev/null || true
# If `bun` is still not found, open a fresh login shell (e.g. log out and back
# in, or run `exec $SHELL -l`).
bun --version
```

## Step 3: Install GitHub CLI

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

## Step 4: Install Deployment Platform

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

## Step 5: Open the firewall for the platform UIs

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

## Step 6: Install Wrangler (if edge deployment chosen)

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

## Step 7: Cloud CLI for Orchestration (orchestrator role only)

See [references/orchestrator-setup.md](orchestrator-setup.md) for the full step-by-step.
