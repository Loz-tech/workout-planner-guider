# Workout Planner & Guider — Design Spec

**Date:** 2026-09-16
**Status:** Draft for review (implementation plan follows after approval)
**Decisions locked:** D1 = idempotent outbox sync (1a); D2 = AI proposes / human confirms, no AI deletes (2a); D3 = feasibility spike gates version pins (3a)

---

## 1. Purpose

A cross-platform mobile app (iOS + Android) for gym-goers who train on a personal program and want to log sessions fast mid-workout. Core value: capture reps/weights in seconds during rest periods, keep full training history, and let an LLM assistant read that history to coach, analyze, and draft program changes — with the human approving anything that modifies their program.

**Users:** several accounts on a shared backend (personal training partners, small group). Each user's data is strictly their own.

## 2. Scope

**In scope at launch:**
- Email/password accounts (Supabase Auth), several users, per-user data isolation via RLS
- Personal programs with full revision history; seeded beginner templates to copy from
- Offline-first session logging (sets, reps, weights, optional RPE), rest timer
- Background synchronization of local data to Supabase (outbox design, §6)
- AI (Koog, multi-provider BYOK): quick text/voice set entry, coach chat with history analysis, program creation/editing as confirmed proposals, local anomaly guard on logged sets
- Progress tracking: PRs, volume, simple charts; kg/lb toggle

**Out of scope at launch:**
- Goals/targets tracking (deferred — user chose strength & volume basics only)
- Social features, sharing programs between users
- Wearables, health-platform integration, video form analysis
- Realtime multi-device push (pull-based sync only; realtime is a later addition)
- AI ability to delete or overwrite history (permanently excluded per D2)

## 3. Platform & stack

