---
name: friday
description: >
  Use when the user asks to "friday", "friday this", "run the friday loop",
  "friday these tasks", "implement this task list with the friday workflow",
  or "plan with Fable and implement with Codex" — i.e. wants a batch of coding
  tasks driven through an orchestration loop for web, native mobile, backend,
  CLIs, or libraries. Also triggers in execute mode ("friday this plan",
  "friday plan.md", "run friday on <file>.md") when the user supplies a
  pre-existing plan file.
user-invocable: true
argument-hint: "[task description | <plan-file>.md for execute mode]"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, AskUserQuestion, Skill, TodoWrite
---

# friday — Plan / Codex / Review / Evidence-Test Orchestration Loop

Implement the user's tasks one at a time through a fixed per-task pipeline. This
session never writes feature code — it delegates planning, implementation,
review, and testing, and only coordinates, gates on approval, and does git.

## Modes

- **Full loop** (default) — the user gives tasks; each runs the whole pipeline,
  including planning and the approval gate.
- **Execute mode** — the user gives a ready plan file ("friday this plan",
  "run friday on <file>.md"). Treat it as already-approved: skip steps 0–2 and
  start at Implement, piping that file instead of a generated
  `plan_<timestamp>.md`. Never edit the user's file; if it isn't self-contained
  for a stateless codex run (missing repo paths, targets, or verification steps),
  pipe a context-augmented copy from the scratchpad. Steps 4–6 run unchanged. If
  the file holds multiple tasks/phases, run them sequentially, each with its own
  review → smoke-test → ship.

## Goal-driven completion

**First action of every run: arm a `/goal` whose condition IS the run's match
criteria** (invoke it before step 0). It installs a session Stop-hook that blocks
stopping until the condition holds and auto-clears when it does — this is what
makes the loop self-continue through fix → re-review → re-smoke instead of
stopping early. Set the condition to:

> Every task is (a) **every** review lens = **approve** (zero findings across the
> whole review team), (b) evidenced smoke test = **PASS**, and (c) committed
> locally (commit only, no push).

- If the user gave explicit acceptance criteria, fold them in **verbatim**.
- In execute mode, phrase the condition over the plan file's tasks/phases.
- Never treat the goal as met while any task has open findings, an unproven or
  failed smoke test, or uncommitted work. Don't tell the user to `/goal clear` —
  it clears itself on success.
- The `/goal` only enforces *not stopping early*; it doesn't replace the per-task
  loops below.
- **If `/goal` is unavailable** (not offered, or invoking it errors), fall back
  to a `TodoWrite` checklist whose items are the same criteria — the bar is
  identical, you just self-continue manually.

## Pipeline (per task)

0. **Brainstorm** — `superpowers:brainstorming` with the user *(skip in execute mode)*
1. **Plan** — Fable subagent writes a timestamped `plan_<timestamp>.md` *(skip in execute mode)*
2. **Approve** — get explicit user approval of the plan *(skip in execute mode)*
3. **Implement** — `codex exec` with the plan (background, ≤30 min); keep the user posted on progress
4. **Review** — a parallel team of Codex adversarial reviewers, one per lens (correctness / security / performance / approach+simplicity+surgical); any lens with findings blocks → rework to step 3 and re-review the whole team — loop until every lens approves
5. **Smoke test** — Opus subagent PROVES it works with captured evidence; root-cause any failure
6. **Ship** — commit only (no push), then next task

## The One Rule That Matters

Every subagent and codex session is **stateless** — no memory of this
conversation, prior tasks, or other subagents. Every dispatch must carry full
context: task statement, file paths, repo location, conventions, constraints,
prior-task summaries, and the exact deliverable. A vague prompt produces a
useless plan, patch, or review. Use the templates in
`references/subagent-prompts.md`.

## Coding Principles

Four principles govern every task. Two are already enforced by the pipeline's
structure; the other two must be **actively carried into the plan and every
codex round** (Codex is stateless and never sees this file — see steps 1, 3, 4).

- **Think Before Coding** — *enforced by* step 0 Brainstorm + step 2 Approval:
  surface assumptions and tradeoffs, don't guess silently, stop when confused.
- **Goal-Driven Execution** — *enforced by* the `/goal` + step 5 evidence test:
  verifiable success criteria, loop until proven.
