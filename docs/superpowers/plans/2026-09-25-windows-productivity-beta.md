# Windows Productivity Tracker Beta Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a distributable Windows beta that measures active app and Chrome/Edge domain usage, classifies activity, manages goals and focus sessions, provides non-blocking coaching, and optionally synchronizes encrypted user data.

**Architecture:** Use a pnpm monorepo with a Tauri 2 desktop app, React/TypeScript UI, Rust native collectors and domain logic, a Manifest V3 browser extension, SQLite as the local source of truth, and Supabase for optional authentication and synchronization. Events from Windows and the browser extension are normalized into immutable activity sessions; policy, UI, and sync consume those sessions through narrow interfaces.

**Tech Stack:** Tauri 2, Rust stable, React, TypeScript, Vite, Vitest, Testing Library, Playwright, SQLite/SQLCipher, Manifest V3, Native Messaging, Supabase Auth/PostgreSQL.

**Spec:** `docs/superpowers/specs/2026-09-25-desktop-productivity-tracker-design.md`

## Global Constraints

- Windows 10 22H2 and supported Windows 11 releases are the beta targets.
- The app must remain fully usable without an account or internet connection.
- Store only app identifiers or normalized domains, timestamps, active duration, and classification by default.
- Never collect keystroke content, page body content, screenshots, or document content.
- Measure only the foreground app or active browser tab; idle, locked, suspended, and signed-out time is excluded.
- Do not block apps or websites; intervention is limited to notification, a 5–15 second mindful pause, and user choice.
- Detailed local activity is retained for 90 days by default; aggregates persist until user deletion.
- Default sync includes settings, goals, classification rules, and aggregates. Detailed-session sync is explicit opt-in.
- Raw timestamps are UTC; local date boundaries use the timezone stored with the applicable goal and aggregate.
- Target daily measurement error is ±5%; idle resource targets are 1% average CPU and 150 MB memory.
- Keep `.superpowers/` out of version control.

## Review Focus

- Clock rollback, timezone change, and midnight crossing must split sessions without negative or duplicated duration; Task 4 owns these tests.
- Browser/native messages from an unapproved extension or malformed payload must be rejected without stopping desktop tracking; Task 6 owns these tests.
- Offline edits followed by reconnect, token expiry, or duplicate delivery must converge without losing local data; Task 12 owns these tests.
- Permission revocation, screen lock, sleep, and collector failure must close the active session and expose a truthful degraded state; Tasks 5 and 13 own these tests.
- Deleting synchronized data must propagate tombstones and must not resurrect records from another device; Task 12 owns these tests.

---

## Phase A — Local-first desktop core

### Task 1: Bootstrap the monorepo and executable desktop shell

**Files:**
- Create: `.gitignore`
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `apps/desktop/package.json`
- Create: `apps/desktop/src/App.tsx`
- Create: `apps/desktop/src/App.test.tsx`
- Create: `apps/desktop/src-tauri/Cargo.toml`
- Create: `apps/desktop/src-tauri/src/main.rs`
- Create: `packages/contracts/package.json`
- Create: `packages/contracts/src/index.ts`

**Interfaces:**
- Produces: workspace scripts `lint`, `test`, `test:rust`, `build`; Tauri command `health_check() -> HealthStatus`.
- Produces: shared TypeScript types exported by `@productivity/contracts`.

- [ ] **Step 1: Initialize version control and workspace metadata**

Run:

```powershell
git init
corepack enable
pnpm init
```

Create `.gitignore` with `node_modules/`, `target/`, `dist/`, `.env*`, `.superpowers/`, `*.sqlite*`, and Tauri signing artifacts.

- [ ] **Step 2: Write the failing desktop smoke test**

```tsx
import { render, screen } from '@testing-library/react';
import { App } from './App';

it('shows the local-first status', () => {
  render(<App />);
  expect(screen.getByText('로컬에서 기록 중')).toBeVisible();
});
```

- [ ] **Step 3: Run the test and verify the intended failure**

Run: `pnpm --filter desktop test -- App.test.tsx`

Expected: FAIL because `App` and test configuration do not exist.

- [ ] **Step 4: Add the smallest working Tauri/React shell**

Implement `App` as a named component that renders a navigation shell and the text `로컬에서 기록 중`. Add Vitest, Testing Library, ESLint, TypeScript strict mode, and the Tauri 2 build configuration. In Rust expose:

