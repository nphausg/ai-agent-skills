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

Implement all tasks given by the user, one at a time, through a fixed per-task
pipeline. The orchestrating session (this session) never writes feature code
itself — it delegates planning, implementation, review, and testing, and only
handles coordination, approval gates, and git.

## Modes

- **Full loop** (default) — the user gives tasks; every task runs the whole
  pipeline below, including planning and the approval gate.
- **Execute mode** — the user gives a pre-existing plan as a markdown file
  ("friday this plan", "execute <path>.md with friday"). Treat the file as
  the already-approved plan: skip steps 0–2 (brainstorm, plan, approve) and start at Implement, piping the
  given file to codex instead of a generated `plan.md`. Read the file first —
  if it is not self-contained enough for a stateless codex session (missing
  repo paths, target files, or verification steps), prepend a short context
  preamble to a copy in the scratchpad and pipe that copy; never edit the
  user's file. Review, smoke test, and ship stages run unchanged, using the
  provided plan's content wherever the templates ask for "plan.md content". If
  the file contains multiple tasks/phases, execute them as sequential tasks,
  each with its own review → smoke-test → ship cycle.

## Goal-driven completion (`/goal` until match criteria)

**First action of every run: arm a `/goal` whose condition IS the run's match
criteria.** `/goal` installs a session Stop-hook that blocks the session from
stopping until the condition holds (and auto-clears the moment it does) — this is
what makes the loop *self-continue* through fix → re-review → re-smoke instead of
stopping early. Invoke it via the Skill/command tool before step 0.

Set the condition to the concrete, checkable match criteria — the same PASS bar
steps 4–6 already enforce, phrased so "goal met" == "every task shipped
review-clean and proven":

> Every task in this run is (a) `/codex:adversarial-review` = **approve** (zero
> open findings), (b) evidenced smoke test = **PASS** with captured evidence, and
> (c) committed + pushed.

- If the user gave an explicit acceptance / match criterion, fold it into the
  condition **verbatim** (it's the authoritative bar).
- In execute mode, phrase the condition over the plan file's tasks/phases.
- Never treat the goal as met — or let the session stop — while any task still
  has open review findings, an unproven or failed smoke test, or uncommitted
  work. Do NOT tell the user to `/goal clear`; it clears itself on success.
- The `/goal` only enforces *not stopping early*; it does not replace the
  per-task loops below — it's the outer guarantee that they run to the bar.

**If `/goal` is unavailable in this environment** (the command/skill is not
offered, or invoking it errors), do not stop — fall back to a manual
todo-driven loop: keep a `TodoWrite` checklist whose items ARE the match
criteria (per task: review-clean, smoke-proven, committed + pushed), and do not
consider the run finished until every item is checked. The self-continuation is
then your responsibility rather than the Stop-hook's; the bar is identical.

## Pipeline (per task)

0. **Brainstorm** — run `superpowers:brainstorming` FIRST to explore intent, requirements, and design with the user before any planning *(skipped in execute mode)*
1. **Plan** — dispatch a Fable subagent to write `plan.md` for the task, grounded in the brainstorming outcome *(skipped in execute mode)*
2. **Approve** — present the plan to the user; do NOT proceed without explicit approval *(skipped in execute mode)*
3. **Implement** — run `codex exec` with the approved plan (background, up to 30 min)
4. **Review** — run `/codex:adversarial-review` (openai/codex-plugin-cc) on the diff against the plan; Fable subagent fallback
5. **Evidence smoke test** — dispatch an Opus subagent to PROVE the change works end-to-end with captured evidence; root-cause failures via systematic-debugging
6. **Ship** — commit and push, then move to the next task

## Model Selection & Fallback

The planner (step 1) runs on **Fable** by default — dispatch it with
`model: "fable"`. The **reviewer (step 4)** runs on **Codex** via
`/codex:adversarial-review` (see step 4); it only falls back to a Fable
subagent when the Codex plugin is unavailable.

**If Fable is unavailable in this environment** (the `fable` model is not
offered, or an Agent dispatch with `model: "fable"` errors as an unknown/
unsupported model), fall back to the second-best Claude model: **Opus 4.8
(1M context)** — dispatch with `model: "opus"` and set reasoning effort to
**extra high (`xhigh`)**. Do not silently fall through to a default model;
choosing the fallback should be a deliberate substitution so plan and review
quality stay as close to the Fable baseline as possible.

Apply the same rule wherever `model: "fable"` appears in this skill and in
`references/subagent-prompts.md`. The smoke-test subagent (step 5) always runs
on Opus and is unaffected.

## The One Rule That Matters

Every subagent and every codex session is **completely independent** — it has
no memory of this conversation, prior tasks, or other subagents. Every
dispatch must carry the full context it needs: the task statement, relevant
file paths, repo location, conventions, constraints, what was already done in
prior tasks, and the exact deliverable expected. A vague prompt produces a
useless plan, patch, or review. Use the templates in
`references/subagent-prompts.md` as the baseline for every dispatch.

## Step Details

### 0. Brainstorm

**Before planning, invoke `superpowers:brainstorming`** (the orchestrator runs it
inline with the user — not a subagent) to pin down what the task actually is:
intent, requirements, constraints, edge cases, and the intended design/approach.
Resolve open questions with the user here, while it's cheap — a plan (and the
stateless codex run that executes it) is only as good as the shared understanding
behind it. Skip for a trivially-specified task or in execute mode.