- **Simplicity First** — *plan + codex must carry this*: the minimum code that
  solves the task. No features beyond what was asked, no speculative
  abstractions/config, no error handling for impossible cases. If 200 lines
  could be 50, rewrite it — a senior engineer shouldn't call it overcomplicated.
- **Surgical Changes** — *plan + codex must carry this*: touch only what the task
  requires. Don't refactor, reformat, or "improve" unrelated code; match the
  existing style even if you'd do it differently; remove only the orphans your
  own change creates; never delete pre-existing dead code (mention it instead).

## Model Selection & Fallback

- **Planner (step 1)** and **fallback reviewer**: Fable (`model: "fable"`).
- **Implementer (step 3)**: Codex `gpt-5.5` @ `xhigh` (pinned — never downgrade).
- **Reviewers (step 4)**: a parallel team of Codex adversarial-review runs, one
  per lens (see step 4); fallback is parallel Fable subagents.
- **Smoke tester (step 5)**: always Opus (`model: "opus"`) — unaffected by the rule below.
- **If Fable is unavailable** (the model isn't offered, or a `model: "fable"`
  dispatch errors as unknown), deliberately substitute **Opus 4.8 (1M)** at
  `xhigh` — never silently fall through to a default. Applies everywhere
  `model: "fable"` appears here and in `references/subagent-prompts.md`.

## Step Details

### 0. Brainstorm

Invoke `superpowers:brainstorming` inline with the user (not a subagent) to pin
down intent, requirements, constraints, edge cases, and design. Resolve open
questions now, while it's cheap. Feed the outcome into the planner prompt (step
1) so the plan reflects it rather than re-deriving it. If the task turns out
under-specified or wrong, stop and realign before planning. Skip for a trivial
task or in execute mode.

### 1. Plan

Dispatch the Agent tool with `model: "fable"`. Have the subagent explore the repo
and write a concrete plan to a **timestamped** `plan_<timestamp>.md` in the repo
root, grounded in the step-0 outcome. Use the planner template in
`references/subagent-prompts.md`.

Generate the timestamp once per task as local `YYYYMMDD-HHMMSS` (e.g.
`plan_20260712-143022.md`) and pass the absolute path into the prompt so the
subagent writes exactly that file. Each task gets its own plan — never overwrite
a prior one. Track that path; steps 2–4 and 6 all use it.

A good plan is executable with no other context: files to touch,
functions/components to change, data flow, edge cases, and how to verify. It
must embody the **Coding Principles** (Simplicity First + Surgical Changes) — the
minimal change, no speculative abstractions — and its out-of-scope list must name
what NOT to touch.

### 2. Approve

Show the user the plan (summary + path to `plan_<timestamp>.md`) and get approval
before implementing — **every** task, no exceptions. On change requests,
re-dispatch the planner with the feedback and the previous plan, then ask again.

### 3. Implement with Codex

```bash
codex --yolo exec -c model="gpt-5.5" -c model_reasoning_effort="xhigh" < plan_<timestamp>.md
```

In execute mode, substitute the user's plan file (or its scratchpad copy). Run
with `run_in_background: true` — runs take up to 30 min, longer than a foreground
Bash timeout. Wait for completion before review; don't review while codex is
still running. Because Codex is stateless and can't read this file, the **Coding
Principles** (Simplicity + Surgical) ride *with* the plan into every codex
dispatch. Full invocation and failure handling: `references/codex-exec.md`.