```rust
#[derive(serde::Serialize)]
struct HealthStatus { status: &'static str }

#[tauri::command]
fn health_check() -> HealthStatus { HealthStatus { status: "ok" } }
```

- [ ] **Step 5: Verify the workspace**

Run: `pnpm lint; pnpm test; pnpm test:rust; pnpm build`

Expected: all commands exit 0 and the Tauri bundle compiles in debug mode.

- [ ] **Step 6: Commit the bootstrap**

```powershell
git add .gitignore package.json pnpm-workspace.yaml apps packages
git commit -m "chore: bootstrap productivity tracker workspace"
```

### Task 2: Define cross-boundary contracts and validation

**Files:**
- Create: `packages/contracts/src/activity.ts`
- Create: `packages/contracts/src/classification.ts`
- Create: `packages/contracts/src/goals.ts`
- Create: `packages/contracts/src/sync.ts`
- Create: `packages/contracts/src/activity.test.ts`
- Modify: `packages/contracts/src/index.ts`

**Interfaces:**
- Produces: `ActivityEventSchema`, `ActivitySession`, `Classification`, `DailyGoal`, `FocusSession`, `InterventionEvent`, `Device`, `SyncOperation`.
- Consumes: no runtime dependency other than Zod.

- [ ] **Step 1: Write contract tests for valid and forbidden input**

```ts
it('accepts a normalized domain event and drops forbidden content fields', () => {
  const parsed = ActivityEventSchema.parse({
    eventId: 'evt-1', source: 'browser', identifier: 'chatgpt.com',
    observedAt: '2026-09-25T01:00:00.000Z', active: true,
    pageBody: 'must not survive'
  });
  expect(parsed.identifier).toBe('chatgpt.com');
  expect(parsed).not.toHaveProperty('pageBody');
});

it('rejects a non-UTC timestamp', () => {
  expect(() => ActivityEventSchema.parse({
    eventId: 'evt-2', source: 'windows', identifier: 'code.exe',
    observedAt: '2026-09-25T10:00:00+09:00', active: true
  })).toThrow();
});
```

- [ ] **Step 2: Run the contract test and verify failure**

Run: `pnpm --filter @productivity/contracts test`

Expected: FAIL because `ActivityEventSchema` is undefined.

- [ ] **Step 3: Implement exact shared types and schemas**

Use discriminated values:

```ts
export type ActivitySource = 'windows' | 'browser';
export type Classification = 'productive' | 'unproductive' | 'neutral' | 'unclassified';
export interface ActivitySession {
  id: string; deviceId: string; source: ActivitySource; identifier: string;
  startedAt: string; endedAt: string; activeSeconds: number;
  classification: Classification; timezone: string;
}
```

Configure Zod objects with `.strip()` and UTC strings ending in `Z`. Define all entity fields exactly as specified in the design document.

- [ ] **Step 4: Verify contracts and type checking**

Run: `pnpm --filter @productivity/contracts test; pnpm --filter @productivity/contracts typecheck`

Expected: PASS.

- [ ] **Step 5: Commit contracts**

```powershell
git add packages/contracts
git commit -m "feat: define shared activity and sync contracts"
```

### Task 3: Create encrypted local persistence and repositories

**Files:**
- Create: `apps/desktop/src-tauri/migrations/0001_initial.sql`
- Create: `apps/desktop/src-tauri/src/storage/mod.rs`
- Create: `apps/desktop/src-tauri/src/storage/models.rs`
- Create: `apps/desktop/src-tauri/src/storage/activity_repository.rs`
- Create: `apps/desktop/src-tauri/src/storage/settings_repository.rs`
- Create: `apps/desktop/src-tauri/tests/storage_test.rs`
- Modify: `apps/desktop/src-tauri/src/main.rs`

**Interfaces:**
- Produces: `ActivityRepository::insert_session`, `list_range`, `delete_range`, `purge_expired`; `SettingsRepository::get`, `set`.
- Produces: `Database::open(path: &Path, key: SecretString) -> Result<Database>`.
- Consumes: Task 2 entity names serialized through Tauri DTOs.

- [ ] **Step 1: Write failing migration and repository tests**

```rust
#[test]
fn insert_is_idempotent_and_range_delete_creates_tombstone() {
    let db = test_database();
    let session = fixture_session("session-1");
    db.activities().insert_session(&session).unwrap();
    db.activities().insert_session(&session).unwrap();
    assert_eq!(db.activities().list_range(day()).unwrap().len(), 1);
    db.activities().delete_range(day()).unwrap();
    assert!(db.activities().list_range(day()).unwrap().is_empty());
    assert_eq!(db.pending_tombstones().unwrap().len(), 1);
}
```

