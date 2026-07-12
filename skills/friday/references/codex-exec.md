# Codex Exec — Invocation and Handling

## Canonical command

```bash
codex --yolo exec -c model="gpt-5.5" -c model_reasoning_effort="xhigh" < plan_<timestamp>.md
```

Always run codex with exactly this model (`gpt-5.5`) and reasoning effort
(`xhigh`) — do not downgrade either for "small" tasks or fix rounds.

- `--yolo` — full-access non-interactive mode; codex edits files and runs
  commands without prompting. This is intentional for the friday loop: the plan
  was already human-approved.
- `exec` — non-interactive run; with stdin piped, the piped file is the
  instructions.
- `-c model=... -c model_reasoning_effort=...` — config overrides. These are
  mandatory pins, not defaults: if the local codex install rejects
  `gpt-5.5` or `xhigh`, stop and report the error to the user — do not
  silently fall back to another model or a lower effort.

## Coding principles (prepend to every piped plan/fix file)

Codex is stateless and never sees `SKILL.md`, so the **Coding Principles** must
travel *inside* what you pipe it. Prepend this block to the plan (and every fix
file) before piping:

```markdown
## Coding principles (follow strictly)
- Simplicity First: implement the minimum that satisfies the plan — no features
  beyond it, no speculative abstractions/config, no error handling for
  impossible cases. If it could be much shorter, make it shorter.
- Surgical Changes: change only what the plan requires. Do not refactor,
  reformat, or "improve" unrelated code; match existing style; remove only the
  orphans your change creates; leave pre-existing dead code alone.
```

## Running it

Always launch with `run_in_background: true` and redirect output to a log in
the scratchpad, e.g.:

```bash
codex --yolo exec -c model="gpt-5.5" -c model_reasoning_effort="xhigh" < plan_<timestamp>.md > "$SCRATCHPAD/codex-task-N.log" 2>&1
```

Allow up to 30 minutes. Wait for the background completion notification —
do not poll in a tight loop and do not start the review stage early. After
completion, read the tail of the log to confirm codex finished cleanly and to
capture its summary of what it changed.

## Verifying the run

Codex does not commit. After the run:

```bash
git status --short && git diff --stat
```

An empty diff after a "successful" codex run means the run failed silently —
read the full log before retrying.

## Feeding back review findings

For a fix round, write a fix-instruction file **to the scratchpad directory**
(not the repo — it must never end up in a commit) and pipe it the same way:

```markdown
# Fix round for: <task title>

## Coding principles (follow strictly)
<the Simplicity First + Surgical Changes block above>

## Original plan
<full plan content>

## What was implemented
<one-paragraph summary of the current diff>

## Review findings to fix (fix ALL of these — surgically; change nothing else)
1. <file:line — finding>
2. ...
```

```bash
codex --yolo exec -c model="gpt-5.5" -c model_reasoning_effort="xhigh" < "$SCRATCHPAD/fixes-task-N.md"
```

Codex sessions are stateless between invocations — the fix file must restate
the plan and findings in full, never "as discussed" or "see above".

## Failure handling

- **Non-zero exit / crash**: read the log; if transient (network, rate
  limit), retry once. Otherwise surface the error to the user.
- **Ran but wrong direction**: do not hand-patch large gaps in the
  orchestrator session. Write a corrective instruction file and run another
  codex round. Reserve direct edits for trivial one-liners (typo, import).
- **Timeout at 30 min**: kill the background task, check `git status` — if
  the diff is substantially complete, proceed to review; if not, split the
  plan into smaller pieces and re-run per piece.
- **Resume**: `codex exec resume --last` continues the most recent session if
  a run was interrupted mid-stream.
