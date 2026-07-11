# Subagent Prompt Templates

Every subagent is a fresh context. Fill every placeholder — never assume the
subagent can see this conversation, the task list, or earlier subagents'
output. When a template asks for "prior tasks completed", include a one-line
summary per prior task; the planner and reviewer both need it to avoid
re-doing or breaking earlier work.

> Model: use `model: "fable"` for the planner. The **reviewer** is normally
> Codex's `/codex:adversarial-review` (openai/codex-plugin-cc) — the template
> below is the **fallback** used only when that plugin is unavailable. When you
> do use it, run it on `model: "fable"` (fallback Opus 4.8 1M @ `xhigh`). See
> "Model Selection & Fallback" and step 4 in SKILL.md.

## Planner (Agent tool, `model: "fable"` — fallback: Opus 4.8 1M @ `xhigh`)

```
You are planning one implementation task in the repo at <ABSOLUTE REPO PATH>.

## Task
<full task statement from the user, verbatim>

## Context
- Project: <one-paragraph description: stack, framework, package manager, and how to build / run / test the project>
- Prior tasks already completed in this batch: <list or "none">
- Constraints: <conventions, files to avoid, perf/security requirements, or "none stated">

## Your job
1. Explore the repo enough to plan concretely: find the files, patterns, and
   existing abstractions this task should build on.
2. Write the plan to <ABSOLUTE REPO PATH>/plan.md (overwrite it).

## plan.md requirements
The plan will be executed verbatim by a separate coding agent with NO other
context, so it must be self-contained:
- Restate the task and the repo-relative paths of every file to create/modify
- For each file: what changes and why, with function/component names
- Data flow and integration points with existing code
- Edge cases to handle
- How to verify the change works (commands, requests)
- Explicitly out of scope: what NOT to touch

Return the plan.md content as your final message as well.
```

## Reviewer — FALLBACK ONLY (Agent tool, `model: "fable"` — fallback: Opus 4.8 1M @ `xhigh`)

> Primary reviewer is `/codex:adversarial-review` (see SKILL.md step 4). Use this
> subagent template only when the Codex plugin is unavailable. The same steer —
> diff vs plan + task, verdict `approve` | `findings` with `file:line` — applies
> to either path.

```
You are reviewing an uncommitted implementation in the repo at <ABSOLUTE REPO PATH>.
Run `git status --short` and `git diff` to see it. Do not modify any files.

## Task it implements
<full task statement>

## Prior tasks completed in this batch
<list or "none" — the diff must not break or re-do this work>

## Approved plan
<full plan.md content>

## Review for
1. Plan conformance — everything in the plan implemented, nothing out of scope
2. Correctness — logic errors, unhandled edge cases, broken integrations with
   existing code (read surrounding code, not just the diff)
3. Regressions — does the diff break anything that previously worked?
4. Repo conventions — style, naming, and idiom consistent with neighbors

## Verdict format (final message)
Either:
APPROVE — <one-line reason>
Or:
FINDINGS:
1. <file>:<line> — <defect and why it is wrong>
2. ...
Only list real defects that must be fixed before commit; put style
suggestions, if any, under a separate "Optional:" heading.
```

## Evidence Smoke Tester (Agent tool, `model: "opus"`)

```
You are smoke-testing a newly implemented feature in the repo at <ABSOLUTE REPO PATH>.
Do not modify any files — only run, observe, and report. If something is
broken, report it with evidence; do NOT fix it.

You MUST prove the feature works with captured evidence — never assert PASS
without showing the observed behavior. Load and follow the
`superpowers:verification-before-completion` discipline (evidence before
assertions). On ANY failure, load and follow `superpowers:systematic-debugging`
before you report (see "On failure" below).

## Feature under test
<task statement + one-paragraph summary of what was implemented>

## How to exercise the project
<stack-specific verification commands — pick what fits:
 - Tests: e.g. `xcodebuild test -scheme <X>`, `./gradlew test`, `pytest`,
   `cargo test`, `go test ./...`, `pnpm test`
 - App run: launch on iOS Simulator / Android emulator / desktop shell and
   drive the flow (the `agent-device` skill can help for iOS/Android; capture a
   screenshot/recording)
 - Endpoints: dev command (e.g. `pnpm dev`), port, required env vars, then
   HTTP requests against the new routes
 - CLI: exact command + arguments
Include the concrete commands, ports, env vars, or fixtures the subagent
will need. Scratchpad for evidence artifacts: <SCRATCHPAD DIR>.>

## Your job
1. Bring the change into a runnable state (build/install/start whatever this
   stack needs) and wait until it is ready.
2. Exercise the NEW feature with real interactions that hit the new code
   path — actual test runs, app interactions on device/simulator, HTTP
   requests, or CLI invocations:
   <2-4 concrete scenarios derived from the plan, including one edge case>
3. Confirm at least one pre-existing flow still works: <name it>.
4. Capture EVIDENCE for every scenario: the exact command/request, the raw
   observed output (exit code, response body, decisive log lines) pasted in,
   and any screenshot/recording/log saved under <SCRATCHPAD DIR> with its path.
5. Tear down anything you started (dev server, simulator, emulator, temp
   processes) when done.

## On failure — root-cause with evidence (do NOT guess, do NOT fix)
If any scenario FAILs, run `superpowers:systematic-debugging` Phase 1 before
reporting:
- Read the error/output completely (line numbers, codes, stack traces).
- Reproduce it consistently; record the exact steps.
- In a multi-component path (CI→build→sign, API→service→DB, UI→VM→data), gather
  evidence at EACH boundary (log what enters/exits each layer) to show WHERE it
  breaks, then investigate that component.
- Trace the bad value back to its source.
Report the failing boundary and the PROVING evidence — the actual output that
demonstrates the root cause — not a symptom or an unproven hypothesis.

## Report format (final message)
For each scenario, a block:
- Scenario: <name>
- Command/request: <exact>
- Observed (raw): <pasted output / exit code / response, or evidence path>
- Verdict: PASS | FAIL

Then an overall verdict:
PASS — every scenario behaved as specified, each backed by shown evidence.
FAIL — followed by, for each failure: expected, observed, the systematic-debugging
ROOT CAUSE (failing component/boundary + proving evidence), and relevant log output.
Never report PASS for a scenario without pasted/attached evidence of the observed behavior.
```
