# 🪅 vibe.md

The end-to-end skill for spinning up a 24/7 production-ready full-stack dev and deploy environment. One interview, one clean pass: VPS provisioned, Bun installed, GitHub CLI wired, deployment platform running, and your project scaffolded and shipping.

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
- **Workflow**: spec → `/ship` → PR → auto-deploy

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

---

Part of [amajorai/skills](https://github.com/amajorai/skills). For more skills check out the full collection.
