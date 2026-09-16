# Workout App — Plan 1 of 5: Spike, Scaffold, Lint Gate, Auth (M0 + M1)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Validate the toolchain on-device (spike), scaffold the KMP workspace with the lint gate active, create the Supabase schema with verified RLS, and ship a working email/password auth screen — Android-runnable end to end.

**Architecture:** Four Gradle modules (`composeApp`, `core:domain`, `core:data`, `core:ai`) on Kotlin Multiplatform with Compose Multiplatform UI. Supabase (Auth + Postgres + RLS + RPC) is the backend; supabase-kt is wrapped behind an interface in `core:data`. Lint (ktlint + detekt) gates `check` before any product code lands.

**Tech Stack:** Kotlin 2.2.x, Compose Multiplatform 1.12.x, Koog 1.2.0 (used in Plan 4), supabase-kt 3.x + Ktor 3.x, SQLDelight 2.3.2 (used in Plan 2), Koin 4.2.x, ktlint + detekt.

**Spec:** `docs/superpowers/specs/2026-09-16-workout-planner-guider-design.md` (architecture §3–§9, §13 spike), `docs/superpowers/specs/2026-09-16-code-style-guide.md` (S1–S8), `docs/superpowers/specs/2026-09-16-workout-app-ui-design.md` (§3.1 Auth). Executors read all three.

**Companion plans (written sequentially as predecessors complete):**
- Plan 2: `2026-09-16-02-offline-logging-session-ui.md` (M2)
- Plan 3: `2026-09-16-03-sync-engine.md` (M3)
- Plan 4: `2026-09-16-04-ai-quick-entry.md` (M4)
- Plan 5: `2026-09-16-05-programs-coach-progress.md` (M5–M7)

## Global Constraints

