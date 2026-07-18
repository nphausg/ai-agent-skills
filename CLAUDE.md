# MUST
- MUST only communicate to me in ENGLISH, and start every reply with "Hi Leo, "
- Never commit unless asked. On "commit", auto-generate the message from the diff and commit immediately, no confirmation.
- NEVER create GitLab/GitHub MRs or PRs without being explicitly asked. Diagnosing or finding a fix does NOT mean create the MR -- only explain the fix and wait for an explicit instruction.
- Never push unless explicitly told to; after committing, ask before pushing.
- NEVER start implementing without completing the full planning workflow below. No exceptions.
- NEVER use typographic Unicode in authored content. Use ASCII only: `->` (not the arrow char), `-` or `--` (not em/en dashes), straight quotes `"` `'` (not curly quotes), `...` (not the ellipsis char). Reason: skill/marketplace/flow files are JSON-serialized with ensure_ascii=True, so any non-ASCII char becomes an ugly `\uXXXX` escape. Applies everywhere (SKILL.md frontmatter/descriptions, JSON, commit messages, docs, prose), not just JSON.

# Coding Principles & Workflow

The principles are the mindset; the workflow enforces them. Think before you type; do not skip steps.

## The Four Principles

The mindset behind the workflow. Each is a one-line rule plus its test; the workflow below is how they get enforced per task.

| Principle | Guards against | The rule (and its test) |
|-----------|----------------|-------------------------|
| Think Before Coding | wrong assumptions, hidden confusion | State assumptions, surface tradeoffs, and ask when ambiguous or confused -- never pick an interpretation silently. |
| Simplicity First | overengineering, bloat | Minimum code that solves it -- no speculative features, abstractions, config, or error-handling for impossible cases. Test: a senior engineer wouldn't call it overcomplicated. |
| Surgical Changes | orthogonal edits | Touch only what the task needs; match existing style; don't refactor or "improve" unrelated code; remove only the orphans your change creates, never pre-existing dead code (mention it). Test: every changed line traces to the request. |
| Goal-Driven Execution | vague success, "make it work" | Turn the task into a verifiable goal -- tests-first (reproduce/validate, then make pass), a verify check per step, loop until proven. |

## Workflow -- Anti-Vibe-Coding (every task, in order)

You are reviewing and implementing code in a large modular codebase. Every task follows this strict sequence.

### Step 1 -- Architecture Impact Analysis (ask first)
Before any code is written:
- Identify which modules/packages/layers are affected and why
- Identify layer boundaries crossed (e.g. data / domain / presentation / infra / API / UI)
- Flag any risk of tight coupling, leaking abstractions, or boundary violations
- **Ask the user to confirm the scope before proceeding**

### Step 2 -- Reuse Detection (ask before creating anything new)
- Search for existing utilities, helpers, base classes, hooks, or abstractions that already solve this
- Prefer reusing over creating
- List candidates and ask the user which to reuse
- **Do not create new files/classes/functions if an equivalent already exists**

### Step 3 -- Task DAG (break down before coding)
- Decompose the task into a dependency-ordered list (DAG) of atomic sub-tasks
- Each sub-task must be independently reviewable and testable
- Present the breakdown and get user approval before proceeding
- **Never implement the whole feature in one shot**

### Step 4 -- Minimal Diff Only
- Generate only the code required for the current sub-task
- Do not rewrite entire files unless explicitly necessary and user-approved
- Prefer `Edit` over `Write` -- patch, don't replace
- Each diff must be explainable line-by-line

### Step 5 -- Unit Tests
- Add or update unit tests for every behavioral change
- Tests must cover: happy path, edge cases, error states
- Do not skip tests unless the user explicitly says so

### Step 6 -- Review Pass (verify after generation)
After generating code, perform a self-review pass:
- [ ] No blocking calls on performance-critical paths (e.g. main thread, event loop, render thread)
- [ ] No unnecessary new abstractions, thread pools, or global state introduced
- [ ] Module/package boundaries respected -- no internal cross-boundary imports
- [ ] No unnecessary new dependencies added
- [ ] Diff is minimal -- no unrelated changes included
- [ ] Tradeoffs explained to the user

### Step 7 -- Explain Tradeoffs
Always state:
- Why this approach over alternatives
- What was deliberately excluded and why
- Any tech debt introduced and how to address it later

### Hard Rules
- **Always ask, then plan, then implement, then verify.** The specific gates -- plan approval, no blocking of performance-critical paths, no cross-boundary internal imports, no whole-file rewrites, no new dependency without flagging -- are enforced in the steps above; do not violate them.

# Git AI Attribution Workflow

## Before Commit
1. Check for unpushed commits (`git log @{u}..HEAD`) and staged/unstaged changes (`git status`, `git diff`)
2. Stage all relevant changes (`git add <files>`)
3. Commit (see Commit section below)

## Commit
- Follow conventional commits format: `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`, `test:`
- Always use `--no-verify` on every `git commit` to skip pre-commit hooks (e.g. ktlint, formatters). Never let hooks auto-format code.
- Always append Claude attribution to every commit message:
  ```
  🤖 Generated with [Claude Code](https://claude.com/claude-code)

  Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
  ```

## Pull Request / Merge Request
- When asked to create an MR or PR: always auto-generate the description based on the diff/commits -- do NOT ask for approval first. Include a summary of changes and a changed-files table.
- Always create MRs as **Draft** (`draft: true`) when using the GitLab MCP tool.
- Always use `$GITLAB_TOKEN` from the shell environment -- not from `.env` file.
- Always append Claude attribution to every PR/MR description:
  ```
  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  ```

## Pre-existing File Rule (prevent attribution failures before they happen)
For files created or modified **outside** Claude Code (e.g., by external pipelines, scripts, or manual edits):
- **New/untracked files**: Delete the file first (`rm <file>`), then recreate using the Write tool. Never use Write tool to overwrite pre-existing content -- git-ai only attributes the diff, not unchanged lines.
- **Modified tracked files**: Run `git restore <file>` first to reset to committed state, then apply changes via Edit tool (only the diff is needed for 100% AI).

# Dangerous Command Confirmation

When a Bash command is blocked by a hook (PreToolUse hook exits with code 2), follow this flow:

1. Use `AskUserQuestion` to ask the user for authorization, showing the full command
2. User approves -> append `# --claude-approved` to the end of the command and retry
3. User rejects -> abort, do not retry

Example:
- Original command: `rm -rf /some/path`
- Retry command:    `rm -rf /some/path # --claude-approved`