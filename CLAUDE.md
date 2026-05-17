# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a curated library of Cursor AI rule files (`.mdc` files) that define coding standards, architectural patterns, and best practices for Android, iOS, and frontend development. Rules are consumed by Cursor IDE to improve AI code generation quality.

## Repository Structure

```
.cursor/rules/
  android/    # 8 rule files — MVVM, Hilt DI, coroutines, testing, WorkManager
  ios/        # Empty — no rules yet (opportunity to contribute)
  frontend/   # React Hooks best practices
  general/    # Clean code principles, performance tips
scripts/
  analyze_token_usage.py      # Token efficiency analysis for rule files
  run_ai_prompt_tests.py      # AI compliance testing via OpenAI API
```

## Scripts

```bash
# Analyze token usage across all rule files (flag files > 1500 tokens)
python scripts/analyze_token_usage.py

# Test rule compliance against expected keywords (requires OPENAI_API_KEY)
OPENAI_API_KEY=... python scripts/run_ai_prompt_tests.py
```

## Rule File Format (`.mdc`)

Each rule file has a YAML frontmatter header followed by markdown content:

```yaml
---
description: Short description of the rule's purpose
globs: ["**/*.kt", "**/*ViewModel.kt"]   # file patterns this rule applies to
alwaysApply: true   # true = automatic, false = optional
---
```

Rules are composable — multiple rules can apply to the same file (e.g., naming conventions stack with architecture rules).

## Architecture Encoded in Rules

**Android rules** enforce:
- MVVM pattern: `ViewModel` + `StateFlow` + `UiState` sealed classes
- Clean Architecture: Domain / Data / Presentation layer separation; all business logic in `UseCase` classes
- Dependency injection via Hilt (`@HiltViewModel`, `@Inject`, `@Module`)
- Coroutines: `viewModelScope`, `Dispatchers.IO` for I/O, `Dispatchers.Main` for UI
- Testing: BDD-style naming (`given_when_then`), MockK/Mockito, `runTest` for coroutines

**Frontend rules** enforce React Hooks best practices: cleanup in `useEffect`, stable references with `useCallback`/`useMemo`, functional components only.

**General rules** cover cross-cutting concerns: function size, DRY, lazy initialization, reflection avoidance, Compose recomposition optimization.

## Contributing Rules

1. Reference existing Android rules in `.cursor/rules/android/` for format examples.
2. Keep rules under 1500 tokens (checked by `analyze_token_usage.py`).
3. Use the PR template checklist: verify `globs` accuracy, clarity, conciseness, consistency with other rules, and token load.
4. iOS rules directory is currently empty — new iOS rules should follow the same `.mdc` format and align with Swift/SwiftUI conventions.
