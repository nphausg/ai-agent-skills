# friday — full pipeline reference

Read this before running the loop. SKILL.md is the lean overview; this file holds
the per-step procedure, coding principles, model selection, completion tracking,
and the discipline gates.

## Coding Principles

Four principles govern every task. Two are enforced by the pipeline's structure;
the other two must be **actively carried into the plan and every codex round**
(Codex is stateless and never sees the skill — see steps 1, 3, 4).

- **Think Before Coding** — *enforced by* step 0 Brainstorm + step 2 Approval:
  surface assumptions and tradeoffs, don't guess silently, stop when confused.
- **Goal-Driven Execution** — *enforced by* the completion checklist + step 5
  evidence test: verifiable success criteria, loop until proven.
- **Simplicity First** — *plan + codex must carry this*: the minimum code that
  solves the task. No features beyond what was asked, no speculative
  abstractions/config, no error handling for impossible cases. If 200 lines
  could be 50, rewrite it.
- **Surgical Changes** — *plan + codex must carry this*: touch only what the task
  requires. Don't refactor, reformat, or "improve" unrelated code; match existing
  style; remove only the orphans your own change creates; never delete pre-existing
  dead code (mention it instead).

## Model Selection & Fallback

- **Planner (step 1)** and **fallback reviewer**: Fable (`model: "fable"`).
- **Implementer (step 3)**: Codex `gpt-5.5` @ `xhigh` (pinned — never downgrade).
- **Reviewers (step 4)**: a parallel team of Codex adversarial-review runs, one per
  lens; fallback is parallel Fable subagents.
- **Smoke tester (step 5)**: always Opus (`model: "opus"`).
- **If Fable is unavailable** (not offered, or a `model: "fable"` dispatch errors as
  unknown), deliberately substitute **Opus 4.8 (1M)** at `xhigh` — never silently
  fall through to a default. Applies everywhere `model: "fable"` appears here and in
  `subagent-prompts.md`.

## Completion tracking

**First action of every run: create a `TodoWrite` checklist whose items ARE the
run's match criteria** (before step 0). This is what makes the loop self-continue
through fix -> re-review -> re-smoke instead of stopping early — you do not stop
while any item is open. The bar for every task:

> (a) **every** review lens = **approve** (zero findings), (b) evidenced smoke
> test = **PASS**, (c) committed locally (commit only, no push).

- Fold explicit user acceptance criteria in **verbatim**.
- In execute mode, phrase the items over the plan file's tasks/phases.
- Never treat a task as done with open findings, an unproven/failed smoke test, or
  uncommitted work. The checklist only enforces *not stopping early*; it doesn't
  replace the per-task loops below.

## Step Details

### 0. Brainstorm
Invoke `superpowers:brainstorming` inline with the user (not a subagent) to pin
down intent, requirements, constraints, edge cases, and design. Resolve open
questions now. Feed the outcome into the planner prompt. If the task is
under-specified or wrong, stop and realign. Skip for a trivial task or execute mode.

### 1. Plan
Dispatch the Agent tool with `model: "fable"`. The subagent explores the repo and
writes a concrete plan to a **timestamped** `plan_<timestamp>.md` in the repo root,
grounded in step 0. Use the planner template in `subagent-prompts.md`. Generate the
timestamp once per task as local `YYYYMMDD-HHMMSS` and pass the absolute path in so
the subagent writes exactly that file. Never overwrite a prior plan. A good plan is
executable with no other context: files to touch, functions/components to change,
data flow, edge cases, how to verify. It must embody Simplicity First + Surgical
Changes, and its out-of-scope list must name what NOT to touch.

### 2. Approve
Show the user the plan (summary + path) and get approval before implementing —
every task, no exceptions. On change requests, re-dispatch the planner with the
feedback and previous plan, then ask again.

### 3. Implement with Codex

```bash
codex --yolo exec -c model="gpt-5.5" -c model_reasoning_effort="xhigh" < plan_<timestamp>.md
```

In execute mode, substitute the user's plan file (or its scratchpad copy). Run with
`run_in_background: true` — runs take up to 30 min. Wait for completion before
review. The Coding Principles ride *with* the plan into every codex dispatch. Full
invocation and failure handling: `codex-exec.md`. Keep the user posted: launch
status (task, plan file, `gpt-5.5` @ `xhigh`, background, log path), periodic short
progress notes (~1–2 min, no busy-loop), completion summary (codex summary +
`git status --short` / `git diff --stat`). Status lines, not raw log dumps.

### 4. Review (parallel lens team)
Review the uncommitted diff (`git diff` / `git status`) with a **team of reviewers
run in parallel**, one per lens — diverse lenses catch failure modes a single pass
misses.

Lenses (scale to the diff): **correctness** (logic, edge cases, regressions);
**security** (injection, authz, unsafe input, secret leaks); **performance**
(hot-path cost, N+1s, leaks, blocking work); **approach/simplicity/surgical** (right
approach vs bandaid; over-engineering, speculative abstractions, out-of-scope
edits). Use all four for a substantial diff; drop to correctness + approach for a
tiny one.

