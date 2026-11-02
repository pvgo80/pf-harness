---
name: grill-me
description: >
  Use when user wants to stress-test a plan or design through relentless
  interviewing, or mentions "grill me". Maps the plan as a design tree and
  works it in rounds — asking every question whose prerequisites are settled,
  each with a recommended answer — until no branch is left silently assumed.
---

Interview the user relentlessly until you reach a shared understanding of the plan. Map it as a **design tree**: every decision branches into the decisions that hang off it.

## Before you start

**Name the destination.** State in one line what "done" looks like for this plan, and confirm it with the user. Every question should serve reaching it.

**Go breadth-first before depth-first.** First surface *every* open decision across the whole plan — don't start resolving yet. This survey seeds the tree and stops you rabbit-holing down one branch while other branches sit unexplored.

**Apply the fog test.** Only pose a question when you can state it *precisely* now — not when you can answer it now. If a decision clearly matters but you can't phrase it sharply yet, park it as "not yet sharp" and return to it once the frontier advances. Don't pre-slice fog into fake questions.

## Work the tree in rounds

The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

Format a round like so:

```
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. When a frontier question needs a fact from the environment (filesystem, codebase, tools, etc.), dispatch a sub-agent to find it; don't ask the user for anything you could look up yourself. Don't block on it: a running exploration is an unsettled prerequisite, so only the questions downstream of it wait for the sub-agent to report; ask the rest of the frontier now. The _decisions_ are the user's: put each to them and wait.

## Done

The session is done when the frontier is empty and nothing is still parked as "not yet sharp": every branch of the design tree visited, nothing left silently assumed. Do not act on the plan until the user confirms you have reached a shared understanding.