| Concern | Choice | Notes |
|---|---|---|
| Language/UI | Kotlin Multiplatform + Compose Multiplatform | One shared UI, Android + iOS targets |
| LLM framework | Koog `1.2.0` (stable `koog-agents` umbrella) | Class-based tools only (annotation-based tools are JVM-only per Koog docs) |
| LLM providers | OpenAI, Anthropic (stable Koog clients); Google Gemini (beta client — allowed, flagged); OpenRouter optional | BYOK |
| Backend | Supabase: Auth (email/password), Postgres + RLS, RPC | Client: `supabase-kt` 3.x (community-maintained — wrapped behind an interface) |
| Ktor | 3.x required by supabase-kt 3.x | Darwin engine on iOS, OkHttp/Android engine on Android |
| Local DB | SQLDelight `2.3.2` (android-driver / native-driver) | Offline-first source of truth for UI reads |
| DI | Koin `4.2.x` | expect/actual platform modules |
| Navigation | Compose Multiplatform Navigation (stable release line, not the docs' alpha example) | Pinned by spike |
| Units | kg canonical storage; lb is a UI/display conversion | One rounding rule: 2.5 kg / 5 lb steps |
| Voice | On-device STT (iOS Speech framework, Android SpeechRecognizer) | Optional accelerator; capability-checked; text fallback always present |
| Targets | Android minSdk 26 (supabase-kt floor), iOS 14+ | iOS validated only on Mac/CI — not in this Windows workspace |

**Version pins are confirmed by the Phase 0 spike before implementation starts (§13).** Doc-derived versions above are starting points, not commitments.

### 3.1 Modules

```
composeApp/          Compose UI, screens, Koin wiring (androidMain, iosMain, commonMain)
core/domain/         Pure Kotlin: entities, Command layer, validation, use cases. No platform deps.
core/data/           SQLDelight schema, sync engine, Supabase client wrapper, repositories
core/ai/             Koog agents, class-based tools (thin adapters over core:domain commands/queries), prompts
```

Dependency rule: `composeApp → core:{domain,data,ai}`; `core:data → core:domain`; `core:ai → core:domain` (never `core:ai → core:data` — AI touches storage only through the command/query layer).

## 4. Data model

All user-owned tables carry: `id uuid`, `owner_id uuid`, `created_at`, `updated_at`, `revision int`, `deleted_at timestamptz null` (soft delete / tombstone). `revision` is bumped by a DB trigger on every update; it is the optimistic-concurrency token for sync. All timestamps UTC. Weights are stored in **kg** (numeric(6,2)); lb is display-only.

### 4.1 Server tables (Supabase Postgres)

```sql
profiles (
  id uuid PK,                       -- = auth.users.id
  display_name text NOT NULL,
  unit_system text NOT NULL DEFAULT 'kg',   -- 'kg' | 'lb'
  created_at, updated_at, revision, deleted_at
)

exercises (                          -- exercise catalog
  id uuid PK,
  owner_id uuid NULL,                -- NULL = builtin catalog row (read-only to clients)
  name text NOT NULL,                -- display name
  canonical_name text NOT NULL,      -- normalized identity ("barbell_bench_press")
  aliases text[] NOT NULL DEFAULT '{}',
  muscle_group text NOT NULL,
  equipment text NOT NULL,
  load_kind text NOT NULL,           -- 'external' | 'bodyweight' | 'assisted'
  created_at, updated_at, revision, deleted_at
)
-- RLS: builtin rows (owner_id IS NULL): authenticated SELECT only.
-- User rows: full CRUD, owner-only. Clients never write builtin rows.

programs (
  id uuid PK, owner_id uuid NOT NULL,
  name text NOT NULL, description text,
  active_revision_id uuid NULL,      -- points into program_revisions
  created_at, updated_at, revision, deleted_at
)

program_revisions (                  -- immutable snapshot; a "version" of a program
  id uuid PK, program_id uuid NOT NULL,
  revision_number int NOT NULL,      -- 1..n per program
  source text NOT NULL,              -- 'manual' | 'llm' | 'template'
  notes text,
  created_at, updated_at, revision, deleted_at
)

program_days (
  id uuid PK, revision_id uuid NOT NULL,
  day_index int NOT NULL,            -- ordering within revision
  name text NOT NULL,                -- e.g. "Pull A"
  created_at, updated_at, revision, deleted_at
)

program_exercises (
  id uuid PK, day_id uuid NOT NULL,
  exercise_id uuid NOT NULL,         -- FK exercises
  order_index int NOT NULL,
  notes text,
  created_at, updated_at, revision, deleted_at
)

planned_sets (                       -- per-set prescription (NOT one row per exercise)
  id uuid PK, program_exercise_id uuid NOT NULL,
  set_index int NOT NULL,
  set_type text NOT NULL,            -- 'warmup' | 'working' | 'top' | 'backoff'
  target_reps int NULL,
  target_reps_max int NULL,          -- for ranges (e.g. 8–12)
  target_weight_kg numeric(6,2) NULL,
  created_at, updated_at, revision, deleted_at
)

sessions (
  id uuid PK, owner_id uuid NOT NULL,
  started_at timestamptz NOT NULL,
  completed_at timestamptz NULL,
  status text NOT NULL,              -- 'active' | 'completed' | 'discarded'
  source_revision_id uuid NULL,      -- program revision trained (NULL = ad-hoc session)
  source_day_id uuid NULL,
  day_label text NOT NULL,           -- denormalized label, stable even if program later edited
  notes text,
  created_at, updated_at, revision, deleted_at
)

session_exercises (
  id uuid PK, session_id uuid NOT NULL,
  exercise_id uuid NOT NULL,
  order_index int NOT NULL,
  planned_snapshot jsonb NOT NULL,   -- copied planned_sets + program_exercise ref at session start
  notes text,
  created_at, updated_at, revision, deleted_at
)
-- History immutability rule: performed data references the snapshot; later program
-- edits never alter what a past session planned.

performed_sets (
  id uuid PK, session_exercise_id uuid NOT NULL,
  set_index int NOT NULL,
  set_type text NOT NULL,            -- copied set_type
  reps int NOT NULL CHECK (reps > 0),
  weight_kg numeric(6,2) NULL,       -- NULL for bodyweight moves
  rpe numeric(3,1) NULL,
  done_at timestamptz NOT NULL,
  created_at, updated_at, revision, deleted_at
)

chat_messages (                      -- coach chat continuity across devices
  id uuid PK, owner_id uuid NOT NULL,
  thread_id uuid NOT NULL,           -- per-user single thread at launch
  role text NOT NULL,                -- 'user' | 'assistant' | 'tool_summary'
  content text NOT NULL,
  created_at, updated_at, revision, deleted_at
)
```

Out-of-history integrity rules (DB-enforced where practical): `revision_number` unique per program; `set_index` unique per (session_exercise, set_index); FK cascades are soft (tombstones), deletes only via `deleted_at`.

### 4.2 Local tables (SQLDelight)

Mirror of every server table (same columns, minus RLS) **plus**:

```sql
sync_outbox (
  op_id uuid PK,                     -- stable operation id → idempotency key
  table_name text NOT NULL,
  row_id uuid NOT NULL,
  op_type text NOT NULL,             -- 'upsert' | 'tombstone' | 'composite' (whole-subtree JSON)
  base_revision int NULL,            -- local revision at enqueue time
  payload jsonb NOT NULL,
  state text NOT NULL,               -- 'pending' | 'in_flight' | 'failed'
  attempts int NOT NULL DEFAULT 0,
  last_error text NULL,
  created_at, updated_at
)

sync_state (
  table_name text PK,
  last_pulled_at text NOT NULL,      -- per-table pull cursor
  last_push_ok_at text NULL
)

device_registry (device_id uuid PK, created_at)   -- local-only device identity
```

The local DB is the single read source for the UI. The server is the reconciliation authority; the client never assumes its local write "won" until the server confirms it.

## 5. Command layer (the only write path)

Everything that mutates data — manual UI **and** AI — goes through typed commands in `core:domain`:

```kotlin
sealed interface Command {
    data class StartSession(val sourceRevisionId: Uuid?, val sourceDayId: Uuid?, val dayLabel: String) : Command
    data class LogSet(val sessionExerciseId: Uuid, val setType: SetType, val reps: Int,
                      val weightKg: Double?, val rpe: Double?) : Command
    data class EditLastSet(val sessionExerciseId: Uuid, val reps: Int, val weightKg: Double?) : Command
    data object UndoLastSet : Command                       // un-does most recent set in session
    data object EndSession : Command
    data class CreateProgramFromTemplate(val templateId: Uuid, val name: String) : Command
    data class ApplyProgramRevision(val programId: Uuid, val proposal: ProgramProposal) : Command
    data class UpdateSettings(val unitSystem: UnitSystem, val displayName: String) : Command
}
```

- Each command validates against domain rules (positive reps, set-type consistency, exercise exists, etc.) and applies in **one local SQLite transaction** that also enqueues the outbox row(s).
- Program revisions are created as **whole-subtree proposals**: a `ProgramProposal` (new days → exercises → planned_sets tree) is applied atomically as a new `program_revisions` row + children, locally and remotely (§6.4). Past revisions are never mutated.
- `ApplyProgramRevision` is the **only** path that changes what future sessions plan. Sessions keep their snapshot regardless.

## 6. Sync design (D1 = 1a: outbox + idempotent ops + server revisions)

### 6.1 Push

1. Command layer writes: row(s) + outbox entry in one transaction, `state='pending'`.
2. Sync worker (foreground/resume/connectivity-restored; **never** relies on OS background execution to persist a set) drains the outbox oldest-first in ordered batches.
3. Batches are pushed via one Supabase RPC: `apply_ops(ops jsonb)` — a security-definer function that, per op, inside one DB transaction:
   - no-ops if `op_id` already logged in `applied_ops` (idempotent retries),
   - for `upsert`: inserts, or updates **only if** `base_revision` equals the server's current `revision` (or row doesn't exist); mismatch → returns `conflict` with server revision,
   - for `tombstone`: sets `deleted_at`, bumps revision,
   - for `composite`: applies the whole program-revision subtree atomically,
   - logs each applied `op_id` in `applied_ops(op_id PK, applied_at)`.
