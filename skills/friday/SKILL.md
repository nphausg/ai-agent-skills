---
name: friday
description: >
  Use when the user wants a batch of coding tasks driven through the friday
  orchestration loop (plan -> Codex implement -> adversarial review -> evidence
  smoke test -> commit) for web, mobile, backend, CLIs, or libraries. Triggers:
  "friday this", "run the friday loop", or execute mode "friday this plan
  <file>.md" when a ready plan file is supplied.
user-invocable: true
argument-hint: "[task description | <plan-file>.md for execute mode]"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, AskUserQuestion, Skill, TodoWrite
---

# friday — Plan / Codex / Review / Evidence-Test Orchestration Loop

friday implements the user's tasks one at a time through a fixed per-task
pipeline. This session coordinates only — it never writes feature code; it
delegates planning, implementation, review, and testing, gates on approval, and
does git. Every subagent and Codex run is **stateless**, so each dispatch must
carry full context (task, paths, repo, conventions, constraints, deliverable).

**Read `references/pipeline.md` before running** — it holds the full per-step
procedure, coding principles, model selection, completion tracking, and the
discipline gates.

## Pipeline (per task)

0. **Brainstorm** — `superpowers:brainstorming` with the user *(skip in execute mode)*.
1. **Plan** — Fable subagent writes a timestamped `plan_<timestamp>.md` *(skip in execute mode)*.
2. **Approve** — explicit user approval of the plan *(skip in execute mode)*.
3. **Implement** — `codex exec` runs the plan in the background (≤30 min); post progress.
4. **Review** — a parallel team of Codex adversarial reviewers, one per lens (correctness / security / performance / approach); any finding blocks -> rework to step 3 and re-review the whole team until every lens approves.
5. **Smoke test** — Opus subagent PROVES it works with captured evidence; root-cause any failure.
6. **Ship** — commit only (no push), then the next task.

## Modes

- **Full loop** (default) — run the whole pipeline including plan + approval.
- **Execute mode** — the user supplies a ready plan file; treat it as approved,
  skip steps 0–2, start at Implement piping that file. Never edit the user's file
  (pipe an augmented scratchpad copy if it isn't self-contained). Multiple tasks
  run sequentially, each with its own review -> smoke -> ship.

## Completion bar

First action of every run: create a `TodoWrite` checklist whose items ARE the
run's match criteria, so the loop self-continues instead of stopping early. A
task is done only when (a) every review lens approves, (b) the evidenced smoke
test PASSes, and (c) it is committed locally (no push). Fold user acceptance
criteria in verbatim.

## Models

Planner + fallback reviewer: Fable. Implementer: Codex `gpt-5.5` @ `xhigh`
(pinned). Smoke tester: Opus. If Fable is unavailable, substitute Opus 4.8 (1M)
@ `xhigh` — never a silent default.

## References

- `references/pipeline.md` — full step details, discipline gates, coding principles.
- `references/codex-exec.md` — codex exec invocation, monitoring, retry.
- `references/subagent-prompts.md` — planner / reviewer / smoke-test prompt templates.
