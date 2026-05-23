# 🪅 vibe.md

The end-to-end skill for spinning up a production-ready full-stack dev and deploy environment. One interview, one clean pass — VPS provisioned, Bun installed, GitHub CLI wired, deployment platform (Dokploy or Coolify) running, and your Better T Stack project scaffolded and deployed.

## Skills

| Skill | What it does |
|-------|------------------------------------------------------------------------------------------------------|
| [`/vibe`](skills/vibe/SKILL.md) | Interview-driven full-stack setup. Detects local vs. server, provisions a VPS (Hetzner/OVH/AWS), installs Bun + GitHub CLI + Dokploy/Coolify, scaffolds a Better T Stack project, and wires up auto-deploy |

## What gets set up

- **Cloud provider** — Hetzner, OVH, or AWS (your choice, with CLI provisioned)
- **Runtime** — Bun on the server
- **Source control** — GitHub CLI authenticated
- **Deployment platform** — Dokploy or Coolify (or both), with CLI authenticated and webhook configured
- **Edge** — Wrangler (optional, for Cloudflare Workers/Pages)
- **Project** — Better T Stack scaffolded and pushed to GitHub
- **Workflow** — spec → `/ship` → PR → auto-deploy

## Quickstart

```bash
npx skills add amajorai/vibe.md
```

### Claude Code plugin

```
/plugin marketplace add amajorai/vibe.md
/plugin install vibemd@amajorai
```

Invoke as `/vibemd:vibe <project name>`.

---

Part of [amajorai/skills](https://github.com/amajorai/skills). For more skills check out the full collection.