- [ ] **Step 2: Run the storage test and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml --test storage_test`

Expected: FAIL because storage modules and migration are absent.

- [ ] **Step 3: Implement the schema and repositories**

Create tables for `activity_sessions`, `classification_rules`, `daily_goals`, `focus_sessions`, `intervention_events`, `daily_aggregates`, `devices`, `sync_operations`, and `settings`. Use primary IDs for idempotency, UTC text timestamps, foreign keys, and indices on `(device_id, started_at)` and `(identifier, started_at)`. Store the SQLCipher key only through a credential-store adapter; never log it.

- [ ] **Step 4: Verify migration, CRUD, deletion, and 90-day purge**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml storage`

Expected: PASS, including boundary records exactly 90 days old.

- [ ] **Step 5: Commit persistence**

```powershell
git add apps/desktop/src-tauri/migrations apps/desktop/src-tauri/src/storage apps/desktop/src-tauri/tests/storage_test.rs
git commit -m "feat: add encrypted local activity storage"
```

### Task 4: Build the session normalizer

**Files:**
- Create: `apps/desktop/src-tauri/src/activity/mod.rs`
- Create: `apps/desktop/src-tauri/src/activity/normalizer.rs`
- Create: `apps/desktop/src-tauri/src/activity/clock.rs`
- Create: `apps/desktop/src-tauri/tests/normalizer_test.rs`
- Modify: `apps/desktop/src-tauri/src/main.rs`

**Interfaces:**
- Produces: `SessionNormalizer::observe(ActivityObservation) -> Vec<SessionChange>`.
- Produces: `SessionChange::{Opened, Updated, Closed, MeasurementGap}`.
- Consumes: observations `{source, identifier, observed_at_utc, is_idle, timezone}`.

- [ ] **Step 1: Write failing boundary tests**

Cover identical-event merging, app switches, browser-over-process precedence, idle transition, midnight split, clock rollback, timezone change, and duplicate event IDs. Pin the clock with fixtures rather than sleeping.

```rust
#[test]
fn clock_rollback_closes_without_negative_duration() {
    let mut n = fixture_normalizer();
    n.observe(obs("code.exe", "2026-09-25T02:00:10Z"));
    let changes = n.observe(obs("code.exe", "2026-09-25T01:59:00Z"));
    assert!(changes.iter().all(|c| c.duration_seconds().unwrap_or(0) >= 0));
    assert!(matches!(changes[0], SessionChange::Closed(_)));
}
```

- [ ] **Step 2: Run and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml --test normalizer_test`

Expected: FAIL because `SessionNormalizer` is absent.

- [ ] **Step 3: Implement a deterministic state machine**

Use event timestamps, not wall-clock sleeps. Close and reopen when identifier, source precedence, idle state, local date, timezone, or non-monotonic clock changes. Ignore duplicate `event_id`. Emit a measurement gap instead of inferring missing time.

- [ ] **Step 4: Verify all time-boundary tests**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml normalizer`

Expected: PASS.

- [ ] **Step 5: Commit normalizer**

```powershell
git add apps/desktop/src-tauri/src/activity apps/desktop/src-tauri/tests/normalizer_test.rs
git commit -m "feat: normalize foreground observations into sessions"
```

### Task 5: Implement the Windows foreground and idle collector

**Files:**
- Create: `apps/desktop/src-tauri/src/platform/mod.rs`
- Create: `apps/desktop/src-tauri/src/platform/windows.rs`
- Create: `apps/desktop/src-tauri/src/platform/windows_api.rs`
- Create: `apps/desktop/src-tauri/tests/windows_collector_test.rs`
- Modify: `apps/desktop/src-tauri/src/main.rs`

**Interfaces:**
- Produces: trait `ActivityCollector { fn poll(&self) -> Result<CollectorState>; }`.
- Produces: `CollectorState::{Active { app_id, observed_at }, Idle, Locked, Suspended, Unavailable { reason }}`.
- Consumes: Task 4 normalizer.

- [ ] **Step 1: Write tests against a fake Win32 adapter**

Test foreground changes, configurable five-minute idle threshold, lock, suspend, access denial, and collector recovery. Assert that process ID and executable identity are returned but window title is not persisted.