- Provisional versions below are **re-pinned in Task 1** from spike evidence; if the spike contradicts a value, the spike result wins and `gradle/libs.versions.toml` is updated before continuing.
  - Kotlin `2.2.20`; Compose Multiplatform `1.12.0` (Jetpack Compose artifacts `1.12.0`); SQLDelight `2.3.2`; Koog `1.2.0` (stable `koog-agents` umbrella only); Koin `4.2.x`; supabase-kt BOM `3.x` (newest stable, spike-confirmed); Ktor `3.x` (spike-confirmed); navigation-compose for CMP: newest **stable** release line (never the docs' alpha example).
- Android `minSdk 26`, `compileSdk 36`, target JVM toolchain 17. iOS targets `iosArm64`, `iosSimulatorArm64`, `iosX64`; iOS 14+ floor.
- `core:domain`, `core:data`, `core:ai` use Kotlin **explicit API mode** (`explicitApi()`); `composeApp` does not.
- Style guide S1–S8 are binding from Task 3 onward: trailing commas, 120 cols, no wildcard imports, no `GlobalScope`, sealed `AppResult<T>` errors in domain, Conventional Commits.
- Windows dev box: Android + JVM targets must build/test here; iOS targets must *configure* (Gradle task `:composeApp:iosSimulatorArm64Klibs` or `linkDebugFrameworkIosSimulatorArm64` may be run only on a Mac — on Windows, `tasks` resolution must not fail). iOS runtime validation is deferred to Mac/CI and gated in Plan 5 (M7).
- Every task: red test → green implementation → full `check` green → commit. Commits are Conventional Commits and stay buildable.
- No product code may be committed before Task 3 (lint gate) is green.
- Secrets: Supabase URL/anon key read from `local.properties` (gitignored) or env vars; never committed. API keys never appear in code or tests.

---

### Task 1: Phase 0 spike — Koog tool plumbing + Supabase auth round-trip (throwaway)

**Files:**
- Create (temp, outside repo): `%TEMP%/opencode/spike-koog-supabase/` — settings/build scripts, two JVM test files
- Create: `docs/superpowers/spikes/2026-09-16-koog-supabase-spike.md`
- Modify: `.gitignore` (append `local.properties`, `**/build/`, `.idea/`, `*.iml`, `local.db`)

**Interfaces:**
- Consumes: nothing (first task)
- Produces: spike note with **verified** versions + go/no-go verdict consumed by Task 2 (pins) and later plans

- [ ] **Step 1: Create the throwaway Gradle project**

In `%TEMP%\opencode\spike-koog-supabase\` create `settings.gradle.kts` + `build.gradle.kts` (plain JVM project — Koog and supabase-kt both run on JVM; this validates API shape and versions, not mobile-specific packaging):

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.2.20"
    application
}
repositories { mavenCentral() }
dependencies {
    implementation("ai.koog:koog-agents:1.2.0")
    implementation(platform("io.github.jan-tennert.supabase:bom:3.2.2"))
    implementation("io.github.jan-tennert.supabase:auth-kt")
    implementation("io.github.jan-tennert.supabase:postgrest-kt")
    implementation("io.ktor:ktor-client-cio:3.3.2")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.2")
    testImplementation(kotlin("test"))
}
kotlin { jvmToolchain(17) }
```

If any resolved version differs (e.g. supabase BOM 3.x is newer), keep the newest stable and note it in the spike file.

- [ ] **Step 2: Create a dev Supabase project and local creds**

Sign up at supabase.com (free tier), create project `workout-guider-dev`. Wait for provisioning. Create `local.properties` (in spike dir, gitignored) with `supabase.url` and `supabase.anonKey` from Project Settings → API. These are the same creds Tasks 5–8 use.

- [ ] **Step 3: Prove Koog class-based tool plumbing with a mocked executor**

Write `src/main/kotlin/spike/KoogSpike.kt` + `src/test/kotlin/spike/KoogSpikeTest.kt`. The tool is **class-based** (`SimpleTool` with serializable args — annotation tools are JVM-only, but `SimpleTool` is the portable pattern we reuse on mobile in Plan 4):

```kotlin
package spike

import ai.koog.agents.core.tools.SimpleTool
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.annotations.LLMDescription
import kotlinx.serialization.Serializable
import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.clients.openai.OpenAIModels

@Serializable
data class LogSetArgs(
    @LLMDescription("exercise name") val exercise: String,
    @LLMDescription("reps performed") val reps: Int,
    @LLMDescription("weight in kg") val weightKg: Double
)

class LogSetTool(private val sink: MutableList<LogSetArgs>) :
    SimpleTool<LogSetArgs>(argsType = typeTokenOf<LogSetArgs>(), name = "log_set",
        description = "Log a performed set") {
    override suspend fun execute(args: LogSetArgs): String {
        sink.add(args); return "logged"
    }
}
```

The test uses Koog's test utilities (`agents-test` artifact, `MockLLMBuilder`/canned tool-call response) to drive an `AIAgent` whose `toolRegistry` contains `LogSetTool`; assert the sink received the parsed args. Exact mock API per `docs.koog.ai/testing/` — adapt import paths to the version resolved in Step 1 and note them.

- [ ] **Step 4: Prove Supabase auth + RLS round-trip**

```kotlin
suspend fun spike(supabase: SupabaseClient) {
    val email = "spike+${Clock.System.now().toEpochMilliseconds()}@test.local"
    supabase.auth.signUpWith(Email) { this.email = email; password = "password123" }
    val session = supabase.auth.currentSessionOrNull() ?: error("no session after signup")
    // Insert into a scratch table `spike_notes(owner_id, text)` (created via dashboard SQL)
    supabase.from("spike_notes").insert(mapOf("text" to "hello"))
    val rows = supabase.from("spike_notes").select().decodeList<Map<String, Any?>>()
    check(rows.isNotEmpty()) { "round trip failed" }
}
```

Run `.\gradlew.bat test` — both tests green. If the Google client is needed later, also smoke the beta `prompt-executor-google-client` resolve only (no runtime test).

- [ ] **Step 5: Record results and verdict**

Write `docs/superpowers/spikes/2026-09-16-koog-supabase-spike.md`: exact resolved versions table, working import paths for Koog mock testing, supabase-kt notes (Ktor engine constraints, session storage config), any gotchas, and the go/no-go verdict. If no-go on a core assumption → STOP, report to the human before scaffolding.

- [ ] **Step 6: Commit the spike note only**

```bash
git add docs/superpowers/spikes/ .gitignore
git commit -m "docs: spike results - koog tools + supabase round trip validated, versions pinned"
```

---

### Task 2: Gradle KMP workspace scaffold (4 modules, version catalog)

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `gradle/libs.versions.toml`
- Create: `core/domain/build.gradle.kts`, `core/domain/src/commonMain/kotlin/...` (empty package marker), `core/domain/src/commonTest/kotlin/…`
- Create: `core/data/build.gradle.kts`, `core/ai/build.gradle.kts` (same shape as domain)
- Create: `composeApp/build.gradle.kts`, `composeApp/src/commonMain/kotlin/…/App.kt` (placeholder "Hub — Plan 2" text), `composeApp/src/androidMain/…/MainActivity.kt`, `composeApp/src/iosMain/kotlin/…/MainViewController.kt`, `composeApp/src/androidMain/AndroidManifest.xml`

**Interfaces:**
- Consumes: pins from Task 1
- Produces: `./gradlew.bat build` green across all modules; module coordinates used by every later task (`core:domain`, `core:data`, `core:ai`, `composeApp`)

- [ ] **Step 1: Version catalog + settings + root build**

`gradle/libs.versions.toml` (versions provisional except SQLDelight/Koog — spike-verified values overwrite the provisional ones):

```toml
[versions]
kotlin = "2.2.20"            # spike-verify
composeMultiplatform = "1.12.0"
koin = "4.2.0"               # spike-verify newest 4.2.x
sqldelight = "2.3.2"         # confirmed via docs
koog = "1.2.0"               # confirmed via docs
supabaseBom = "3.2.2"        # spike-verify
ktor = "3.3.2"               # spike-verify
coroutines = "1.10.2"
serialization = "1.9.0"      # spike-verify
datetime = "0.7.1"           # spike-verify newest stable
navigation = "2.9.0"         # newest STABLE CMP navigation; never the docs alpha

[libraries]
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }
koin-compose = { module = "io.insert-koin:koin-compose", version.ref = "koin" }
koin-viewmodel = { module = "io.insert-koin:koin-viewmodel", version.ref = "koin" }
supabase-auth = { module = "io.github.jan-tennert.supabase:auth-kt" }
supabase-postgrest = { module = "io.github.jan-tennert.supabase:postgrest-kt" }
supabase-bom = { module = "io.github.jan-tennert.supabase:bom", version.ref = "supabaseBom" }
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
koog-agents = { module = "ai.koog:koog-agents", version.ref = "koog" }
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }
navigation-compose = { module = "org.jetbrains.androidx.navigation:navigation-compose", version.ref = "navigation" }
kotlinx-datetime = { module = "org.jetbrains.kotlinx:kotlinx-datetime", version.ref = "datetime" }

[plugins]
kotlinMultiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
composeMultiplatform = { id = "org.jetbrains.compose", version.ref = "composeMultiplatform" }
composeCompiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlinSerialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
androidApplication = { id = "com.android.application", version = "8.9.1" }
androidLibrary = { id = "com.android.library", version = "8.9.1" }
sqldelight = { id = "app.cash.sqldelight", version.ref = "sqldelight" }
```

`settings.gradle.kts` includes the four modules with `rootProject.name = "workout-planner-guider"`. Root `build.gradle.kts` declares `plugins { alias(libs.plugins.kotlinMultiplatform) apply false … }`.

- [ ] **Step 2: Module build files**

`core/domain/build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidLibrary)
    alias(libs.plugins.kotlinSerialization)
}
kotlin {
    explicitApi()
    androidTarget()
    iosArm64(); iosSimulatorArm64(); iosX64()
    sourceSets {
        commonMain.dependencies {
            implementation(libs.coroutines.core)
            implementation(libs.serialization.json)
            implementation(libs.kotlinx.datetime)
        }
        commonTest.dependencies { implementation(kotlin("test")) }
    }
}
android { namespace = "com.workoutplanner.guider.core.domain"; compileSdk = 36
    defaultConfig { minSdk = 26 } }
```

`core/data` adds supabase-bom + auth-kt + postgrest-kt + ktor-core (+ sqldelight plugin wiring in Plan 2); `core/ai` adds koog-agents; both with `explicitApi()`. `composeApp` is an Android application + iOS entry with compose plugins, depends on all three core modules, minSdk 26.

- [ ] **Step 3: Placeholder app entry points**

`App.kt` shows `Text("Hub — built in Plan 2")` centered on the theme background; `MainActivity` and `MainViewController` call it. Manifest with `MainActivity` exported, no permissions yet.

- [ ] **Step 4: Verify build + commit**

Run `.\gradlew.bat build` — all modules compile, commonTest smoke (`core/domain` gets one trivial test: `fun scaffoldCompiles() = Unit` guarded assertion) passes. Expected: BUILD SUCCESSFUL.

```bash
git add settings.gradle.kts build.gradle.kts gradle/ gradle.properties core/ composeApp/ .gitignore
git commit -m "feat: scaffold KMP workspace with 4 modules and version catalog"
```

---

### Task 3: Lint gate — EditorConfig + ktlint + detekt + pre-commit hook

**Files:**
- Create: `.editorconfig` (exact contents from style guide §2.1), `config/detekt/detekt.yml`, `scripts/hooks/pre-commit`, `scripts/hooks/install-hooks.ps1`
- Modify: `build.gradle.kts` (root: ktlint + detekt plugin wiring), `gradle/libs.versions.toml` (ktlint `12.x` via `org.jmailen.kotlinter`? — **decision: use `org.jmailen.gradle:kotlinter` Gradle plugin** for ktlint integration; detekt `io.gitlab.arturbosch.detekt`)
- Modify: module build files to apply `lint` conventions

**Interfaces:**
- Consumes: scaffold from Task 2
- Produces: `.\gradlew.bat check` fails on style violations; hook enforces on commit — prerequisite for all later commits

- [ ] **Step 1: Add .editorconfig verbatim**

Copy the ini block from `docs/superpowers/specs/2026-09-16-code-style-guide.md` §2.1 into `/.editorconfig` (repo root).

- [ ] **Step 2: Write detekt.yml with the documented thresholds**

```yaml
build:
  maxIssues: 0
complexity:
  CyclomaticComplexMethod: { active: true, threshold: 10 }
  NestedBlockDepth: { active: true, threshold: 4 }
  LongMethod: { active: true, threshold: 60 }
  LongParameterList: { active: true, functionThreshold: 5, constructorThreshold: 6 }
  TooManyFunctions: { active: true, thresholdInFiles: 10 }
  LargeClass: { active: true, threshold: 200 }
naming: { active: true }
style:
  MagicNumber: { active: false }
  UnusedPrivateMember: { active: true }
imports:
  ForbiddenImport:
    active: true
    imports: [ 'GlobalScope.*', 'java.util.Date' ]
```

- [ ] **Step 3: Wire plugins into root build**

Apply `io.gitlab.arturbosch.detekt` (version `1.23.x`, spike-verify newest stable) + `org.jmailen.kotlinter` (version `5.x`, spike-verify) at the root with `subprojects` configuration so every module gets `lintKotlin`/`detekt`; make root `check` depend on both:

```kotlin
subprojects {
    apply(plugin = "io.gitlab.arturbosch.detekt")
    apply(plugin = "org.jmailen.kotlinter")
    detekt { config.setFrom(rootProject.files("config/detekt/detekt.yml")) }
}
tasks.named("check") { dependsOn(subprojects.mapNotNull { it.tasks.findByName("detekt") }) }
```

- [ ] **Step 4: Prove the gate rejects violations (test the linter itself)**

Temporarily add a badly formatted file (e.g. a line of 130 chars + wildcard import) to `core/domain`, run `.\gradlew.bat check`. Expected: FAIL with ktlint errors. Delete the file, re-run: PASS.

- [ ] **Step 5: Pre-commit hook + installer**

`scripts/hooks/pre-commit` (bash; runs under Git Bash on Windows):

```bash
#!/usr/bin/env bash
set -e
STAGED=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.kt[s]?$' || true)
[ -z "$STAGED" ] && exit 0
./gradlew.bat lintKotlin detekt 2>/dev/null || ./gradlew lintKotlin detekt
```

`install-hooks.ps1`: `git config core.hooksPath scripts/hooks`. Run it. Verify hook fires on a deliberately mis-formatted `git add` attempt, then commit normally.

```bash
git add .editorconfig config/ scripts/ build.gradle.kts gradle/libs.versions.toml
git commit -m "feat: lint gate - editorconfig, ktlint, detekt, pre-commit hook"
```

---

### Task 4: core:domain foundation — AppResult, DomainError, entities, value IDs

**Files:**
- Create: `core/domain/src/commonMain/kotlin/com/workoutplanner/guider/core/domain/model/AppResult.kt`
- Create: `.../model/DomainError.kt`, `.../model/Ids.kt`, `.../model/Exercise.kt`, `.../model/Program.kt`, `.../model/Session.kt`, `.../model/Profile.kt`
- Test: `core/domain/src/commonTest/kotlin/com/workoutplanner/guider/core/domain/model/AppResultTest.kt`, `.../ValidationTest.kt`

**Interfaces:**
- Consumes: Task 2 scaffold, Task 3 lint gate
- Produces (used by Plans 2–5): `AppResult<T>` (`Success`/`Failure`), `DomainError` sealed hierarchy, value IDs, entity types below — **these exact names/signatures are frozen here**

```kotlin
package com.workoutplanner.guider.core.domain.model

public sealed interface AppResult<out T> {
    public data class Success<T>(public val value: T) : AppResult<T>
    public data class Failure(public val error: DomainError) : AppResult<Nothing>
}
public inline fun <T> AppResult<T>.onSuccess(block: (T) -> Unit): AppResult<T>
public inline fun <T> AppResult<T>.onFailure(block: (DomainError) -> Unit): AppResult<T>

public sealed class DomainError public constructor(public val message: String) {
    public data class Validation(override val message: String, public val field: String? = null) : DomainError(message)
    public data class NotFound(override val message: String) : DomainError(message)
    public data class SyncConflict(public val table: String, public val rowId: String, public val serverRevision: Int) : DomainError("conflict")
    public data class AiUnavailable(override val message: String) : DomainError(message)
    public data class Network(override val message: String) : DomainError(message)
    public data class Unauthorized(override val message: String) : DomainError(message)
}
```

IDs (all `@JvmInline value class X(val raw: String)`): `ExerciseId, ProgramId, RevisionId, ProgramDayId, ProgramExerciseId, PlannedSetId, SessionId, SessionExerciseId, PerformedSetId, ChatMessageId, ThreadId`.

Enums: `SetType { WARMUP, WORKING, TOP, BACKOFF }`, `LoadKind { EXTERNAL, BODYWEIGHT, ASSISTED }`, `UnitSystem { KG, LB }`, `SessionStatus { ACTIVE, COMPLETED, DISCARDED }`, `RevisionSource { MANUAL, LLM, TEMPLATE }`.

Entity data classes per architecture spec §4.1 (immutable, e.g.):

```kotlin
public data class Exercise(
    public val id: ExerciseId,
    public val ownerId: String?,
    public val name: String,
    public val canonicalName: String,
    public val aliases: List<String>,
    public val muscleGroup: String,
    public val equipment: String,
    public val loadKind: LoadKind,
)
public data class PlannedSet(public val id: PlannedSetId, public val setIndex: Int,
    public val setType: SetType, public val targetReps: Int?, public val targetRepsMax: Int?,
    public val targetWeightKg: Double?)
public data class ProgramRevision(public val id: RevisionId, public val programId: ProgramId,
    public val revisionNumber: Int, public val source: RevisionSource, public val notes: String?,
    public val days: List<ProgramDay>)
public data class ProgramDay(public val id: ProgramDayId, public val dayIndex: Int,
    public val name: String, public val exercises: List<ProgramExercise>)
public data class ProgramExercise(public val id: ProgramExerciseId, public val exerciseId: ExerciseId,
    public val orderIndex: Int, public val notes: String?, public val plannedSets: List<PlannedSet>)
public data class PerformedSet(public val id: PerformedSetId, public val setIndex: Int,
    public val setType: SetType, public val reps: Int, public val weightKg: Double?,
    public val rpe: Double?, public val doneAt: Instant)  // kotlinx-datetime Instant
```

- [ ] **Step 1: Write failing tests** — `AppResultTest`: Success/Failure construction, `onSuccess`/`onFailure` do not consume, exhaustive `when` over `DomainError` compiles (a `when` with all five branches used as expression).
- [ ] **Step 2: Run** `.\gradlew.bat :core:domain:allTests` — Expected: FAIL (types not defined).
- [ ] **Step 3: Implement** the files above with KDoc on every public declaration (S3) and `explicitApi()` satisfied.
- [ ] **Step 4: Run** `.\gradlew.bat :core:domain:allTests check` — Expected: PASS.
- [ ] **Step 5: Commit**

```bash
git add core/domain/
git commit -m "feat: domain foundation - AppResult, DomainError, ids, core entities"
```

---

### Task 5: Supabase schema migration — tables, RLS, triggers, RPCs, seed

**Files:**
- Create: `supabase/migrations/0001_init.sql`, `supabase/config.toml` (from `supabase init`), `supabase/seed.sql`

**Interfaces:**
- Consumes: entity shapes from Task 4 (column names must match)
- Produces: live schema on `workout-guider-dev`; table/RPC names consumed by `core:data` (Task 7) and Plans 2–3 (`apply_ops`, `pull_changes`)

- [ ] **Step 1: Init Supabase CLI + link dev project** — `supabase init`, `supabase link --project-ref <dev>` (creds from Task 1 Step 2; `supabase/config.toml` is committed minus any secrets).
- [ ] **Step 2: Write `0001_init.sql`** — all tables from architecture spec §4.1 with: `id uuid primary key default gen_random_uuid()`, `owner_id uuid references auth.users`, `created_at/updated_at timestamptz not null default now()`, `revision int not null default 1`, `deleted_at timestamptz null`; CHECK constraints from spec (reps > 0; unique `(session_exercise_id, set_index)`; `revision_number` unique per program; `exercises.load_kind` and enum-text columns constrained); indexes on `(owner_id, updated_at)` per table and `(program_id, revision_number)` unique where not deleted.
- [ ] **Step 3: Triggers** — `bump_row()` (on update: `updated_at = now()`, `revision = revision + 1`), `stamp_commit_ts()` (sets `commit_ts = pg_current_xact_timestamp()` on insert/update), plus `updated_at` on insert defaults.
- [ ] **Step 4: RLS** — enable on every user table; policies: `owner_all` (all ops, `owner_id = auth.uid()`), catalog/template `exercises`/`programs` where `owner_id is null` → SELECT for authenticated; deny all for `anon`. Recreate grants per supabase-kt docs (`grant select, insert, update, delete … to authenticated`).
- [ ] **Step 5: RPCs** — `apply_ops(ops jsonb) returns jsonb` (security definer; per-op: idempotency via `applied_ops`, base_revision check → insert-or-conditional-update, tombstone, `composite` subtree in one transaction, returns per-op status) and `pull_changes(cursors jsonb) returns jsonb` (rows + tombstones where `commit_ts > cursor`, ordered by commit). Full bodies per architecture spec §6.1/§6.3 — write them as security-definer functions with `set search_path = public`.
- [ ] **Step 6: Seed** — `supabase/seed.sql` inserts builtin exercises (15 core barbell/dumbbell/bodyweight moves with canonical names + aliases) and 1 beginner 3-day template program (owner null, is_template true) with a revision 1 tree.
- [ ] **Step 7: Apply + verify** — `supabase db push`; then run a smoke SQL: select seeded exercises as anon key via PostgREST (`curl`), expect 401 for user tables and 200 for catalog. Expected: catalog readable, `sessions` forbidden.
- [ ] **Step 8: Commit**

```bash
git add supabase/
git commit -m "feat: supabase schema - tables, rls, triggers, sync rpcs, seed"
```

---

### Task 6: RLS negative suite (integration test, CI-gated later)

**Files:**
- Create: `core/data/src/jvmTest/kotlin/com/workoutplanner/guider/core/data/rls/RlsNegativeSuite.kt`
- Modify: `core/data/build.gradle.kts` (jvmTest source set, reads `local.properties`)

**Interfaces:**
- Consumes: Task 5 schema; Task 2 module
- Produces: `RlsNegativeSuite` — the reusable gate every later schema change must keep green

- [ ] **Step 1: Write the suite** — JVM-only integration test (runs when `SUPABASE_URL`/`SUPABASE_ANON_KEY` env present, else skipped via `Assume`): creates users A and B via `signUpWith`, then asserts user A: cannot SELECT B's `sessions`, INSERT with `owner_id = B.id` fails (or is coerced to own id), UPDATE/DELETE of B's rows affects 0 rows; anon client cannot read `programs`/`sessions`; both can read builtin `exercises`.
- [ ] **Step 2: Run** `.\gradlew.bat :core:data:jvmTest` with env set — Expected: all assertions PASS (they're negative expectations against the server).
- [ ] **Step 3: Commit**

```bash
git add core/data/
git commit -m "test: rls negative suite - cross-user isolation verified"
```

---

### Task 6b: core:data — Supabase client wrapper + AuthRepository

**Files:**
- Create: `core/data/src/commonMain/kotlin/com/workoutplanner/guider/core/data/SupabaseClientFactory.kt`, `.../auth/AuthRepository.kt`, `.../auth/SupabaseAuthRepository.kt`, `.../di/DataModule.kt`
- Test: `core/data/src/commonTest/kotlin/com/workoutplanner/guider/core/data/auth/AuthRepositoryTest.kt`

**Interfaces:**
- Consumes: Task 4 `AppResult`/`DomainError`; Task 2 deps
- Produces: `AuthRepository` interface — used by Task 7 UI and Plan 2 session bootstrap

```kotlin
public interface AuthRepository {
    public val sessionState: StateFlow<AuthState>
    public suspend fun signUp(email: String, password: String): AppResult<Unit>
    public suspend fun signIn(email: String, password: String): AppResult<Unit>
    public suspend fun signOut(): AppResult<Unit>
}
public sealed interface AuthState {
    public data object Unauthenticated : AuthState
    public data object Loading : AuthState
    public data class Authenticated(public val userId: String) : AuthState
}
```

- [ ] **Step 1: Failing test** — `FakeAuthBackend` implements a seam `AuthApi` (thin supabase-kt port); test signUp success → `Authenticated`, bad password → `Failure(DomainError.Unauthorized)`, network throw → `Failure(DomainError.Network)`; assert sessionState transitions.
- [ ] **Step 2: Run** — Expected FAIL.
- [ ] **Step 3: Implement** — `SupabaseAuthRepository` maps supabase exceptions (`AuthRestException`/`HttpRequestException`) to `DomainError.Unauthorized`/`Network`; `SupabaseClientFactory` builds client from platform config (URL/key injected); `DataModule.kt` Koin wiring. All exceptions map to `AppResult` (style guide §6).
- [ ] **Step 4: Run** `.\gradlew.bat :core:data:allTests check` — Expected: PASS.
- [ ] **Step 5: Commit** — `feat: auth repository with supabase client wrapper`

---

### Task 7: composeApp — navigation shell + Auth screen

**Files:**
- Create: `composeApp/src/commonMain/kotlin/com/workoutplanner/guider/ui/App.kt` (NavHost: `auth`, `hub` placeholder), `.../ui/screens/auth/AuthScreen.kt`, `.../ui/screens/auth/AuthViewModel.kt`, `.../ui/theme/Theme.kt` (dark-first tokens from UI spec §2), `.../di/AppModule.kt`
- Test: `composeApp/src/commonTest/kotlin/com/workoutplanner/guider/ui/screens/auth/AuthViewModelTest.kt`

**Interfaces:**
- Consumes: Task 6b `AuthRepository`/`AuthState`; UI spec §3.1 + tokens
- Produces: `AppNavGraph` with routes `auth` and `hub` (hub placeholder) — Plan 2 adds session/programs routes

- [ ] **Step 1: Failing ViewModel test** — given invalid email → `showValidationError`; given fake repo failure `Unauthorized` → inline error state; given success → `navigatesToHub` event. (Fake `AuthRepository`.)
- [ ] **Step 2: Run** `.\gradlew.bat :composeApp:testDebugUnitTest` (or `:composeApp:allTests` with jvm target if added for UI tests) — Expected: FAIL.
- [ ] **Step 3: Implement** — `AuthViewModel` exposes `StateFlow<AuthUiState>` (email/password/submitting/error), calls `AuthRepository` mapping to `AppResult`; `AuthScreen` per UI spec §3.1 (email + password + submit + inline `InlineError` + sign-up toggle), `Theme.kt` with token table, `NavHost` in `App.kt` switching on `AuthState`.
- [ ] **Step 4: Run** full `.\gradlew.bat check` — Expected: PASS, lint gate green.
- [ ] **Step 5: Manual Android verification** — `.\gradlew.bat :composeApp:installDebug` on an emulator/device; sign up a fresh account; confirm hub placeholder appears and app restart restores session.
- [ ] **Step 6: Commit**

```bash
git add composeApp/
git commit -m "feat: auth screen with dark-first theme and navigation shell"
```

---

### Task 8: Plan-1 verification gate

- [ ] **Step 1:** `.\gradlew.bat check` from root — green (ktlint + detekt + all tests).
- [ ] **Step 2:** `.\gradlew.bat :core:data:jvmTest` with Supabase env — RLS suite green.
- [ ] **Step 3:** Android smoke checklist recorded in `docs/superpowers/spikes/2026-09-16-koog-supabase-spike.md` appendix: fresh signup → hub → sign out → sign in; app-kill relaunch keeps session; airplane-mode login shows `Network` error inline.
- [ ] **Step 4:** Version pins in `libs.versions.toml` match the spike note; discrepancies resolved.
- [ ] **Step 5:** Commit any final touches — `chore: plan 1 verification gate passed`.

**Plan 1 done when:** spike note committed with go verdict; workspace builds; lint gate blocks violations; schema + RLS live and verified; auth works on Android. Plan 2 may now be written.