**Each reviewer = a Codex adversarial-review run via the companion script directly —
NOT the Skill tool** (the `/codex:adversarial-review` command is
`disable-model-invocation`). Write each lens's full contract (diff = target; plan +
task = judge-against; the lens focus) to a scratchpad file, then launch one
background run per lens at once:

```bash
COMPANION=$(ls -t ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | head -1)
node "$COMPANION" adversarial-review --background \
  "Adversarially review the uncommitted working-tree diff through the CORRECTNESS lens only. Full contract in $SCRATCHPAD/review-task-N-correctness.md — read it first. Verdict: exactly 'approve' or a 'findings' list with file:line."
# repeat for security / performance / approach, each with its own steer file
```

Wait for **all** lenses before merging; don't start the smoke test until every lens
returns. Verdict per lens = `approve` or `findings` (`file:line`).

**Fallback:** if the companion isn't found or Codex isn't set up (`/plugin
marketplace add openai/codex-plugin-cc` -> `/plugin install codex@openai-codex` ->
`/codex:setup`), dispatch the lens team as **parallel Fable subagents**. Prefer
Codex — independent non-Claude reviewers are the point.

**Merge = any lens blocks (union).** Dedupe findings across lenses by `file:line` +
mechanism. The stage passes only when **every** lens returns `approve`. On any
findings -> rework to step 3: feed the combined, deduped findings to a single codex
fix round (fix-instruction file + original plan — see `codex-exec.md`), then re-run
the **whole lens team**. Loop 3↔4 until every lens approves.

### 5. Evidence Smoke Test
Dispatch an Opus subagent (`model: "opus"`) to PROVE the change works end-to-end —
exercise whatever hits the changed path: the project's test suite; an app run on
device/simulator (`agent-device`, capture screenshot/recording); endpoints (start
service, hit routes, capture request + full response); or CLI (real args, capture
command + stdout/stderr + exit code).

**Evidence is mandatory.** Per scenario, record the exact command/request, the raw
output (exit code, response body, decisive log lines, or a saved
screenshot/recording path), and PASS/FAIL. A bare "looks fine" or a PASS with no
shown evidence is **rejected** — treat as failed and re-dispatch. (Mirrors
`superpowers:verification-before-completion`.)

**On FAIL, root-cause before reporting** via `superpowers:systematic-debugging`
Phase 1: read the error fully; reproduce; in a multi-component path gather evidence
at each boundary to show *where* it breaks; trace the bad value to its source.
Report the failing boundary with proving evidence, not a symptom or hypothesis.

The smoke test is **observe-only** — never fix. Evidenced findings feed back to a
codex fix round (step 4's loop), then re-review and re-smoke.

### 6. Ship
Only after review approval + a passing evidenced smoke test: commit with a
conventional message. **Commit only — do NOT push.** Stage files explicitly; never
commit orchestration artifacts (`plan_*.md`, fix-instruction files); delete the
generated `plan_<timestamp>.md` after ship. In execute mode the plan file is the
user's — leave it, just exclude it from the commit. Then start the next task. The
run ends only when the completion checklist holds for **every** task.

## Task Tracking

With multiple tasks, keep a todo checklist of all tasks and the active task's stage.
Process strictly sequentially — one task's implementation is context for the next
task's planner prompt.

## Discipline Gates — Don't Rationalize Around Them

Violating the letter of a gate is violating the spirit of it.

| Excuse | Reality |
|--------|---------|
| "The plan is obvious, skip approval" | Every task's plan needs explicit user approval before Implement (execute mode is the only skip: the supplied file stands in for an approved plan). |
| "Evidence is overkill" | A PASS with no shown evidence is rejected and re-dispatched. Evidence before assertions. |
| "I'll hand-patch this finding" | Fixes go through a codex fix round. Reserve direct edits for trivial one-liners (typo, import). |
| "I'll review it myself, skip Codex" | Reviewers are independent, non-Claude adversaries. The orchestrator never self-reviews. Use the Fable fallback only if Codex is truly unavailable. |
| "One reviewer is enough" | Review is a lens team. Run the full set for a substantial diff; shrink to correctness + approach only for a genuinely tiny one. |
| "Only approach flagged it, ship it" | Any lens with a real finding blocks. The bar is every lens approves, not a majority. |
| "This abstraction may help later" | Speculative code violates Simplicity First. Build the minimum; mention the future idea, don't build it. |
| "The failure cause is obvious" | Root-cause via systematic-debugging with proving evidence — a hypothesis stated as fact is not a root cause. |
| "I'll fix it in the smoke-test step" | Smoke test is observe-only. Findings feed back to a codex fix round. |
| "Committing plan_*.md is harmless" | Never commit orchestration artifacts. Stage feature files explicitly; delete generated `plan_*.md` after ship. |
| "Everything's basically done" | Not done until the match criteria hold for every task. |

### Red Flags — STOP if you think any of these
"Skip approval, the plan is fine" · "PASS — looks fine" (no pasted output) · "I'll
patch this one finding myself" · "I'll review the diff myself" · "The cause is
obvious, no need to debug systematically" · "Close enough, I'll stop here". Each
means: stop, return to the gate you're skipping, run it properly.