- [ ] **Step 2: Run and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml --test windows_collector_test`

Expected: FAIL because the collector trait is absent.

- [ ] **Step 3: Implement the Win32 adapter and collector**

Wrap `GetForegroundWindow`, `GetWindowThreadProcessId`, `QueryFullProcessImageNameW`, and `GetLastInputInfo` behind `WindowsApi`. Listen for session lock/unlock and power suspend/resume. Poll every five seconds while awake; emit `Unavailable` on errors and continue retrying without terminating the app.

- [ ] **Step 4: Verify lifecycle and degraded-state tests**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml windows_collector`

Expected: PASS.

- [ ] **Step 5: Commit Windows collection**

```powershell
git add apps/desktop/src-tauri/src/platform apps/desktop/src-tauri/tests/windows_collector_test.rs
git commit -m "feat: collect active Windows application time"
```

## Phase B — Browser visibility and local product behavior

### Task 6: Build the Chrome/Edge extension and secure Native Messaging host

**Files:**
- Create: `apps/extension/manifest.json`
- Create: `apps/extension/package.json`
- Create: `apps/extension/src/background.ts`
- Create: `apps/extension/src/domain.ts`
- Create: `apps/extension/src/domain.test.ts`
- Create: `apps/desktop/src-tauri/src/browser/mod.rs`
- Create: `apps/desktop/src-tauri/src/browser/message.rs`
- Create: `apps/desktop/src-tauri/tests/native_message_test.rs`
- Create: `scripts/install-native-host.ps1`

**Interfaces:**
- Produces: browser message `{eventId, extensionId, domain, observedAt, active}`.
- Produces: `normalize_domain(url: &str) -> Result<String>` and `BrowserMessage::validate(allowed_extension_ids)`.
- Consumes: Task 4 `ActivityObservation`; Task 2 `ActivityEventSchema`.

- [ ] **Step 1: Write domain normalization and security tests**

```ts
expect(normalizeDomain('https://www.YouTube.com/watch?v=1')).toBe('youtube.com');
expect(() => normalizeDomain('chrome://settings')).toThrow();
expect(() => normalizeDomain('file:///C:/secret.txt')).toThrow();
```

In Rust test malformed JSON, oversized frames, unknown extension IDs, replayed event IDs, and a valid domain event. The expected behavior is rejection with a structured error while Windows collection remains healthy.

- [ ] **Step 2: Run and verify both suites fail**

Run: `pnpm --filter extension test; cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml native_message`

Expected: FAIL because the extension and host do not exist.

- [ ] **Step 3: Implement active-tab observation**

Use `tabs.onActivated`, `tabs.onUpdated`, and `windows.onFocusChanged`. Send only the registrable domain, never the path, query, page title, or content. Send an inactive event when the browser loses focus.

- [ ] **Step 4: Implement and register the Native Messaging host**

Validate the four-byte frame length, cap payload size, parse through the shared schema, verify the configured extension ID, and deduplicate event IDs. Make `install-native-host.ps1` write per-browser manifests for Chrome and Edge using the built host executable path.

- [ ] **Step 5: Verify extension and host integration**

Run: `pnpm --filter extension test; cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml native_message`

Expected: PASS.

- [ ] **Step 6: Commit browser support**

```powershell
git add apps/extension apps/desktop/src-tauri/src/browser apps/desktop/src-tauri/tests/native_message_test.rs scripts/install-native-host.ps1
git commit -m "feat: capture active Chrome and Edge domains"
```

### Task 7: Implement classification, aggregates, and goals

**Files:**
- Create: `apps/desktop/src-tauri/src/productivity/mod.rs`
- Create: `apps/desktop/src-tauri/src/productivity/classifier.rs`
- Create: `apps/desktop/src-tauri/src/productivity/aggregator.rs`
- Create: `apps/desktop/src-tauri/src/productivity/goals.rs`
- Create: `apps/desktop/src-tauri/resources/default-classifications.json`
- Create: `apps/desktop/src-tauri/tests/productivity_test.rs`

**Interfaces:**
- Produces: `Classifier::classify(target) -> Classification` with user > default > unclassified precedence.
- Produces: `aggregate_day(sessions, timezone) -> DailyAggregate`.
- Produces: `GoalEvaluator::evaluate(aggregate, goal) -> GoalProgress`.
- Consumes: Tasks 2–3 entities and repositories.

- [ ] **Step 1: Write calculation tests with exact expected values**

Use a fixture with 120 productive, 30 unproductive, 20 neutral, and 10 idle minutes. Assert productivity ratio `120 / (120 + 30) = 0.8`; neutral and idle are excluded. Test custom classification overriding the bundled `youtube.com` rule and an unclassified target.

