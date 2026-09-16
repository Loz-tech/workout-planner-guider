# Workout Planner & Guider — UI/UX Spec

**Date:** 2026-09-16
**Status:** Draft for review
**Parent spec:** `2026-09-16-workout-planner-guider-design.md` (architecture & data)
**Source mockups:** `.superpowers/brainstorm/` (navigation-structure, session-screen, visual-style)

---

## 1. Locked decisions

| # | Decision | Choice |
|---|---|---|
| U1 | Navigation | **Hub + back navigation** — no bottom tabs; one hub screen, every section reached from it, back to return |
| U2 | Session layout | **Exercise accordion + per-exercise "+"** with set chips; quick text/voice bar pinned bottom; rest timer strip |
| U3 | Theme | **Dark-first "gym mode"** — near-black surfaces, lime accent, glowing rest timer; "follow system" toggle in Settings |
| U4 | Hub content | Start-session card, recent-activity strip, coach tip card, section link cards (all four) |

## 2. Design tokens (dark-first theme)

| Token | Value |
|---|---|
| `surface/base` | `#121214` |
| `surface/card` | `#1D1D21` |
| `surface/divider` | `#2C2C30` |
| `text/primary` | `#E8E8E8` |
| `text/secondary` | `#9A9AA0` |
| `accent` | `#C8F656` (lime — primary actions, LOG/confirm) |
| `accent/success-onDark` | `#4ADE80` (completed set chips, rest timer) |
| `error` | `#F87171` |
| `warning` | `#FBBF24` |
| `light variant` | Inverse surfaces (`#FFFFFF`/`#F6F8FA`), same accent hues, auto-derived when "follow system" is on |

Rules: minimum touch target 48dp; primary action always in the bottom third (one-hand reach); contrast AA minimum on dark surfaces; numeric fields open numeric keyboard.

## 3. Screen inventory & key layouts

### 3.1 Auth
Sign in / sign up (email + password), error inline under fields, no deeplink OAuth at launch.

### 3.2 Hub (home)
Top-to-bottom:
1. **Header** — greeting/date, pending-sync badge (subtle count), settings gear.
2. **Start-session card** *(primary, largest)* — next planned day from active program: "Pull A — 4 exercises · ~45 min" + **Start** button; secondary "Start ad-hoc" text link (no program attached).
3. **Recent-activity strip** — last workout summary (exercises count, total volume, duration) + weekly volume mini-figure.
4. **Coach tip card** — one-line LLM analysis headline ("Bench +2.5kg over 4 weeks"), tap → Coach chat; hidden when offline/no key.
5. **Section link cards** — Programs, History, Progress, Coach — each a full-width tappable card with count/summary subtitle.
Empty states: no program → Start-session card shows "Create or copy a program" leading to Programs.

### 3.3 Session (the core screen)
Full-screen over the hub (back = confirm dialog if sets exist).
- **Header:** back chevron, "Pull A" + elapsed timer, "✓ End".
- **Body:** collapsible exercise cards (accordion). Each card: name + planned sets badge; expanded shows **set chips** — `8×60 ✓` per logged set (tap chip = edit), `+ set` chip prefilled with last values (one tap logs, triggers rest timer). Set-type prefix shown for warmup/backoff (`1·warm 8×40`).
- **Quick entry bar** (pinned above rest strip): mic icon + text field placeholder "bench 8 x 60" → parses to a **ConfirmChip** ("Bench · 8 × 60 kg ✓ / ✗") — one tap applies.
- **Rest timer strip** (bottom, accent glow while running): countdown, skip/+30s, auto-starts after each logged set.
- **Anomaly confirm dialog** on suspicious entries (per architecture spec §7.5).

### 3.4 Programs
List (personal programs + template section) → detail viewer (days → exercises → planned sets) → editor (manual CRUD, saved as new revision) → **Create with AI** (prompt screen → proposal → **Revision diff review**: old vs new side-by-side, Approve/Discard).

### 3.5 History
Session list (date, day label, volume, duration) → session detail: per-exercise planned-vs-performed comparison, notes.

### 3.6 Progress
Per-exercise PR cards (est. 1RM + best set), weekly volume bar chart, simple trend lines; range selector (4w/12w/1y).

### 3.7 Coach
Chat thread; assistant messages can embed **action chips** — log-set confirm, program-diff preview — same confirmation semantics as everywhere (AI proposes, human confirms). History loaded from `chat_messages`.

### 3.8 Settings
Provider + model + API key (masked, Keychain/Keystore-stored), units kg/lb, display name, theme (dark / follow system).

## 4. Component inventory (`composeApp/ui/components`)

`SetChip`, `ExerciseCard` (collapsible), `RestTimerStrip`, `QuickEntryBar`, `ConfirmChip`, `RevisionDiffView`, `PendingSyncBadge`, `SectionCard`, `EmptyState`, `InlineError`, `OfflineBanner`.

## 5. States

Every data screen defines: loading (shimmer), empty (with next-action CTA), error (inline + retry), offline (banner where relevant). The Session screen never blocks logging on network state.

## 6. Accessibility & platform notes

- Voice input behind a runtime capability check; text field always available.
- Back navigation: system back works everywhere; destructive actions (End session with unlogged sets) confirm.
- Numeric steppers/inputs for reps/weight; no free-text-only entry.
- Both platforms tested for dark-surface contrast and one-hand reach at launch.
