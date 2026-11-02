# PRDs and Issues

How the building and one-shot skills should find and write PRDs and their issues.

**Location:** `prds/`

**Issues:** `tracker`

Override these when the skill writes `docs/agents/prds.md` for a specific repo.
`Issues` is `tracker` (the PRD's issues live on the issue tracker configured in
`docs/agents/issue-tracker.md`) or `file` (they live in a local
`<location>/<feature-name>-issues.md`). Every skill in the PRD lane —
`/prd-to-issues`, `/issue-worker`, `/issue-swarm`, and the pipelines that drive
them — reads this one field, so the lane is either all-tracker or all-file.

## How each skill uses this

- **`/write-a-prd`** writes its output to `<location>/<feature-name>.md`. Feature name comes from the user; slugify if needed.
- **`/prd-to-issues`** — if multiple PRDs exist, ask the user which one.
  - `tracker`: creates one issue per slice on the tracker, in dependency order, each labelled `prd:<feature-name>` and linking the PRD, with blocking edges set as `docs/agents/issue-tracker.md` describes. No local file.
  - `file`: writes `<location>/<feature-name>-issues.md`.
- **`/issue-worker`** / **`/issue-swarm`** pick the next issue(s) in dependency order whose blockers are done.
  - `tracker`: the open `prd:<feature-name>` issues; an issue is done once a commit carries `Implements issue: #<n> <title>` (plus `Closes #<n>`, which closes it when the commit reaches the default branch).
  - `file`: the issues in `<location>/*-issues.md`; an issue is done once a commit carries `Implements issue: <title>`.

## Scratch, not durable

PRDs and issue files are **scratch** planning artefacts. `/issue-worker` retires
each issue as it ships (removes it from the file in `file` mode; the commit's
`Closes #<n>` closes it in `tracker` mode), and deletes the PRD (and the issues
file, in `file` mode) once the last issue lands, staged into that commit. The
durable record is the code + tests + ADRs (see `docs/agents/one-shot.md`, where
the one-shot plugin is configured), not the planning docs.

## Conventions

- One PRD per feature
- `file` mode: the issues file lives alongside its parent PRD, with the suffix `-issues.md`
- `tracker` mode: one label per PRD, `prd:<feature-name>`
- Issues are committed one at a time; the commit's `Implements issue:` trailer is what lets subsequent runs know where to pick up