- [ ] **Step 2: Run and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml --test productivity_test`

Expected: FAIL because classification and aggregation modules are absent.

- [ ] **Step 3: Implement rules, aggregation, and goal evaluation**

Bundle a small reviewed default list including Codex/ChatGPT as productive and YouTube/Instagram as unproductive. Do not infer category from page text. Recompute affected aggregates transactionally when a user changes a classification.

- [ ] **Step 4: Verify exact arithmetic and boundary behavior**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml productivity`

Expected: PASS, including zero-denominator ratio returning `None` rather than NaN.

- [ ] **Step 5: Commit productivity logic**

```powershell
git add apps/desktop/src-tauri/src/productivity apps/desktop/src-tauri/resources apps/desktop/src-tauri/tests/productivity_test.rs
git commit -m "feat: classify activity and evaluate daily goals"
```

### Task 8: Implement non-blocking coaching and focus sessions

**Files:**
- Create: `apps/desktop/src-tauri/src/coaching/mod.rs`
- Create: `apps/desktop/src-tauri/src/coaching/policy.rs`
- Create: `apps/desktop/src-tauri/src/coaching/focus.rs`
- Create: `apps/desktop/src-tauri/tests/coaching_test.rs`
- Create: `apps/desktop/src/features/coaching/MindfulPause.tsx`
- Create: `apps/desktop/src/features/coaching/MindfulPause.test.tsx`

**Interfaces:**
- Produces: `CoachingPolicy::evaluate(progress) -> CoachingAction`.
- Produces: `CoachingAction::{None, NotifyApproaching, ShowMindfulPause { seconds }}`.
- Produces: `FocusTimer::{start, pause, resume, complete}`.
- Consumes: Task 7 `GoalProgress`; writes `InterventionEvent` through Task 3 storage.

- [ ] **Step 1: Write failing policy and UI tests**

Assert 79% gives `None`, 80% gives one approach notification, 100% gives a mindful pause, repeated polling does not duplicate an intervention, and the pause never calls a blocking API. In React assert the continue/transition buttons stay disabled only for the configured countdown and both become available afterward.

- [ ] **Step 2: Run and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml coaching; pnpm --filter desktop test -- MindfulPause.test.tsx`

Expected: FAIL.

- [ ] **Step 3: Implement coaching and focus state machines**

Persist state transitions so restart does not duplicate notifications. Clamp pause duration to 5–15 seconds. `계속 사용` dismisses the UI; `생산적 활동으로 전환` dismisses it and opens the dashboard suggestion list. Neither action terminates or hides another process.

- [ ] **Step 4: Verify policy, countdown, restart, and no-block guarantees**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml coaching; pnpm --filter desktop test -- MindfulPause.test.tsx`

Expected: PASS.

- [ ] **Step 5: Commit coaching**

```powershell
git add apps/desktop/src-tauri/src/coaching apps/desktop/src-tauri/tests/coaching_test.rs apps/desktop/src/features/coaching
git commit -m "feat: add non-blocking coaching and focus sessions"
```

### Task 9: Build the approved desktop information architecture

**Files:**
- Create: `apps/desktop/src/app/router.tsx`
- Create: `apps/desktop/src/app/theme.css`
- Create: `apps/desktop/src/components/AppShell.tsx`
- Create: `apps/desktop/src/features/today/TodayPage.tsx`
- Create: `apps/desktop/src/features/analytics/AnalyticsPage.tsx`
- Create: `apps/desktop/src/features/activity/ActivityPage.tsx`
- Create: `apps/desktop/src/features/goals/GoalsPage.tsx`
- Create: `apps/desktop/src/features/classification/ClassificationPage.tsx`
- Create: `apps/desktop/src/features/sync/SyncPage.tsx`
- Create: `apps/desktop/src/features/settings/SettingsPage.tsx`
- Create: `apps/desktop/src/features/today/TodayPage.test.tsx`
- Modify: `apps/desktop/src/App.tsx`

**Interfaces:**
- Produces: route pages `today`, `analytics`, `activity`, `goals`, `classification`, `sync`, `settings`.
- Consumes: typed Tauri query hooks over Tasks 3, 7, and 8.

- [ ] **Step 1: Write the failing Today-page behavior test**