4. Per-op result (`applied` / `conflict:server_revision`) returns to the client.

### 6.2 Conflicts

- On `conflict`: the local pending write is moved to a `conflict_versions` side row (both versions retained); the server row is pulled; UI surfaces a small "edited on another device" prompt. **No silent overwrite in either direction.**
- Append-only rows (`performed_sets`, `sessions`, `chat_messages`) use client-generated UUIDs, so true conflicts are practically limited to `programs/revisions/planned_sets/profiles` — exactly where the revision check applies.

### 6.3 Pull

- Per-table cursor (`last_pulled_at`): pull via RPC `pull_changes(cursors jsonb)` returning rows where server `updated_at > cursor` **ordered by commit order**, plus tombstones. Cursor advances to the last committed change's boundary transaction; server stores `commit_ts` per row via trigger to make the cursor durable and gap-free.
- Tombstoned rows are applied locally as deletes; server garbage-collects tombstones after 90 days.
- On reconnect after failure: cursors are the recovery state; a full re-pull is the fallback if cursors look inconsistent.

### 6.4 Multi-row atomicity

Program edits enqueue **one** `composite` op containing the whole revision subtree — never N independent row ops — so a program can never sync as a half-applied state.

### 6.5 Sync triggers

Foreground/resume, connectivity regained, and after each command completes. Pending count shown subtly in the UI ("3 sets pending sync"). Sessions are saved locally at the moment of logging; sync is eventual.

