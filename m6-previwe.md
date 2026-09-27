Discovery complete. Worktree verified clean. Producing the freeze document.

```text
══════════════════════════════════════════════════════════
HARF HESAB
M6 EVOLUTION — ARCHITECTURE & UX FREEZE
══════════════════════════════════════════════════════════

RUN DATE:     2026-09-27
PROJECT:      harf_hesab  (Flutter 3.47.5 / Dart 3.13.4, Android-only)
HEAD:         44fb44f  docs(m5): add content expansion report
BRANCH:       master
WORKTREE:     CLEAN  (git status --short → empty; diff-index → empty)

Mode: READ-ONLY. Zero files created, modified, or deleted in the repository.
Only artifacts written: none. (Previous phase left only gitignored
android/.gradle/ cache timestamps, from the pre-check build.)


──────────────────────────────────────────────────────────
1. EXECUTIVE SUMMARY
──────────────────────────────────────────────────────────

OBSERVED — the application is a 34-file, ~3,540-line Flutter app built on a
deliberately narrow foundation:

• Feature-first folder layout, no `domain/`/`data/` layering ceremony.
• Riverpod 3.4.3 with four provider kinds only (Provider, FutureProvider,
  AsyncNotifier, Notifier). No codegen, no freezed, no repository interfaces.
• One SQLite database (`harf_hesab.db`, **version 1**) with exactly three
  tables: `answer_records`, `day_map`, `settings`.
• Navigation is a 3-tab `IndexedStack` with **zero named routes and no router
  package**.
• 400 questions in 8 categories, bundled as one JSON asset, validated
  structurally at runtime and strictly at build/CI time.
• Anti-repetition is **already solved elegantly**: `day_map` doubles as a
  *presentation ledger*, so a question is marked "seen" by the act of
  persisting the day that shows it. No seen-table, no migration.
• 213 tests across 21 files. **No `integration_test/` directory exists.**
• One notification architecture exists (daily reminder, id 1701) behind an
  abstract `ReminderService`.

INFERRED — the architecture is unusually disciplined: every non-trivial rule
is a pure, Flutter-free function (`selection.dart`, `result_engine.dart`,
`streak.dart`, `calendar.dart`, `notifications.dart`, `content_bundle.dart`)
with a thin notifier/widget above it, and platform I/O is always behind an
overridable provider. That is exactly the shape Phase 1 and Phase 2 need.

PROPOSED — M6 Evolution is achievable as a **pure extension**:

• **Phase 1 needs almost no new domain model.** `QuestionCategory` already
  carries `wireName` + Persian `faLabel` for all 8 categories, `Option`/
  `Question` already carry `category`, and remaining counts are *derivable*
  from the existing `seenQuestionIds()`. This is EXTEND, not REBUILD.
• **The single highest risk is narrow and specific**: `day_map` has
  `day_key` as PRIMARY KEY, so exactly one question is frozen per day
  globally. Introducing same-day category switching would require
  deleting that row, which silently re-consumes questions and can re-show
  one already seen. ADR-003 resolves this in favour of an *immutable daily
  question*.
• **Phase 2 requires one schema migration (v1→v2)** adding `goals` and
  `tasks`. Additive only; no existing table is touched, so no user data
  can be lost.
• **The UI does not use the bundled Vazirmatn font** — `AppTypography` sets
  no `fontFamily`, so the app renders in the platform's default Persian face;
  Vazirmatn is loaded *only* for share-card PNG export. Wiring the
  already-present font into the UI is a zero-dependency modernization.
• **There is no Jalali calendar anywhere.** `fa_date.dart` renders Persian
  *Gregorian* month names. Due dates will need an explicit decision.

STATUS: architecture is freeze-ready. Three product decisions (§22, D1–D3)
require sign-off before Phase 2 coding, but none of them is an architectural
unknown.


──────────────────────────────────────────────────────────
2. CURRENT ARCHITECTURE
──────────────────────────────────────────────────────────

OBSERVED structure (line counts from `lib/`):

```

lib/ ├── main.dart 71 bootstrap: DB → bundle → reminders → runApp ├── app/ │ ├── app.dart 93 MaterialApp(fa) + \_TabShell (3-tab IndexedStack) │ ├── providers.dart 21 appDatabaseProvider, contentBundleProvider, │ │ todayDayKeyProvider │ └── theme/ │ ├── tokens.dart 51 AppColors / AppSpacing / AppRadius (ONLY place for raw values) │ ├── typography.dart 43 7 named TextStyles — NO fontFamily (system font) │ └── app_theme.dart 72 ColorScheme + textTheme, light + dark ├── core/ │ ├── date_keys.dart 37 YYYY-MM-DD local day identity (Gregorian, pure) │ ├── fa_date.dart 33 Persian DISPLAY of Gregorian month names — NOT Jalali │ └── fa_digits.dart 7 Latin→Persian digits, display only ├── data/ │ ├── models.dart 267 QuestionCategory (8 + legacy aliases), Option, │ │ Question, AnswerRecord, DayMapping, SettingKeys │ ├── content_bundle.dart 399 validateBundle (structural) + validateLibrary (400/50) │ ├── content_audit.dart 141 Persian normalization + Dice near-duplicate detection │ ├── content_loader.dart 20 rootBundle → parseStructureOnly │ └── app_database.dart 221 sqflite, version 1, 3 tables, all accessors └── features/ ├── today/ │ ├── today_providers.dart 271 TodayNotifier + sealed TodayState + submitAnswer │ ├── today_screen.dart 182 LOADING/READY/ANSWERED/ERROR/EXHAUSTED rendering │ ├── selection.dart 127 selectUnseenQuestion (pure, deterministic) │ ├── result_engine.dart 68 resultFor(question, optionId) (pure lookup) │ └── widgets/ │ ├── question_card.dart 149 option list + optional free text + submit │ └── result_card.dart 98 insight + action (+ share footer) ├── history/ │ ├── history_providers.dart 20 historyProvider, statsProvider │ ├── history_screen.dart 263 stats strip + month calendar + day detail │ ├── history_entry.dart 92 sealed HistoryEntry (Answered | Unavailable) │ ├── streak.dart 76 computeStats (current/longest/total) │ ├── calendar.dart 87 buildMonthView, adjacentMonth (pure, Gregorian) │ └── widgets/month_calendar.dart 154 ├── settings/ │ ├── settings_screen.dart 131 daily reminder toggle + time picker ONLY │ ├── retention_providers.dart 78 reminderServiceProvider, resyncReminderFromStorage │ ├── reminder_service.dart 160 abstract ReminderService + Local + Fake │ ├── notifications.dart 100 pure policy: ids, channel, plan, next fire │ └── tz_location.dart 54 fixed-offset tz setup └── share/ ├── share_card.dart 112 ShareCard value object + builders ├── share_card_view\.dart 324 off-screen render host + private sub-widgets ├── share_card_style.dart 396 layout/measurement/text styles for export ├── share_card_renderer.dart 132 ShareCardRenderer, ShareImageEncoder, SharePort ├── share_port.dart 37 PlatformSharePort (share_plus) ├── share_sheet.dart 248 modal sheet + ShareReflectionButton └── share_providers.dart 202 ShareCardController, stamps recorder
content/questions/ 24 authored sources: \_a|b|c.json (\~26–36 KB each) assets/content/questions_v1.json 809,351 B (bundle version 1, 400 questions) assets/fonts/ Vazirmatn Regular/Medium/SemiBold/Bold + OFL.txt tool/build_content.dart, tool/content_merge.dart merge + audit + --check test/ 21 files, 213 tests integration_test/ DOES NOT EXIST

```text

**UI layer** — OBSERVED: three tab screens, one modal bottom sheet
(`ShareCardSheet`), one off-screen render host (`ShareCardCanvas`), one
startup error app. **No routing package, no named routes, no `go()`/`Navigator`
push anywhere** — the only `Navigator` use is `showDialog`/`showModalBottomSheet`
pop. Components are *not* abstracted into a shared library: only
`QuestionCard`, `ResultCard` and `MonthCalendar` are shared between screens
(`ResultCard` is the only genuinely reused one: Today + History). Everything
else builds raw Material widgets inline.

**State layer** — OBSERVED: **Riverpod 3.4.3.** No Bloc, GetX, ChangeNotifier,
ValueNotifier, or setState-for-app-state. Only `setState` is for local widget
state (selected month, selected day, notice text). Pattern for state change is
`ref.invalidateSelf()` / `ref.invalidate(target)`. Provider count: 9
production providers (appDatabase, contentBundle, todayDayKey,
reminderService, reminderSettings, todayState, history, stats,
shareRenderer, shareStampsRecorder, shareCardController).

**Domain layer** — OBSERVED: no repository interfaces, no use-case classes, no
DI container. The "domain" is 7 pure functions/engines. This is a deliberate
choice, documented in-file as M0–M5 lessons.

**Data layer** — OBSERVED: sqflite 2.4.4 (single file DB) + one bundled JSON
asset + no SharedPreferences, no Hive, no Isar, no secure storage. All
preferences live in the `settings` key/value table behind typed accessors on
`AppDatabase`.

**Data flow (OBSERVED, end to end):**
```

main() → AppDatabase.open() [fail → \_StartupErrorApp] → loadContentBundle() [parseStructureOnly; fail → \_StartupErrorApp] → resyncReminderFromStorage() [unawaited, swallows all errors] → ProviderScope(overrides: db, reminderService, shareRenderer) \_TabShell → IndexedStack([Today, History, Settings]) Today → todayStateProvider → \_resolve() → selection.dart → day_map/answer_records History → historyProvider → allAnswers() + bundle → resolveHistory → StreakStats Settings → reminderSettingsProvider → settings kv → ReminderService.syncReminder() Share → ShareReflectionButton → ShareCardSheet → ShareCardController → ShareCardRenderer → SharePort(share_plus) → DbShareStampsRecorder → narrow column UPDATE

```text

**Android config (OBSERVED):** `namespace`/`applicationId` = `com.harfhesab`;
`compileSdk`/`targetSdk`/`minSdk`/`versionCode`/`versionName` all delegated to
`flutter.*` (resolved on device: minSdk 24, targetSdk 36, compileSdk 36,
versionCode 1, versionName 1.0.0); `MainActivity : FlutterActivity`;
`android:windowSoftInputMode="adjustResize"`; `taskAffinity=""`;
`configChanges` includes locale/layoutDirection/fontScale; NDK reference
removed; `path_provider_android` pinned to 2.2.23 (documented M4 constraint:
CMake 3.22.1 and the NDK are unavailable in this build environment).


──────────────────────────────────────────────────────────
3. CURRENT SCREEN INVENTORY
──────────────────────────────────────────────────────────

| Field | SCREEN 1 |
|---|---|
| ID / Name | `S1` — Today |
| File | `lib/features/today/today_screen.dart` |
| Route entry | `IndexedStack` index 0 (default), `app.dart:52`; also the notification-tap target (`app.dart:63-65`) |
| Purpose | One reflection question per day → answer → result |
| Primary components | `QuestionCard` (headline + 4 `_OptionTile` + optional `TextField` + 52dp `FilledButton`), `ResultCard`, `ShareReflectionButton`, `_ExhaustedView`, `_ErrorView` |
| State used | `todayDayKeyProvider`, `todayStateProvider` (sealed `TodayState`) |
| Data used | `contentBundleProvider`, `day_map`, `answer_records` |
| Persistence | Writes `day_map` on presentation; writes `answer_records` on submit; both inside `db.transact` for submit |
| User actions | select option, type optional note, submit, share reflection |
| Dependencies | `share_plus`, `AppDatabase`, `ContentBundle` |
| Current risks | `_visibleDayKey` is assigned **inside `build()`** (line 84) — build-time mutation; `IndexedStack` keeps the tab alive so day changes depend on the resume hook; **no category affordance anywhere** |

| Field | SCREEN 2 |
|---|---|
| ID / Name | `S2` — History |
| File | `lib/features/history/history_screen.dart` |
| Route entry | `IndexedStack` index 1 |
| Purpose | Recall past reflections + streak stats |
| Primary components | `_StatsStrip` (3 `_Stat`), `MonthCalendar`, `_DayDetail`, `ResultCard` (shrinkWrap), `ShareReflectionButton` |
| State used | local `_year`, `_month`, `_selectedDayKey`; `historyProvider`, `statsProvider`, `todayDayKeyProvider` |
| Data used | `answer_records` (all), `contentBundleProvider` |
| Persistence | Read-only here; share stamps are narrow column updates |
| User actions | browse months, select a day, share a past reflection |
| Dependencies | `appDatabaseProvider`, `contentBundleProvider`, share module |
| Current risks | Forward month navigation stops at the current month (`adjacentMonth` returns null); `UnavailableEntry` branch exists for content drift |

| Field | SCREEN 3 |
|---|---|
| ID / Name | `S3` — Settings |
| File | `lib/features/settings/settings_screen.dart` |
| Route entry | `IndexedStack` index 2 |
| Purpose | **Daily reminder only** — enable toggle + time picker |
| Primary components | `SwitchListTile`, `ListTile` + `showTimePicker`, rationale `AlertDialog`, inline notice |
| State used | `reminderSettingsProvider` |
| Data used | `settings` kv (`notification_enabled`, `notification_time`) |
| Persistence | Writes kv first, then `syncReminder` (cancel-then-replace) |
| User actions | toggle reminder, pick time |
| Dependencies | `ReminderService` (abstract), `flutter_local_notifications` |
| Current risks | **No other settings exist.** No theme switch (light/dark come from the OS), no retention control despite the file being named `retention_*` |

| Field | SCREEN 4 |
|---|---|
| ID / Name | `S4` — Share Card sheet (modal) |
| File | `lib/features/share/share_sheet.dart` (`ShareCardSheet`, `ShareReflectionButton`, `showShareCardSheet`) |
| Route entry | Footer button on `ResultCard` — Today **and** History day detail |
| Purpose | Render a Persian reflection card to PNG and open the native share sheet |
| Primary components | `_Preview`, `_StatusLine`, `_Actions` |
| State used | `shareCardControllerProvider` (`ShareCardController extends Notifier<ShareState>`) |
| Data used | `ShareCard` value object; writes `card_generated_at_utc` / `shared_at_utc` |
| Persistence | Narrow column UPDATE via `DbShareStampsRecorder` wrapped in `SafeShareStampsRecorder` (never blocks a share) |
| User actions | preview, generate, share |
| Dependencies | `share_plus` (`SharePort`), `ShareImageEncoder` |
| Current risks | Off-screen `GlobalKey` capture host must be laid out at export size; `shareText` is stubbed to `''` in the History path (`history_screen.dart:230`) |

| Field | SCREEN 5 |
|---|---|
| ID / Name | `S5` — Share card canvas (off-screen, not user-navigable) |
| File | `lib/features/share/share_card_view.dart` (`ShareCardCanvas`, `_Masthead`, `_Rule`, `_AnswerBlock`, `_ActionBlock`, `_Footer`) |
| Route entry | None — mounted invisibly by the sheet at export size |
| Purpose | Deterministic PNG layout |
| State | none (StatelessWidget) |
| Data | `ShareCardContent` from `share_card_style.dart` |
| Persistence | none |
| Dependencies | `ShareCardTextStyles` (the **only** place Vazirmatn is used) |
| Current risks | Layout is hand-measured; any text-length change can overflow |

| Field | SCREEN 6 |
|---|---|
| ID / Name | `S6` — Startup error app |
| File | `lib/main.dart:57-82` (`_StartupErrorApp`) |
| Route entry | Replaces the whole app if DB open or bundle validation fails |
| Purpose | Fail loudly, never a silent broken shell |
| Components | `MaterialApp` + `Scaffold` + centered `Text` |
| State | none |
| Data | error message string |
| Persistence | none |
| Dependencies | none |
| Current risks | Does **not** use the app theme (its own bare `MaterialApp`) |

| Field | SCREEN 7 |
|---|---|
| ID / Name | `S7` — Day detail (embedded, not a route) |
| File | `lib/features/history/history_screen.dart:190-258` (`_DayDetail`) |
| Route entry | Rendered inside History below the calendar |
| Purpose | Show one specific day's stored reflection |
| Components | `ResultCard` (shrinkWrap) or one of 3 text states |
| State | `HistoryEntry?` |
| Data | `AnsweredEntry` / `UnavailableEntry` |
| Persistence | read-only |
| User actions | share |
| Dependencies | share module |
| Current risks | 4 mutually exclusive `Text` branches; empty-string `shareText` injected into `TodayResult` |

**NOT FOUND (RULE 2) — searched `lib/**/*.dart` for `onboarding`, `splash`,
`walkthrough`, `intro`:**
* Onboarding screen: **NOT FOUND**. `SettingKeys.onboardingDone` exists at
  `models.dart:299` with **zero references** anywhere in `lib/`, `test/`, `tool/`.
* Task/goal list, task detail, statistics dashboard, search, category browser:
  **NOT FOUND** (searched `class Goal`, `class Task`, `goal`, `task_`, `tasks`,
  `dueDate`, `checklist`, `todo`, `milestone`, `habit` — the only `habit`
  hit is the category label `habits_and_discipline`).


──────────────────────────────────────────────────────────
4. CURRENT QUESTION ENGINE
──────────────────────────────────────────────────────────

**Model (OBSERVED, `lib/data/models.dart`):**
* `Question { id, category: QuestionCategory, answerType: AnswerType, question, options: List<Option>, contentVersion }` — immutable, `==`/`hashCode` by value, `toMap`/`fromMap`.
* `Option { id, title, result, action, shareText }` — **the result, the action and the share text are authored per option**, not computed.
* `QuestionCategory` — 8 members, each with `wireName` (snake_case, persisted) and `faLabel` (Persian, never persisted). Legacy aliases `general|self → selfAwareness`, `business → workAndBusiness`, `relations → relationships`, `lifestyle → lifeAndValues` are read-only and used **only** when parsing persisted rows.
* `AnswerType { singleChoice, textOptional }` — only `self_001` is `textOptional`.
* `AnswerRecord` — 9 columns, `dayKey` PK, carries `card_generated_at_utc` and `shared_at_utc` for M4.

**Source (OBSERVED):** 24 authored files `content/questions/<category>_{a,b,c}.json`,
each `{ "category": "<wireName>", "questions": [...] }`. Merged deterministically
by `tool/content_merge.dart` into `assets/content/questions_v1.json`
(bundle `version: 1`, 400 questions) by `tool/build_content.dart`, which also
runs `validateLibrary` and can `--check` for drift.

**Loading (OBSERVED):** `contentLoaderProvider` → `loadContentBundle()` reads
`assets/content/questions_v1.json` via `rootBundle` → `parseStructureOnly`.
**Deliberately does NOT run the 400/50 library contract at runtime** — that is
enforced by the build tool and `content_library_test.dart` (rationale in
`content_bundle.dart:410-416`).

**Filtering:** `ContentBundle.byId(id)` only. There is **no category query on
the bundle** (no `byCategory`). `category` is a field on `Question`, so
filtering is a one-line `where`, not a new capability.

**Answer representation:** `optionId` (string) + optional `freeText`.
`submitAnswer` validates that the option belongs to the question
(`today_providers.dart:112-116`) and throws `ResultException` otherwise.

**Exact selection algorithm (OBSERVED, `today_providers.dart:160-231` + `selection.dart:75-141`):**

```

