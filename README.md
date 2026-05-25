# 🪅 vibe.md

The end-to-end skill for spinning up a 24/7 production-ready full-stack dev and deploy environment. One interview, one clean pass: VPS provisioned, Bun installed, GitHub CLI wired, deployment platform running, and your project scaffolded and shipping.

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
|-------|------------------------------------------------------------------------------------------------------|
| [`/vibe`](skills/vibe/SKILL.md) | Interview-driven full-stack setup. Detects local vs. server, provisions a VPS (Hetzner/OVH/AWS), installs Bun + GitHub CLI + Dokploy/Coolify, scaffolds a Better T Stack project, and wires up auto-deploy |

## Tech Stack

- **Cloud provider**: Hetzner, OVH, or AWS (your choice, with CLI provisioned)
- **Runtime**: Bun on the server
- **Source control**: GitHub CLI authenticated
- **Deployment platform**: Dokploy or Coolify (or both), with CLI authenticated and webhook configured
- **Edge**: Wrangler (optional, for Cloudflare Workers, Pages, and DNS)
- **Project**: Better T Stack scaffolded and pushed to GitHub

## Quickstart

```bash
npx skills add amajorai/vibe.md
```

### Auto-Update

`/vibe` can update itself before running. Auto-update is opt-in: pass `--update` to your command or set `SKILLS_AUTO_UPDATE: true` in your project CLAUDE.md, and it runs `npx skills update vibe -y` at the start of the invocation, then asks you to re-run if a new version was installed.

### Claude Code plugin

```
/plugin marketplace add amajorai/vibe.md
/plugin install vibemd@amajorai
```

Invoke as `/vibemd:vibe <project name>`.

## Star History

<a href="https://www.star-history.com/#amajorai/vibe.md&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=amajorai/vibe.md&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=amajorai/vibe.md&type=Date" />
   <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=amajorai/vibe.md&type=Date" />
 </picture>
</a>
