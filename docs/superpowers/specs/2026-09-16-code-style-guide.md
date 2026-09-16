# Workout Planner & Guider — Code Style Guide

**Date:** 2026-09-16
**Status:** Draft for review
**Applies to:** all modules in this repository

---

## 1. Locked decisions

| # | Decision | Choice |
|---|---|---|
| S1 | Enforcement stack | ktlint + detekt; shared `.editorconfig`; pre-commit hook; CI gate |
| S2 | Style profile | Official Kotlin coding conventions; 120-col limit; trailing commas ON |
| S3 | API discipline | Explicit API mode on `core:*`; KDoc on domain public API |
| S4 | Error modeling | Sealed `Result<T, DomainError>` in domain; exceptions wrapped at IO boundaries |
| S5 | Coroutines | Structured concurrency; injected dispatchers; no `runBlocking` in production |
| S6 | Compose | One screen composable per file; state hoisted to ViewModels; previews optional |
| S7 | Tests | camelCase names + `// given / // when / // then` comments |
| S8 | Git | Conventional Commits; pre-commit hook; CI full check |

## 2. Enforcement (how the guide is checked)

| Checkpoint | What runs |
|---|---|
| Editor | `.editorconfig` shared by IDEs and ktlint |
| Pre-commit hook | `ktlintFormat` on staged files + `detekt` on changed modules (hook script checked into repo) |
| Gradle | `check` depends on `ktlintCheck` + `detekt` — no build passes with violations |
| CI | Full `./gradlew check` gate on every PR; iOS compile check on Mac runner (M7) |

Setup is an **M0 task in implementation plan 1** — no product code is written before the lint gate exists.

### 2.1 `.editorconfig` baseline (exact values)

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4
max_line_length = 120

[*.{kt,kts}]
ij_kotlin_name_count_to_use_star_import = 2147483647
ij_kotlin_name_count_to_use_star_import_with_member = 2147483647
insert_final_newline = true
ktlint_code_style = intellij_idea
ij_kotlin_allow_trailing_comma = true
ij_kotlin_allow_trailing_comma_on_call_site = true
```

## 3. Kotlin style rules (ktlint-managed)

- Official Kotlin conventions as the base; 4-space indent; 120-col hard limit.
- **Trailing commas on** (declaration + call sites) — enforced, not optional.
- **No wildcard imports**; no unused imports (ktlint errors).
- When-style exhaustiveness: `when` over sealed types must be exhaustive; prefer expression `when`.
- Data classes for DTOs/value types; value classes for domain IDs (`SessionId`, `ExerciseId`).
- Explicit visibility: `internal` by default for anything not crossing a module boundary (S3).

## 4. detekt baseline (balanced, documented thresholds)

Style/compose rules **delegated to ktlint** (no overlap). detekt owns complexity & smells:
- `LongMethod` ≤ 60 lines, `LongParameterList` ≤ 5, `TooManyFunctions` ≤ 10/file, `LargeClass` ≤ 200 lines
- `MagicNumber` off (units and UI tokens make constants noisy); naming rules on; `UnusedPrivateMember` on
- `ForbiddenImport`: `GlobalScope.*`, `java.util.Date`
- `Complexity`: cyclomatic ≤ 10, nested block depth ≤ 4
- Thresholds live in `config/detekt/detekt.yml` (checked into repo); deviations require a comment + issue link, never silent `@Suppress`

## 5. API & documentation discipline

- `core:domain`, `core:data`, `core:ai` compile with **explicit API mode** (`explicitApi()`): every public member needs an explicit modifier.
- **KDoc required** on every public declaration in `core:domain` (commands, entities, use cases, `DomainError` types). Elsewhere: KDoc on non-obvious public APIs, not on composables or trivial overrides.
- No comments explaining *what* — code reads itself; comments only for *why* (constraints, trade-offs, links to spec sections).

## 6. Error modeling (S4)

- Domain commands and use cases return `Result<T, DomainError>` (sealed class in `core:domain`, exhaustive `when` at call sites).
- `DomainError` subtypes per aggregate (e.g. `ValidationError`, `NotFound`, `SyncConflict`, `AiUnavailable`) — no stringly-typed errors.
- Exceptions are real only at IO boundaries (SQLDelight, supabase-kt, Koog executors); `core:data`/`core:ai` catch and map them into `Result`. Rethrowing across the domain boundary is a review-blocking violation.
- Never swallow: every caught exception maps to a `DomainError` or is logged-and-rethrown.

## 7. Coroutines rules (S5)

- Structured concurrency only: launched within a scope owned by the caller (viewModelScope, application scope); **`GlobalScope` forbidden** (detekt `ForbiddenImport`).
- Dispatchers injected via a small `CoroutineDispatchers` interface (io/default/main); no hardcoded `Dispatchers.IO` in domain logic.
- `runBlocking` forbidden in production code (test code allowed).
- Reactive reads are `Flow` from SQLDelight; UI collects via lifecycle-aware APIs; no callback-based async.

## 8. Compose conventions (S6)

- One screen composable per file, named `<Feature>Screen` in `composeApp/ui/screens/<feature>/`.
- State hoisted: screens take state + callbacks; ViewModels (multiplatform) own state via Koin.
- Reusable pieces go to `ui/components` with `@Composable` functions; no KDoc required (S5 exception); previews optional.
- Theming only via the token table (UI spec §2) — no hardcoded colors in screens.

## 9. Naming & file layout

- Files named after their primary declaration; one public purpose per file.
- Packages by aggregate, not by layer: `core:domain/{session,program,exercise}` each containing entities + commands + validation.
- Command handlers suffixed `Handler` (`LogSetHandler`); repositories named after aggregate (`SessionRepository`); fakes prefixed `Fake` in tests.
- Test classes: `<Subject>Test`; test methods camelCase describing behavior + `// given / when / then` comments (S7).

## 10. Git conventions (S8)

- Conventional Commits: `feat:`, `fix:`, `docs:`, `test:`, `chore:`, `refactor:` — matches existing repo history.
- Branches: `<type>/<short-topic>` (e.g. `feat/session-logging`).
- Commits stay small and buildable; every task in the implementation plan ends in a commit.
