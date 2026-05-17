# AI Agent Skills


A curated collection of rules, skills, and patterns for AI coding agents (Cursor, Claude Code, and similar tools) — designed to generate high-quality, production-ready code across multiple platforms.

## What's Inside

| Category | Path | Description |
|---|---|---|
| Android | `.cursor/rules/android/` | MVVM, Hilt DI, Coroutines, testing, WorkManager |
| Frontend | `.cursor/rules/frontend/` | React Hooks best practices |
| General | `.cursor/rules/general/` | Clean code, performance tips |
| iOS | `.cursor/rules/ios/` | *(coming soon)* |
| Skills | `skills/` | Claude Code slash commands (e.g., `/debug`) |

## Cursor Rules (`.mdc` files)

Rules live in `.cursor/rules/` and are picked up automatically by Cursor IDE. Each file targets specific file patterns via `globs` metadata.

### How to use

**Option 1 — Automatic (recommended):** Place the repo or its `.cursor/rules/` folder in your project; Cursor loads matching rules automatically.

**Option 2 — Manual:** Copy the relevant rule block and prepend it to your prompt:
```
Please follow this rule strictly:

[paste rule content here]

Now, here's my request: ...
```

### Key principles encoded in the rules

**Android**
- MVVM: business logic in `ViewModel`, views only observe `StateFlow`
- Clean Architecture: Domain / Data / Presentation layer separation; all business logic in `UseCase`
- Hilt dependency injection (`@HiltViewModel`, `@Inject`, `@Module`)
- Coroutines: `viewModelScope`, `Dispatchers.IO` for I/O, no heavy work on main thread
- Testing: BDD-style naming (`given_when_then`), MockK/Mockito, `runTest`

**Frontend**
- Functional components only
- Cleanup in `useEffect`, stable references with `useCallback` / `useMemo`

**General**
- Single Responsibility, DRY, `lazy` initialization, avoid reflection

## Claude Code Skills (`skills/`)

Skills extend Claude Code with slash commands usable in any project.

| Skill | Command | Purpose |
|---|---|---|
| debug | `/debug` | Structured bug diagnosis — gather context, find root cause, show minimal fix |

To install a skill into a project, copy its `SKILL.md` into `.claude/skills/<name>/SKILL.md`.

## Rule Review Checklist

| Item | Details |
|---|---|
| Naming & location | File name descriptive? Correct folder? |
| `globs` accuracy | Patterns match intended files? |
| Clarity | Rules are actionable and unambiguous? |
| Conciseness | No redundancy; under ~100 lines |
| Consistency | No conflicts with existing rules |
| Token efficiency | Validated with `scripts/analyze_token_usage.py` |
| AI compliance | Tested with `scripts/run_ai_prompt_tests.py` |

## Scripts

```bash
# Check token usage across all rule files (flag files > 1500 tokens)
python scripts/analyze_token_usage.py

# Test AI compliance against expected keywords (requires OPENAI_API_KEY)
OPENAI_API_KEY=... python scripts/run_ai_prompt_tests.py
```

## Contributing

Pull requests welcome. Please use the PR template and run through the review checklist before submitting.

<a href="https://revolut.me/nphausg" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="nphausg" style="height: 41px !important;width: 174px !important;box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;-webkit-box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;" ></a>

## Author

<p>
    <a href="https://nphausg.medium.com" target="_blank">
    <img src="https://avatars2.githubusercontent.com/u/13111806?s=400&u=f09b6160dbbe2b7eeae0aeb0ab4efac0caad57d7&v=4" width="96" height="96">
    </a>
</p>
