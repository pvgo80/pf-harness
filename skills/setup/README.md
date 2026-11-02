# Setup

Per-repo configuration for the pf-skills plugins.

## Skills

- **[setup-pf-skills](./setup-pf-skills/SKILL.md)** — Scaffold the per-repo `## Agent skills` block + `docs/agents/` that the building, engineering, and one-shot plugins read: domain-doc layout, PRD/issue/plan locations, issue tracker + triage labels, and one-shot's ADR-write target and auto-mode posture. Detects which plugins are installed and asks only what applies. Run once per repo.

## Provenance

Merges several upstream installers, including `setup-matt-pocock-skills` (mattpocock/skills), which all wrote overlapping `docs/agents/` files, into one wizard.
