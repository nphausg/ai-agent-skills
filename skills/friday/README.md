# friday

A per-task orchestration loop that drives a batch of coding tasks to completion
**one at a time**, each through a fixed pipeline:

```
brainstorm → plan (Fable) → approve → implement (Codex gpt-5.5)
  → review (Codex adversarial-review) → evidence smoke-test (Opus) → ship
```

The driving session **only coordinates** — it gates on approval, dispatches
subagents, and does the git work. It never writes feature code itself; planning,
implementation, review, and testing are all delegated to independent agents.
Stack-agnostic: web, native mobile (iOS/Android), backend, CLIs, and libraries.

## When to use

Use when you have one or more coding tasks you want implemented end-to-end with
strong quality gates and minimal hand-holding — especially when you want an
**independent, non-Claude reviewer** (Codex) and **proof** the change actually
runs, not just that it compiles.

**Not for:** a single trivial edit (the pipeline overhead isn't worth it), or
read-only investigation.

## Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Full loop** (default) | "friday", "run the friday loop", "friday these tasks" | Every task runs the whole pipeline, including brainstorm + planning + the approval gate. |
| **Execute mode** | "friday this plan", "friday `<file>.md`", "run friday on `<file>.md`" | The user supplies a pre-existing plan file. Brainstorm/plan/approve are skipped — the file *is* the approved plan and goes straight to implement → review → smoke-test → ship. The user's file is never edited; if it's not self-contained for a stateless codex run, a context-augmented copy is piped from the scratchpad. |

## The pipeline (per task)

0. **Brainstorm** — `superpowers:brainstorming` with the user to pin down intent,
   requirements, constraints, and design *(skipped in execute mode)*.
1. **Plan** — a **Fable** subagent explores the repo and writes a self-contained
   `plan.md` *(skipped in execute mode)*.
2. **Approve** — the plan is shown to the user; nothing proceeds without explicit
   approval *(skipped in execute mode)*.
3. **Implement** — `codex --yolo exec -c model=gpt-5.5 -c model_reasoning_effort=xhigh < plan.md`,
   run in the background (up to 30 min). Codex edits files but does not commit.
4. **Review** — `/codex:adversarial-review` (an independent, non-Claude adversary
   that tries to break the change) judges the diff against the plan; verdict is
   **approve** or **findings** (`file:line`). Findings loop back through a codex
   fix round until the review approves.
5. **Evidence smoke test** — an **Opus** subagent proves the change works
   end-to-end (tests / app run / endpoints / CLI) with captured evidence. A PASS
   without shown evidence is rejected. Failures are root-caused via
   `superpowers:systematic-debugging` and fed back to a codex fix round —
   observe-only, the tester never edits.
6. **Ship** — commit (staging feature files explicitly, never orchestration
   artifacts) and push, then move to the next task.

## Goal-driven completion

The first action of every run arms a **`/goal`** whose condition is the run's
match criteria (every task: review-clean, smoke-proven, committed + pushed). This
installs a session Stop-hook so the loop **self-continues** through
fix → re-review → re-smoke instead of stopping early, and auto-clears once every
task meets the bar. If `/goal` is unavailable, it falls back to a `TodoWrite`-driven
manual loop with the identical bar.

## Model selection & fallback

- **Planner (step 1):** Fable → fallback **Opus 4.8 (1M)** at `xhigh` if Fable
  is unavailable.
- **Reviewer (step 4):** Codex `/codex:adversarial-review` → fallback Fable
  subagent (same steer) only if the Codex plugin is unavailable. An independent
  non-Claude reviewer is the whole point.
- **Implementer (step 3):** Codex `gpt-5.5` at `xhigh` — pinned, never downgraded.
- **Smoke tester (step 5):** always Opus.

## Prerequisites

- **Codex plugin** (openai/codex-plugin-cc) for implementation and adversarial
  review — install once:
  `/plugin marketplace add openai/codex-plugin-cc` → `/plugin install codex@openai-codex` → `/codex:setup`.
- **superpowers** skills: `brainstorming`, `systematic-debugging`,
  `verification-before-completion`.

## Discipline gates

The loop only works if every gate holds under pressure — approval before
implement, evidence before a PASS, independent review (no self-review), fixes
through a codex round (no orchestrator hand-patching beyond one-liners),
observe-only smoke tests, and never committing orchestration artifacts. SKILL.md
carries a rationalization table + red-flags list that spell these out.

## Files

- `SKILL.md` — the full loop, step details, modes, goal-driven completion, model
  selection, and discipline gates.
- `references/codex-exec.md` — codex exec invocation, config pins, background
  monitoring, and failure/retry handling.
- `references/subagent-prompts.md` — full-context prompt templates for the
  planner, reviewer (fallback), and evidence-smoke-tester subagents.