Carry the brainstorming outcome forward: feed the agreed intent + design +
constraints into the planner's prompt (step 1) so the plan reflects it rather
than re-deriving it. If brainstorming surfaces that the task is under-specified
or the wrong thing to build, stop and realign with the user before planning.

### 1. Plan

Dispatch via the Agent tool with `model: "fable"` (or the Opus 4.8 1M @
`xhigh` fallback — see **Model Selection & Fallback**). Instruct the subagent
to explore the repo, then write a concrete implementation plan to `plan.md` in
the repo root (overwrite any previous task's plan) — grounded in the step-0
brainstorming outcome (intent, design, constraints), which you pass into its
prompt. Use the planner template in `references/subagent-prompts.md`.

A good `plan.md` is directly executable by an engineer with no other context:
files to touch, functions/components to add or change, data flow, edge cases,
and how to verify.

### 2. Approve

Show the user the plan (summary plus path to `plan.md`) and ask for approval.
Require approval for **every** task's plan before implementation begins. If
the user requests changes, re-dispatch the planner with the feedback and the
previous plan, then ask again.

### 3. Implement with Codex

Run codex non-interactively with the plan as its instructions:

```bash
codex --yolo exec -c model="gpt-5.5" -c model_reasoning_effort="xhigh" < plan.md
```

In execute mode, substitute the user-provided plan file (or its
context-augmented scratchpad copy) for `plan.md`.

Run it with `run_in_background: true` — codex runs may take up to 30 minutes,
longer than the foreground Bash timeout allows. Wait for completion before
proceeding; do not start review while codex is still running. Full invocation
details, monitoring, and failure handling are in `references/codex-exec.md`.

### 4. Review

Review the resulting diff (`git diff` / `git status` — codex does not commit)
with **Codex's adversarial review** from the
[openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) plugin —
a steerable challenge review that tries to break the change, not rubber-stamp it:

```
/codex:adversarial-review
```

Steer it at THIS task: in the invocation, point it at the uncommitted diff and
give it the **plan content + task statement** as the contract to judge against,
and require its verdict to be one of **approve** or **findings** (a concrete
list of defects with `file:line`). `/codex:adversarial-review` is read-only and
runs as a background Codex job — use `/codex:status` to wait for it and
`/codex:result` to collect the verdict (`/codex:cancel` to abort a hung run);
do NOT start the smoke test until the review has returned.

**Prerequisite / fallback.** This requires the Codex plugin (install once:
`/plugin marketplace add openai/codex-plugin-cc` → `/plugin install
codex@openai-codex` → `/codex:setup`). If the plugin's commands are unavailable
in this environment, fall back to dispatching a **Fable subagent** (Agent tool,
`model: "fable"`, or the Opus 4.8 1M @ `xhigh` fallback — see **Model Selection
& Fallback**) with the same steer: review the diff against the plan + task, emit
`approve` | `findings`. Prefer the Codex adversarial review whenever available —
an independent (non-Claude) reviewer is the point.

On findings: feed them back through another codex exec run (pipe a fix
instruction file containing the findings and the original plan), then
re-review with `/codex:adversarial-review` again. Loop until the review approves.

### 5. Evidence Smoke Test

Dispatch an Opus subagent (Agent tool, `model: "opus"`) to verify the feature
actually works end-to-end **and to PROVE it with evidence, not assert it.**
The subagent must exercise whatever this stack provides — pick the one(s) that
actually hit the changed code path:

- **Tests** — run the project's unit / UI / integration suite (e.g.
  `xcodebuild test`, `./gradlew test` or `connectedAndroidTest`, `pytest`,
  `cargo test`, `go test`, `pnpm test`).
