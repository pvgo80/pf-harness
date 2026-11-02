---
name: write-an-ia
description: Use when starting a new project and needing to map the Information Architecture before any UI work — capturing the screens, user flows, and project glossary in one canonical document. Triggered by "let's map the IA", "draft a sitemap", "what screens does this product have", or when a brief exists but no IA file does.
---

# Write an IA

Produces `docs/design/ia.md`: a canonical Markdown document with a glossary, a sitemap, named user flows, and any open questions.

This document is the canonical input to `build-lofi` and a reference for `write-a-prd`. Get the screen names and flow names right here — downstream skills will refer to them verbatim.

## When to use

- A new project has a brief but no IA file yet.
- The screen list and flow names need to become canonical for downstream skills.

When NOT to use:
- Mid-project edits to an existing IA — edit `docs/design/ia.md` directly.
- Producing wireframes or layout sketches — that's `build-lofi`.

## Output location

Default: `docs/design/ia.md` at the repo root.

If `docs/agents/design.md` exists, read the `**Location:**` line in it and use that path instead.

If a file already exists at the target, ask whether to edit it in place (default) or overwrite from scratch.

## Process

### 1. Gather input

Read whatever the user provided: a pasted brief, a file path, or the existing `BRIEF.md`/`README.md` in the repo. If none of those exist and the user gave a one-line request, **stop and ask for the brief** before drafting anything.

### 2. Draft three sections in order

**Glossary first.** Pull project-specific nouns from the brief — terms a new joiner wouldn't know. Format: `- Term: one-sentence definition.` Skip standard English nouns ("login", "settings"). If the brief invents a noun ("note", "digest", "workspace"), it goes here.

**Sitemap next.** A nested Markdown list. Each line: `- Screen name — one-line purpose.` Children indented two spaces. Sentence case for screen names ("Meeting detail", never "MEETING_DETAIL" or "meetingDetail"). The purpose is **what the user does or sees there**, not what components are on it.

**Flows last.** Each named user journey is an H3 heading. Body is a single arrow-string referencing sitemap nodes verbatim:

```
Login → Workspace home → Meeting detail → Record note → Meeting detail
```

Only write flows the brief or user explicitly mentioned. Don't pad.

### 3. Interview where input is sparse

- A screen has a vague purpose → ask.
- A flow's terminal screen isn't named → ask.
- The brief implies a screen but doesn't name it → propose a name, get confirmation.

Don't invent screens or flows the user didn't mention. Surface gaps as `## Open questions` instead.

### 4. Validate

Every screen referenced inside a flow MUST appear in the sitemap with **the exact same name**. Orphan references are an error — fix before writing the file.

### 5. Write

Write to the target location. Print one line: where you wrote it, how many screens and flows, how many open questions.

## Output format

```markdown
# IA — TalkTime

## Glossary
- Workspace: the company or team account; all notes live inside one workspace.
- Note: a 30-second voice recording attached to a meeting.
- Digest: a weekly auto-generated summary of recurring themes.

## Sitemap
- Workspace home — list of meetings the user has access to
  - Meeting detail — notes from one specific meeting
    - Record note — voice capture for the meeting
- Weekly digest — recurring themes across one project
- Settings — account, workspace, privacy

## Flows

### Record a note after a meeting
Login → Workspace home → Meeting detail → Record note → Meeting detail

### Browse weekly themes
Login → Workspace home → Weekly digest

## Open questions
- Does Settings need its own flow, or is it ad-hoc from a global menu?
```

## Common mistakes

- **Inventing flows.** One mentioned flow → one written flow. Don't pad with hypothetical journeys.
- **Vague purposes.** "Dashboard — main page" tells nothing. Use "Workspace home — pick a meeting to open."
- **Name drift.** "Meeting page" in the sitemap and "Meeting detail" in a flow is an error. Validate exact-match.
- **Skipping the glossary.** Any term invented by the brief belongs there.
- **Writing UI in the IA.** No components, no layout, no visuals. That belongs in the lo-fi.
- **Generic names.** "Dashboard", "Page", "Home" without qualification — push back, ask for a name that says what the user does there.

## Red flags — STOP and ask

- You're guessing at a screen's purpose without source material.
- A flow contains a screen name that isn't in the sitemap.
- You're about to write more than 10 top-level screens — the scope is probably too wide for v1.
- The user said "make me an IA" with no brief. Ask for the brief first.