```tsx
it('prioritizes today metrics and next action', async () => {
  renderToday({ productiveSeconds: 12240, unproductiveSeconds: 2520, goalRatio: 0.68 });
  expect(await screen.findByText('3시간 24분')).toBeVisible();
  expect(screen.getByText('42분')).toBeVisible();
  expect(screen.getByRole('button', { name: '집중 세션 시작' })).toBeEnabled();
});
```

- [ ] **Step 2: Run and verify failure**

Run: `pnpm --filter desktop test -- TodayPage.test.tsx`

Expected: FAIL because the page does not exist.

- [ ] **Step 3: Implement routes and the color system**

Use navy for navigation, green for productive activity, red for unproductive activity, purple for coaching, blue for local data, teal for sync, and amber/orange for warnings. Meet WCAG AA contrast and never use color as the sole status cue.

- [ ] **Step 4: Implement each page with loading, empty, degraded, and error states**

The Today page is action-first. Analytics owns weekly comparisons. Activity owns the timeline and classification correction. Keep each feature's components, query hooks, and tests in its feature folder.

- [ ] **Step 5: Verify UI and accessibility**

Run: `pnpm --filter desktop test; pnpm --filter desktop lint; pnpm --filter desktop typecheck`

Expected: PASS with no accessibility violations in rendered route tests.

- [ ] **Step 6: Commit the desktop UI**

```powershell
git add apps/desktop/src
git commit -m "feat: add action-first productivity dashboard"
```

### Task 10: Add onboarding and privacy controls

**Files:**
- Create: `apps/desktop/src/features/onboarding/OnboardingFlow.tsx`
- Create: `apps/desktop/src/features/onboarding/OnboardingFlow.test.tsx`
- Create: `apps/desktop/src/features/settings/PrivacyControls.tsx`
- Create: `apps/desktop/src-tauri/src/privacy/export.rs`
- Create: `apps/desktop/src-tauri/src/privacy/delete.rs`
- Create: `apps/desktop/src-tauri/tests/privacy_test.rs`

**Interfaces:**
- Produces: Tauri commands `export_data(path, range)`, `delete_data(range)`, `reset_all_data(confirmation)`.
- Consumes: Task 3 repositories and Task 5/6 capability states.

- [ ] **Step 1: Write failing onboarding and privacy tests**

Test disclosure before tracking starts, denied permissions, skipped extension, offline account skip, excluded app/domain, CSV/JSON export without forbidden fields, range deletion, and full reset confirmation.

- [ ] **Step 2: Run and verify failure**

Run: `pnpm --filter desktop test -- OnboardingFlow.test.tsx; cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml privacy`

Expected: FAIL.

- [ ] **Step 3: Implement onboarding and controls**

Use the sequence install status → privacy explanation → Windows collector check → extension connection → classification review → goals → optional account. Tracking begins only after the disclosure step is acknowledged. Add an explicit Windows-startup toggle backed by the Tauri autostart plugin; disabling it must remove the startup registration without stopping the current session.

- [ ] **Step 4: Implement safe export and deletion**

Export normalized records and aggregates only. Exclude credentials, internal encryption material, window titles, and diagnostic logs. On deletion, close intersecting active sessions and enqueue tombstones for synchronized records.

- [ ] **Step 5: Verify all privacy flows**

Run: `pnpm --filter desktop test -- OnboardingFlow.test.tsx; cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml privacy`

Expected: PASS.

- [ ] **Step 6: Commit onboarding and privacy controls**

```powershell
git add apps/desktop/src/features/onboarding apps/desktop/src/features/settings/PrivacyControls.tsx apps/desktop/src-tauri/src/privacy apps/desktop/src-tauri/tests/privacy_test.rs
git commit -m "feat: add onboarding and privacy data controls"
```

## Phase C — Optional cloud, resilience, and beta release

### Task 11: Create Supabase authentication and row-level security

**Files:**
- Create: `supabase/config.toml`
- Create: `supabase/migrations/202609250001_sync_schema.sql`
- Create: `supabase/tests/rls.test.sql`
- Create: `apps/desktop/src/services/auth.ts`
- Create: `apps/desktop/src/features/sync/AuthPanel.tsx`
- Create: `apps/desktop/src/features/sync/AuthPanel.test.tsx`

**Interfaces:**
- Produces: `AuthService::{signIn, signOut, session}` and user-scoped sync tables.
- Consumes: Task 9 Sync page; stores tokens through the credential adapter from Task 3.

- [ ] **Step 1: Write failing RLS and UI tests**

Create two users and assert each can read and mutate only rows whose `user_id = auth.uid()`. Assert the desktop remains in `로컬 전용` state when login is skipped or fails.

