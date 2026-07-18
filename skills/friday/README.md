# friday

A per-task orchestration loop that drives a batch of coding tasks to completion
**one at a time**, each through a fixed pipeline:

```
brainstorm -> plan (Fable) -> approve -> implement (Codex gpt-5.5)
  -> review (parallel Codex lens team) -> evidence smoke-test (Opus) -> ship
```

The driving session **only coordinates** — it gates on approval, dispatches
subagents, and does the git work. It never writes feature code itself; planning,
implementation, review, and testing are all delegated to independent agents.
Stack-agnostic: web, native mobile (iOS/Android), backend, CLIs, and libraries.

Four **Coding Principles** run through it: *Think Before
Coding* and *Goal-Driven Execution* are enforced structurally (brainstorm +
approval; completion checklist + evidence test), while *Simplicity First* and *Surgical
Changes* are baked into the plan and the Codex contract (since Codex is stateless
and can't see the skill) and policed by the review team's approach/simplicity/
surgical lens.

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
| **Execute mode** | "friday this plan", "friday `<file>.md`", "run friday on `<file>.md`" | The user supplies a pre-existing plan file. Brainstorm/plan/approve are skipped — the file *is* the approved plan and goes straight to implement -> review -> smoke-test -> ship. The user's file is never edited; if it's not self-contained for a stateless codex run, a context-augmented copy is piped from the scratchpad. |

## The pipeline (per task)

0. **Brainstorm** — `superpowers:brainstorming` with the user to pin down intent,
   requirements, constraints, and design *(skipped in execute mode)*.
1. **Plan** — a **Fable** subagent explores the repo and writes a self-contained,
   timestamped `plan_<timestamp>.md` *(skipped in execute mode)*.
2. **Approve** — the plan is shown to the user; nothing proceeds without explicit
   approval *(skipped in execute mode)*.
3. **Implement** — `codex --yolo exec -c model=gpt-5.5 -c model_reasoning_effort=xhigh < plan_<timestamp>.md`,
   run in the background (up to 30 min). Codex edits files but does not commit.
   The loop keeps the user informed as it runs — a launch status line, periodic
   log tails, and a final change summary (`git diff --stat`) — so you can see
   what Implement is doing rather than staring at silence.
4. **Review** — a **parallel team** of Codex adversarial reviewers, one per lens
   (correctness / security / performance / approach+simplicity+surgical), each an independent
   non-Claude adversary judging the diff against the plan. Invoked by calling the
   companion script directly (`node codex-companion.mjs adversarial-review …`),
   because the `/codex:adversarial-review` slash command is
   `disable-model-invocation` and can't be triggered via the Skill tool; parallel
   Fable subagents are the fallback. **Any lens with findings blocks** (union;
   pass = every lens approves). On findings, **rework back to step 3** (feed the
   combined findings to a codex fix round), then re-review the whole team — the
   3↔4 loop runs until every lens returns zero comments.
5. **Evidence smoke test** — an **Opus** subagent proves the change works
   end-to-end (tests / app run / endpoints / CLI) with captured evidence. A PASS
   without shown evidence is rejected. Failures are root-caused via
   `superpowers:systematic-debugging` and fed back to a codex fix round —
   observe-only, the tester never edits.
6. **Ship** — commit only (staging feature files explicitly, never orchestration
   artifacts); **do not push** — pushing is left to the user. Then move to the next task.

## Completion tracking

The first action of every run creates a **`TodoWrite`** checklist whose items are
the run's match criteria (every task: review-clean, smoke-proven, committed
locally — no push). The loop **self-continues** through fix -> re-review ->
re-smoke instead of stopping early: no item is closed until it holds, and the run
ends only once every task meets the bar.

## Model selection & fallback

- **Planner (step 1):** Fable -> fallback **Opus 4.8 (1M)** at `xhigh` if Fable
  is unavailable.
- **Reviewers (step 4):** a parallel team of Codex adversarial-review runs, one
  per lens -> fallback parallel Fable subagents only if the Codex plugin is
  unavailable. Independent non-Claude reviewers are the whole point.
- **Implementer (step 3):** Codex `gpt-5.5` at `xhigh` — pinned, never downgraded.
- **Smoke tester (step 5):** always Opus.

## Prerequisites

- **Codex plugin** (openai/codex-plugin-cc) for implementation and adversarial
  review — install once:
  `/plugin marketplace add openai/codex-plugin-cc` -> `/plugin install codex@openai-codex` -> `/codex:setup`.
- **superpowers** skills: `brainstorming`, `systematic-debugging`,
  `verification-before-completion`.

## Discipline gates

The loop only works if every gate holds under pressure — approval before
implement, evidence before a PASS, independent review (no self-review), fixes
through a codex round (no orchestrator hand-patching beyond one-liners),
observe-only smoke tests, and never committing orchestration artifacts. SKILL.md
carries a rationalization table + red-flags list that spell these out.

## Files

- `SKILL.md` — the full loop, step details, modes, completion tracking, model
  selection, and discipline gates.
- `references/codex-exec.md` — codex exec invocation, config pins, background
  monitoring, and failure/retry handling.
- `references/subagent-prompts.md` — full-context prompt templates for the
  planner, reviewer (fallback), and evidence-smoke-tester subagents.
