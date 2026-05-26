# 🪅 vibe.md

The end-to-end skill for spinning up a 24/7 production-ready full-stack dev and deploy environment. One interview, one clean pass: VPS provisioned, Bun installed, GitHub CLI wired, deployment platform running, and your server ready to ship.

[![Status](https://shieldcn.dev/badge/status-experimental-orange.svg)](https://github.com/amajorai/vibe.md)
[![Stars](https://shieldcn.dev/github/stars/amajorai/vibe.md.svg)](https://github.com/amajorai/vibe.md)
[![Forks](https://shieldcn.dev/github/forks/amajorai/vibe.md.svg)](https://github.com/amajorai/vibe.md)
[![License](https://shieldcn.dev/github/license/amajorai/vibe.md.svg)](https://github.com/amajorai/vibe.md)
[![Issues](https://shieldcn.dev/github/issues/amajorai/vibe.md.svg)](https://github.com/amajorai/vibe.md/issues)

## Works great with

- 📦 **[ship.md](https://github.com/amajorai/ship.md)** to build features with a quality-gated pipeline once your environment is running: interview → explore → plan → implement → verify.
- 🎉 **[party.md](https://github.com/amajorai/party.md)** to keep building autonomously 24/7. GitHub Projects as your interface; drop in issues and the agent ships them while you sleep.
- ⚡ **[amajorai/skills](https://github.com/amajorai/skills)** for edge cases, E2E, payments, auth, SEO, icons, CI, observability, and 20+ more.

## Skills

| Skill | What it does |
|-------|--------------|
| [`/vibe`](skills/vibemd/SKILL.md) | Full setup skill. Detects local vs. server, provisions a VPS (Hetzner/DigitalOcean/OVH/AWS) with region and size chosen from live CLI queries, installs Bun + GitHub CLI + Dokploy/Coolify + optional Wrangler + optional AI coding CLIs (Claude Code, Codex), and writes `/etc/vibemd/server.json`. Re-running on an already-configured server shows current state and offers next steps instead of re-running setup. |
| [`/vibe-provision-worker`](skills/vibemd-provision-worker/SKILL.md) | Add a worker server to an existing orchestrator. Reads `/etc/vibemd/server.json` for provider, platform, and AI CLI preferences, spins up a new VPS, registers it with Dokploy or Coolify, optionally installs AI coding CLIs, and records the worker in the config. |
| [`/vibe-reconfigure`](skills/vibemd-reconfigure/SKILL.md) | Change role, platform, or installed tools on an existing server. Detects current state from `/etc/vibemd/server.json` first and only applies the delta — switch master ↔ orchestrator, swap or add Dokploy/Coolify, install or remove Wrangler, and add or remove AI coding CLIs. Safe to run on a live server. |

## Tech Stack

- **Cloud providers**: Hetzner, DigitalOcean, OVH, or AWS (your choice; region and server size chosen dynamically via live CLI queries after authentication)
- **Runtime**: Bun on the server
- **Source control**: GitHub CLI authenticated
- **Deployment platform**: Dokploy or Coolify (or both), with CLI authenticated and webhook configured
- **Edge**: Wrangler (optional, for Cloudflare Workers, Pages, and DNS)
- **AI coding CLIs**: Claude Code (`@anthropic-ai/claude-code`) and/or Codex CLI (`@openai/codex`) — optional, installable on the server and/or on worker nodes
- **Server config**: `/etc/vibemd/server.json` written on first setup; subsequent skill runs read this to show current state and skip re-provisioning

## Quickstart

```bash
npx skills add amajorai/vibe.md
```

### Auto-Update

`/vibe` can update itself before running. Auto-update is opt-in: pass `--update` to your command or set `SKILLS_AUTO_UPDATE: true` in your project CLAUDE.md, and it runs `npx skills update amajorai/vibe.md -y` at the start of the invocation, then asks you to re-run if a new version was installed.

### Claude Code plugin

```
/plugin marketplace add amajorai/vibe.md
/plugin install vibemd@amajorai
```

Invoke as `/vibemd:vibe`, `/vibemd:vibe-reconfigure`, or `/vibemd:vibe-provision-worker`.

## Star History

<a href="https://www.star-history.com/#amajorai/vibe.md&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=amajorai/vibe.md&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=amajorai/vibe.md&type=Date" />
   <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=amajorai/vibe.md&type=Date" />
 </picture>
</a>
