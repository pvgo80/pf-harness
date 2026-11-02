---
name: prototype
description: Build a throwaway prototype to flesh out a design before committing to it. Routes by question and audience — a runnable terminal app for state/business-logic questions deep in the build, a single shareable HTML demo when a stakeholder (PM, designer, domain expert) needs to feel a still-forming state model, or several radically different UI variations toggleable from one route. Use when the user wants to prototype, sanity-check a data model or state machine, mock up a UI, explore design options, build a clickable demo for a non-developer, or says "prototype this", "let me play with it", "try a few designs", "make something I can send to the PM".
---

# Prototype

A prototype is **throwaway code that answers a question**. The question — and who needs to feel the answer — decides the shape.

## Pick a branch

Identify which question is being answered — from the user's prompt, the surrounding code, or by asking if the user is around:

- **"Does this logic / state model feel right?"** — then ask *who needs to drive it*:
  - **A developer, deep in the build** ("what could fit here?") → [LOGIC.md](LOGIC.md). A tiny interactive terminal app in the host project's own language that pushes the state machine through cases that are hard to reason about on paper. The pure logic module is written to be lifted into the real codebase afterward.
  - **A stakeholder, early in the design** — a PM, designer, or domain expert, while the model itself is still up for debate → [DEMO.md](DEMO.md). A single self-contained HTML file (free-play buttons plus tabbed guided walkthroughs, labelled in domain language) that a non-developer can double-click and drive.
- **"What should this look like?"** → [UI.md](UI.md). Generate several radically different UI variations on a single route, switchable via a URL search param and a floating bottom bar.

The branches produce very different artifacts — getting this wrong wastes the whole prototype. If the question is genuinely ambiguous and the user isn't reachable, default by context: a backend module → logic; a page or component → UI; for logic, the terminal branch unless a non-developer audience was mentioned. State the assumption at the top of the prototype.

## Rules that apply to every branch

1. **Throwaway from day one, and clearly marked as such.** Locate the prototype code close to where it will actually be used (next to the module or page it's prototyping for) so context is obvious — but name it so a casual reader can see it's a prototype, not production. For throwaway UI routes, obey whatever routing convention the project already uses; don't invent a new top-level structure.
2. **Trivial to run.** Terminal and UI prototypes start from one command in the project's existing task runner — `pnpm <name>`, `python <path>`, `bun <path>`, etc. A demo is a single HTML file the recipient double-clicks. Either way, no thinking required to start it.
3. **No persistence by default.** State lives in memory. Persistence is the thing the prototype is _checking_, not something it should depend on. If the question explicitly involves a database, hit a scratch DB or a local file with a clear "PROTOTYPE — wipe me" name.
4. **Skip the polish.** No tests, no error handling beyond what makes the prototype _runnable_, no abstractions. The point is to learn something fast and then delete it.
5. **Surface the state.** After every action (logic, demo) or on every variant switch (UI), print or render the full relevant state so the user can see what changed.
6. **Delete or absorb when done.** When the prototype has answered its question, either delete it or fold the validated decision into the real code — don't leave it rotting in the repo. A demo is the one exception worth keeping as a primary source: being a single self-contained file, commit it to a throwaway branch out of main (or attach it to the issue) next to the captured answer, where it stays trivially re-runnable.

## When done

The _answer_ is the only thing worth keeping from a prototype. Capture it somewhere durable (commit message, ADR, issue, or a `NOTES.md` next to the prototype) along with the question it was answering. If the user is around, that capture is a quick conversation; if not, leave the placeholder so they (or you, on the next pass) can fill in the verdict before deleting the prototype.