**Keep the user posted (don't go silent for 30 min).** Post a launch status
(task, plan file, `gpt-5.5` @ `xhigh`, running in background, log path
`$SCRATCHPAD/codex-task-N.log`); periodically tail the log (~1–2 min, no
busy-loop) and relay a short progress note; on completion summarize what changed
(codex's own summary + `git status --short` / `git diff --stat`). Status lines,
not raw log dumps.

### 4. Review (parallel lens team)

Review the uncommitted diff (`git diff` / `git status` — codex doesn't commit)
with a **team of reviewers run in parallel**, each looking through a different
lens. The stage chain stays sequential (you can't review before implement), but
the review *itself* is a fan-out — diverse lenses catch failure modes a single
holistic pass misses.

**Lenses (default set — scale to the diff):**
- **correctness** — logic errors, unhandled edge cases, broken integrations, regressions
- **security** — injection, authz/authn, unsafe input handling, secret/credential leaks
- **performance / resources** — hot-path cost, N+1s, leaks, blocking work, needless allocation
- **approach / simplicity / surgical** — is this the right approach for the plan+task (or a fragile bandaid)? Plus **Coding Principles** violations: over-engineering, speculative abstractions, bloated APIs, and out-of-scope / unrelated edits.

Use all four for a substantial diff; drop to 1–2 (correctness + approach) for a
tiny one. Each lens is a **separate reviewer dispatched concurrently.**

**Each reviewer = a Codex adversarial-review run, invoked via the companion
script directly — NOT the Skill tool.** The `/codex:adversarial-review` slash
command is `disable-model-invocation`, so `Skill(...)` fails with *"cannot be
used with Skill tool due to disable-model-invocation"*. Run the companion the
command shells out to (its `Bash(node:*)` is whitelisted — same mechanism, not a
safety bypass). For each lens, write its full contract — the diff is the target,
plan + task are what to judge against, plus the specific lens focus — to a
scratchpad file, then launch one background run per lens (all at once):

```bash
COMPANION=$(ls -t ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | head -1)
# One background run per lens (launch all, then wait on all):
node "$COMPANION" adversarial-review --background \
  "Adversarially review the uncommitted working-tree diff through the CORRECTNESS lens only. Full contract in $SCRATCHPAD/review-task-N-correctness.md — read it first. Verdict: exactly 'approve' or a 'findings' list with file:line."
# …repeat for security / performance / approach, each with its own steer file…
```

Launch every lens in the background at once, then monitor each Bash task and
**wait for all of them** before merging; do NOT start the smoke test until every
lens has returned. (`--wait` only if you're running a single lens on a tiny diff.)
Each reviewer's verdict is **approve** or **findings** (defects with `file:line`).

**Fallback.** Requires the Codex plugin installed/set up (`/plugin marketplace
add openai/codex-plugin-cc` → `/plugin install codex@openai-codex` →
`/codex:setup`). If the companion isn't found or Codex isn't set up, dispatch the
lens team as **parallel Fable subagents** (one per lens, same per-lens steer,
emit `approve` | `findings`). Prefer Codex — independent non-Claude reviewers are
the point.

**Merge = any lens blocks (union).** Collect every lens's verdict:
- Dedupe findings across lenses by `file:line` + mechanism (different lenses
  often flag the same defect — keep one, note which lenses raised it).
- The stage **passes only when every lens returns `approve`** (zero findings
  across the whole team). If **any** lens returns findings, the stage fails.

**On any findings — rework to step 3.** Don't advance with open findings. Feed
the **combined, deduped** findings from all lenses to a single codex fix round (a
fix-instruction file with the findings + original plan — see
`references/codex-exec.md`), then re-run the **whole lens team** (step 4). Loop
3↔4 under the `/goal` until *every* lens returns `approve`. Only then go to
step 5.

### 5. Evidence Smoke Test

Dispatch an Opus subagent (`model: "opus"`) to PROVE the change works end-to-end
— exercise whatever hits the changed code path:

- **Tests** — the project's suite (`xcodebuild test`, `./gradlew test` /
  `connectedAndroidTest`, `pytest`, `cargo test`, `go test`, `pnpm test`).
- **App run** — launch on a device/simulator and drive the new flow
  (`agent-device` can automate iOS/Android; capture a screenshot/recording).
- **Endpoints** — start the service and hit the new routes; capture request +
  full response.
- **CLI** — invoke with real args; capture command + stdout/stderr + exit code.

**Evidence is mandatory.** For every scenario, record the exact command/request,
the **raw output** (exit code, response body, decisive log lines, or a
screenshot/recording saved with its path), and PASS/FAIL. A bare "looks fine" or
a PASS with no shown evidence is **rejected** — treat it as a failed test and
re-dispatch. (Mirrors `superpowers:verification-before-completion`.)

**On FAIL, root-cause before reporting — don't guess.** Run
`superpowers:systematic-debugging` Phase 1: read the error fully (line numbers,
codes, stack traces); reproduce it consistently; in a multi-component path
(CI→build→sign, API→service→DB) gather evidence at each boundary to show *where*
it breaks; trace the bad value to its source. Report the failing boundary with
proving evidence, not a symptom or an unproven hypothesis.

The smoke test is **observe-only** — never fix anything. Evidenced findings feed
back like review findings: to a codex fix round (step 4's loop), then re-review
and re-smoke.

### 6. Ship

Only after review approval and a passing evidenced smoke test: commit with a
conventional message describing the task. **Commit only — do NOT push** (the user
decides when to push). Stage files explicitly; never commit orchestration
artifacts (`plan_*.md`, fix-instruction files), and delete the generated
`plan_<timestamp>.md` after ship. In execute mode the plan file is the user's —
leave it in place, just exclude it from the commit. Then start the next task
(step 1, or step 3 in execute mode). The run ends only when the `/goal` criteria
hold for **every** task (all review-clean, all smoke-proven, all committed
locally); the Stop-hook then auto-clears.

## Task Tracking

With multiple tasks, keep a todo checklist of all tasks and the active task's
current stage. Process strictly sequentially — one task's implementation is
context for the next task's planner prompt.

## Discipline Gates — Don't Rationalize Around Them

This loop only works if every gate holds under pressure. **Violating the letter
of a gate is violating the spirit of it.**

| Excuse | Reality |
|--------|---------|
| "The plan is obvious, I'll skip the approval gate" | In the full loop, every task's plan needs explicit user approval before Implement — no exceptions. (Execute mode is the *only* skip: the user-supplied file already stands in for an approved plan.) |
| "The smoke test clearly passes, evidence is overkill" | A PASS with no shown evidence is rejected and re-dispatched. Evidence before assertions, always. |
| "This finding is tiny, I'll just hand-patch it in the orchestrator" | Fixes go through a codex fix round. Reserve direct edits for trivial one-liners (typo, import) only — never large gaps. |
| "The diff looks right, I'll review it myself and skip Codex" | The reviewers are *independent, non-Claude* adversaries. The orchestrator never self-reviews its delegated work. Use the Fable fallback only if Codex is truly unavailable. |
| "One reviewer is enough, I'll skip the other lenses" | Review is a lens *team* — different lenses catch different failure modes. Run the full set for a substantial diff; only shrink to correctness+approach for a genuinely tiny one. |
| "Only the approach lens flagged it, the others approved — ship it" | Any lens with a real finding blocks. The pass bar is *every* lens approves, not a majority. |
| "This abstraction/flag might be useful later, I'll add it now" | Speculative code violates **Simplicity First** (YAGNI). Build the minimum the task needs; mention the future idea, don't build it. |
| "The smoke test failed but the cause is obvious" | The tester must root-cause via `superpowers:systematic-debugging` with proving evidence — a hypothesis stated as fact is not a root cause. |
| "I'll fix the failure right here in the smoke-test step" | Smoke test is observe-only. Findings feed back to a codex fix round; the tester never edits. |
| "I'll commit plan_<timestamp>.md / the fix file too, it's harmless" | Never commit orchestration artifacts. Stage feature files explicitly; delete generated `plan_*.md` after ship. |
| "Everything's basically done, I can stop now" | Not done until the match criteria hold for **every** task. The `/goal` (or its todo fallback) is what proves it. |

### Red Flags — STOP if you catch yourself thinking any of these

- "Skip approval, the plan is fine"
- "PASS — looks fine" (with no pasted output/screenshot/exit code)
- "I'll just patch this one finding myself"
- "I'll review the diff myself instead of dispatching the reviewer"
- "The failure cause is obvious, no need to debug systematically"
- "Close enough, I'll stop here"

**Each of these means: stop, return to the gate you're skipping, and run it properly.**

## Additional Resources

- **`references/codex-exec.md`** — codex exec invocation, config overrides,
  background monitoring, and failure/retry handling
- **`references/subagent-prompts.md`** — full-context prompt templates for the
  planner, reviewer, and evidence-smoke-test subagents
- **`superpowers:systematic-debugging`** — root-cause method the smoke-test
  subagent runs on any failure (Phase 1 evidence-gathering)
- **`superpowers:verification-before-completion`** — evidence-before-assertions
  discipline the smoke test enforces