- [ ] **Step 2: Run and verify failure**

Run: `supabase test db; pnpm --filter desktop test -- AuthPanel.test.tsx`

Expected: FAIL because schema and service are absent.

- [ ] **Step 3: Implement auth schema and client**

Create user-owned tables for devices, settings, goals, rules, aggregates, optional encrypted session payloads, and tombstones. Enable RLS on every exposed table. Never store refresh tokens in SQLite or browser local storage.

- [ ] **Step 4: Verify isolation and local-only behavior**

Run: `supabase test db; pnpm --filter desktop test -- AuthPanel.test.tsx`

Expected: PASS.

- [ ] **Step 5: Commit auth and RLS**

```powershell
git add supabase apps/desktop/src/services/auth.ts apps/desktop/src/features/sync
git commit -m "feat: add optional account authentication"
```

### Task 12: Implement encrypted, offline-first synchronization

**Files:**
- Create: `apps/desktop/src-tauri/src/sync/mod.rs`
- Create: `apps/desktop/src-tauri/src/sync/crypto.rs`
- Create: `apps/desktop/src-tauri/src/sync/merge.rs`
- Create: `apps/desktop/src-tauri/src/sync/worker.rs`
- Create: `apps/desktop/src-tauri/tests/sync_test.rs`
- Modify: `apps/desktop/src/features/sync/SyncPage.tsx`

**Interfaces:**
- Produces: `SyncWorker::run_once() -> SyncReport` and `MergeEngine::merge(local, remote) -> MergeResult`.
- Consumes: Task 3 sync queue, Task 11 authenticated API, credential-store key material.

- [ ] **Step 1: Write failing convergence tests**

Test offline create/update/delete, exponential retry, expired-token refresh, duplicate upload, activity union by immutable ID, settings last-write-wins, clock ties resolved by revision/device ID, and tombstone propagation preventing resurrection.

```rust
#[test]
fn remote_old_copy_cannot_resurrect_deleted_record() {
    let result = merge(local_tombstone("s1", 4), remote_record("s1", 3));
    assert!(matches!(result, MergeResult::Deleted { id } if id == "s1"));
}
```

- [ ] **Step 2: Run and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml --test sync_test`

Expected: FAIL.

- [ ] **Step 3: Implement encryption and queue processing**

Encrypt sensitive payloads before upload with an authenticated cipher and per-user data key. Upload settings, goals, rules, and aggregates by default. Refuse detailed-session upload unless the local opt-in flag is true. Process queue entries idempotently and retain failed entries.

- [ ] **Step 4: Implement merge and transparent status**

Expose last success time, pending operation count, current error, and retry action. Signing out stops the worker without deleting local data. Account deletion sends server deletion first, then clears local credentials.

- [ ] **Step 5: Verify all sync failure modes**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml sync`

Expected: PASS.

- [ ] **Step 6: Commit synchronization**

```powershell
git add apps/desktop/src-tauri/src/sync apps/desktop/src-tauri/tests/sync_test.rs apps/desktop/src/features/sync/SyncPage.tsx
git commit -m "feat: add encrypted offline-first synchronization"
```

### Task 13: Add lifecycle recovery and privacy-safe diagnostics

**Files:**
- Create: `apps/desktop/src-tauri/src/lifecycle/mod.rs`
- Create: `apps/desktop/src-tauri/src/diagnostics/mod.rs`
- Create: `apps/desktop/src-tauri/tests/lifecycle_test.rs`
- Create: `apps/desktop/src/features/settings/DiagnosticsPanel.tsx`
- Create: `apps/desktop/src/features/settings/DiagnosticsPanel.test.tsx`

**Interfaces:**
- Produces: `recover_open_session(last_heartbeat) -> RecoveryResult`.
- Produces: `DiagnosticsSnapshot` containing capability states and redacted technical errors only.
- Consumes: collectors, extension connection, storage, and sync health states.

- [ ] **Step 1: Write failing recovery and redaction tests**

Test crash recovery stops at the last heartbeat; lock/sleep closes immediately; permission revocation sets degraded status; diagnostics exclude domains, app identifiers, tokens, database keys, and user paths.

