# Solutions Guide Template

This is the template the `compound` skill writes to a repo's `docs/agents/solutions.md` (on first run, with consent). It tells other agents that the solution store exists and how to use it — mirroring the role `docs/agents/domain.md` and `docs/agents/prds.md` play for their stores.

Write it verbatim, adjusting only the location line if the repo deviates from the default.

---

```md
# Solutions store

**Location:** `docs/solutions/`

A searchable knowledge base of solved problems for this repo. Two kinds of entry,
distinguished by the `track` frontmatter field:

- `track: bug` — a postmortem: symptom, what failed, fix, root cause, prevention.
- `track: knowledge` — a how-it-works note: context, guidance, why it matters, when to apply.

## Before solving a problem

Search here first. Grep `docs/solutions/` for the component, the error message, or
the symptom. If a matching entry exists, read it before debugging from scratch — the
fix or the root cause may already be recorded.

## After solving a problem

Run the `compound` skill (`/compound`) to capture the learning. Do not hand-write
entries — the skill handles classification, overlap detection, and cross-product
promotion to memory.

## Relationship to other docs

- `CONTEXT.md` — the glossary. Terms only, no fixes.
- `docs/adr/` — decisions: *why* the code is shaped this way.
- `docs/solutions/` — fixes and how-it-works notes: *what broke and how*, *how a thing actually works*.

A question of the form "why did we choose X?" is an ADR. "X broke, here's the fix"
or "here's how X actually works" is a solution.
```