## 7. AI layer (Koog, D2 = 2a)

### 7.1 Principles

- AI never touches SQLDelight or Supabase directly. All effects flow through the same `Command` layer as manual UI (§5).
- AI proposals require human confirmation. No delete/overwrite tools exist at launch (permanently excluded per D2).
- LLM output is untrusted input: typed tool parameters, enum/length/range validation, no free-form SQL or table names from the model.

### 7.2 Agents (core:ai)

| Agent | Purpose | Shape |
|---|---|---|
| **QuickEntryAgent** | Parse "bench 3x8 60kg" / dictation into a `LogSet`-shaped proposal | One-shot, low temperature, structured output, **no tool loop**; executes only `ProposeLogSet` |
| **CoachAgent** | Chat: read history, analyze, propose programs | Tool loop over read + propose tools; history compression on; chat memory persisted locally (synced via `chat_messages`) |

### 7.3 Tools (class-based only)

Read tools (bounded, current user only):
- `get_active_program()` → active revision tree
- `get_recent_sessions(n ≤ 10)`
- `get_progress_summary(window_days ≤ 90)` → volume/PR aggregates
- `get_prs()` → per-exercise bests
- `list_exercises(query)` → catalog lookup for identity resolution

Propose tools:
- `propose_log_set(exercise, setType, reps, weightKg, rpe?)` → returns parsed entry → **UI confirm chip** (one tap applies; wrong parse → user edits inline)
- `propose_program_revision(programId, proposal)` → stored as pending proposal → **UI renders revision diff** → user approves → `ApplyProgramRevision` command fires

Forbidden at launch: any tool that deletes rows, mutates history, or writes without confirmation. Ambiguous voice input → agent asks a clarifying question rather than guessing.

### 7.4 Provider & key handling

- Settings: user picks provider (OpenAI / Anthropic / Gemini / OpenRouter) + model (sensible defaults per provider) + pastes their API key.
- Keys live in iOS Keychain / Android Keystore-backed storage only; held in memory solely for the active agent call; never logged, never in SQLDelight, never in `multiplatform-settings` plain storage.
- A provider call fails → clear inline error + retry button; Koog's built-in retries handle transient provider errors.

### 7.5 Anomaly guard (no LLM on the hot path)

After each `LogSet`, a local heuristic compares against the exercise's recent history: weight jump >15% vs. prior set, or >2.5× existing PR → confirmation dialog before the set is persisted. Fast, offline, deterministic; the CoachAgent can do richer analysis later in chat.

## 8. UX screens (composeApp)

1. **Auth** — sign up / sign in (email/password), deeplink-safe.
2. **Today** — active program's next day, "Start session", pending-sync indicator.
3. **Session** *(the core screen)* — exercise list from snapshot; big "+" per exercise with last-set values prefilled; quick text/voice bar (parse → confirm chip); rest timer auto-starts after logging; set edit/undo; end-session summary.
4. **Programs** — list; detail viewer; editor (manual); "Create with AI" (prompt → proposal → diff review); "Copy from template" (beginner templates seeded server-side, copied into personal programs before first edit).
5. **History** — session list + detail (planned vs performed per set).
6. **Progress** — per-exercise PRs, weekly volume, simple trend charts.
7. **Coach** — chat thread; suggestions surface as actionable chips (log-set confirm, program diff).
8. **Settings** — provider/model/API key, units (kg/lb), display name.

Offline behavior: everything except AI and auth works fully offline; AI surfaces a clear offline banner; nothing queues for later LLM processing.

## 9. Security