- **App run** — launch the app on a device or simulator (iOS Simulator,
  Android emulator, Electron shell, desktop binary) and drive the new flow.
  The `agent-device` skill can help automate this for iOS/Android — capture a
  screenshot/recording as evidence.
- **Endpoints** — start the dev server or service and hit the new routes
  (`curl`, HTTP client, RPC call) — capture request + full response.
- **CLI** — invoke the CLI directly with real arguments — capture command +
  stdout/stderr + exit code.

**Evidence is mandatory (this is the enhancement over a plain smoke test).**
For EVERY scenario the subagent must record: the exact command/request it ran,
the **raw observed output** (exit code, response body, the decisive log lines,
or a screenshot/recording saved to the scratchpad **with its path**), and
PASS/FAIL. A bare "looks fine" / a PASS with no shown evidence is **rejected** —
treat it as a failed smoke test and re-dispatch. This mirrors
`superpowers:verification-before-completion`: evidence before assertions, always.

**On FAIL, root-cause before reporting — do NOT guess.** The subagent MUST run
`superpowers:systematic-debugging` (Phase 1 — Root Cause Investigation):
1. Read the error/output completely (line numbers, codes, stack traces).
2. Reproduce it consistently; note the exact steps.
3. In a multi-component path (CI → build → sign, API → service → DB), gather
   evidence at EACH boundary — log what enters/exits each layer — to show
   *where* it breaks, then investigate that component.
4. Trace the bad value back to its source; report the failing boundary and the
   **proving evidence**, not a symptom or a hypothesis stated as fact.

The smoke test is **observe-only** — the subagent must NOT fix anything. Its
evidenced root-cause findings feed back exactly like review findings: to a
codex fix round (step 4's loop), then re-review and re-smoke-test.

### 6. Ship

Only after review approval and a passing **evidenced** smoke test: commit with
a conventional message describing the task, and push. Stage files explicitly —
never commit orchestration artifacts (`plan.md`, fix-instruction files); delete
the generated `plan.md` from the repo root after the task ships. In execute
mode the plan file belongs to the user: leave it in place and simply exclude
it from the commit. Then start the next task from step 1 (or step 3 in
execute mode). The run ends only when the `/goal` match criteria hold for
**every** task (all review-clean, all smoke-proven, all shipped) — at which
point the Stop-hook auto-clears; until then the session keeps working the loop.

## Task Tracking

With multiple tasks, keep a visible checklist (todo list) of all tasks and the
current pipeline stage of the active task. Process tasks strictly
sequentially — one task's implementation is context for the next task's
planner prompt.

## Discipline Gates — Don't Rationalize Around Them

This loop only works if every gate holds under pressure. **Violating the letter
of a gate is violating the spirit of it.** Common rationalizations and why they
are wrong:

| Excuse | Reality |
|--------|---------|
| "The plan is obvious, I'll skip the approval gate" | In the full loop, every task's plan needs explicit user approval before Implement — no exceptions. (Execute mode is the *only* skip: the user-supplied file already stands in for an approved plan.) |
| "The smoke test clearly passes, evidence is overkill" | A PASS with no shown evidence is rejected and re-dispatched. Evidence before assertions, always. |
| "This finding is tiny, I'll just hand-patch it in the orchestrator" | Fixes go through a codex fix round. Reserve direct edits for trivial one-liners (typo, import) only — never large gaps. |
| "The diff looks right, I'll review it myself and skip Codex" | The reviewer is an *independent, non-Claude* adversary. The orchestrator never self-reviews its delegated work. Use the Fable fallback only if the Codex plugin is truly unavailable. |
| "The smoke test failed but the cause is obvious" | The tester must root-cause via `superpowers:systematic-debugging` with proving evidence — a hypothesis stated as fact is not a root cause. |
| "I'll fix the failure right here in the smoke-test step" | Smoke test is observe-only. Findings feed back to a codex fix round; the tester never edits. |
| "I'll commit plan.md / the fix file too, it's harmless" | Never commit orchestration artifacts. Stage feature files explicitly; delete generated `plan.md` after ship. |
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
- **`superpowers:systematic-debugging`** — the root-cause method the
  smoke-test subagent runs on any failure (Phase 1 evidence-gathering)
- **`superpowers:verification-before-completion`** — evidence-before-assertions
  discipline the smoke test enforces