- [ ] **Step 2: Run and verify failure**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml lifecycle; pnpm --filter desktop test -- DiagnosticsPanel.test.tsx`

Expected: FAIL.

- [ ] **Step 3: Implement heartbeat recovery and capability health**

Persist a heartbeat every five seconds while active. On startup, close an orphaned session at its final heartbeat and add a recovery marker. Present collector, browser, storage, and sync status independently so one failure cannot masquerade as total success.

- [ ] **Step 4: Implement opt-in diagnostic export**

Generate a user-previewable JSON file with versions, capability states, error codes, and timestamps. Apply allow-list serialization rather than post-hoc string replacement.

- [ ] **Step 5: Verify recovery and redaction**

Run: `cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml lifecycle diagnostics; pnpm --filter desktop test -- DiagnosticsPanel.test.tsx`

Expected: PASS.

- [ ] **Step 6: Commit resilience and diagnostics**

```powershell
git add apps/desktop/src-tauri/src/lifecycle apps/desktop/src-tauri/src/diagnostics apps/desktop/src-tauri/tests/lifecycle_test.rs apps/desktop/src/features/settings/DiagnosticsPanel*
git commit -m "feat: add crash recovery and safe diagnostics"
```

### Task 14: Verify end-to-end behavior, performance, packaging, and updates

**Files:**
- Create: `tests/e2e/onboarding.spec.ts`
- Create: `tests/e2e/daily-flow.spec.ts`
- Create: `tests/e2e/offline-sync.spec.ts`
- Create: `tests/performance/collector_benchmark.rs`
- Create: `scripts/verify-privacy.ps1`
- Create: `scripts/package-beta.ps1`
- Create: `.github/workflows/ci.yml`
- Create: `.github/workflows/release.yml`
- Modify: `apps/desktop/src-tauri/tauri.conf.json`
- Create: `docs/beta-test-guide.md`

**Interfaces:**
- Produces: signed Windows installer, extension bundle, native-host installer, update manifest, and beta test guide.
- Consumes: all preceding tasks.

- [ ] **Step 1: Write end-to-end acceptance tests**

Automate onboarding, app-to-domain precedence, idle exclusion, classification correction, goal notification, mindful pause without blocking, focus session completion, offline operation, reconnect sync, export, and deletion. Use fake collector and fake clock test modes; keep one Windows-only smoke suite against real APIs.

- [ ] **Step 2: Add privacy and performance verification scripts**

`verify-privacy.ps1` must inspect exported data, diagnostic output, and test logs for seeded sentinel page text, keystrokes, tokens, and file paths and fail if any appear. Benchmark 8 simulated hours and assert daily duration error ≤5%, idle CPU target ≤1%, and memory target ≤150 MB; record hardware and treat resource thresholds as release gates on the reference machine.

- [ ] **Step 3: Run the complete verification matrix**

Run:

```powershell
pnpm lint
pnpm typecheck
pnpm test
pnpm test:rust
pnpm exec playwright test
powershell -ExecutionPolicy Bypass -File scripts/verify-privacy.ps1
cargo bench --manifest-path apps/desktop/src-tauri/Cargo.toml --bench collector_benchmark
pnpm build
```

Expected: every command exits 0; acceptance, privacy, and performance thresholds pass.

- [ ] **Step 4: Configure signed packaging and rollback-safe updates**

Read signing secrets only from CI secret storage. Build MSI/NSIS artifacts, extension ZIP, and native-host installer. Sign application and update metadata. Verify an invalid signature is rejected and an interrupted update leaves the prior version runnable.

- [ ] **Step 5: Test on supported Windows/browser combinations**

Run the smoke suite on Windows 10 22H2 and current supported Windows 11 with current stable Chrome and Edge. Record app tracking, domain tracking, lock/sleep, startup, update, and uninstall results in `docs/beta-test-guide.md`.

- [ ] **Step 6: Commit release automation**

```powershell
git add tests scripts .github apps/desktop/src-tauri/tauri.conf.json docs/beta-test-guide.md
git commit -m "build: add Windows beta verification and packaging"
```

## Final acceptance gate

- [ ] All task-specific tests pass from a clean checkout.
- [ ] The app works in local-only mode with network access disabled.
- [ ] Chrome and Edge report only normalized active domains.
- [ ] Lock, idle, sleep, crashes, and clock changes do not inflate time.
- [ ] No UI or native path blocks another app or website.
- [ ] Sync is opt-in, encrypted, idempotent, and deletion-safe.
- [ ] Export and diagnostics contain no forbidden content.
- [ ] Windows 10/11 packaging, signing, installation, update, rollback, and uninstall are verified.
- [ ] The beta guide documents permissions, privacy behavior, known limitations, and feedback collection.