\_resolve(db, bundle, dayKey):

1. mapping = db.getDayMapping(dayKey)
2. if mapping == null: seen = db.seenQuestionIds() // SELECT DISTINCT question_id FROM day_map outcome = selectUnseenQuestion(bundle, dayKey, seen) if outcome is QuestionsExhausted -> return TodayExhausted // NO mapping written mapping = DayMapping(dayKey, outcome.question.id) db.putDayMapping(mapping) // <-- this write IS the "seen" mark
3. question = bundle.byId(mapping.questionId) // else TodayContentError
4. record = db.getAnswer(dayKey) if record != null: return TodayAnswered(result: resultFor(record's question/option)) else: return TodayReady(question)

selectUnseenQuestion(bundle, dayKey, seenIds): if bundle.questions.isEmpty -> throw SelectionException unseen = questions not in seenIds, sorted by id if unseen.isEmpty -> return QuestionsExhausted(total, seen) exposure = per category, counting only seenIds resolvable in the bundle (ids from retired content are skipped, never thrown) available= categories that still own >=1 unseen question lowest = min(exposure over available) eligible = all categories whose exposure == lowest candidates = unseen questions in eligible categories best = argmin over candidates of fnv1a32('$dayKey#${bundle.version}#${questionId}') return QuestionSelected(best, eligibleCount: candidates.length)

````text

**Why the hash is min-hash and not modulo (OBSERVED, `selection.dart:128-130`):**
modulo reshuffles every candidate whenever the pool size changes; min-hash
behaves like a fixed pseudo-random priority order consumed one item per day.

**Daily state (OBSERVED):** `currentDayKey()` is the *only* production caller of
`DateTime.now()` for day identity. `app.dart:93` watches `todayDayKeyProvider`
so the shell rebuilds; `today_screen.dart:46-54` on `AppLifecycleState.resumed`
compares `_visibleDayKey` to a live read and, on change, invalidates
`todayDayKeyProvider` + calls `TodayNotifier.refresh()`. `historyProvider` and
`statsProvider` both `ref.watch(todayDayKeyProvider)` so they refresh with it.

**History relationship (OBSERVED):** `answer_records` is the single source for
History and stats. `resolveHistory()` joins records to the bundle; unknown
question/option ids degrade to `UnavailableEntry` — **a drifted record is
preserved and shown neutrally, never reinterpreted.**


──────────────────────────────────────────────────────────
5. CURRENT ANTI-REPETITION SYSTEM
──────────────────────────────────────────────────────────

**The mechanism (OBSERVED):** there is **no seen-table**. `day_map` is
*already* a presentation ledger — this is the single most important
architectural fact in this document. Rationale is documented at
`app_database.dart:186-200`.

| Question | Answer (OBSERVED) |
|---|---|
| 1. Where are answered IDs stored? | `day_map.question_id` (distinct values). Never a separate table, never a settings key. |
| 2. When are they written? | `TodayNotifier._resolve` at `today_providers.dart:191`, **immediately after selection and before the user sees anything** — so a question the user merely *looked at* is consumed. Written again inside `submitAnswer` (`:119`) but idempotently (`ConflictAlgorithm.replace`, same value). |
| 3. When are they read? | Only in `_resolve` **when `mapping == null`** (`:170`), i.e. exactly once per new day, and then passed to `selectUnseenQuestion`. |
| 4. Global or category-specific? | **Global.** The seen set spans all 400 ids. Category awareness exists only as *exposure balancing* (prefer least-exposed categories), never as a filter. |
| 5. After reinstall? | **LOST.** `day_map` is app-private SQLite; uninstall deletes it. There is no backup, export, or seed. All 400 questions become unseen again. Existing `answer_records` are equally lost. |
| 6. After app restart? | Preserved (file on disk). |
| 7. After force-stop? | Preserved (file on disk). |
| 8. When one category is exhausted? | **Not a representable state today.** A category with 0 unseen questions is simply excluded from `available` (`selection.dart:115-118`) and is never selected. Nothing is surfaced to the user and no error occurs. In practice all 8 categories drain together — the M5 400-day test asserts exactly 50 exposures per category. |
| 9. When all are exhausted? | `QuestionsExhausted` → `TodayExhausted(total, seen)` state → `_ExhaustedView` (`today_screen.dart:142-175`). Calm, terminal, **no retry button**, and **no `day_map` row is written**, so a 401st question cannot even be represented. |
| 10. Can category filtering reset repetition state? | **This is the live risk and the reason ADR-003 exists.** `day_map` PK is `day_key`, so there is exactly one frozen question per day. A naive same-day category switch must either (a) overwrite/delete the day's row — re-consuming a question and potentially re-presenting one the user has already seen that day, or (b) ignore the choice silently. The safe answer is: **the day's question is immutable once created.** |

**Corroborating test evidence (OBSERVED, `test/selection_anti_repetition_test.dart`, 15 tests):** 400-day real-notifier + real-file-backed-SQLite run delivering 400 unique ids; day 401 → `TodayExhausted`; `mappedDayCount()` stays 400; exposure exactly 50 per category; persistence proven across a real file close/reopen; the golden vector `2026-09-23 → relationships_004` is pinned.


──────────────────────────────────────────────────────────
6. CURRENT PERSISTENCE
──────────────────────────────────────────────────────────

| Data | Storage | Key / Table | Write trigger | Read trigger | Restart | Force-stop | Reinstall |
|---|---|---|---|---|---|---|---|
| Question bundle | **Bundled asset** | `assets/content/questions_v1.json` (809,351 B) | build tool only | `contentBundleProvider` at startup | ✅ | ✅ | ✅ |
| Day → question (ledger) | SQLite | `day_map(day_key PK, question_id)` | first presentation of a day (`:191`) + every submit | `_resolve` when mapping is null | ✅ | ✅ | ❌ lost |
| Answered day | SQLite | `answer_records(day_key PK, question_id, option_id, free_text, category, answered_at_utc, updated_at_utc, card_generated_at_utc, shared_at_utc)` | `submitAnswer` inside `transact` | `getAnswer(dayKey)`, `allAnswers()` | ✅ | ✅ | ❌ lost |
| Streak / stats | **Derived** | none — `computeStats(answeredDayKeys, todayKey)` | n/a (derived) | `statsProvider` | ✅ | ✅ | ❌ lost |
| History entries | **Derived** | none — `resolveHistory(allAnswers(), bundle)` | n/a | `historyProvider` | ✅ | ✅ | ❌ lost |
| Selected question | SQLite | `day_map` (same row) | with the ledger | `_resolve` | ✅ | ✅ | ❌ |
| Reminder enabled | SQLite | `settings['notification_enabled']` | toggle | `reminderSettingsProvider` | ✅ | ✅ | ❌ |
| Reminder time | SQLite | `settings['notification_time']` (`HH:MM` regex-validated) | time picker | same | ✅ | ✅ | ❌ |
| Share stamps | SQLite | `answer_records.card_generated_at_utc` / `shared_at_utc` | share flow (best-effort) | not read back by `fromMap` | ✅ | ✅ | ❌ |
| Theme | **None** | — | — | follows OS via `darkTheme` | ✅ | ✅ | ✅ |
| **Selected category** | **NONE — does not exist** | `SettingKeys.enabledCategories` declared at `models.dart:298`, **0 references** | — | — | — | — | — |

**Schema (OBSERVED, `_createV1`):** 3 tables, no indices beyond the two
implicit PKs, **no foreign keys**, no `settings` seed rows. `databaseVersion = 1`.
`_upgrade` has a `case 1:` stub with a comment "v1 → v2 step goes here when v2
exists" — the ladder is already in place. `migrateForTest` is exposed
`@visibleForTesting` for exactly this purpose.

**Transactions (OBSERVED):** `AppDatabase.transact` refuses nesting
(`StateError`) and returns a transaction-scoped `AppDatabase`. Only
`submitAnswer` uses it, so "a day never has a mapping without an answer (or
vice versa)" is atomic.


──────────────────────────────────────────────────────────
7. CURRENT NOTIFICATION SYSTEM
──────────────────────────────────────────────────────────

**OBSERVED — exactly one notification architecture exists. Phase 2 must extend
it, never add a second library.**

* **Service:** abstract `ReminderService` (`reminder_service.dart:15-37`) with
  `init`, `areNotificationsEnabled`, `requestPermission`, `syncReminder`,
  `cancelReminder`, `Stream<void> taps`. Production `LocalReminderService`
  wraps `flutter_local_notifications` 22.3.1. Test `FakeReminderService` is an
  in-memory recorder (no platform I/O). Overridden via
  `reminderServiceProvider`.
* **Policy (pure, `notifications.dart`):** `dailyReminderId = 1701`;
  channel `harf_daily` / «یادآور روزانه»; copy `reminderTitle = 'حرف حساب'`,
  `reminderBody = 'حرف حساب امروز منتظره.'`, `reminderPayload = 'today'`;
  `defaultReminderTime = '21:00'`; `parseReminderTime` / `formatReminderTime`
  (strict `HH:MM`); `nextReminderFire` uses **calendar-field arithmetic, never a
  24h span**; `planReminderSync` returns a `ReminderPlan`.
* **Permission handling:** rationale `AlertDialog` is shown **before** the OS is
  ever asked (`settings_screen.dart:102-125`); then
  `areNotificationsEnabled()` → if false, `requestPermission()` once. Denial
  keeps the local preference and shows an inline notice — **no prompt loop**, app
  stays fully usable. Returns true pre-Android-13 where no prompt exists.
* **Scheduling:** `syncReminder` **cancels unconditionally first**, then
  schedules only if `plan.schedule`. `androidScheduleMode:
  AndroidScheduleMode.inexactAllowWhileIdle` — a reminder, not an alarm;
  **no `SCHEDULE_EXACT_ALARM`** (deliberate, documented). `matchDateTimeComponents:
  DateTimeComponents.time` for daily recurrence. Wall-clock scheduling via
  `wallClockIn(plan.firstFire!, tz.local)`.
* **Cancellation:** `cancelReminder()`; and the unconditional cancel at the top
  of `syncReminder` is what makes replacement duplicate-free across repeated app
  starts.
* **Persistence:** the *preference* is in `settings`; the *schedule* lives in
  the plugin. **Reboot/timezone recovery is app-driven:**
  `resyncReminderFromStorage()` is called once at startup (unawaited,
  `main.dart:41`) and on every `AppLifecycleState.resumed` (`app.dart:81-86`).
* **Timezone handling:** `_applyDeviceTimezone()` sets `tz.local` to
  `fixedOffsetLocation(DateTime.now().timeZoneOffset) ?? tz.UTC`. The file
  comment concedes a fixed offset carries no DST transitions — acceptable
  because the app's target locale (`fa`) has no DST zone in use. **No tz
  database is loaded** despite `timezone` 0.11.1 being a dependency.
* **Android manifest:** `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`,
  `ScheduledNotificationReceiver`, and `ScheduledNotificationBootReceiver` with
  `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `QUICKBOOT_POWERON`,
  `com.htc.intent.action.QUICKBOOT_POWERON`. No exact-alarm permission. No
  `SCHEDULE_EXACT_ALARM`, no `USE_FULL_SCREEN_INTENT`, no `VIBRATE`.
* **Existing UI:** one `SwitchListTile` + one `ListTile`/`showTimePicker` in
  Settings. There is **no notification list, no per-item state, and no
  permission-status screen**.
* **Extensibility for Phase 2:** `flutter_local_notifications` already supports
  many scheduled ids on additional channels. The gaps to design for are
  (a) a **stable per-task notification id**, (b) a **second channel** for task
  reminders, (c) per-task cancel on delete/complete/time-change, and
  (d) a **resync entry point for task schedules** on boot/timezone change
  (the current `resyncReminderFromStorage` is hard-coded to the daily reminder).
* **NOT FOUND:** any second notification library, any `AlarmManager`, any
  `WorkManager`, any notification history/log, any tap routing beyond
  `payload == 'today'`.


──────────────────────────────────────────────────────────
8. CURRENT DESIGN SYSTEM
──────────────────────────────────────────────────────────

**Principle (OBSERVED, `tokens.dart:1-5`):** "the only place raw
color/spacing/radius values live". This is a genuinely enforced rule.

**Colors (`tokens.dart`)** — calm paper-and-ink, light and dark in one family:

| Token | Light | Dark |
|---|---|---|
| page background | `#F7F3EC` | `#171410` |
| primary surface | `#FFFFFF` | `#211D17` |
| secondary surface | `#EFE7D8` | `#2B261D` |
| elevated surface | `#FFFFFF` | `#2B261D` |
| primary text | `#23201A` | `#F1EADF` |
| secondary text | `#6E675C` | `#B8AE9D` |
| disabled text | `#B3AA9C` | `#6B6355` |
| **accent** | `#2F6B5E` (deep teal-green) | `#7FB5A4` |
| accent foreground | `#FFFFFF` | `#10201B` |
| divider | `#E2D9C8` | `#38312A` |
| success / warning / error | `#3E7C4F` / `#B07A2A` / `#B23A3A` | (same, not themed) |

**No glass/blur component exists** — `secondarySurface` is an opaque cream, not
translucency. (Noted because the brief mentions "glass panels"; that aesthetic
is NOT currently in the product.)

**Spacing:** 4-pt scale — `xs 4, sm 8, md 16, lg 24, xl 32, xxl 48`.
**Radii:** `sm 8, md 12, lg 16, xl 24, full 999`.

**Typography (`typography.dart`) — 7 styles, and a significant finding:**

| Style | size | height | weight |
|---|---|---|---|
| `display` | 32 | 1.5 | 700 |
| `headline` | 24 | 1.6 | 700 |
| `title` | 20 | 1.6 | 600 |
| `body` | 16 | 1.9 | 400 |
| `bodyEmphasized` | 16 | 1.9 | 600 |
| `label` | 14 | 1.7 | 500 |
| `caption` | 12 | 1.7 | 400 |

Line heights are deliberately generous (1.5–1.9) for Persian glyphs.
**No `fontFamily` is set on any of them, and `app_theme.dart` does not set one
on `ThemeData`.** The in-app UI therefore renders in the *platform* Persian
face. The bundled **Vazirmatn** family (4 weights, SIL OFL) is declared in
`pubspec.yaml` and used **only** by `share_card_style.dart` for PNG export.
Wiring `fontFamily: 'Vazirmatn'` into the UI is a **zero-new-dependency**
modernization and is the single highest-leverage Phase 1 UI change.

**Components:** there is **no shared component library**. Each screen composes
raw Material widgets. `FilledButton`, `SwitchListTile`, `ListTile`,
`showTimePicker`, `showDialog`, `showModalBottomSheet`, `InkWell` + `Container`
with `BoxDecoration`, `Semantics`. Extracted/reused widgets: `QuestionCard`
+ `_OptionTile` (Today only), `ResultCard` (Today + History), `MonthCalendar`
(History only), `ShareReflectionButton` (Today + History). Icons: Material
`Icons.*` only — **no custom icon set**.

**Motion (OBSERVED):** no `AnimationController`, no `Hero`, no page transitions.
Only implicit Material animation: `CircularProgressIndicator` (Today loading,
History loading, submit button spinner) and the disabled-button transition.

**RTL (OBSERVED):** app-level only — `locale: Locale('fa')`,
`supportedLocales: [Locale('fa')]`, Material/Widgets/Cupertino localizations
delegates. The file comment at `app.dart:3` states the rule explicitly: "RTL is
app-level (locale fa) — no screen sets direction itself." Evidence of correct
convention: `EdgeInsetsDirectional.only(start:/end:)` throughout,
`MainAxisAlignment` / `CrossAxisAlignment` rather than `left`/`right`, radio
icon placed first in the `Row` so it lands on the right, and `faDigits()` for
every displayed number (stats, dates, totals). Persian month names come from
`gregorianMonthNamesFa` and the Iran week starts on شنبه
(`weekDayHeadersFa = ['ش','ی','د','س','چ','پ','ج']`, `calendar.dart:26`).
**Confirmed on the physical device in the M6 pre-check screenshot:** title
«امروز» right-aligned, option radios on the right, bottom nav ordered
right-to-left with امروز active, clock `۰۸:۴۳` in Persian digits, ZWNJ in
«ویژگی‌ات» rendering correctly, no overflow, no tofu.

**Jalali: NOT FOUND.** `fa_date.dart:1-4` states identity is Gregorian and the
file "never converts to/from Jalali". A search for `jalali|shamsi|هجری|شمسی`
across `lib/` and `pubspec.yaml` returned **3 hits, all in comments saying Jalali
is deliberately not used**.


──────────────────────────────────────────────────────────
9. PHASE 1 UX PROPOSAL
──────────────────────────────────────────────────────────

### 9.1 Where category selection belongs

PROPOSED: **a modal bottom sheet opened from a persistent category chip inside
the Today screen.** Not a new tab, not a new route, not Settings.

Reasoning against the alternatives, in the terms the brief asks for:

| Option | Interactions | Discoverability | Persistence fit | Verdict |
|---|---|---|---|---|
| Inline on Today (always-visible list of 8) | 0 extra taps | Max | Perfect | **Rejected** — 8 cards above the question destroys the calm focus of the Today screen and forces a decision before every answer |
| **Bottom sheet from a Today chip** | **1 extra tap, only when changing** | High (the chip *is* the affordance, and it always shows the current state) | Perfect (it is a per-day choice) | **PROPOSED** |
| New tab | 1 tap, but +1 permanent destination | High | Poor (a tab for a transient choice) | **Rejected** — §25 forbids blindly adding nav, and this is not a destination |
| Settings only | 3+ taps, on a low-traffic screen | **Low** — a first-time user never opens Settings | Poor | **Rejected** — the brief makes category choice a *core-loop* concern ("what category do I want **today**"), not a preference |

Scalability: the sheet is a `ListView`, so 8 categories fit on a 720×1600 screen
with room for a 9th without a redesign.

### 9.2 The screen

```text
┌──────────────────────────────────────────────┐  ← bottom sheet, drag handle
│                                              │
│              امروز از کدام حوزه؟              │  TITLE
│                                              │
│   حوزه‌ای را انتخاب کن که امروز می‌خواهی      │  DESCRIPTION
│   روی آن مکث کنی. این انتخاب تا پایان امروز   │
│   عوض نمی‌شود.                                │
│                                              │
│  ┌──────────────────────────────────────────┐│
│  │ 🌀  همهٔ حوزه‌ها            ۳۵۲ باقی‌مانده ││  ALL (default, full width)
│  └──────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────┐│
│  │ 👁  خودشناسی و خودآگاهی        ۴۴ باقی‌مانده││  CATEGORY CARD
│  │     ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░  ││  EXPOSURE BAR
│  └──────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────┐│
│  │ ⚖  تصمیم‌گیری و تفکر             ۵۰ باقی‌مانده││
│  └──────────────────────────────────────────┘│
│  …  6 more …
│  ┌──────────────────────────────────────────┐│
│  │ 🕘  عادت‌ها، نظم و عمل‌گرایی       ۰ باقی‌مانده││  EXHAUSTED (dimmed, disabled)
│  └──────────────────────────────────────────┘│
│                                              │
│  ┌──────────────────────────────────────────┐│
│  │           ادامه                          ││  CTA (disabled until a pick)
│  └──────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
````

| **Element**        | **PROPOSED spec**                                                                                                                                                                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Header             | Reuses the app **`AppBar`** title «انتخاب حوزه»; sheet is **`showModalBottomSheet`** with **`isScrollControlled: true`**                                                                                                                                                     |
| Title              | «امروز از کدام حوزه؟»                                                                                                                                                                                                                                                        |
| Description        | «حوزه‌ای را انتخاب کن که امروز می‌خواهی روی آن مکث کنی. این انتخاب تا پایان امروز عوض نمی‌شود.» — states the immutability rule up front, which is the honest way to handle ADR-003                                                                                           |
| Category card      | **`minHeight 64`**, **`AppRadius.md`**, **`AppSpacing.md`** padding, **`AppColors.primarySurface`** fill, **`AppColors.divider`** 1dp border, 2dp **`accent`** border when selected + **`accent @ 12%`** fill (exactly the selected idiom already used by **`_OptionTile`**) |
| Category icon      | Material **`Icons.*`**, one per category, **`AppColors.secondaryText`**, RTL-safe (no directional icons)                                                                                                                                                                     |
| Category name      | **`text.title`** (20/1.6/600) — **the existing `faLabel`, never re-authored**                                                                                                                                                                                                |
| Remaining count    | **`text.label`** + **`faDigits(...)`** → «۴۴ باقی‌مانده»                                                                                                                                                                                                                     |
| Progress bar       | 3dp **`ClipRRect(AppRadius.full)`**, track **`AppColors.secondarySurface`**, fill **`accent`**. Width = **`remaining / 50`**, laid out **RTL** (fills from the right)                                                                                                        |
| All-categories row | Separate, full-width, first in the list; label «همهٔ حوزه‌ها»; count = total unseen                                                                                                                                                                                          |
| Selected state     | 2dp accent border + accent-tinted fill + **`Icons.radio_button_checked`** (consistent with **`_OptionTile`**)                                                                                                                                                                |
| CTA                | **`FilledButton`**, 52dp (same height as the submit button), full width, label «ادامه». Disabled when nothing is chosen or the chosen category is exhausted                                                                                                                  |
| Back / cancel      | Sheet drag or system back = cancel, no write                                                                                                                                                                                                                                 |

**Copy set (PROPOSED, §35):**

| **State**                       | **Text**                                                                                                            |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Title                           | «امروز از کدام حوزه؟»                                                                                               |
| Subtitle                        | «حوزه‌ای را انتخاب کن که امروز می‌خواهی روی آن مکث کنی. این انتخاب تا پایان امروز عوض نمی‌شود.»                     |
| All categories                  | «همهٔ حوزه‌ها»                                                                                                      |
| Remaining                       | «۴۴ باقی‌مانده»                                                                                                     |
| CTA                             | «ادامه»                                                                                                             |
| Exhausted badge                 | «تمام شد»                                                                                                           |
| Empty/loading                   | «در حال آماده‌کردن حوزه‌ها…»                                                                                        |
| Error                           | «حوزه‌ها بارگذاری نشدند.» + «تلاش دوباره»                                                                           |
| Post-answer / day-answered lock | «سؤال امروز مشخص شده. از فردا می‌توانی حوزه را عوض کنی.»                                                            |
| Exhausted-selected state        | «همهٔ سؤال‌های این حوزه را مرور کردی.» / «حوزهٔ دیگری انتخاب کن یا همهٔ حوزه‌ها را ببین.» / CTA «انتخاب حوزهٔ دیگر» |

### 9.3 Category states

| **State** | **Definition**                             | **PROPOSED behavior**                                                                      |
| --------- | ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| DEFAULT   | available, not chosen                      | tappable, **`divider`** border, secondary-text icon                                        |
| SELECTED  | user picked it this session                | 2dp accent border, accent-tinted fill, checked icon, CTA enabled                           |
| PRESSED   | pointer/tap feedback                       | Material **`InkWell`** ripple within **`AppRadius.md`** (matches **`_OptionTile`**)        |
| DISABLED  | **`remaining == 0`**                       | opacity \~0.45, **`onTap == null`**, no ripple, «تمام شد» label, counts **`۰ باقی‌مانده`** |
| EXHAUSTED | same as DISABLED but semantically distinct | see §14.4 / §18                                                                            |
| LOADING   | counts not yet computed                    | **`CircularProgressIndicator`** in place of the progress bar, taps ignored                 |
| ERROR     | bundle/DB read failed                      | whole sheet shows the error state + «تلاش دوباره»; never a partially-populated list        |

### 9.4 Category exhaustion (explicit, as required)

```text
```

PROPOSED, three separate surfaces:

1. **In the sheet** — the card is DISABLED, shows «تمام شد» and «۰ باقی‌مانده», and cannot be picked. The user is never shown a category that cannot deliver.
2. **If it becomes exhausted while selected** (e.g. it was the last remaining question in that category) — the day resolves to a new **`TodayCategoryExhausted(cat, remaining: 0)`** state rendering:
   - «همهٔ سؤال‌های این حوزه را مرور کردی.»
   - «حوزهٔ دیگری انتخاب کن یا به همهٔ حوزه‌ها برگرد.»
   - CTA «انتخاب حوزهٔ دیگر» → reopens the sheet, pre-filtered to non-exhausted categories.
   - **No `day_map` row is written. Nothing is consumed.** Exactly mirroring the existing **`TodayExhausted`** discipline at **`today_providers.dart:184-187`**.
3. **Never a silent fallback to "all categories."** Auto-switching would consume from an unrequested scope and quietly violate the user's choice. This is a deliberate rejection, not an oversight.

**`remaining(cat) = 50 − |{seenIds ∩ bundle.byId(id).category == cat}|`** — derived, never stored, so it can never drift from the ledger.
────────────────────────────────────────────────────────── 10. PHASE 1 ARCHITECTURE ──────────────────────────────────────────────────────────

### 10.1 Data model — EXTEND, do not duplicate

OBSERVED: the question model **already fully supports category filtering**. **`Question.category`** is a **`QuestionCategory`** field and **`QuestionCategory`** already carries the Persian **`faLabel`** and the **`wireName`** that the UI will persist. **`ContentBundle.byId`** is the only query, but filtering is **`bundle.questions.where((q) => q.category == c)`**.
**PROPOSED: no change to `models.dart`'s data classes. No new category table, no category metadata duplication, no new JSON.** The only additions are a settings key and typed accessors, plus one query helper on the bundle.
One design consequence to record: **`ContentLibraryPolicy`** currently *requires exactly 50 per category* (**`content_bundle.dart:281-302`**) and that check runs in the build tool and **`content_library_test.dart`**. **A per-category filter does not affect this** (it validates the bundle, not the selection). But if the product ever wants unequal category sizes, that validator must change first.

### 10.2 State model

| **Piece**                                                    | **PROPOSED owner**                                                                 | **Persisted?**                        | **Derived?**                            |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------- |
| **`selectedCategory`** (**`QuestionCategory?`**, null = all) | new **`categoryFocusProvider`** (**`AsyncNotifier<CategoryFocus?>`**)              | ✅ **`settings['selected_category']`** | —                                       |
| **`availableCategories`**                                    | plain **`List<QuestionCategory>`** from **`QuestionCategory.values`**              | ❌                                     | ✅ enum                                  |
| **`remainingQuestions`**                                     | new **`categoryCountsProvider`** (**`FutureProvider<Map<QuestionCategory,int>>`**) | ❌                                     | ✅ from bundle + **`seenQuestionIds()`** |
| **`categoryStatus`**                                         | plain enum **`CategoryStatus { available, exhausted }`** per category              | ❌                                     | ✅                                       |
| the day's question                                           | **unchanged** — **`day_map`** + **`TodayNotifier`**                                | ✅                                     | —                                       |

**Why here (PROPOSED):**

- **`categoryFocusProvider`** belongs in a new **`lib/features/today/category_providers.dart`** beside **`today_providers.dart`**, because **it is a Today-loop concern**, not a global app preference. It must not live in **`app/providers.dart`** (that file is the M1 foundation graph) and must not live in Settings (which is currently reminder-only).
- It is an **`AsyncNotifier`**, not a **`Notifier`**, because the value lives in SQLite and every read is async — exactly mirroring **`ReminderSettingsNotifier`**.
- It **owns** writes; the sheet only calls **`setFocus`**. The sheet never touches **`AppDatabase`**, matching the M3 precedent where **`SettingsScreen`** calls the notifier and the notifier owns both the DB write *and* the side effect.
- Persistence is a single **`settings`** row. **`db.setSetting`** / **`db.getSetting`** already exist (**`app_database.dart:220-237`**) and are **`ConflictAlgorithm.replace`**, so no schema change is needed at all.

### 10.3 New DB accessors (Phase 1) — no migration

```dart
// app_database.dart — typed accessors beside the existing ones (§ symmetry)
Future<QuestionCategory?> getSelectedCategory();   // null == all categories
Future<void> setSelectedCategory(QuestionCategory? c);
```

Backed by the existing **`settings`** table with a new key **`SettingKeys.selectedCategory = 'selected_category'`**.
**Decision on the dead `enabledCategories` constant (models.dart:298):** PROPOSED — do **not** use it. Its name and comment ("comma-separated") describe a *multi-select filter set*, which is a different feature from the single-focus choice the brief specifies. Two similarly-named settings keys with different semantics is a future bug. Either use it later for a real multi-select filter, or delete it. **It stays untouched in Phase 1** (deleting it is out of scope here) but must not be silently co-opted.

### 10.4 Selection contract (the only behavioural change)

PROPOSED: **`selectUnseenQuestion`** gains one **optional, purely additive** parameter. Existing behaviour with the default argument is bit-for-bit identical.

```dart
SelectionOutcome selectUnseenQuestion({
  required ContentBundle bundle,
  required String dayKey,
  required Set<String> seenIds,
  QuestionCategory? categoryFilter = null,   // NEW, defaults to null == M5 behaviour
})
```

Inserted after the **`unseen.isEmpty`** check, before exposure balancing:

```text
if (categoryFilter != null) {
  unseen = unseen.where((q) => q.category == categoryFilter);
  if (unseen.isEmpty) return CategoryExhausted(category: categoryFilter);
}
```

**Critical detail — the two exhaustion outcomes must be distinguishable.** **`CategoryExhausted`** is a *new* member of the **`SelectionOutcome`** sealed class, so the existing **`switch`** in **`_resolve`** is exhaustively checked by the compiler and **cannot silently fall through**. Existing tests that call **`selectUnseenQuestion`** without the new argument keep passing unchanged.

### 10.5 The immutability rule (ADR-003 in code)

PROPOSED, enforced in **`TodayNotifier`**:

```text
The day's question is created once and is immutable for that day.
```

- If **`day_map`** already has today's row → the sheet opens **read-only**, every card is disabled, and the notice «سؤال امروز مشخص شده. از فردا می‌توانی حوزه را عوض کنی.» is shown. No DB write is possible from that path.
- The persisted **`selected_category`** is still updated (it governs *tomorrow* and the remaining counts), but **`_resolve`** passes the **filter only when `mapping == null`**, i.e. when today's question is about to be created.

**This is the whole reason the feature is safe.** Because **`categoryFilter`** is consulted *only* on the **`mapping == null`** branch, a category change can never delete or overwrite a **`day_map`** row, can never re-consume a question, and can never re-present a question the user already saw today.

### 10.6 Lifecycle interaction

- **App restart / force-stop / cold start:** **`categoryFocusProvider`** reads **`settings`** like **`reminderSettingsProvider`** does. The sheet's in-progress selection is intentionally **not** persisted until CTA — a user who opens the sheet and backs out leaves the stored focus untouched.
- **New day:** nothing to do. **`selected_category`** persists across days; the new day creates a new **`day_map`** row using it. This matches the existing pattern where the day key alone drives rollover.
- **Returning from Result / History:** the sheet is a modal; dismissal needs no coordination. **`IndexedStack`** keeps the Today tab alive, so the chip must read from the provider (not local state) to stay truthful.
- **Resuming after midnight:** **`today_screen.dart`**'s existing resume hook already invalidates **`todayDayKeyProvider`** + refreshes **`todayStateProvider`**. The chip will re-read **`selected_category`** (unchanged) and **`categoryCountsProvider`** must **`ref.watch(todayDayKeyProvider)`** so remaining counts advance at the same moment. **This is a required detail** — a counts provider that does not watch the day key would show yesterday's numbers.

### 10.7 Dependency policy for Phase 1

**PROPOSED: ZERO new dependencies.** Everything needed already exists: Riverpod (state), the **`settings`** table (persistence), **`faLabel`** (labels), **`faDigits`** (numerals), **`AppColors`**/**`AppSpacing`**/**`AppRadius`** (design), **`Material`** bottom sheet (UI). See §21/§33.
────────────────────────────────────────────────────────── 11. PHASE 2 DOMAIN MODEL ──────────────────────────────────────────────────────────
OBSERVED: **no Goal, Task, reminder, due-date, checklist, or progress concept exists anywhere in `lib/`** (searched **`class Goal`**, **`class Task`**, **`goal`**, **`task_`**, **`tasks`**, **`dueDate`**, **`checklist`**, **`todo`**, **`milestone`** — zero hits).

### 11.1 Goal

```sql
CREATE TABLE goals(
  id                 TEXT PRIMARY KEY,     -- uuid v4, generated in Dart
  title              TEXT NOT NULL,        -- 1..80 chars, trimmed
  description        TEXT NULL,            -- 0..500 chars
  category           TEXT NULL,            -- QuestionCategory.wireName, nullable
  status             TEXT NOT NULL,        -- 'active' | 'completed'
  created_at_utc     INTEGER NOT NULL,
  updated_at_utc     INTEGER NOT NULL,
  completed_at_utc   INTEGER NULL,
  target_date        TEXT NULL,            -- dayKey 'YYYY-MM-DD' (same identity as day_map)
  source_question_id TEXT NULL,            -- §21
  source_option_id   TEXT NULL,
  source_day_key     TEXT NULL             -- the day the reflection was recorded
);
CREATE INDEX idx_goals_status ON goals(status);
```

Deliberate omissions from the brief's sample model: **`startDate`** (a goal starts when it is created — a separate field invites a lie), and **`progress`** / **`taskIds`** (**both are derivable** — progress is a **`COUNT`** over **`tasks`**, and task ids are a **`SELECT`** by **`goal_id`**. Storing them would create two sources of truth and a whole class of desync bugs. The brief lists them as a *minimum model*; this proposal deliberately does not store them.)
**`Goal.progress`** is therefore a **computed** value: **`completedTasks / totalTasks`** over non-deleted tasks, exposed as a getter on the model and never persisted.

### 11.2 Task

```sql
CREATE TABLE tasks(
  id                 TEXT PRIMARY KEY,
  goal_id            TEXT NULL,            -- NULL == free-floating
  title              TEXT NOT NULL,        -- 1..120 chars, trimmed
  notes              TEXT NULL,            -- 0..1000 chars
  created_at_utc     INTEGER NOT NULL,
  updated_at_utc     INTEGER NOT NULL,
  due_date           TEXT NULL,            -- 'YYYY-MM-DD' (Gregorian, same identity as day_key)
  due_time           TEXT NULL,            -- 'HH:MM' wall-clock, validated by the EXISTING
                                            -- parseReminderTime() — reuse, do not re-invent
  completed          INTEGER NOT NULL DEFAULT 0,   -- 0/1
  completed_at_utc   INTEGER NULL,
  reminder_enabled   INTEGER NOT NULL DEFAULT 0,   -- 0/1
  notification_id    INTEGER NULL          -- reserved for the plugin schedule
);
CREATE INDEX idx_tasks_goal    ON tasks(goal_id);
CREATE INDEX idx_tasks_due     ON tasks(due_date);
CREATE INDEX idx_tasks_open    ON tasks(completed, due_date);
```

**`notification_id`** is stored rather than derived because a stable integer id must survive process death for cancellation to work; deriving it from a hash of the uuid would work but makes the mapping un-inspectable when a cancel silently fails. Proposal: allocate monotonically from **`max(notification_id) + 1`** inside the same transaction that creates the task, in a reserved band (see §7/ADR-007).

### 11.3 Status semantics

- **`Goal.status`**: **`active ⇄ completed`**. No **`draft`** state — a goal is either live or done, and creating it *is* the draft. This avoids a third state with no distinct UI in a calm product.
- **`Task.completed`**: boolean + **`completed_at_utc`** (needed for "completed 3 days ago" and for undo windowing). No **`cancelled`** state in v1 — a task that will not happen is deleted, not archived.

────────────────────────────────────────────────────────── 12. PHASE 2 UX ──────────────────────────────────────────────────────────

### 12.1 Goals

```text
┌──────────────────────────────────────────┐
│  ←                 اهداف                 │
│                                          │
│  از بازتاب، به قدم                      │  Subtitle
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ تمرین منظم را جدی بگیرم      ⋯     │  │  title + overflow menu
│  │ ▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░  ۶۰٪        │  │  progress bar (RTL fill)
│  │ ۳ از ۵ کار · تا ۱۵ مهر             │  │  tasks + target date
│  │ [🏷 عادت‌ها، نظم و عمل‌گرایی]        │  │  category chip (optional)
│  └────────────────────────────────────┘  │
│                                          │
│              + هدف جدید                   │  FAB / bottom button
└──────────────────────────────────────────┘
```

| **Aspect**      | **PROPOSED**                                                                                                                                 |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Goal list       | 2-segment control at the top: «اهداف» / «کارها» (one screen, two views — no second tab)                                                      |
| Goal creation   | Modal bottom sheet, not a route. Title (required), توضیح (optional), حوزه (optional, reuses **`faLabel`**), تاریخ هدف (optional date picker) |
| Goal detail     | Header (title + progress) → task list grouped «باز» / «انجام‌شده» → «افزودن کار»                                                             |
| Goal progress   | Derived; bar + «۳ از ۵ کار»; no manual progress slider (would be a lie the data cannot support)                                              |
| Goal editing    | Same sheet as creation, pre-filled                                                                                                           |
| Goal completion | Explicit «کامل شد» toggle with confirmation; sets **`completed_at_utc`**; tasks are **not** auto-completed (the user may still owe them)     |
| Goal deletion   | Confirmation **`AlertDialog`** stating the task count and offering **both** options — see §20                                                |
| Empty state     | «هنوز هدفی نساخته‌ای.» + «از یک بازتاب شروع کن» (deep-links to Today)                                                                        |
| Active state    | **`status='active'`**, accent border-left-free (RTL-safe: no directional border)                                                             |
| Completed state | Muted to **`secondaryText`**, progress bar at 100%, collapsed task list by default                                                           |

### 12.2 Tasks

```text
┌──────────────────────────────────────────┐
│              افزودن کار                   │
│                                          │
│  عنوان                                   │
│  ┌────────────────────────────────────┐  │
│  │ چه کاری می‌خواهی انجام دهی؟         │  │
│  └────────────────────────────────────┘  │
│                                          │
│  مربوط به هدف (اختیاری)                  │
│  ┌────────────────────────────────────┐  │
│  │ بدون هدف                       ▾   │  │
│  └────────────────────────────────────┘  │
│                                          │
│  تاریخ (اختیاری)        ساعت (اختیاری)    │
│  ┌──────────────┐      ┌──────────────┐  │
│  │ ۲۷ سپتامبر   │      │  ۰۹:۳۰      │  │
│  └──────────────┘      └──────────────┘  │
│                                          │
│  ☐ یادآور قبل از موعد                   │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │              ذخیره                  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

| **Aspect**  | **PROPOSED**                                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Create      | Bottom sheet, title required, everything else optional                                                                                                                                                               |
| Edit        | Same sheet; pre-filled; if due time changes and a reminder is on, **cancel-then-replace** the schedule                                                                                                               |
| Complete    | Checkbox inline. On completion: set **`completed`**/**`completed_at_utc`**, **cancel any pending reminder**                                                                                                          |
| Undo        | **`SnackBar`** with «برگرداندم» for \~5 s, restoring the prior row (no soft-delete column needed)                                                                                                                    |
| Delete      | Confirm **`AlertDialog`** «این کار حذف شود؟» + cancel any pending reminder                                                                                                                                           |
| Due date    | Optional; stored as a **`dayKey`** in the **exact same Gregorian `YYYY-MM-DD` identity as `day_map`** (one date system for the whole app)                                                                            |
| Due time    | Optional; **`HH:MM`** wall-clock, validated by the **existing** **`parseReminderTime`**                                                                                                                              |
| Reminder    | A single **`reminder_enabled`** boolean per task. PROPOSED: fires **30 minutes before** due time, or **at 09:00 on the due day** when no time is set. One rule, no per-task offset picker (low cognitive load)       |
| Empty state | «کاری برای این هدف ثبت نشده.» + «افزودن کار»                                                                                                                                                                         |
| Overdue     | **`due_date < today && !completed`** → date in **`AppColors.warning`**, one-line «این کار از موعد گذشته»                                                                                                             |
| Completed   | Title struck through in **`secondaryText`**, checkbox checked, section collapsed                                                                                                                                     |
| Validation  | Title 1..120 after trim, empty rejected with an inline error under the field (not a snackbar); due time must match **`^\d{2}:\d{2}$`**; a due time with no due date is rejected inline («برای ساعت، تاریخ لازم است») |

### 12.3 Question → Goal

```text
Question → Answer → Result
                    ↓
              «این را به هدف تبدیل کن»   (secondary button, next to share)
                    ↓
              [ pre-filled goal sheet: title = option.action ]
```

PROPOSED rationale: the product thesis is REFLECT → UNDERSTAND → CHOOSE → ACT. The **`tiny action`** string in every **`Option`** is already an action statement, so it is a natural, honest default goal title. The affordance lives on the Result card only — it is not a modal, not a tab, and it is never required.

### 12.4 Reminder UX states

| **State**              | **PROPOSED behavior**                                                                                                                   |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Reminder OFF           | **`reminder_enabled = 0`**; no schedule exists; **`notification_id`** may be null                                                       |
| Reminder ON            | schedule created on save                                                                                                                |
| Permission not granted | Reuse the **existing** M3 pattern exactly: the setting is saved, a notice explains, no prompt loop                                      |
| Scheduled              | **`notification_id`** non-null; the task row shows «یادآور: ۳۰ دقیقه قبل»                                                               |
| Cancelled              | **`notification_id`** set to NULL after a successful plugin cancel                                                                      |
| Triggered              | The plugin owns delivery; **the app does not write a "fired" flag** (avoids a race and a redundant column). Tap → deep-link to the task |
| Timezone change        | Reuse the existing foreground **`resync…`** hook, extended to tasks (ADR-007)                                                           |
| Device restart         | **`ScheduledNotificationBootReceiver`** already handles rescheduling; the app re-registers pending schedules on next start              |

**Behaviour on user actions (PROPOSED):** disabling notifications globally → cancel all task schedules; changing a task's time → cancel + replace; deleting a task → cancel; completing a task → cancel; changing timezone → re-derive from the stored wall-clock (identical to the daily reminder's logic); restarting the device → plugin receiver + app re-registration.
────────────────────────────────────────────────────────── 13. NAVIGATION EVOLUTION ──────────────────────────────────────────────────────────
**CURRENT (OBSERVED):** **`_TabShell`** **`IndexedStack`** + **`BottomNavigationBar`**, three destinations, Persian labels, icons **`Icons.today / Icons.history / Icons.settings`**, ordered **`['امروز', 'تاریخچه', 'تنظیمات']`**. All three pages are constructed once in a **`static const`** list, so **every tab stays mounted for the app's lifetime.** No routes exist.
**PROPOSED:**

```text
CURRENT                          PROPOSED
┌──────────────┐                 ┌──────────────┐
│ امروز         │ index 0        │ امروز         │ index 0   (unchanged)
│ تاریخچه       │ index 1        │ تاریخچه       │ index 1   (unchanged)
│ تنظیمات      │ index 2        │ اهداف        │ index 2   (NEW)
│              │                 │ تنظیمات      │ index 3
└──────────────┘                 └──────────────┘
```

**Decision: add a 4th tab, do not bury Goals.** Justification, in the brief's terms:

- §25 says don't *blindly* add a nav item. This is not blind: the product is explicitly becoming a two-sided app (reflect / act), and hiding "act" behind Today + History makes Goals undiscoverable and un-deep-linkable.
- 4 items at 720dp width = 180dp each. The label «اهداف» is 5 characters and fits comfortably (the existing «تاریخچه» is 8 and already fits). **If** the label needed to be «اهداف و کارها», this would break — which is exactly why the Goals screen carries an internal اهداف/کارها segment instead.
- Goals + Tasks stay in **one** destination (segmented) to keep the bar at 4.
- Today and History must remain index 0 and 1 so the M3 notification tap → Today (**`app.dart:63-65`**) and the cold-start-on-Today behaviour are untouched.

**`IndexedStack` consequence (the risk §30 asks about):** all four tabs stay mounted, so a Goal created from a Today Result will **not** be visible in the Goals tab until that provider is invalidated. PROPOSED: **`GoalListNotifier`** / **`TaskListNotifier`** are explicit notifiers that mutate their own state on write, so no cross-tab invalidation is needed for changes made inside Goals — and a Goal created from Result calls **`ref.invalidate(goalsProvider)`**, mirroring the existing **`ref.invalidate(historyProvider); ref.invalidate(statsProvider);`** pair at **`today_providers.dart:278-279`**. This is the established pattern; the M6.0 runtime gate already flagged stale-IndexedStack state as a class of defect.
────────────────────────────────────────────────────────── 14. DATA MIGRATION ──────────────────────────────────────────────────────────
**PROPOSED: exactly one migration, v1 → v2, purely additive.**
**`databaseVersion: 1 → 2`**, with the existing stub at **`app_database.dart:98`** (**`case 1:`**) filled in as:

```dart
case 1:
  await db.execute('CREATE TABLE goals(...)');
  await db.execute('CREATE TABLE tasks(...)');
  await db.execute('CREATE INDEX ...');
  break;
```

**Why no other approach is needed:**

| **Concern**                           | **Answer**                                                                                                                                                                              |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Schema change required?               | **Yes** — two new tables. **`goals`**/**`tasks`** cannot be expressed in the kv table without destroying query performance and referential integrity.                                   |
| Migration required?                   | **Yes**, one additive step. **`onCreate`** for a *fresh* install must be updated to build goals+tasks at v2 as well (or the ladder is only exercised by upgraders — a real bug source). |
| New fields need safe defaults?        | Phase 2 introduces **no new columns on existing tables.** **`answer_records`**, **`day_map`** and **`settings`** are untouched.                                                         |
| How do existing users transition?     | Nothing to transition. The upgrade adds two empty tables; the app opens on Today exactly as before.                                                                                     |
| Does existing history stay intact?    | ✅ **`answer_records`** is not read, written, or migrated. **`resolveHistory`** and **`computeStats`** are untouched.                                                                    |
| Does streak data stay intact?         | ✅ **`StreakStats`** derives from **`answer_records`** day keys only.                                                                                                                    |
| Do answered question ids stay intact? | ✅ **`day_map`** untouched, so **`seenQuestionIds()`** and the never-repeat guarantee are bit-identical.                                                                                 |

**Hard rule: `onUpgrade` must never `DROP` or rewrite a user table.** This is already the file's stated policy (**`app_database.dart:4-5`**).
**Phase 1 requires NO migration at all** — it uses the existing **`settings`** kv table with one new key. New keys are invisible to old builds (a downgrade simply ignores an unknown key), so a v1 build remains functional against a v2 database. That asymmetry is safe in Phase 1 because Phase 1 adds **no** table or column.
**`NOTIFY`/`migrateForTest` note:** **`migrateForTest`** (**`@visibleForTesting`**) already exists specifically to test the ladder, and the current v1 has no observable step. After this migration it becomes genuinely testable — that is the test to write first (a v1 fixture DB → **`migrateForTest(1, 2)`** → assert all three original tables and their rows survive byte-for-byte).
────────────────────────────────────────────────────────── 15. BACKWARD COMPATIBILITY ──────────────────────────────────────────────────────────

| **Existing feature**                          | **Phase 1 impact**                                                                                                                           | **Phase 2 impact**                                                                                       | **Must preserve** |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------- |
| Questions (400, 8 cats)                       | **None** — bundle untouched; the 50/category validator is unaffected because it validates the bundle, not the selection                      | None                                                                                                     | **YES**           |
| Answers (**`optionId`** + optional free text) | None — option set and **`AnswerType`** unchanged                                                                                             | None                                                                                                     | **YES**           |
| Result (insight + tiny action)                | None — **`resultFor`** untouched; **`option.action`** gains a *consumer* (goal prefill), never a change                                      | Gains a "make this a goal" affordance                                                                    | **YES**           |
| History (calendar + day detail)               | None — **`answer_records`** is the single source; a category is already stored per row and is simply not displayed                           | None                                                                                                     | **YES**           |
| Streak (current/longest/total)                | **None** — **`computeStats`** works on day keys and cannot see categories                                                                    | None                                                                                                     | **YES**           |
| Statistics                                    | None                                                                                                                                         | Goals add a *new* stat; existing three untouched                                                         | **YES**           |
| Settings                                      | Reminder controls remain; nothing removed                                                                                                    | Reminder controls remain; task reminders are per-task, not a new global section                          | **YES**           |
| Theme (light/dark)                            | None — tokens unchanged unless the Vazirmatn decision (D3) is taken                                                                          | None                                                                                                     | **YES**           |
| Anti-repetition (never repeat)                | **Touched — the one intentional change.** Filtered selection, but only on the **`mapping == null`** branch; **`day_map`** is never rewritten | None                                                                                                     | **YES**           |
| Exhaustion                                    | **`QuestionsExhausted`** behaviour preserved verbatim; a *new sibling* **`CategoryExhausted`** is added                                      | None                                                                                                     | **YES**           |
| Daily rollover                                | None — the day key remains the only day identity                                                                                             | Reused for **`due_date`**                                                                                | **YES**           |
| Notifications (daily)                         | None — untouched                                                                                                                             | **Extended, not replaced**: same library, same cancel-then-replace, same rationale-first permission flow | **YES**           |
| Share Card                                    | None                                                                                                                                         | None                                                                                                     | **YES**           |
| Notification tap → Today                      | None                                                                                                                                         | Gains task payload routing; **`'today'`** payload behaviour unchanged                                    | **YES**           |
| Legacy category aliases                       | None — needed only for M1–M4 rows and kept for that                                                                                          | None                                                                                                     | **YES**           |
| **`databaseVersion`**                         | **Unchanged (1)**                                                                                                                            | 1 → 2, additive                                                                                          | —                 |

────────────────────────────────────────────────────────── 16. REGRESSION RISKS ──────────────────────────────────────────────────────────

| **#**   | **Risk**                                                                 | **Cause**                                                                                                | **Impact**                                                             | **Mitigation**                                                                                                                                                                                                                    |
| ------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **R1**  | **Same-day category switch re-consumes or duplicates a question**        | **`day_map`** PK is **`day_key`**; a naive switch must delete/overwrite the row                          | **HIGH** — breaks the M5 product promise                               | ADR-003: filter is consulted **only** when **`mapping == null`**; the day's question is immutable; sheet is read-only once a mapping exists. Add a test asserting the **`day_map`** row is byte-identical after a category change |
| **R2**  | Category counts go stale at midnight                                     | A **`FutureProvider`** for remaining counts that doesn't watch **`todayDayKeyProvider`**                 | MEDIUM — wrong "۴۴ باقی‌مانده" on the first screen of the day          | **`categoryCountsProvider`** must **`ref.watch(todayDayKeyProvider)`**, exactly as **`historyProvider`**/**`statsProvider`** already do                                                                                           |
| **R3**  | Stale **`IndexedStack`** after creating a goal from a Result             | All tabs stay mounted for the app lifetime                                                               | MEDIUM — user believes the goal wasn't saved                           | Mirror **`today_providers.dart:278-279`**: **`ref.invalidate(goalsProvider)`** on creation; Goals notifiers own their own state                                                                                                   |
| **R4**  | Notification id collision between the daily reminder (1701) and tasks    | Both use the same plugin instance                                                                        | MEDIUM — cancelling a task reminder kills the daily one, or vice versa | Reserved id bands (daily 1700–1799; tasks 2000+, allocated transactionally). Assert the band in a test                                                                                                                            |
| **R5**  | Task reminders survive a timezone change incorrectly                     | The existing code uses a **fixed offset**, not a real tz database (a documented concession for **`fa`**) | LOW–MEDIUM — a reminder could sit an hour off                          | Reuse the same **`_applyDeviceTimezone`** + foreground resync semantics; do not introduce a second tz path. If Persian DST is ever a concern, that is a separate decision                                                         |
| **R6**  | Duplicate task notifications after reboot/app start                      | Plugin receiver reschedules, and the app may re-register too                                             | MEDIUM — the user gets N reminders                                     | Cancel-then-schedule unconditionally, exactly as **`syncReminder`** does today; store **`notification_id`** so cancel is precise                                                                                                  |
| **R7**  | Goals/tasks visible but History/streak corrupted by the migration        | An **`onUpgrade`** that rebuilds a user table                                                            | **HIGH** — irreversible data loss                                      | **`_createV2`** must only **`CREATE`**. A migration test loads a real v1 fixture, migrates, and asserts all original rows survive                                                                                                 |
| **R8**  | RTL break in new UI                                                      | Hard-coded **`left`**/**`right`**, **`EdgeInsets.only`**, or Latin digits                                | MEDIUM — visibly wrong in an RTL-first product                         | **`EdgeInsetsDirectional`** + **`faDigits`** only; add a widget test asserting **`Directionality`** is RTL and numerals are Persian                                                                                               |
| **R9**  | UI modernization regresses Persian rendering                             | Switching the UI to bundled Vazirmatn changes metrics/line-breaking everywhere                           | MEDIUM — overflow, or a different look from the share card             | Make the font change its own isolated commit with a full visual pass; it is currently a *candidate*, not a requirement                                                                                                            |
| **R10** | **`validateLibrary`** blocks a future unequal-category library           | It hard-requires exactly 50 per category                                                                 | LOW (today)                                                            | Documented; only matters if content policy changes. Phase 1 filtering does **not** affect it                                                                                                                                      |
| **R11** | Install-time data loss makes the anti-repetition promise untestable      | Uninstall clears **`day_map`**                                                                           | LOW — existing, documented, out of scope                               | Note it; a backup/export feature is a separate milestone, not M6                                                                                                                                                                  |
| **R12** | Over-engineering Phase 2 into a generic productivity app                 | Brief explicitly warns against this                                                                      | MEDIUM — product identity loss                                         | §34 principles as review gates; no streaks/badges/points; tasks have no priority, no subtasks, no recurrence, no tags in v1                                                                                                       |
| **R13** | A new **`settings`** key collides with the dead **`enabled_categories`** | Two similar keys, different semantics                                                                    | LOW–MEDIUM — future confusion                                          | New key is **`selected_category`**; the dead constant is left alone and explicitly not co-opted (§10.3)                                                                                                                           |
| **R14** | Due-date calendar confusion                                              | No Jalali anywhere; the app renders Persian **Gregorian** month names                                    | MEDIUM for a Persian audience                                          | Decision D2 (§22). Storage stays **`YYYY-MM-DD`** regardless — only *display* is at stake                                                                                                                                         |

────────────────────────────────────────────────────────── 17. TEST STRATEGY ──────────────────────────────────────────────────────────
**Baseline to protect: 213 tests / 21 files, all currently green.** The M0–M5 contract is that no existing test is deleted, skipped, or weakened. Net-new only.
**PROPOSED Phase 1 additions:**

| **Area**                 | **Test**                                                                                       | **Kind**                         |
| ------------------------ | ---------------------------------------------------------------------------------------------- | -------------------------------- |
| Category selection       | sheet renders 8 cards + "all"; **`remaining`** counts match the ledger                         | widget                           |
| Category selection       | selected state is visually distinct and announced (**`Semantics(selected: true)`**)            | widget                           |
| Category persistence     | **`selected_category`** survives a real DB close/reopen                                        | integration (real SQLite)        |
| Category persistence     | choosing «همهٔ حوزه‌ها» clears the key; the default is null                                    | unit                             |
| Category switching       | changing focus **after** a mapping exists leaves **`day_map`** byte-identical                  | integration — **R1 guard**       |
| Category switching       | changing focus before a mapping exists changes the day's question                              | integration                      |
| Category exhaustion      | a 50/50-seen category is DISABLED with «تمام شد»                                               | widget                           |
| Category exhaustion      | **`selectUnseenQuestion(categoryFilter: exhaustedCat)`** returns **`CategoryExhausted`**       | unit                             |
| Category exhaustion      | **`CategoryExhausted`** writes **no** **`day_map`** row and consumes nothing                   | integration — **R1 guard**       |
| **Anti-repetition**      | 400 days with a filter set still yield 400 unique ids                                          | integration                      |
| **Anti-repetition**      | filtering never re-offers a question from another category                                     | integration                      |
| **Back-compat**          | every existing **`selectUnseenQuestion`** call (no filter) is unchanged                        | the existing 15 tests, untouched |
| Question selection       | default (null filter) reproduces the pinned golden vector **`2026-09-23 → relationships_004`** | unit                             |
| App restart / force-stop | focus + **`day_map`** + answer all survive; no duplicate entry                                 | integration                      |
| History                  | unaffected by focus; today's entry still appears once                                          | widget                           |
| Day rollover             | focus persists; counts advance; a new question is selected                                     | integration                      |
| RTL                      | new widgets render under **`Directionality.rtl`**; counts use Persian digits                   | widget                           |
| Visual regression        | goldens of the sheet in light + dark, RTL                                                      | golden                           |
| Content                  | **`tool/build_content.dart --check`** still clean; asset unmodified                            | existing                         |

**PROPOSED Phase 2 additions:**

| **Area**                  | **Tests**                                                                                                              |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Goal creation             | validation (title trim/length/empty), persistence, defaults                                                            |
| Goal lifecycle            | complete → reopen; **`completed_at_utc`** set/cleared; reopening keeps tasks                                           |
| Goal deletion             | with 0, 1, and 7 tasks; both confirmation outcomes; no orphans                                                         |
| Task creation             | standalone (**`goal_id`** NULL) and attached; due date/time validation                                                 |
| Task persistence          | survives close/reopen; **`updated_at_utc`** changes                                                                    |
| Task completion           | toggles both columns; undo restores exactly                                                                            |
| Task deletion             | confirm; cancels a pending schedule                                                                                    |
| Goal progress             | derived correctly for 0/partial/all tasks; **never stored**                                                            |
| Reminder scheduling       | schedule on enable; cancel on disable/delete/complete; cancel-then-replace ordering                                    |
| Notification permission   | granted / denied / pre-33 — reusing the existing **`FakeReminderService`** pattern                                     |
| Notification delivery     | **on-device only** (M6.0 runtime gate)                                                                                 |
| Notification cancellation | pending list contains no task ids after completion                                                                     |
| Timezone                  | foreground resync re-derives wall-clock; the existing **`notifications_test.dart`** conventions extend to tasks        |
| Device reboot             | boot receiver present in the manifest; app re-registers on next start                                                  |
| **Migration**             | real v1 fixture → v2: all three original tables and every row intact; new tables empty; **`migrateForTest`** exercised |
| Existing data             | a user with N days of history upgrades and sees byte-identical History, streak and question                            |
| Anti-repetition           | seen set identical pre/post migration                                                                                  |

**PROPOSED on-device tests:** **`integration_test/`** **does not exist today.** Creating it requires adding **`integration_test: sdk: flutter`** to **`dev_dependencies`** — a **`pubspec.yaml`** change, which needs explicit approval (§33). Tests to add there: cold launch, answer→result, force-stop→reopen, day rollover, share sheet, notification delivery, task reminder delivery. Until that directory exists, **no automated test can prove a notification was delivered** — that gap must be stated in every M6 release report rather than assumed away.
────────────────────────────────────────────────────────── 18. ADRs ──────────────────────────────────────────────────────────
**ADR-001 — Category Selection Architecture** CONTEXT: users must choose the category they want a question from. OBSERVED: 3-tab **`IndexedStack`**, no routes, no router; **`QuestionCard`** is the sole Today affordance; **`faLabel`** already exists on the enum. OPTIONS: (a) inline list on Today; (b) modal bottom sheet from a Today chip; (c) new nav destination; (d) Settings toggle. TRADE-OFFS: (a) adds 8 cards to every answer — worst for calm focus. (c) turns a transient per-day choice into a permanent destination. (d) is 3+ taps on a low-traffic screen and the least discoverable. (b) is 1 tap, discoverable (because the chip displays the current state), needs no nav change, and scales to a 9th category in a **`ListView`**. **PROPOSED DECISION: (b) — modal bottom sheet from a persistent chip on Today.** CONSEQUENCES: one new screen-like widget + one chip; no nav change; the day's question becomes immutable (ADR-003); the sheet needs a read-only mode.
**ADR-002 — Category Persistence** CONTEXT: which category the user wants must survive restarts. OBSERVED: **`settings`** kv table with **`getSetting`**/**`setSetting`** and typed accessors; **`ReminderSettingsNotifier`** is the precedent (write DB, then re-sync the side effect); **`SettingKeys.enabledCategories`** exists but is **unreferenced**. OPTIONS: (a) reuse **`enabled_categories`**; (b) new **`selected_category`** key; (c) no persistence (per-session only). TRADE-OFFS: (a) reuses a constant but its name/comment describe a *multi-select set*; two similarly named keys with different semantics is a latent bug. (c) makes the app forget the user's daily intent every launch — the feature loses its point. (b) is one row, no migration, invisible to older builds, and matches the existing M3 pattern exactly. **PROPOSED DECISION: (b) — `settings['selected_category']`, single `wireName`, `null` = all categories.** A typed **`getSelectedCategory()`**/**`setSelectedCategory()`** pair is added beside the notification accessors. The dead constant is left untouched and explicitly not co-opted. CONSEQUENCES: no migration; new key ignored by a v1 build; the count UI must treat **`null`** as "all" rather than as an error.
**ADR-003 — Anti-Repetition Integration** *(the critical one)* CONTEXT: category filtering must not weaken the M5 never-repeat guarantee. OBSERVED: **`day_map(day_key PK, question_id)`** **is** the seen-ledger; a question is marked seen by the act of persisting the day that shows it; the day's question is frozen on first view and re-read thereafter. OPTIONS: (a) filter only when creating a new day's question (immutable day); (b) allow same-day switching by rewriting the day's row; (c) keep two question/day (one per category). TRADE-OFFS: (b) requires deleting/overwriting **`day_map`** → re-consumes a question, can re-present a question the user already saw that day, and inflates the seen set (silently eroding the 400-question promise). (c) multiplies the consumption rate and breaks the "one reflection per day" product promise that Streak is built on. (a) costs the user the ability to change their mind mid-day — a real but small cost, which the sheet's copy states up front. **PROPOSED DECISION: (a) — the day's question is immutable once created. The category filter is passed to `selectUnseenQuestion` only on the `mapping == null` branch. Once a mapping exists the sheet is read-only with «سؤال امروز مشخص شده. از فردا می‌توانی حوزه را عوض کنی.»** CONSEQUENCES: one new **`SelectionOutcome`** member (**`CategoryExhausted`**), which the compiler forces **`_resolve`**'s switch to handle; the filter parameter defaults to **`null`** so all 15 existing anti-repetition tests pass unchanged; a new integration test asserts **`day_map`** is untouched by a category change.
**ADR-004 — Goal Domain Model** CONTEXT: the app needs goals. OBSERVED: no goal concept exists; **`day_key`** is the only date identity; ids elsewhere are authored strings; the codebase rejects unnecessary indirection. OPTIONS: (a) store derived fields (**`progress`**, **`taskIds`**); (b) derive them. TRADE-OFFS: (a) gives faster reads but creates two sources of truth and a class of desync bugs; **`startDate`** is also a field a user can lie to. (b) costs one **`COUNT`** query. **PROPOSED DECISION: (b) — persist identity, status, copy and timestamps only. `progress` and `taskIds` are computed. No `startDate`. Status is `active | completed` (no `draft`).** Provenance columns are nullable and add-only (ADR-009). CONSEQUENCES: no denormalisation bugs; **`Goal`** is a thin value object.
**ADR-005 — Task Domain Model** CONTEXT: the app needs tasks with due dates/times and reminders. OBSERVED: the app already validates wall-clock **`HH:MM`** in **`parseReminderTime`**; the day identity is a Gregorian **`YYYY-MM-DD`** string; there is no Jalali anywhere. OPTIONS: store due date as a **`DateTime`**; store a day key + optional time string. TRADE-OFFS: a **`DateTime`** introduces a timezone/precision trap and a second date identity. Two fields reuse two existing, already-validated representations. \*\*PROPOSED DECISION: **`due_date TEXT`** (**`YYYY-MM-DD`**, same identity as **`day_key`**)

- **`due_time TEXT`** (**`HH:MM`**, validated by the existing **`parseReminderTime`**). **`completed`** is an INTEGER 0/1 with **`completed_at_utc`**. **`notification_id`** is stored so cancellation is exact after process death.\*\* CONSEQUENCES: no new date library, no new validation logic, one identity for the whole app. Display of Persian dates remains a display concern (D2).

**ADR-006 — Goal↔Task Relationship** CONTEXT: tasks and goals need an explicit relationship. OPTIONS: (a) strict 1-to-many, **`goal_id NOT NULL`**; (b) optional **`goal_id`**; (c) many-to-many join table. TRADE-OFFS: (a) forces a goal before any task — the highest-friction possible path, directly against §34's low-cognitive-load principle. (c) is a join table, a second model, and extra UI, for a relationship no user has asked for. **PROPOSED DECISION: (b) — one Goal → many Tasks; `tasks.goal_id` is NULLABLE. A Task may exist without a Goal; a Goal may exist with zero Tasks. A Task belongs to at most ONE Goal (no join table — YAGNI per §41).** Deletion semantics: **deleting a Goal re-parents its Tasks to `goal_id = NULL`** (zero data loss, calm, no cascade dialog) and the confirmation dialog states the count. *Alternative considered and rejected: cascade delete, which is more predictable but destroys work the user may still want.* Deleting a Task never touches its Goal. Completing a Goal does **not** auto-complete its Tasks. A completed Goal **can be reopened** (the one direction reversal the product allows), which restores **`status='active'`** and clears **`completed_at_utc`**. CONSEQUENCES: two orphan states are *normal*, not errors; every list query must handle **`goal_id IS NULL`** explicitly.
**ADR-007 — Reminder Architecture** CONTEXT: tasks need reminders; the app already has a reminder system. OBSERVED: **`flutter_local_notifications`** 22.3.1, one channel, one id (1701), abstract **`ReminderService`** + fake, cancel-then-replace, inexact scheduling, **`fixedOffsetLocation`**, boot receiver in the manifest, rationale-first permission. OPTIONS: (a) extend **`ReminderService`**; (b) a second notification library; (c) **`AlarmManager`**/exact alarms. TRADE-OFFS: (b) duplicates scheduling, permission, and boot logic and would need a second manifest receiver set. (c) needs **`SCHEDULE_EXACT_ALARM`** and a **`USE_EXACT_ALARM`** justification, contradicting the deliberate M3 decision that a reminder is not an alarm. **PROPOSED DECISION: (a) — extend the existing service. Add a second channel (`harf_tasks`), allocate task ids in a reserved band (daily 1700–1799, tasks 2000+, transactionally), store `notification_id` on the task, keep `inexactAllowWhileIdle`, and generalize the resync hook into `resyncNotificationsFromStorage()` (which re-syncs both the daily reminder and all task schedules) called from the same two places it is called today (startup + foreground). One reminder rule: 30 minutes before due time, or 09:00 on the due day when no time is set.** CONSEQUENCES: no new dependency, no new manifest receiver, no exact-alarm permission; the abstract service keeps widget tests I/O-free; a new id-allocation invariant must be tested (R4).
**ADR-008 — Notification Architecture** *(architecture-level, distinct from ADR-007's mechanism)* CONTEXT: Phase 2 adds a second kind of notification to a single-channel app. OBSERVED: one channel created in **`_prepare()`**; **`requestPermission()`** is idempotent and already Android-13-aware; the rationale dialog lives in Settings and is reminder-specific. OPTIONS: (a) one channel for everything; (b) a channel per concern, created lazily; (c) per-task channels. TRADE-OFFS: (a) makes task reminders indistinguishable from the daily reflection in system settings and prevents per-concern muting. (c) is a user-hostile channel explosion. (b) gives two durable channels the user can mute independently, and the app already knows how to create one. **PROPOSED DECISION: (b) — exactly two channels, both created during `init()`: `harf_daily` (unchanged, «یادآور روزانه») and `harf_tasks` («کارها»). Permission is requested once, at the first moment either feature needs it, and the rationale copy is generalized to cover both.** CONSEQUENCES: no new permission, no new receiver; a small copy change in the existing rationale dialog (must be reviewed, not silently reworded).
**ADR-009 — Question-to-Goal Relationship** CONTEXT: should a reflection be able to become a goal? OBSERVED: **`Option.action`** is already an authored action statement; **`ResultCard`** is shared by Today and History; **`AnswerRecord`** already stores **`question_id`**, **`option_id`** and **`day_key`**. OPTIONS: (a) no link; (b) full copy of the reflection into the goal; (c) three nullable provenance columns. TRADE-OFFS: (a) loses the product's core narrative (reflect → act). (b) duplicates content that already lives in **`answer_records`** and could drift out of sync. (c) is minimal, additive, nullable, and lets History render provenance without duplicating anything. **PROPOSED DECISION: (c) — `goals.source_question_id`, `source_option_id`, `source_day_key`, all NULLABLE. The "make this a goal" affordance is a secondary button on the Result card only, pre-filling the title from `option.action`. No source text is copied; provenance is rendered by reference via the existing `bundle.byId`, degrading to a neutral line if the content ever drifts (exactly like `UnavailableEntry`).** CONSEQUENCES: three nullable columns, zero migration risk to existing rows; the relationship can be omitted without affecting the goal's validity, so it must never be required.
**ADR-010 — Navigation Evolution** CONTEXT: Goals + Tasks need a home. OBSERVED: 3 tabs in a permanently-mounted **`IndexedStack`**; no routes; the notification tap hard-codes index 0. OPTIONS: (a) 4th tab with an internal Goals/Tasks segment; (b) a 5th tab; (c) no tab, enter from Result/History only. TRADE-OFFS: (b) over-crowds a 4-item bar and needs two new labels. (c) makes existing goals undiscoverable — you could only reach a goal through the reflection that created it. (a) keeps the bar at 4, keeps «اهداف» and «کارها» as views of one destination, and fits 4 items at 720dp. **PROPOSED DECISION: (a) — 4 tabs: امروز (0) / تاریخچه (1) / اهداف (2) / تنظیمات (3). Today stays index 0 so notification-tap routing is unchanged. Goals and Tasks share tab 2 behind a 2-segment control.** CONSEQUENCES: one new screen pair; **`IndexedStack`** staleness handled by explicit invalidation (R3); no new nav dependency, no routes introduced.
**ADR-011 — Persistence / Migration Strategy** CONTEXT: users have real data; the app must not lose it. OBSERVED: **`databaseVersion = 1`**; three tables; an **`onUpgrade`** ladder with a ready **`case 1:`** stub; **`migrateForTest`** exposed for the ladder; the file's stated policy is "never DROP user tables in upgrades"; M5 deliberately needed **no** migration at all. OPTIONS: (a) kv-table-only storage for goals/tasks; (b) new tables with an additive v1→v2 migration; (c) rewrite history. TRADE-OFFS: (a) is implementable but makes list queries, progress counting and cascade semantics unworkable in SQL and would put logic in Dart string manipulation. (c) is unacceptable. (b) is one additive step, touches no existing table, and is testable with the existing test seam. **PROPOSED DECISION: (b) — Phase 1 requires NO migration (one new `settings` key). Phase 2 does exactly one additive v1→v2 migration creating `goals`, `tasks` and three indices. `onCreate` must build the v2 schema for fresh installs. The FIRST Phase 2 test is a real-v1-fixture → v2 migration test asserting all three original tables and every row survive.** No user table is ever dropped, rewritten, or re-typed. CONSEQUENCES: zero data-loss risk if the migration test passes; the migration ladder becomes genuinely exercised for the first time, which is a small win on its own.
────────────────────────────────────────────────────────── 19. FILE-LEVEL IMPLEMENTATION MAP ──────────────────────────────────────────────────────────
**Phase 1 — Category Selection**

| **File**                                                    | **Current role**                                                                      | **Proposed change**                                                                                                                                                                                  | **Reason**                                                                         | **Risk** | **Depends on**                                                                    |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------- | --------------------------------------------------------------------------------- |
| **`lib/features/today/category_providers.dart`**            | **does not exist**                                                                    | **NEW** — **`categoryFocusProvider`** (AsyncNotifier), **`categoryCountsProvider`** (FutureProvider watching the day key)                                                                            | own the focus + derived counts                                                     | R2       | **`appDatabaseProvider`**, **`contentBundleProvider`**, **`todayDayKeyProvider`** |
| **`lib/features/today/category_sheet.dart`**                | **does not exist**                                                                    | **NEW** — **`showCategorySheet()`**, **`_CategoryCard`**, states DEFAULT/SELECTED/DISABLED/LOADING/ERROR                                                                                             | the 8-card bottom sheet                                                            | R8       | **`category_providers`**, tokens, **`faDigits`**                                  |
| **`lib/data/app_database.dart`**                            | SQLite, v1, 3 tables, typed accessors                                                 | **ADD** **`getSelectedCategory()`** / **`setSelectedCategory()`** beside the notification accessors (**`:239-253`**)                                                                                 | reuse the existing kv pattern; no schema change                                    | low      | **`models.dart`**                                                                 |
| **`lib/data/models.dart`**                                  | **`QuestionCategory`** (8 + aliases), models, **`SettingKeys`**                       | **ADD** **`SettingKeys.selectedCategory = 'selected_category'`** next to the existing keys (**`:295-300`**)                                                                                          | key constant lives with its siblings                                               | R13      | —                                                                                 |
| **`lib/features/today/selection.dart`**                     | **`selectUnseenQuestion`** + sealed **`SelectionOutcome`**                            | **ADD** optional **`QuestionCategory? categoryFilter = null`**; **ADD** **`CategoryExhausted`** outcome; filter **`unseen`** **after** the **`unseen.isEmpty`** check, **before** exposure balancing | filter the pool; the compiler then forces **`_resolve`** to handle the new outcome | R1       | **`models.dart`**, **`content_bundle`**                                           |
| **`lib/features/today/today_providers.dart`**               | **`TodayNotifier._resolve`** (**`:160-231`**), sealed **`TodayState`**                | **ADD** **`TodayCategoryExhausted`** state; read the focus provider; pass the filter **only inside the `mapping == null` branch**; handle **`CategoryExhausted`**                                    | the single behavioural change                                                      | **R1**   | **`selection.dart`**, **`category_providers`**                                    |
| **`lib/features/today/today_screen.dart`**                  | renders the 6 Today states; owns **`_freeText`**, day-rollover resume hook            | **ADD** the category chip above **`QuestionCard`**; **ADD** a render branch for **`TodayCategoryExhausted`**; **ADD** the read-only sheet mode                                                       | surface the feature; new state needs a view                                        | R2       | **`category_sheet`**, **`category_providers`**                                    |
| **`lib/app/theme/app_theme.dart`**                          | **`ThemeData`** from tokens, light + dark                                             | **OPTIONAL** **`fontFamily: 'Vazirmatn'`** (decision D3) — isolated, separate commit                                                                                                                 | the UI does not currently use the bundled font                                     | **R9**   | **`typography.dart`**                                                             |
| **`lib/features/today/today_providers.dart`** (submit path) | **`ref.invalidate(historyProvider); ref.invalidate(statsProvider)`** (**`:278-279`**) | **NO CHANGE**                                                                                                                                                                                        | History/streak are category-agnostic                                               | —        | —                                                                                 |

**Phase 1 — explicitly UNTOUCHED**

| **File**                                                                                              | **Why it must not change**                                              |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **`lib/data/app_database.dart`** (schema/**`_createV1`**/**`_upgrade`**)                              | Phase 1 needs no migration; **`databaseVersion`** stays 1               |
| **`lib/data/content_bundle.dart`**                                                                    | the bundle and its 400/50 contract are orthogonal to filtering          |
| **`lib/data/content_audit.dart`**, **`tool/**`**, **`content/questions/**`**, **`assets/content/**`** | content is frozen; **`build_content.dart --check`** must stay clean     |
| **`lib/features/history/**`**                                                                         | derived from **`answer_records`**; a category is already stored per row |
| **`lib/features/share/**`**                                                                           | unaffected                                                              |
| **`lib/features/settings/**`**                                                                        | reminder-only; category choice is a Today concern (ADR-001)             |
| **`lib/app/app.dart`**                                                                                | the tab shell is not modified in Phase 1                                |
| **`android/**`**, **`pubspec.yaml`**                                                                  | no manifest, permission, or dependency change                           |

**Phase 2 — Goals + Tasks**

| **File**                                             | **Current role**                                        | **Proposed change**                                                                                                                                             | **Reason**                                                   | **Risk**   | **Depends on**                                        |
| ---------------------------------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | ---------- | ----------------------------------------------------- |
| **`lib/features/goals/models.dart`**                 | **does not exist**                                      | **NEW** — **`Goal`**, **`Task`**, **`GoalStatus`**, computed **`progress`**                                                                                     | domain                                                       | low        | **`core/date_keys`**                                  |
| **`lib/features/goals/goals_repository.dart`**       | **does not exist**                                      | **NEW** — all **`goals`**/**`tasks`** SQL (CRUD, progress count, due queries)                                                                                   | one place for the two new tables' SQL                        | low        | **`appDatabaseProvider`**                             |
| **`lib/features/goals/goals_providers.dart`**        | **does not exist**                                      | **NEW** — **`goalsProvider`**, **`goalDetailProvider`**, **`tasksProvider`**, **`GoalListNotifier`**, **`TaskListNotifier`**                                    | mirrors the M3 notifier pattern (DB first, then side effect) | R3         | repository                                            |
| **`lib/features/goals/goals_screen.dart`**           | **does not exist**                                      | **NEW** — goal list + «کارها» segment                                                                                                                           | nav destination 2                                            | R3         | providers                                             |
| **`lib/features/goals/goal_detail_screen.dart`**     | **does not exist**                                      | **NEW** — header, progress, task list                                                                                                                           | —                                                            | R3         | providers                                             |
| **`lib/features/goals/task_sheet.dart`**             | **does not exist**                                      | **NEW** — create/edit form with date+time+reminder                                                                                                              | the single write form                                        | low        | providers, **`notifications.dart`** (time validation) |
| **`lib/features/settings/reminder_service.dart`**    | abstract + local + fake, daily only                     | **EXTEND** the interface: task channel, **`scheduleTask`**/**`cancelTask`**, **`syncTasks`**; keep **`FakeReminderService`** in step                            | extend, don't replace                                        | **R4**, R6 | **`notifications.dart`**                              |
| **`lib/features/settings/notifications.dart`**       | pure policy for the daily reminder                      | **ADD** **`taskReminderChannelId/Name`**, task copy, id-band constants, **`planTaskReminder`**; **REUSE** **`parseReminderTime`**                               | policy stays pure + testable                                 | R4         | —                                                     |
| **`lib/features/settings/retention_providers.dart`** | **`resyncReminderFromStorage`** for the daily reminder  | **RENAME/EXTEND** to **`resyncNotificationsFromStorage`** covering daily **and** task schedules                                                                 | boot/tz recovery must cover both                             | R5, R6     | **`appDatabaseProvider`**                             |
| **`lib/features/today/widgets/result_card.dart`**    | insight + action + share footer                         | **ADD** an optional secondary «این را به هدف تبدیل کن» button                                                                                                   | the reflect→act bridge                                       | R3         | goals providers                                       |
| **`lib/data/app_database.dart`**                     | v1, 3 tables                                            | **`databaseVersion 1→2`**; fill the **`case 1:`** stub with **`CREATE TABLE goals/tasks`** + 3 indices; **update `_createV1` → `_createV2` for fresh installs** | additive migration                                           | **R7**     | **`models.dart`**                                     |
| **`lib/app/app.dart`**                               | 3-tab **`IndexedStack`**                                | **ADD** tab 2 «اهداف»; shift تنظیمات to 3; add a **`GoalsScreen`** page                                                                                         | ADR-010                                                      | R3         | goals screen                                          |
| **`lib/main.dart`**                                  | bootstrap + overrides                                   | **NO CHANGE** (new providers need no override — they read **`appDatabaseProvider`**)                                                                            | —                                                            | —          | —                                                     |
| **`android/app/src/main/AndroidManifest.xml`**       | POST_NOTIFICATIONS, RECEIVE_BOOT_COMPLETED, 2 receivers | **NO CHANGE** — a second channel needs no manifest entry; **`flutter_local_notifications`** receivers are already declared                                      | avoid permission creep                                       | —          | —                                                     |
| **`pubspec.yaml`**                                   | 4 runtime deps                                          | **NO CHANGE** unless D1 (Jalali) is approved                                                                                                                    | dependency policy                                            | —          | —                                                     |

────────────────────────────────────────────────────────── 20. IMPLEMENTATION ORDER ──────────────────────────────────────────────────────────
Adjusted from the brief's template only where discovery demanded it.

```text
STEP 1   Architecture preparation
         ADR sign-off on 001/002/003. Add `SettingKeys.selectedCategory` +
         the two typed accessors. NO provider, NO UI. Ship as its own commit
         so the persistence seam is provable before any behaviour moves.
         Why first: ADR-003 is the whole risk profile; getting the seam in
         place and tested alone makes the rest additive.

STEP 2   Design-system / UI foundations
         Resolve D3 (Vazirmatn) as an ISOLATED commit with a full visual pass.
         Optionally extract the repeated "option card" box decoration from
         `_OptionTile` and `ResultCard` into one tokenised widget — the
         selected-state idiom (2dp accent border + accent @12% fill) is
         already duplicated and the new sheet must match it exactly.
         Why here: the sheet must not invent a second look.

STEP 3   Phase 1 — category selection, read-only half
         `category_providers.dart` (focus + counts) + `category_sheet.dart`
         rendering only. No selection is applied yet. Proves the counts against
         the real ledger and the persistence round-trip.
         Why split: the read path and the write path have different risk.

STEP 4   Question engine integration  ← THE CRITICAL STEP
         `selection.dart`: optional `categoryFilter` + `CategoryExhausted`.
         `today_providers.dart`: apply the filter **only** when
         `mapping == null`; add `TodayCategoryExhausted`.
         Commit 4a = selection.dart alone (all 15 existing tests must stay
         green untouched). Commit 4b = today_providers.dart. Commit
         4c = the new R1 guard tests. The day's question stays immutable.
         Why split: this is the only step that can break the M5 promise.

STEP 5   Phase 1 persistence + UI wiring
         Chip on Today, exhausted view, read-only sheet mode, CTA,
         `categoryCountsProvider` watching the day key.

STEP 6   Phase 1 regression testing
         Full suite + the Phase 1 matrix (§17). `flutter analyze` clean.
         `dart run tool/build_content.dart --check` still clean (the asset
         must be byte-unchanged). Debug + release builds. THEN the M6.0
         runtime gate on hardware: category switch, exhaustion, anti-repetition
         across a real day rollover, RTL, History/Streak unaffected.

STEP 7   Goal domain  — models + repository only. No UI, no providers.
STEP 8   Task domain  — same.
STEP 9   Goal/Task persistence — v1→v2 migration, `_createV2` for fresh
         installs, and the migration test FIRST (R7) before any feature code
         depends on the tables.
STEP 10  Goal/Task UI — Goals tab (list + detail + sheet + task sheet).
         Invalidate `goalsProvider` from the Result action.
STEP 11 Reminder infrastructure — extend `ReminderService`, add the
         `harf_tasks` channel, id bands, `syncTasks`, generalize the resync
         hook. Fake service updated in the same commit.
STEP 12 Android notifications — on-device verification only: delivery,
         icon, tap routing, cancellation, reboot reschedule, timezone change.
STEP 13 Question → Goal integration — the secondary Result affordance,
         pre-filled from `option.action`, provenance columns populated.
STEP 14 Full regression — 213 baseline tests + all new tests; analyze;
         builds; content gate.
STEP 15 Runtime verification — full M6.0 A–L gate on hardware, including the
         400-day anti-repetition and exhaustion checks in their deterministic
         (repository) form, with physical limits stated honestly.
```

────────────────────────────────────────────────────────── 21. IMPLEMENTATION CONTRACT ──────────────────────────────────────────────────────────
**MUST CHANGE**

- **`lib/features/today/selection.dart`** — optional **`categoryFilter`** (default **`null`**); new **`CategoryExhausted`** outcome.
- **`lib/features/today/today_providers.dart`** — filter applied only when **`mapping == null`**; new **`TodayCategoryExhausted`** state; handle the new outcome.
- **`lib/data/app_database.dart`** — **`getSelectedCategory()`** / **`setSelectedCategory()`** (Phase 1). **`databaseVersion 1→2`** + **`CREATE goals, tasks, 3 indices`** + **`_createV2`** for fresh installs (Phase 2 only).
- **`lib/data/models.dart`** — **`SettingKeys.selectedCategory`** (Phase 1); **`databaseVersion`** stays 1 in Phase 1.
- **`lib/features/today/category_providers.dart`** — **NEW**.
- **`lib/features/today/category_sheet.dart`** — **NEW**.
- **`lib/features/today/today_screen.dart`** — category chip + new state view.
- Phase 2: **`lib/features/goals/{models,goals_repository,goals_providers, goals_screen,goal_detail_screen,task_sheet}.dart`** — **NEW**.
- Phase 2: **`lib/features/settings/{reminder_service,notifications, retention_providers}.dart`** — extended, never replaced.
- Phase 2: **`lib/app/app.dart`** — one added tab.
- Phase 2: **`lib/features/today/widgets/result_card.dart`** — optional secondary affordance only.

**MUST NOT CHANGE**

- The M5 never-repeat guarantee; the day's **`day_map`** row must never be deleted or rewritten after creation.
- **`databaseVersion`** during Phase 1 (stays **1**).
- **`lib/data/content_bundle.dart`**, **`content_audit.dart`**, **`content/questions/**`**, **`assets/content/**`**, **`tool/**`** — content is frozen.
- The 400/50 library contract and the near-duplicate hard threshold.
- **`resultFor()`** / **`Option`** semantics — results stay authored, never computed.
- **`computeStats()`** and the three Streak values.
- **`resolveHistory()`** and the **`UnavailableEntry`** degradation.
- **`AnswerRecord`**'s columns and **`DayMapping`**'s shape.
- The 6 prototype question ids and the 5 legacy category aliases.
- The daily reminder's id, channel, copy, inexact schedule mode, and rationale-first permission flow.
- **`MainActivity`**, the manifest receivers/permissions, **`minSdk`**/**`targetSdk`**.
- Any existing test: no deletion, no skipping, no weakening.
- The build environment constraints (no NDK, no CMake 3.22.1, no android-35; **`path_provider_android`** pinned to 2.2.23).

**MAY CHANGE** *(safe, optional, each its own commit)*

- **`lib/app/theme/app_theme.dart`** — adopt the bundled **`Vazirmatn`** (D3).
- **`lib/features/today/typography`** — re-type the sheet's text styles onto existing tokens.
- Extract the duplicated selected-card box decoration into one shared widget.
- **`lib/features/history/history_screen.dart:230`** — the **`shareText: ''`** stub in the History **`TodayResult`** (a genuine pre-existing inconsistency, not part of M6).
- **`lib/app/app.dart:3`** comment — refresh if Phase 1 changes the tab count.

**NEW COMPONENTS** **`category_providers.dart`** · **`category_sheet.dart`** · **`GoalsScreen`** · **`GoalDetailScreen`** · **`GoalEditorSheet`** · **`TaskEditorSheet`** · **`CategoryCard`** · **`GoalProgressBar`** · **`TaskRow`** (all PROPOSED, none created).
**NEW MODELS** **`Goal`** · **`Task`** · **`GoalStatus`** · **`CategoryStatus`** · **`CategoryFocus`** · **`SelectionOutcome.CategoryExhausted`** · **`TodayState.TodayCategoryExhausted`** (PROPOSED, none created). **`progress` and `taskIds` are NOT models** — they are derived (ADR-004).
**NEW SERVICES** **`GoalsRepository`** (SQL only) · extended **`ReminderService`** (task channel, **`scheduleTask`**/**`cancelTask`**/**`syncTasks`**) · **`resyncNotificationsFromStorage`**. **No new third-party service, no event bus, no DI container, no repository interface with a single implementation** (§41).
**NEW STORAGE** Phase 1: one **`settings`** row, key **`selected_category`** — **no migration**. Phase 2: tables **`goals`**, **`tasks`**; indices **`idx_goals_status`**, **`idx_tasks_goal`**, **`idx_tasks_due`**, **`idx_tasks_open`** — **one additive migration, v1 → v2**.
**NEW SCREENS** **`CategorySheet`** (modal) · **`GoalsScreen`** (tab, 2 segments) · **`GoalDetailScreen`** · **`GoalEditorSheet`** (modal) · **`TaskEditorSheet`** (modal) · **`TodayCategoryExhausted`** (in-screen state, not a screen).
**NEW NAVIGATION** One tab added: **`امروز(0) / تاریخچه(1) / اهداف(2) / تنظیمات(3)`**. No routes, no router package, no **`go_router`**. Today stays index 0 so notification-tap routing is byte-identical. Phase 1 adds **no** navigation change.
**NEW DEPENDENCIES**

| **Package**                                                               | **Purpose**                    | **Why existing code cannot handle it**                                                                                                                                   | **Alternative**                                | **Risk**                                                                                                                                                                   |
| ------------------------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **none — Phase 1**                                                        | —                              | Riverpod, the **`settings`** table, **`faLabel`**, **`faDigits`**, tokens and Material bottom sheets already cover it entirely                                           | —                                              | —                                                                                                                                                                          |
| **none — Phase 2 (recommended)**                                          | —                              | **`ReminderService`** extends cleanly; SQLite holds the data; **`flutter_local_notifications`** is already installed and manifest-ready                                  | —                                              | —                                                                                                                                                                          |
| **`shamsi_date`** / **`persian_calendar`** *(only if D1 approves Jalali)* | Persian calendar for due dates | **`fa_date.dart`** is Persian-labelled **Gregorian** and states it never converts to/from Jalali; a Persian audience reading «۲۷ سپتامبر» for 6 مهر is a real UX problem | keep Gregorian display (status quo, zero risk) | **MEDIUM** — a new date dependency must never touch day identity; display-only, behind a pure formatter mirroring **`fa_date.dart`**; must not become a second date system |
| **`integration_test`** (SDK, dev only)                                    | on-device tests                | **`integration_test/`** does not exist; without it nothing can prove a notification was delivered                                                                        | manual M6.0 runtime gate (status quo)          | **LOW** — dev-only, SDK-bundled, no APK size impact                                                                                                                        |

────────────────────────────────────────────────────────── 22. FREEZE DECISION ──────────────────────────────────────────────────────────
**ARCHITECTURE STATUS: READY**
All 22 freeze questions are answerable from the repository:

1. **What exists today?** 3 tabs, 3 SQLite tables, a 400-question bundled library, one daily reminder, one share card, 213 tests. 34 lib files.

2. **How does question selection work?** **`TodayNotifier._resolve`** → mapping reuse, else **`selectUnseenQuestion`**: drop seen → min-exposure categories → min-FNV-hash. Full algorithm in §4.

3. **How does anti-repetition work?** **`day_map`** *is* the presentation ledger; a question is marked seen by persisting the day that shows it. No seen-table, no migration. Detail in §5.

4. **Where is user state persisted?** One SQLite file, 3 tables, kv **`settings`**. Full matrix in §6. Bundled JSON holds content; nothing else persists.

5. **How does navigation work?** **`_TabShell`** **`IndexedStack`** + **`BottomNavigationBar`**, 3 permanently-mounted tabs, **no routes, no router package**.

6. **Where should category selection live?** A modal bottom sheet opened from a chip on Today (ADR-001) — 1 interaction, no nav change, scales in a **`ListView`**, and Settings would be 3+ taps on a low-traffic screen.

7. **How does it interact with anti-repetition?** The filter is consulted **only** when the day has no mapping yet; the day's question is immutable (ADR-003). The one **`day_map`** row is never deleted or rewritten.

8. **How do exhausted categories behave?** A new **`CategoryExhausted`** outcome; card DISABLED in the sheet with «تمام شد»; if it is the selected one, a dedicated calm state with «انتخاب حوزهٔ دیگر»; **never** a silent fallback to all categories, and **never** a **`day_map`** write (§9.4).

9. 9\. \*\*What is the Goal model?\*\* \`id, title, description?, category?, status
   (active|completed), createdAtUtc, updatedAtUtc, completedAtUtc?, targetDate?,
   sourceQuestionId?, sourceOptionId?, sourceDayKey?\`. \`progress\` and
   \`taskIds\` are \*\*derived, not stored\*\* — no \`startDate\`, no \`draft\` state
   (ADR-004, §11.1).

   10\. \*\*What is the Task model?\*\* \`id, goalId?, title, notes?, createdAtUtc,
   updatedAtUtc, dueDate?, dueTime?, completed (0/1), completedAtUtc?,
   reminderEnabled (0/1), notificationId?\`. \`dueDate\` reuses the existing
   Gregorian \`YYYY-MM-DD\` identity; \`dueTime\` reuses the existing
   \`parseReminderTime\` validator. No priority, no tags, no subtasks, no
   recurrence in v1 (ADR-005, §11.2).

   11\. \*\*How are Goals and Tasks related?\*\* One Goal → many Tasks.
   \`tasks.goal_id\` is \*\*nullable\*\*. A Task may exist without a Goal; a Goal
   may exist with zero Tasks; a Task belongs to at most ONE Goal. No join
   table. Deleting a Goal \*\*re-parents\*\* its Tasks to \`goal_id = NULL\`
   (zero data loss); deleting a Task never touches its Goal; completing a
   Goal does not auto-complete its Tasks; a completed Goal \*\*can be reopened\*\*
   (ADR-006).

   12\. \*\*Should Tasks require Goals?\*\* \*\*No.\*\* Requiring one would put a goal
   before every task — the highest-friction path possible, directly against
   §34's low-cognitive-load principle. A free-floating task is a first-class
   object, and "task without a goal" is a normal state, not an error (§12.2).

   13\. \*\*How will reminders work?\*\* By \*\*extending\*\* the existing
   \`flutter_local_notifications\` stack — never a second library, never
   \`AlarmManager\`, never \`SCHEDULE_EXACT_ALARM\`. Second channel \`harf_tasks\`;
   task ids in a reserved band (daily 1700–1799, tasks 2000+, allocated
   transactionally and stored on the task so cancel is exact after process
   death); still \`inexactAllowWhileIdle\`; one rule — \*\*30 minutes before due
   time, or 09:00 on the due day when no time is set\*\*; \`resync…\` generalized to
   cover both daily and task schedules from the same two call sites
   (ADR-007, §12.4).

   14\. \*\*What notification infrastructure already exists?\*\* One architecture:
   abstract \`ReminderService\` + \`LocalReminderService\` + \`FakeReminderService\`,
   policy pure in \`notifications.dart\`, single id \`1701\`, channel \`harf_daily\`,
   cancel-then-replace, rationale-first permission flow, \`fixedOffsetLocation\`
   timezone, boot receiver already in the manifest, and a \`POST_NOTIFICATIONS\`
   permission already granted flow. \*\*Nothing new is required in
   \`AndroidManifest.xml\`\*\* (§7).

   15\. \*\*How will existing user data survive?\*\* Phase 1 needs \*\*no migration at
   all\*\* — one new \`settings\` key, which a v1 build simply ignores. Phase 2 is
   exactly one additive v1→v2 migration creating two empty tables; no existing
   table is read, rewritten, re-typed, or dropped, so \`answer_records\`,
   \`day_map\`, History, Streak and the seen-set are provably untouched. The
   first Phase 2 test is a real-v1-fixture → v2 migration test (ADR-011, §14).

   16\. \*\*What screens will change?\*\* Phase 1: \`TodayScreen\` (adds a category chip
   and one new state view). Phase 2: \`ResultCard\` (one optional secondary
   button), \`app.dart\` (one added tab).

   17\. \*\*What screens will be added?\*\* \`CategorySheet\` (modal) ·
   \`TodayCategoryExhausted\` (in-screen state) · \`GoalsScreen\` (tab) ·
   \`GoalDetailScreen\` · \`GoalEditorSheet\` · \`TaskEditorSheet\` (ADR-001, ADR-010).

   18\. \*\*What files must change?\*\* Phase 1 — exactly \*\*four existing files\*\*:
   \`lib/features/today/selection.dart\`,
   \`lib/features/today/today_providers.dart\`,
   \`lib/features/today/today_screen.dart\`,
   \`lib/data/app_database.dart\` (+ \`models.dart\` for the key constant), plus
   \*\*two new\*\* files: \`category_providers.dart\`, \`category_sheet.dart\`.
   Full table in §19.

   19\. \*\*What files must remain untouched?\*\* \`content_bundle.dart\`,
   \`content_audit.dart\`, all of \`content/questions/\*\*\`,
   \`assets/content/\*\*\`, all of \`tool/\*\*\`, \`app_database.dart\`'s schema +
   \`\_createV1\`/\`\_upgrade\` + \`databaseVersion\` (Phase 1),
   \`result_engine.dart\`, \`streak.dart\`, \`history_entry.dart\`, all of
   \`features/history/\*\*\`, all of \`features/share/\*\*\`, all of
   \`features/settings/\*\*\`, \`app.dart\`, \`main.dart\`, everything under
   \`android/\*\*\`, \`pubspec.yaml\`, and all 21 existing test files.

   20\. \*\*What is the implementation order?\*\* 15 steps in §20, with the two
   deliberate deviations from the brief's template: STEP 1 lands the
   \*persistence seam alone\* (no provider, no UI, no behaviour) so ADR-003's
   risk is isolated and provable up front, and STEP 4 — the only step that can
   break the M5 promise — is split into three commits (\`selection.dart\` alone
   with all 15 existing tests untouched, then the notifier, then the R1 guard
   tests).

   21\. \*\*What are the major regression risks?\*\* Thirteen in §16. The three that
   could actually ship a defect: \*\*R1\*\* same-day category switch re-consuming
   or duplicating a question (mitigated by ADR-003 + a byte-identical
   \`day_map\` guard test); \*\*R7\*\* a migration that rebuilds a user table
   (mitigated by additive-only \`CREATE\` + the migration test written first);
   \*\*R3\*\* stale \`IndexedStack\` making a saved Goal look unsaved (mitigated by
   the established \`ref.invalidate\` pattern at
   \`today_providers.dart:278-279\`).

   22\. \*\*How will every change be tested?\*\* §17. Baseline is 213 tests / 21 files,
   all staying green, none deleted, skipped, or weakened. Phase 1 adds \~19
   tests across selection, persistence, exhaustion, anti-repetition, day
   rollover, History, RTL, and goldens — with two explicit \*\*R1 guard\*\* tests
   asserting \`day_map\` is never rewritten by a category change and never
   written at all on exhaustion. Phase 2 adds the migration test first, then
   goal/task/reminder coverage. \*\*Known gap, stated rather than assumed
   away:\*\* \`integration_test/\` does not exist, so no automated test can prove a
   notification was \*delivered\*; that requires the M6.0 hardware gate, and
   creating the directory needs a \`dev_dependencies\` change (approved per
   §33).

   ──────────────────────────────────────────────────────────
   FREEZE DECISION (continued)
   ──────────────────────────────────────────────────────────

   \*\*ARCHITECTURE STATUS: READY\*\*

   \*\*REASON:\*\* Every one of the 22 freeze questions is answerable from the
   repository, and none of them required an assumption, an invented component, or a
   guess. The two phases are both \*\*pure extensions\*\* of an architecture that was
   built for exactly this: questions already carry a category, the enum already
   carries Persian labels, counts are derivable from a ledger that already exists,
   persistence already has a typed kv pattern to copy, reminders already sit behind
   an abstract service, and the migration ladder already has its \`case 1:\` stub and
   its test seam.

   The single most important finding of this discovery is that \*\*Phase 1's
   dangerous idea — letting the user change category mid-day — is not needed at
   all.\*\* Because \`day_map\` is keyed by \`day_key\`, the day's question is already
   immutable, and the filter can be applied \*only\* on the \`mapping == null\` branch.
   ADR-003 therefore turns the highest-risk feature into a \*\*zero-migration,
   zero-\`day_map\`-write, fully-testable\*\* change, and converts a data-loss-shaped
   risk into a one-line scoping rule.

   Three decisions are product calls, not architectural unknowns, and none blocks
   this freeze:

   \| ID | Decision | Consequence if "yes" | Consequence if "no" |
   \|---|---|---|---|
   \| \*\*D1\*\* | Jalali calendar for Phase 2 due-date display? | adds a display-only date dependency (§33) — must never touch day identity | status quo: Persian-labelled Gregorian, zero risk |
   \| \*\*D2\*\* | Accept that the day's question is immutable once shown? | feature ships as designed; user changes focus from tomorrow | requires same-day switching, which forces \`day_map\` rewrites and puts the M5 never-repeat promise at risk — \*\*not recommended\*\* |
   \| \*\*D3\*\* | Adopt the already-bundled Vazirmatn in the UI? | visible modernization at zero dependency cost; needs an isolated commit + full visual pass (R9) | UI keeps the platform Persian face; the share card and the app then look different |

   \*\*NEXT STEP:\*\* Do not begin implementation yet. Required, in order:

   1\. Approve or amend \*\*ADR-001 through ADR-011\*\* (11 decisions, §18).
   2\. Answer \*\*D1, D2, D3\*\* (§22). D2 is the only one with architectural weight.
   3\. Authorize \*\*STEP 1\*\* (the \`selected_category\` persistence seam alone — four
   additive accessors and one key constant, no provider, no UI, no behaviour
   change) as the first implementation commit.

   The contract another agent needs is §§19–21: the file-level map with per-file
   risk, the MUST / MUST NOT / MAY lists, the 15-step order, and the dependency
   policy. Nothing in this document has been implemented, and no production file
   was created, modified, or deleted.

   Final §44 worktree verification:

   **`git diff-index HEAD`** and **`git status --porcelain`** both return empty — tracked **content** is identical to HEAD. The mtime list needs explaining, not glossing over. Checking the actual timestamps:

   Resolved cleanly — and the correction is worth stating plainly.

   **§44 WORKTREE VERIFICATION: CLEAN**

   | **Check**                             | **Result**                                                             |
   | ------------------------------------- | ---------------------------------------------------------------------- |
   | **`git status --short`**              | *(no output)*                                                          |
   | **`git status --porcelain`**          | *(no output)* — no modified, no untracked                              |
   | **`git diff-index --name-only HEAD`** | *(no output)* — **zero tracked content changes**                       |
   | HEAD / branch                         | **`44fb44f docs(m5): add content expansion report`** / **`master`**    |
   | File counts                           | **`lib/`** 38, **`test/`** 22, **`content/questions/`** 24 — unchanged |

   **Correcting my own last check, because its output looked alarming and wasn't.** I ran a "modified in the last 3 hours" sweep under **`lib/ test/ tool/ content/ assets/ docs/`** and it listed 44 files. That was a badly-chosen window, not a real finding. The actual mtimes are **06:30–07:20**, which is the M5 authoring session (**`be898b5`** 00:22 → **`44fb44f`** 07:20). Current time is **09:14**; this freeze session began after **08:47**, following the M6 pre-check. The newest mtime in the entire list is **`docs/m5-content-expansion-report.md`** at **07:20:09** — 95 minutes *before* this session started. Nothing under **`lib/`**, **`test/`**, **`tool/`**, **`content/`**, or **`assets/`** was written during the freeze.

   Git's own content-level verification is the authoritative one, and it is unambiguous: **`diff-index`** empty means every tracked file's blob matches **`HEAD`** exactly. The mtimes moved because M5 authored and committed those files, and because the M6 pre-check's **`flutter clean`** + release build touched only **`.dart_tool/`** and **`build/`** — both gitignored, neither containing **`lib/`** or **`content/`**.

   ```text
   ══════════════════════════════════════════════════════════
   ```

   **M6 Evolution — Architecture & UX Freeze: complete and READY.**

   What this session produced, and what it did not:

   - **Inspected** 34 production Dart files, 24 authored content sources, the manifest, Gradle config, resolved dependency versions, and all 21 test files — no report, memory, or naming convention was used as a substitute for reading the code.
   - **Marked every claim** OBSERVED / INFERRED / PROPOSED, and returned **`NOT FOUND`** with the search performed for onboarding, routing, a shared component library, Jalali support, and any Goal/Task concept — all genuinely absent.
   - **Found the single most important fact:** **`day_map`** is *already* a presentation ledger, keyed by **`day_key`**. That single observation is what turns Phase 1's most dangerous requirement — same-day category switching — into a **zero-migration, zero-write** change (ADR-003), and it is why the 400-question never-repeat promise survives intact.
   - **Found two things nobody asked about but that change the plan:** the UI does **not** use the bundled Vazirmatn font (only the share-card export does), making a premium modernization available at zero dependency cost; and there is **no Jalali calendar anywhere**, which is a real product decision (D1) before Phase 2 due dates.
   - **Recommended against** three things that looked reasonable: silently falling back to "all categories" on exhaustion, storing **`progress`**/**`taskIds`** as denormalized fields, and requiring a Goal before any Task — the last two because they trade the calm product thesis for conventional data modelling.
   - **Created, modified, and deleted nothing.** No **`pubspec.yaml`** change, no dependency, no new file. Phase 1 needs **zero** new dependencies; Phase 2's recommended path needs **zero** too, and the one candidate dependency is listed with its risk and a zero-risk alternative.

   Awaiting your sign-off on **ADR-001…011** and decisions **D1 / D2 / D3**. Only after that does STEP 1 begin — and STEP 1 is the persistence seam alone, deliberately behaviourless, so that the riskiest decision in the whole programme is provable before any user-visible code moves.