- **BYOK:** Keychain/Keystore-backed secure storage only (§7.4).
- **Supabase anon key** is public-by-design; safety rests on RLS: every user-owned table gets owner-only policies (`owner_id = auth.uid()`), builtin `exercises` read-only.
- **RLS negative tests are CI-gated** (SQL test suite): user A cannot SELECT/INSERT/UPDATE/DELETE user B's rows on any table; unauthenticated clients can read nothing user-owned.
- PostgREST/`supabase-kt` grants follow least privilege (`grant select, insert, update, delete ... to authenticated` on user tables only; `select` on catalog).
- Agent tracing/telemetry off by default; no prompt/response logging containing keys.

## 10. Error handling

| Failure | Behavior |
|---|---|
| Log set with no connectivity | Saved locally instantly; outbox entry enqueued; UI shows pending badge |
| Sync RPC failure / timeout | Batch retried with backoff (Koog-independent); attempts capped, then `failed` state + manual "Retry sync" |
| Sync conflict | Both versions preserved; UI prompt (§6.2) |
| LLM provider error / bad key | Inline error + link to Settings; quick entry falls back to manual "+"; session logging never blocked |
| Parse ambiguity (voice/text) | Agent asks clarifying question; never guesses |
| Anomaly threshold hit | Confirm dialog (§7.5) |
| App killed mid-session | Session state is local; restored on relaunch with active session intact |

## 11. Testing strategy

- **commonTest (multiplatform):** command validation; sync engine against a **fake transport** first (idempotent retry of same op_id, stale-revision conflict, composite atomicity, tombstone pull, cursor recovery); AI tool validation & tool-selection with Koog mock executor / canned responses; PR/volume aggregation math.
- **Android:** instrumented smoke (auth → log set → sync → reload), SQLDelight driver behavior.
- **iOS:** compile + unit tests on Mac/CI; manual smoke checklist (numeric input, STT permission, backgrounding/restore).
- **Integration:** Supabase RLS negative suite; two-device convergence script (device A logs offline, device B edits program, both reconnect → no data loss, conflict surfaced once).
- **Definition of done (launch):** offline set logging works end-to-end; two devices converge without silent loss; AI parse→confirm→apply flow works; no AI path can delete or bypass the command layer; RLS negative suite green.

## 12. Milestones

- **M0** — Phase 0 spike (§13): go/no-go + version pins.
- **M1** — Scaffold (4 modules, Koin, navigation), Supabase schema + RLS + RPCs, auth screen. *Testable: sign in/out.*
- **M2** — Offline logging: domain commands, SQLDelight schema, session UI with prefilled sets, rest timer, anomaly dialog. *Testable: full workout offline.*
- **M3** — Sync engine: outbox, apply_ops, pull_changes, conflicts, two-device test.
- **M4** — AI quick entry: QuickEntryAgent, confirm chip, provider settings, offline banner.
- **M5** — Programs: revisions, editor, templates, `propose_program_revision` diff flow.
- **M6** — Coach chat + progress charts + PRs.
- **M7** — Polish: iOS CI on Mac, empty/error states, settings completion.

## 13. Phase 0 spike (D3 = 3a) — gate before implementation

Throwaway project (never committed as product code), ~1 day, validates on Android/JVM in this Windows workspace:
1. Koog `koog-agents` stable umbrella: one **class-based** tool executed via agent with a mocked executor → proves tool plumbing.
2. `supabase-kt` 3.x + Ktor 3.x: signup/signin + one RLS-protected table round-trip.
3. Records: exact working versions for Kotlin, CMP, navigation, supabase-kt, Ktor, Koog, Koin → written into the implementation plan's Global Constraints.
4. iOS targets compile-flagged only; runtime iOS validation deferred to Mac/CI (M7 gate).

## 14. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Sync correctness (history loss) | High | Outbox + idempotency + revisions (§6); sync tests before UI; two-device gate |
| AI writes bypassing validation | High | Single command layer; propose-only tools; no delete tools (§7) |
| Koog mobile parity gaps | Medium | Spike (§13); class-based tools only; stable modules only; thin `core:ai` boundary |
| `supabase-kt` community maintenance | Medium | Pin versions; wrap behind `core:data` interface; replaceable |
| iOS untestable on this Windows machine | Medium | Android-first dev; Mac/CI checkpoint at M1/M7 |
| Gemini via beta Koog client | Low-Med | Ship OpenAI/Anthropic defaults; Gemini optional |
| Voice recognition quality/device variance | Low | Text fallback always; capability checks |
