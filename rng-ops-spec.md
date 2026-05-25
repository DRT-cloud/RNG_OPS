# RNG OPS — Build Specification (v2.5)

**Project:** RNG OPS
**Purpose:** Event operations admin app for Run and Gun (RNG) biathlon events. Handles competitor check-in, start/finish timing, obstacle penalties, results, and roster management from a single browser window on a local event laptop, with read-only mobile views for staff and competitors via Firebase.
**Current Build Status:** Phase 3 complete · Staff view shipped (`staff.html`) with collapsed sections, wake-lock, reconnect banner, theme parity · Firebase auth-locked · Netlify live at [rng-ops.netlify.app](https://rng-ops.netlify.app).
**Operator account (Block 4):** `rng.ops.operator@gmail.com` (project-specific Gmail, not personal).
**Supersedes:** rng-ops-spec.md v2.4.

### What changed in Block 3 (this revision, 2026‑05‑25)

- **Shared Firebase config.** `rng-fb-config.js` is the single source of truth. Both `rng-ops-firebase.html` and the new `staff.html` load it via `<script src="rng-fb-config.js">`. The inline `FIREBASE_CONFIG` literal and the dead `firebaseConfig` literal in the operator file were removed; `const FIREBASE_CONFIG = window.FIREBASE_CONFIG;` is now the only binding. The competitor view in Block 4 will load the same file.
- **Staff view shipped (`staff.html`).** Mobile-first, read-only, no edit DOM. URL: `staff.html?event=<eventId>`. Anonymous auth. Subscribes to `events/<eventId>/competitors` and `events/<eventId>/config` via `on('value')`. Header shows event name, event ID pill, **Screen lock** pill, and connection dot. No `fbWriteCx`, no `fbWriteCfg`, no `set`/`update`/`remove`/`push` calls on any Firebase ref.
- **Three collapsed sections (all closed by default).** Tap any header to expand:
  - **On Deck** — `ci === 'in' && !st` (checked in, waiting to start).
  - **Running** — `st && !fi && !dnf`; rows append start clock (HH:MM:SS) and live elapsed (`m:ss`).
  - **Done** — `fi || dnf`; rows show final time (raw + obstacle penalty applied), finish clock, and a green **DONE** or red **DNF** pill.
  Section counts (`On Deck N`, `Running N`, `Done N`) update live as Firebase pushes changes.
- **Live elapsed timer is gated.** The 1-second `setInterval` only runs when the Running section is expanded; collapsing it clears the interval (battery on iPhone 16 Pro).
- **Wake-lock.** `navigator.wakeLock.request('screen')` is acquired on page load. A pill near the connection dot shows **Screen lock: on** (green) or **off** (gray). On `visibilitychange` to `visible`, the lock is re-acquired — iOS Safari drops it when the tab backgrounds. If the browser doesn't support the API, the pill reads **Screen lock: unsupported** without blocking the page.
- **Reconnect UX.** When Firebase reports `.info/connected === false`, the connection dot turns red, an amber **Reconnecting…** banner appears at the top, and a **Retry now** button forces a full re-init (`firebase.app().delete()` + new `initializeApp` + new anonymous sign-in). The SDK auto-retries in the background; the banner shows attempt count and auto-dismisses when the dot flips back to green.
- **Event picker.** Opening `staff.html` with no `?event=` query parameter shows a list of every event in `/eventIndex/`, sorted by `updatedAt` desc. Each row shows event name, date, competitor count, and a `· read-only` suffix when the event is locked. Tapping a row navigates to `staff.html?event=<id>`. The footer of the live staff view also has a **switch event** link back to the picker.
- **Theme parity.** Staff view uses the same CSS custom properties as the operator app (`--bg`, `--sur`, `--pri`, `--suc`, `--wrn`, `--dan`, etc.), so the staff phone matches the operator laptop. Mobile-first sizing: `viewport-fit=cover`, safe-area insets respected, minimum 56px touch targets.

### What changed in Block 2 (revision, 2026‑05‑23)

- **Multi-event library.** New top-level Firebase node `/eventIndex/<eventId>` carries small metadata for every event the operator has touched (`name`, `date`, `day1`, `day2`, `cxCount`, `mode`, `createdAt`, `updatedAt`). The operator can hold up to ~8 events live at once (4/year × 2 years of retention) without touching the full `events/<id>/` subtree. Every `fbWriteCx()` and `fbWriteCfg()` call now also writes the index entry, so the library stays in lockstep with reality.
- **Event Library panel on Event Config.** Replaces the old Session row. Shows the current event card (eventId, name, competitor count, mode, last updated as `relTime()`), a **Mode** dropdown (`edit` / `readonly`), and **Switch Event**, **New Event**, and **Delete This Event** buttons. The JSON **Save Session to File** / **Load Session from File** buttons moved into this panel.
- **Three new modals.**
  - **Switch Event** — table of every event in `/eventIndex/`, current event marked, others have a Switch button. Switching swaps the active event ID in `cfg`, re-points the Firebase listeners, and re-renders without a page reload.
  - **New Event** — operator enters a new event ID (validated against `validEventId()` and checked for collision against `/eventIndex/`) plus a name. Creates an empty event with default cfg and writes the index entry.
  - **Delete This Event** — 2-step paranoid flow. Step 1 requires the operator to type both the **Event ID** and **Event Name** exactly (case-sensitive) before Continue enables. Step 2 is a backup gate: **Export JSON First** (downloads the session envelope) or **Delete Permanently**. After delete, a 15-second undo bar appears with an in-memory snapshot (`_delEvSnapshot`) that can restore the event in full.
- **Per-event read-only mode (Level B).** `cfg.mode = 'edit' | 'readonly'` is stored per-event in Firebase. In `readonly`:
  - Timing buttons (**Start**, **Finish**, **DNF**, **Obstacle ±**) are hidden via the `.ro-hide` class (`body.readonly .ro-hide{display:none}`).
  - Check-in status controls are disabled.
  - CSV import section and the Clear All Data button are hidden.
  - Inline field edits (name/email/squad/paid) **remain enabled** — this is the deliberate Level B carve-out so the operator can fix data after the event closes.
  - A persistent amber `#ro-banner` reading "Read-only mode — timing is frozen for this event" is rendered inside `<main>`.
  - Switching to `readonly` is one click; `readonly` → `edit` requires a confirmation prompt.
  - JS guards in every destructive handler (`doStart`, `doFinish`, `doDNF`, `adjObs`, `setCi`, `confirmClear`, `impCSV`) toast "Read-only mode — …" and bail before mutating state, so the lock holds even if a stale DOM element slips through CSS.
- **Silent migration.** On first load of an event that has no `/eventIndex/` entry, `syncCfgToUI()` writes one from the current `cfg` so the legacy single-event install upgrades without operator intervention.
- **Session envelope bumped to `version: 4`.** Same shape as v3; the version number is the migration signal. Loaders accept v3 and v4 envelopes.

### What changed in Block 1.5 (revision, 2026‑05‑22)

- **Seed data removed.** No more hardcoded `loadSample()` 46-row dataset; the app boots with an empty roster. Every data screen renders an empty-state card with **Import CSV** and **Download Template** buttons when `cx.length === 0`.
- **CSV import is now header-driven, not positional.** Required headers: `first, last, division, day, squad`. Optional: `shirt, phone, email, address, paid, pmm link`. Header detection is case-insensitive and accepts aliases (e.g. `First Name`, `FIRST`, `first_name`, `firstname` all resolve to `first`). Column order does not matter. Missing required headers → hard reject with row-level error log. See §5.1.
- **`approval` field removed.** Dropped from data model (`Competitor PII object`), Roster screen column, CSV format, and session-export envelope. No migration step is required — Firebase nodes that still carry it will simply ignore the field on next write.
- **Paid normalization.** Import normalizes any case (`paid`, `PAID`, `Paid`, `yes`, `1`) to canonical `Paid`/`Unpaid`/`Waived`. Fixes Block 1 bug where lowercase `unpaid` rendered as `Paid` in the Check-In dropdown because no option matched.
- **Download Template button** on Event Config emits `rng-ops-roster-template.csv` (headers only, no rows). Replaces the old Load Sample Data button.

### What changed in v2.3 (2026‑05‑22 prior revision)

- Calendar dates removed from the roadmap. Plan is now a sequenced block plan with no fixed deadlines; user has ample runway before the next event.
- Operator email locked to `rng.ops.operator@gmail.com` (Block 4 creates this account).
- §4 retitled "Sequenced Block Plan" — blocks remain ordered, each has a "Done when" gate, but no date ranges.
- §14 "Target" file structure no longer dated.
- Event-day go-live checklist no longer dated.

### What changed in v2.2

- Three-file architecture confirmed; "single file" hard constraint replaced with "inline-everything-per-file, no bundlers, no build step."
- Stale `enablePersistence()` claims removed (the call doesn't exist in shipped code).
- Runtime credential entry removed; Firebase config hardcoded only.
- Event ID model switched from auto-year to operator-named explicit ID.
- PII split into a separate Firebase node behind operator-only email/password auth.
- Auto-snapshot to Firebase every 5 minutes (Block 5).
- Block sizes re-fit to a 9-week feature-work / 4-week cushion shape (Sizing B).
- Design System inlined (§15) — no more dangling v1 references.
- Working agreement aligned to 1–4 questions per Space instructions.
- Event-day checklist updated to reflect current reality.

---

## 1. Context & Deployment

- **Stack per file:** inline HTML, CSS, vanilla JS in each `.html` file. Firebase compat SDK (v10.12.2) loaded via three CDN script tags (`app-compat`, `database-compat`, `auth-compat`).
- **Hosting:** GitHub repo [`DRT-cloud/RNG_OPS`](https://github.com/DRT-cloud/RNG_OPS) → Netlify auto-deploy from `main`. Root URL serves the operator app via `netlify.toml` 200-status redirect. Auto-publishing on.
- **Primary operator:** one laptop at the timing/start line, full browser, online via event wifi or hotspot. Authenticated via email/password (Block 4).
- **Secondary devices (first-class):** read-only iPhone 16 Pro for staff dashboard; read-only iPhone 16 Pro for competitor board. Anonymous auth, scoped reads only.
- **Offline behavior:** no browser-side persistence; all state is in-memory or in Firebase. Brief disconnects (<60s) are handled by the RTDB SDK's automatic write queue and reconnect resync. Prolonged outages are covered by JSON session export (Block 1) and auto-snapshots (Block 5).
- **Theme:** dark mode default, light mode toggle, system preference respected on load.

### Hard constraints (do not break)

- **No bundlers, no frameworks, no build step.** Plain HTML/CSS/JS served as-is by Netlify.
- Each app file is **inline-everything** — its HTML, CSS, and JS live in one file. No shared modules across files. Cross-file consistency is maintained by hand (with help from the release checklist).
- Vanilla JS only. DOM APIs, no external JS libraries beyond Firebase compat SDK.
- Operator app must preserve existing patterns: `cx` competitor array, `cfg` config object, `renderAll()` flow, bibs derived (never persisted as authoritative state).
- Operator app must remain usable on a single laptop with zero secondary devices connected.
- Mobile views must remain read-only — no edit controls in the DOM at all.

---

## 2. Event Structure

- Two-day event: **Friday** and **Saturday** nights.
- Each night has a **configurable start time** (default 21:30 for both).
- Competitors are organized into **Squads** (numbered groups).
- Slots per squad derived from `floor(3600 / interval_seconds)`.
- Default release interval: **5:00** (300 seconds). Adjustable in real time.
- **Bib format:** `SSXX` — two-digit squad + two-digit slot (e.g., Squad 3, Slot 7 = `0307`).
- Special squad ranges:
  - Squads 10–19 = **STAFF** (tagged, excluded from results)
  - Squads 20–29 = **SPONSOR** (tagged, excluded from results)
  - All other squads = regular competitors

### Divisions

- `2-GUN`
- `NV 2-GUN` (night vision)
- `PCC` (pistol-caliber carbine)
- `NV PCC` (night vision PCC)

---

## 3. Implemented Features (Phase 1 + 1.5 + 2 + 3 — shipped)

### Phase 1 (operator app)

1. Single-screen run control — Start and Finish on the same row.
2. Bib auto-assignment at check-in; format `SSXX`.
3. Adjustable release interval (live global control; applies to unstarted competitors only).
4. Scheduled start time per row, auto-calculated from interval + slot.
5. Actual start timestamp via `Date.now()`.
6. Actual finish timestamp via `Date.now()`.
7. Obstacle penalty counter ±1 per row; each obstacle = configurable seconds (default 5:00).
8. DNF flag; blocks finish timestamp while set.
9. Check-in status: Pending / Checked In / Late / No Show.
10. Inline field edits on Check-In: Last, First, Division, Squad, Paid.
11. Change warning badge (`!`) on edited rows; click to see field-level diffs across all screens.
12. Squad grouping with header rows (squad number, type, day, slot fill count) on every screen.
13. Empty/open slot rows on Run Scoring.
14. Day filter (All / Friday / Saturday) in topbar.
15. Search filter (name/bib) on Check-In and Run Scoring.
16. CSV import (header-driven; required: `first, last, division, day, squad`; optional: `shirt, phone, email, address, paid, pmm link`). See §5.1.
17. CSV export.
18. Clear All Data with confirmation modal + 15-second undo.
19. Midnight-crossing elapsed-time handling.
20. Live on-course timer with color shifts at 1:30:00 (warn) and 2:00:00 (alert).
21. Row contrast: 1px divider between every data row.
22. Dark/Light mode toggle.

### Phase 1.5 (Firebase, shipped 2026‑05‑19/20)

- Real-time sync of `competitors` and `config` nodes via `on('value')`.
- Per-edit writes via `fbWriteOne(c)`; bulk writes via `fbWriteCx()`.
- Sidebar connection dot — green (online) / amber (syncing) / red (offline or unauthenticated).
- Firebase config hardcoded in `rng-ops-firebase.html` (FIREBASE_CONFIG block).
- Event ID currently auto-derived from `slug(cfg.name) + '-' + new Date().getFullYear()` — superseded by Block 1 explicit Event ID (§7).
- **Anonymous auth** on every page load; DB rules require `auth != null`.

### Phase 2 (Multi-event library + read-only mode, shipped 2026‑05‑23)

- **Top-level `/eventIndex/<eventId>` Firebase node** — metadata-only registry of every event the operator has touched. Written on every `fbWriteCx()` and `fbWriteCfg()`. Schema: `{ name, date, day1, day2, cxCount, mode, createdAt, updatedAt }`.
- **Event Library panel on Event Config** — current event card, mode dropdown, Switch / New / Delete buttons, JSON Save/Load to file.
- **Switch Event modal** — lists all events from `/eventIndex/`, swaps active event ID without page reload.
- **New Event modal** — validates ID against `validEventId()` and collisions, creates empty event with default cfg.
- **Delete This Event modal** — 2-step gate (type Event ID + Event Name exactly, then Export JSON First or Delete Permanently). 15-second in-memory undo.
- **Per-event read-only mode (Level B)** — `cfg.mode` toggles timing buttons / CSV import / Clear All off via `.ro-hide` + `body.readonly` CSS, while leaving inline field edits enabled. Persistent amber banner. JS guards on `doStart`, `doFinish`, `doDNF`, `adjObs`, `setCi`, `confirmClear`, `impCSV`.
- **Silent eventIndex migration** — first load of a pre-Block-2 event writes its index entry automatically.
- **Session envelope bumped to `version: 4`**. Loaders still accept v3.

### Phase 3 (Staff view + reconnect/wake-lock polish, shipped 2026‑05‑25)

- **Shared Firebase config** — `rng-fb-config.js` is loaded by both `rng-ops-firebase.html` and `staff.html`. Single source of truth; the inline `FIREBASE_CONFIG` literal and dead `firebaseConfig` literal in the operator file were removed.
- **Staff view (`staff.html`)** — mobile-first read-only page at `staff.html?event=<eventId>`. Anonymous auth, no edit DOM, no write calls.
- **Three collapsed sections** — On Deck (`ci === 'in' && !st`), Running (`st && !fi && !dnf`), Done (`fi || dnf`). All closed on load. Live counts. DNF (red) vs DONE (green) pill on Done rows; obstacle badge `+N` when penalties exist; final time = `(fi-st) + ob*penalty*1000` formatted `m:ss`.
- **Wake-lock with status pill** — `navigator.wakeLock.request('screen')` on load. Pill shows on/off/unsupported. Re-acquires on `visibilitychange`.
- **Reconnect UX** — amber banner with **Retry now** button when `.info/connected === false`. SDK auto-retries in background; banner auto-dismisses on green. Retry now button forces a full re-init.
- **Live elapsed timer** — 1 s interval, gated on the Running section being expanded; cleared when collapsed.
- **Event picker** — `staff.html` with no `?event=` shows a list of every event in `/eventIndex/` sorted by `updatedAt` desc.
- **Theme parity** — staff page uses the same CSS custom properties (`--bg`, `--sur`, `--pri`, `--suc`, `--wrn`, `--dan`) as the operator app.

---

## 4. Sequenced Block Plan

No fixed dates. Blocks are executed in order; each has a "Done when" gate that must pass before the next begins. Work respects user's hard rules: no work after 6 PM, Monday = planning, Friday = low cognitive load, weekends = personal projects. Block 6 (synthetic dry run) and Block 8 (real-world dress rehearsal) gate event readiness — neither can be skipped.

### Block 1 · Race-day safety items

- **Explicit Event ID** (§7) — replace auto-year derivation with operator-named ID on Event Config. Add validation: `[a-z0-9-]+`, max 60 chars. Stored in `cfg.eventId`. Default falls back to `slug(cfg.name)` + current year for backward compat with existing data.
- **JSON session export/import** — offline failover. "Save Session" + "Resume Session" buttons on Event Config. Envelope: `{version, exportedAt, eventId, cfg, cx}`. Resume confirmation modal shows imported eventId vs current. Write-through to Firebase on resume.
- **Quoted-field CSV parser** — replace `impCSV()` naive `split(',')` with state-machine handling `"quoted,values"`, escaped `""`, and CRLF. Friday low-load task.

**Done when:** Round-trip session export preserves all timing state; test CSV with comma-bearing addresses imports correctly; new event IDs survive a calendar-year rollover.

### Block 2 · Staff View MVP

- Create `staff-view.html` — separate file, mobile-first, inline-everything, ~10KB target.
- Firebase anonymous auth + read-only listeners on `events/<id>/competitors` and `events/<id>/config`.
- **No-event-yet state:** if `competitors` is empty or the eventId in the URL has no data, render full-screen "Waiting for event…" with a sync indicator and the event ID being watched.
- Layout: event header · status counts (Checked In / On Course / Finished) · On Course live list with elapsed timer + color thresholds (1:30 warn / 2:00 alert) · Next Up panel · sticky footer with sync dot.
- URL format: `rng-ops.netlify.app/staff-view.html?event=<eventId>`.

**Done when:** Open the staff URL on iPhone 16 Pro — state mirrors operator app within 1s of any change.

### Block 3 · Staff View ~~polish~~ (shipped 2026‑05‑25)

Shipped as `staff.html` + shared `rng-fb-config.js`. Detail in §5 (Staff view) and the Block 3 changelog at the top of this document.

- Three collapsed sections (On Deck / Running / Done), tap-to-expand, all closed by default.
- Theme parity with operator app via shared CSS custom properties.
- DNF (red) vs DONE (green) pill on Done rows; obstacle count badge `+N` on rows with penalties.
- Wake-lock acquired on load with status pill near connection dot; re-acquires on `visibilitychange`.
- Reconnect: amber banner with **Retry now** button forces a full re-init; SDK also auto-retries in background; `.info/connected` flip dismisses banner.
- Event picker when `?event=` is missing, sorted by `updatedAt` desc, lists every event in `/eventIndex/`.

**Done when:** Staff could run a full event night without complaints. — Met (Netlify preview verified by operator).

### Block 4 · Operator Auth + PII Lockdown (4a only — 4b deferred)

**Scope decision (2026-05-25):** competitor view (4b) deferred to Block 5+ until the operator has run 1–2 real events and refined the UI/behavior. This block ships 4a only.

**4a. Operator email/password auth + PII split — IMPLEMENTED**

- Email/password provider enabled in Firebase Console; operator account `rng.ops.operator@gmail.com` (UID `Te9HFmyA1cREPARflULb17jstNI3`) created.
- Operator app: replaced anonymous sign-in with email/password. Full-screen sign-in modal (`#mo-signin`) gates all DB access; appears on first load when no cached session and after sign-out. Persistence is `firebase.auth.Auth.Persistence.LOCAL` — session survives page reloads and browser close.
- "Account" section at top of Event Config tab shows the signed-in email and a "Sign out" button. Sign-out shows a `confirm()` dialog ("Sign out? You'll need to sign back in to make changes.") to prevent accidental sign-out mid-event.
- On sign-out: listeners detach, `fbDb`/`fbRef` null out, in-memory `cx` clears, modal reappears. Sign-in cleanly re-attaches listeners.
- PII fields (`phone`, `email`, `address`, `paid`, `pmm`, `shirt`) moved from `events/<id>/competitors/<id>` to `events/<id>/private/<id>`. Defined in code as `PII_FIELDS` constant.
- `splitCxRecord(c)` / `mergeCxRecord(pub, priv)` helpers handle the public/private split and merge.
- `fbWriteCx()` and `fbWriteOne()` use multi-path `update()` so public and private halves write atomically. `fbDeleteOne()` added for symmetric removal from both nodes.
- `bindEventListeners()` attaches a third listener on `private/`. Two in-memory caches (`_pubCache`, `_privCache`) are merged on every change into the unified `cx` array — existing UI code is unchanged.
- Listener-driven UI updates are suppressed (`_migrationRunning` flag) while migration is in flight.
- Auto-migration (`runPiiMigrationOnce(uid)`) runs once per device per operator UID on first successful sign-in. Reads `/eventIndex/`, iterates all events, and for each cx record at `competitors/<id>` that still has any PII field, multi-path-updates the PII to `private/<id>/<field>` and nulls the source field. Idempotent: re-running skips already-empty records. Success flag stored in `localStorage` as `rng_pii_migrated_v1_<uid>`. On failure the flag is not set so it retries on next sign-in. Toast on success when records moved.
- Clear-all-data (`confirmClear`) and event-restore (`undoDeleteEvent`) updated to operate on both `competitors/` and `private/` nodes.
- Spec §10 DB rules tightened to `auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'` on writes and on the `private/`/`access/`/`snapshots/` subtrees. **Rules are NOT published until the operator has signed in once on the Netlify preview and verified migration completed.**

**4b. Competitor view — DEFERRED**

Deferred to Block 5+ until the operator runs 1–2 real events and refines requirements. Spec text retained below for reference.

- Create `competitor-view.html`.
- **Passphrase gate** — operator sets passphrase on Event Config; stored at `events/<id>/access/competitorPassphrase` as SHA-256 hash. Competitor view's gate hashes typed input, compares, stores `'unlocked'` in `localStorage` keyed to event ID. **Note: the passphrase is UX friction, not security — the DB rules are what actually protect PII.**
- Renders full first + last name (matches posted start list). Reads only `events/<id>/competitors` — PII is on a separate node the competitor view's anon user can't read.
- Layout: event header · "Next 5 starts" with live countdowns · "Find my bib" search · "Show all by squad" toggle.
- **No-event-yet state:** same "Waiting for event…" pattern as staff view.
- URL format: `rng-ops.netlify.app/competitor-view.html?event=<eventId>`.

**Done when (4a only):** Operator signs in with email/password; PII is on `private/` node after migration; staff view continues to work (already PII-free); tightened DB rules deny `private/` reads to anyone but the operator UID.

### Block 5 · Competitor View polish + Auto-snapshot

- Live leaderboard toggle (during/after event).
- Squad/division filter.
- "My squad on deck" flash when squad enters the start window.
- Sticky footer with last-sync time.
- **Auto-snapshot** — operator app writes a full session envelope to `events/<id>/snapshots/<ms-timestamp>` every 5 minutes, and immediately before any clear-all or bulk import.
- "Restore from snapshot" UI on Event Config — modal lists the most recent 10 snapshots with timestamps, restore picks one, replaces state, writes through to Firebase.
- Auto-prune snapshots beyond the most recent 50.

### Block 6 · Synthetic dry run

- Import real-event test CSV via header-driven import (see §5.1).
- Laptop (operator, email/password) + iPhone 16 Pro (staff) + iPhone 16 Pro (competitor, passphrase) all on same event ID.
- Walk through a fake 90-minute event covering: check-in, mass start, mid-event laptop refresh (state should survive), mid-event simulated wifi drop, finish processing, results display.
- Compile fix list. The fix list itself isn't part of this block — it bleeds into Block 7.

**Done when:** All blocker-severity items from the dry run are logged with owner + fix plan.

### Block 7 · Fix list + secondary items

- Burn down dry-run fixes.
- Pick up deferred UI improvements (operator app polish — see §5 list).
- **Print start list + print results** (`@media print` CSS in operator app, page breaks between squads / divisions).

**Done when:** All Block 6 blockers closed; print views verified.

### Block 8 · Real-world dress rehearsal

- Run the system at a smaller event or closed practice session — identify a candidate event during Block 5.
- Real venue, real wifi, real glare on phone screens.
- Final fix list.

**Done when:** Dress rehearsal completes without operator-app crash, data loss, or PII leak; final fix list closed.

### Block 9 · Event-day readiness

- **Code freeze** — 7 days before event, only show-stopper fixes after freeze.
- Pre-event checklist (§13 event-day go-live) executed the day before event.
- Event day: go.

**Done when:** §13 event-day go-live checklist passes end-to-end.

### Deferred UI improvements (operator app — Block 7 candidates)

- Button placement / color review on Run Scoring (Start/Finish/DNF clarity at arm's length).
- Visual cue for on-course vs finished vs DNF rows (background tint, not just badge).
- Live timer typography (tabular numerals, larger at warn/alert thresholds).
- Next-Up panel layout (countdown prominence, fewer fields).
- Empty-slot row contrast in dark mode.
- Topbar density on narrow screens.

### Explicitly out of scope for v2.2

- Helper edit roles (e.g., marshal who can tap Finish on assigned competitors). Laptop is the only edit surface.
- Big-screen "TV mode" — staff view at full-screen on a tablet covers it.
- Persistence beyond Firebase + session JSON + auto-snapshots.
- Squad drag-to-reorder, stage scoring, wait-time credit, PDF export.

These remain on the long-term backlog and are not blockers for 2026‑10‑21.

---

## 5. Screen Map

### Operator app (laptop, full functionality)

| Screen | DOM ID | Purpose |
|---|---|---|
| Check-In | `#page-checkin` | Pre-event verification, inline edits, status set |
| Run Scoring | `#page-run` | Live timing — start, finish, obstacles, DNF, edit |
| Results | `#page-results` | Live standings, group by division or squad |
| Roster | `#page-roster` | Read-only reference, sortable, searchable |
| Event Config | `#page-config` | Event setup, **Event Library (switch/new/delete, mode dropdown, JSON save/load)**, data import/export, passphrase, snapshots, clear-all, sign-out |

### Staff view (iPhone 16 Pro, read-only) — shipped Block 3

File: `staff.html`. Mobile-first scrollable page. No edit controls, no input fields, no `contenteditable` elements. Anonymous auth via shared `rng-fb-config.js`. Subscribes to `events/<eventId>/competitors` and `events/<eventId>/config` only.

**URL contract**

- `staff.html?event=<eventId>` — live staff view for that event.
- `staff.html` with no `?event=` — picker listing every event in `/eventIndex/`, sorted by `updatedAt` desc.

**Header** (sticky)

- **RNG OPS staff** brand mark.
- **Screen lock** pill — green `Screen lock: on` when wake-lock is active, gray `off` when released, `unsupported` on browsers without the API.
- **Connection dot** — green (live) / amber (signing in / connecting) / red (offline). Pulled from `.info/connected`.
- Event line below: event name + event ID pill.

**Reconnect banner** (only when `.info/connected === false`)

- Amber sticky banner above the header: `Reconnecting… (attempt N)`.
- **Retry now** button calls `retryNow()` — detaches listeners, deletes the Firebase app, re-initializes, re-signs anonymously, re-binds the event. Useful when the SDK is stuck (rare, but happens on long iOS backgrounding).
- Auto-retries every 5 s in the background; banner auto-dismisses on green.

**Three sections**

All closed by default per operator preference. Tap header to toggle.

| Section | Predicate | Row content |
|---|---|---|
| **On Deck** | `c.ci === 'in' && !c.st` | bib · last, first · division · squad |
| **Running** | `c.st && !c.fi && !c.dnf` | bib · last, first · division · squad · live elapsed `m:ss` · start clock |
| **Done** | `c.fi \|\| c.dnf` | bib · last, first · division · squad · obstacle badge `+N` · **DONE** or **DNF** pill · final time · finish clock |

Section counters tick live. Final time on Done rows = `(fi - st) + ob * cfg.penalty * 1000`, formatted `m:ss`.

**Live elapsed timer**

- 1 s `setInterval` updates `.elapsed` cells inside the Running section.
- Started only when the Running section is expanded; cleared when collapsed. Reduces battery draw on a phone that may sit on the staging table for 3 hours.

**Wake-lock**

- `navigator.wakeLock.request('screen')` acquired on `load`.
- `visibilitychange` re-acquires when the tab returns to visible (iOS Safari drops it on background).
- Pill near the connection dot reflects state.

**No-event-yet state:** event picker renders when `?event=` is absent. Empty `/eventIndex/` renders "No events yet. The operator hasn’t created any."

**Out of scope for Block 3** (deferred to later polish if requested):

- Scheduled-vs-actual start delta on rows.
- Color thresholds on elapsed times.
- Live leaderboard / rank display.

### Competitor view (iPhone 16 Pro, public-facing, read-only)

Anonymous auth + passphrase gate. Reads only `competitors`, not `private/`. Sections:

- Event header — name, day
- Next 5 starts — live countdowns
- Find my bib — search
- Show all by squad — toggle
- Live leaderboard (during/after event) — toggle

**No-event-yet state:** "Waiting for event…" with the watched event ID visible.

---

## 6. Data Model

### Competitor object — PUBLIC (operator app `cx[]`; Firebase `events/<id>/competitors/<id>`)

```javascript
{
  id: Number, bib: String,        // bib derived
  first: String, last: String,
  div: String,        // '2-GUN' | 'NV 2-GUN' | 'PCC' | 'NV PCC'
  day: String,        // 'Friday' | 'Saturday'
  squad: Number, slot: Number,
  ciStatus: String,   // 'Pending' | 'Checked In' | 'Late' | 'No Show'
  startMs: Number|null, finishMs: Number|null,
  obstacles: Number, dnf: Boolean,
  _origLast, _origFirst, _origDiv, _origSquad, _origPaid,
  _changes: Array     // [{field, from, to}]
}
```

### Competitor PII object — PRIVATE (operator app `cxPriv{}`; Firebase `events/<id>/private/<id>`)

```javascript
{
  id: Number,                     // FK to competitors.id
  phone: String, email: String, address: String,
  shirt: String,
  paid: String,                   // 'Paid' | 'Unpaid' | 'Waived'
  pmm: String
}
```

`approval` was removed in Block 1.5 — it was set but never displayed or filtered. Existing Firebase records that still carry the field are ignored on next write.

### Config object

```javascript
{
  eventId: String,         // operator-named, slugified, e.g. 'twilight-2026'
  name: String,            // human-readable, e.g. 'Twilight Biathlon 2026'
  eventDate: String,       // 'YYYY-MM-DD' (Block 2; surfaces in Event Library card)
  day1: String,            // 'YYYY-MM-DD' (Block 2; Friday date)
  day2: String,            // 'YYYY-MM-DD' (Block 2; Saturday date)
  mode: String,            // 'edit' | 'readonly' (Block 2; default 'edit')
  friStart: String,        // 'HH:MM'
  satStart: String,        // 'HH:MM'
  interval: Number,        // seconds
  penalty: Number          // seconds per obstacle
}
```

`mode === 'readonly'` freezes timing/CSV/Clear via the `.ro-hide` CSS class and the JS guards in destructive handlers. Inline field edits remain enabled. See §7.1.

### Session export envelope (Block 1, also used for snapshots in Block 5)

```javascript
{
  version: 4,                     // bumped from 3 in Block 2; v3 envelopes still load
  exportedAt: Number,             // Date.now()
  eventId: String,
  cfg: { ... },                   // includes cfg.mode after Block 2
  cx:  [ ... public competitor fields ],
  cxPriv: { id: {...}, ... }      // present only if operator-exported; absent for snapshot reads by non-operator
}
```

`SESSION_VERSION = 4` in `rng-ops-firebase.html`. The shape is unchanged from v3 — the bump signals that `cfg.mode` is now part of the envelope payload.

### Firebase tree (Block 4 target, with Block 2 additions)

```
eventIndex/                   all authed clients can read (Block 2)
  <eventId>: {
    name, date, day1, day2,
    cxCount, mode,
    createdAt, updatedAt
  }
events/
  <eventId>/
    competitors/              all authed clients can read
      <id>: { public fields }
    private/                  operator only
      <id>: { PII fields }
    config/                   all authed clients can read; includes mode (Block 2)
    access/                   operator only
      competitorPassphrase: "<sha256-hex>"
    snapshots/                operator only
      <ms-timestamp>: { full envelope }
```

`/eventIndex/` is the cheap registry that powers the Switch Event modal without dragging every event's full `competitors/` payload over the wire. Written on every `fbWriteCx()` and `fbWriteCfg()` via `fbWriteEventIndex()`; read once on Event Config open via `fbReadEventIndex(cb)`.

### CSV import format (header-driven, since Block 1.5)

**Required headers** (all must be present after alias normalization; case-insensitive):

`first, last, division, day, squad`

**Optional headers** (silently filled blank if absent):

`shirt, phone, email, address, paid, pmm link`

**Column order does not matter.** The importer reads the header row, maps each canonical field to a column index via `mapColumns()`, and pulls cells by index. If any required header is missing, import is rejected before any row is processed and the error log lists the missing fields.

PII columns (`shirt`, `phone`, `email`, `address`, `paid`, `pmm`) flow into `cxPriv` on import after Block 4. `first`, `last`, `division`, `day`, `squad` remain on the public node.

A **Download Template** button on Event Config emits `rng-ops-roster-template.csv` containing the canonical header row (no data rows), suitable for distributing to registration helpers.

### 5.1 Header aliases & paid normalization (Block 1.5)

Incoming headers are normalized before lookup: lowercased, trailing ` name` stripped, internal spaces / underscores / hyphens removed. Lookup table:

| Canonical field | Required | Accepted normalized aliases                       |
|-----------------|----------|---------------------------------------------------|
| `first`         | Yes      | `first`, `firstname`                              |
| `last`          | Yes      | `last`, `lastname`, `surname`                     |
| `div`           | Yes      | `division`, `div`, `class`                        |
| `day`           | Yes      | `day`, `raceday`                                  |
| `squad`         | Yes      | `squad`, `squadnumber`, `squad#`                  |
| `shirt`         | No       | `shirt`, `shirtsize`, `size`                      |
| `phone`         | No       | `phone`, `phonenumber`, `mobile`                  |
| `email`         | No       | `email`, `emailaddress`                           |
| `address`       | No       | `address`, `streetaddress`                        |
| `paid`          | No       | `paid`, `feestatus`, `payment`                    |
| `pmm`           | No       | `pmmlink`, `pmm`                                  |

Examples that all resolve to `first`: `First Name`, `FIRST`, `first_name`, `firstname`, `First`. Examples that all resolve to `squad`: `Squad`, `SQUAD`, `Squad #`, `squad number`.

**Paid normalization** (`normalizePaid()`):

- `paid`, `yes`, `y`, `true`, `1` → `Paid`
- `unpaid`, `no`, `n`, `false`, `0` → `Unpaid`
- `waived`, `comp`, `complimentary`, `free` → `Waived`
- Blank → `Unpaid`
- Anything else → original value with first letter capitalized (preserved so the operator sees their input)

---

## 7. Event ID Model

**Status as of v2.2:** changing in Block 1.

### Today (problematic)

```javascript
fbEventId = slug(cfg.name) + '-' + new Date().getFullYear();
```

Year flips on Jan 1; same name + same year collides; depends on system clock.

### Block 1 target

- New `cfg.eventId` field on Event Config — operator-entered, free text.
- Validated: lowercase alphanumeric + hyphens, max 60 chars. Live-validated as the operator types.
- On save: written to Firebase at `events/<eventId>/config/eventId`.
- If `cfg.eventId` is empty when the app first loads, fall back to the legacy derivation for backward compat with existing Firebase data.
- Mobile views read the eventId from the URL `?event=<eventId>` parameter; operator app reads from `cfg.eventId`.

**Pros:** unambiguous, stable across calendar boundaries, operator-controlled.
**Trade-off:** one more field to set during event setup. Lives at the top of Event Config.

---

## 7.1 Event Lifecycle (Block 2)

The operator now holds multiple events in a single Firebase database. Lifecycle operations live in the Event Library panel on Event Config.

### Active event

- The active event is whichever `cfg.eventId` is currently loaded. All Firebase listeners (`competitors`, `config`) are scoped to `events/<eventId>/`.
- The current event card in the Event Library shows `eventId`, `name`, competitor count (live from `cx.length`), mode, and `relTime(updatedAt)` from the `/eventIndex/` entry.

### Switch Event

- `openSwitchEventMo()` calls `fbReadEventIndex(cb)` and renders every entry as a row in `#mo-switch-ev`. The current event is marked, all others get a **Switch** button.
- `confirmSwitchEvent(id)` updates `cfg.eventId`, detaches the current Firebase listeners, re-binds against `events/<newId>/`, and triggers `renderAll()`. No page reload.

### New Event

- `openNewEventMo()` prompts for a new event ID and name.
- `validateNewEventId()` runs `validEventId()` (`^[a-z0-9-]{1,60}$`, no leading/trailing hyphen) and checks the typed ID against `/eventIndex/` for collisions; the Create button stays disabled until both pass.
- `confirmNewEvent()` writes an empty `events/<newId>/` (default cfg with `mode: 'edit'`) and a matching `/eventIndex/<newId>` entry, then switches to it.

### Read-only mode

- `cfg.mode` is persisted in `events/<eventId>/config/mode`. Default `'edit'`.
- The Event Library card has a `<select id="cfg-mode">` with options `edit` / `readonly`; `onModeChange()` handles the toggle.
- Going **edit → readonly** is one click; going **readonly → edit** requires `confirm('Re-enable editing for this event?')` to prevent accidental reopens after results are posted.
- `applyMode()` adds or removes `body.readonly`, which the CSS uses to hide `.ro-hide` elements (timing buttons, CSV import section, Clear All button). It also shows or hides `#ro-banner`.
- `isReadOnly()` returns `cfg.mode === 'readonly'`. Every destructive handler (`doStart`, `doFinish`, `doDNF`, `adjObs`, `setCi`, `confirmClear`, `impCSV`) calls it as its first line and bails with a toast.

### Delete Event

Paranoid because deleting an event drops a year of competitor data.

- **Step 1 — Confirm identity.** Operator must type both the **Event ID** and the **Event Name** exactly (case-sensitive). `validateDeleteEvent()` enables Continue only when both strings match.
- **Step 2 — Backup gate.** Two buttons: **Export JSON First** (calls `exportBeforeDelete()`, which downloads the session envelope before proceeding) or **Delete Permanently** (calls `confirmDeleteEvent()`).
- **Delete.** Snapshots `cfg`, `cx`, and the `/eventIndex/` entry into in-memory `_delEvSnapshot`, then removes `events/<eventId>/` and `/eventIndex/<eventId>` from Firebase. Switches to whichever other event is most recent in the index, or to a clean default if none remain.
- **15-second undo.** `showDeleteUndoBar()` renders a sticky banner with a countdown. `undoDeleteEvent()` writes the snapshot back. After 15s the snapshot is cleared and the delete is permanent. The undo timer is held in `_delEvUndoTimer`.

### Silent migration

A pre-Block-2 install has data under `events/<id>/` but no `/eventIndex/<id>` entry. On the first load after upgrade, `syncCfgToUI()` runs `fbReadEventIndex(cb)` and, if the current event ID is not present, calls `fbWriteEventIndex()` with the current cfg. The legacy event is then visible in the Switch Event modal without manual setup.

---

## 8. Bib Assignment Logic

Function: `assignBibs()`

1. Group competitors by `day + squad`.
2. Within each group, sort by `_wantSlot` (if set) then by `id`.
3. Assign `slot = i + 1`.
4. Set `bib = p2(squad) + p2(slot)`.
5. Delete `_wantSlot` after assignment.

**Bibs are derived on every render** — never persisted as authoritative state.

**Bib change via edit modal:** operator enters target bib; system finds current occupant, sets `_wantSlot` on the moving competitor, shifts all at/below target slot down by one.

---

## 9. Scheduled Start Calculation

```javascript
function schedStart(c) {
  const base = c.day === 'Saturday' ? cfg.satStart : cfg.friStart;
  const [h, m] = base.split(':').map(Number);
  const sec = h*3600 + m*60 + (c.slot - 1) * cfg.interval;
  const hh = Math.floor(sec / 3600) % 24;
  const mm = Math.floor((sec % 3600) / 60);
  return p2(hh) + ':' + p2(mm);
}
```

---

## 10. Auth, Rules, and Scoring

### Auth model (Block 4 target)

- **Operator (laptop):** Firebase email/password sign-in as `rng.ops.operator@gmail.com`. One operator account in Firebase Console → Authentication → Users. Session persists across browser restarts via Firebase's default `LOCAL` persistence. Sign-out button on Event Config.
- **Staff (mobile):** anonymous sign-in on page load. No role.
- **Competitor (mobile):** anonymous sign-in on page load + passphrase gate (UX-only).
- **Operator role marker:** the operator's UID is hardcoded in DB rules as the authority. No Cloud Functions required.

### DB rules (Block 4 target)

```json
{
  "rules": {
    "eventIndex": {
      ".read":  "auth != null",
      ".write": "auth != null && auth.uid === 'OPERATOR_UID_HERE'"
    },
    "events": {
      "$eventId": {
        "competitors": {
          ".read":  "auth != null",
          ".write": "auth != null && auth.uid === 'OPERATOR_UID_HERE'"
        },
        "config": {
          ".read":  "auth != null",
          ".write": "auth != null && auth.uid === 'OPERATOR_UID_HERE'"
        },
        "private": {
          ".read":  "auth != null && auth.uid === 'OPERATOR_UID_HERE'",
          ".write": "auth != null && auth.uid === 'OPERATOR_UID_HERE'"
        },
        "access": {
          ".read":  "auth != null && auth.uid === 'OPERATOR_UID_HERE'",
          ".write": "auth != null && auth.uid === 'OPERATOR_UID_HERE'"
        },
        "snapshots": {
          ".read":  "auth != null && auth.uid === 'OPERATOR_UID_HERE'",
          ".write": "auth != null && auth.uid === 'OPERATOR_UID_HERE'"
        }
      }
    }
  }
}
```

`OPERATOR_UID_HERE` will be replaced with the actual UID once the operator account is created in Block 4.

### Scoring

- `total = elapsedMs(startMs, finishMs) + obstacles * cfg.penalty * 1000`.
- DNF excluded from rank.
- Staff (squads 10–19) and Sponsor (squads 20–29) excluded from rank.

### Change tracking

- First edit of any field stores `_origField` on the competitor.
- Subsequent edits diff against `_origField`, not the previous value.
- Badge clears if the current value matches `_origField` (revert).

---

## 11. Known Risks & Mitigations

| Risk | Severity | Mitigation | Status |
|---|---|---|---|
| Firebase Test Mode rules expire / world-readable | **High** | Anonymous auth + `auth != null` rules | ✅ Closed 2026‑05‑20 |
| Operator-app PII (phone/email/address) visible to all authed clients | **High** | Split to `events/<id>/private/<id>`; operator-only DB rules; email/password auth on operator | Block 4 |
| Wifi drop on event night | **High** | RTDB SDK auto-resync covers brief drops; Block 1 JSON export covers prolonged outages; Block 5 auto-snapshot covers laptop failure | Block 1 / Block 5 |
| Operator laptop hardware failure mid-event | **High** | Auto-snapshot to Firebase every 5 min; "Restore from snapshot" on Event Config from any laptop | Block 5 |
| Naive CSV parser breaks on quoted fields | Medium | Block 1 — quoted-field parser | Block 1 |
| No paper backup of start list | Medium | Block 7 — print views | Block 7 |
| Mobile views show stale data after long backgrounding | Medium | Sticky footer with last-sync timestamp; visible sync dot; auto-reconnect on focus | Block 3 |
| Event ID collision across calendar years | Medium | Operator-named explicit event ID | Block 1 |
| Three-file edits drift out of sync (theme tokens, behavior) | Medium | Release checklist enforces cross-file smoke test before merge | Process |
| Single-file HTML edits risk regressions (e.g., Turn 6 auto-connect bug) | Medium | Per-change branch + Netlify preview deploy + release checklist | Process in place |
| `file://` origin blocks Firebase | Low | Always access via Netlify URL; never open the file directly from disk | Documented |
| Bib collisions during rapid squad edits | Low | `assignBibs()` re-derives on every render | Mitigated |
| Operator forgets to set Event ID before sharing mobile URLs | Low | Live-validate field on Event Config; warn if empty; copy URL only when valid | Block 1 |

---

## 12. Working Agreement (how we collaborate on this app)

Per the Space instructions, when starting any new topic (UI redesign, new feature, refactor) I will:

1. Summarize current understanding of the relevant screen/feature in 3–5 bullets.
2. Ask **1–4 focused questions** to reach ≥99% confidence before any code.
3. Identify which screen(s) and which file(s) are affected, and whether the change is operator UX, data integrity, mobile-view UX, or future phase.
4. Flag any conflict with hard constraints (§1) and propose a constrained alternative.

Code answers will:

- Stay inline-everything per file. No bundlers, no frameworks, no build step.
- Use vanilla JS and DOM APIs only.
- Preserve `cx`, `cfg`, `renderAll()`, derived-bib patterns in the operator app.
- Call out any change that affects timing, bib assignment, ranking, auth, or PII handling, with implications.

---

## 13. Release / Verification Checklist (per change)

### Per change (any file)

- [ ] Change made on a branch, not `main` directly.
- [ ] Netlify preview deploy loaded and tested in Chrome.
- [ ] Operator app: sidebar Firebase dot turns green within 5s of load (signed in or signing in).
- [ ] Operator app: Import the canonical test CSV → competitor count renders across all five screens. Empty-state card appears when roster is cleared.
- [ ] Operator app: start one competitor → live timer ticks → finish → appears in Results.
- [ ] Operator app: DNF one competitor → appears at bottom of Results, unranked.
- [ ] Operator app: edit one field → change badge appears; revert → badge clears.
- [ ] Operator app: CSV export → reimport → identical state (modulo derived bibs).
- [ ] (After Block 1) JSON session export → clear all → import → identical state including timing.
- [ ] (After Block 2) Staff view loaded on iPhone 16 Pro at preview URL — state mirrors operator app within 1s of any change.
- [ ] (After Block 4) Competitor view loaded as anonymous user — DevTools network inspection shows no PII fields readable.
- [ ] (Block 2) Event Library panel shows the current event card with correct count and mode; Switch Event modal lists every event in `/eventIndex/`.
- [ ] (Block 2) Toggle Mode to `readonly` → amber banner appears; Start/Finish/DNF/Obstacle buttons disappear; Clear All and CSV import sections hidden; inline name/email edits still save.
- [ ] (Block 2) Toggle Mode back to `edit` requires confirmation; controls return.
- [ ] (Block 2) New Event modal rejects an existing ID, accepts a valid unique one; created event boots empty and appears in `/eventIndex/`.
- [ ] (Block 2) Delete This Event: typing wrong ID or name leaves Continue disabled; exact matches enable it; Export JSON First downloads a session envelope; Delete Permanently removes the event and shows 15-second undo; undo restores the event in full.
- [ ] (Block 2) Silent migration: clearing `/eventIndex/<currentId>` and reloading writes a new index entry from current cfg.
- [ ] (Block 3) Open `staff.html` on iPhone 16 Pro Safari with `?event=<id>` — all three sections collapsed by default; counts correct.
- [ ] (Block 3) Expand Running → elapsed counter ticks every second; collapse → counter stops (network tab quiet).
- [ ] (Block 3) Screen lock pill shows green `on`; lock the phone for 30 s, unlock, pill re-acquires.
- [ ] (Block 3) Pull network plug → red dot + amber banner appear within 5 s; **Retry now** button forces a re-init; restore network → banner auto-dismisses.
- [ ] (Block 3) Operator marks a finish in the operator app → that row moves from Running to Done on staff phone within ~1 s.
- [ ] (Block 3) Open `staff.html` with no `?event=` → picker lists every event in `/eventIndex/`; tapping one navigates to `staff.html?event=<id>`.
- [ ] Merge to `main` → wait for Netlify Published → smoke test prod URLs (operator, staff, competitor where applicable).

### Event-day go-live checklist

- [ ] Operator signed in with email/password; "Signed in as <email>" visible on Event Config.
- [ ] Event ID set on Event Config (matches the year of the event date).
- [ ] Real competitor CSV imported.
- [ ] Squad structure and bibs verified in Roster.
- [ ] Friday/Saturday start times confirmed.
- [ ] Release interval and obstacle penalty confirmed.
- [ ] Competitor passphrase set and tested.
- [ ] Staff view URL distributed to staff (with event ID parameter).
- [ ] Competitor view URL + passphrase posted via QR code at registration.
- [ ] Dry run: check in 3 competitors, start/finish, verify Results.
- [ ] JSON session exported as pre-event snapshot.
- [ ] Auto-snapshot confirmed firing (check Firebase `snapshots/` node has a recent timestamp).
- [ ] Locked DB rules verified — incognito direct-URL hit on `events/<id>/private/.json` returns Permission denied.
- [ ] Print start list generated and posted.

---

## 14. File Structure

### Current

```
RNG_OPS/
├── rng-ops-firebase.html   ← operator app
└── netlify.toml            ← root redirect → rng-ops-firebase.html
```

### Target (end of Block 4)

```
RNG_OPS/
├── rng-ops-firebase.html   ← operator app (laptop, full functionality, email/password)
├── staff-view.html         ← staff dashboard (mobile, read-only, anonymous)
├── competitor-view.html    ← competitor board (mobile, anonymous + passphrase gate)
└── netlify.toml            ← root redirect; mobile views served at their own paths
```

Each file is inline-everything — no shared modules. Cross-file consistency (design tokens, Firebase config, theme switcher) is maintained by hand. Release checklist §13 enforces a cross-file smoke test before merging.

---

## 15. Design System

Inlined here because each file maintains its own copy. When changing a token, change it in **every file** and run the per-change checklist.

### Color tokens (CSS custom properties)

```css
:root {
  /* Dark mode (default) */
  --bg:           #0f1115;
  --bg-elevated: #161922;
  --bg-row:      #1a1e29;
  --bg-row-alt:  #1f2330;
  --border:      #2a3142;
  --text:        #e7eaf2;
  --text-dim:    #9aa3b8;
  --text-faint:  #6b7488;

  --accent:      #4f8cff;          /* primary actions */
  --accent-dim:  #2c5cc7;
  --good:        #3fb27f;          /* checked in, finished */
  --warn:        #e8a23a;          /* late, change badge, 1:30 timer */
  --alert:       #e8553a;          /* DNF, 2:00 timer, errors */
  --staff:       #b07cd9;          /* squads 10–19 */
  --sponsor:     #d9a55c;          /* squads 20–29 */

  --sync-online:  #3fb27f;
  --sync-syncing: #e8a23a;
  --sync-offline: #e8553a;
}

[data-theme="light"] {
  --bg:           #f7f8fb;
  --bg-elevated: #ffffff;
  --bg-row:      #ffffff;
  --bg-row-alt:  #f0f2f7;
  --border:      #d8dde8;
  --text:        #0f1115;
  --text-dim:    #4a5266;
  --text-faint:  #7d869b;
  /* accent/good/warn/alert/staff/sponsor unchanged */
}
```

### Typography

- Family: **Satoshi** via Fontshare CDN; weights 400, 500, 700.
- Fallback: `system-ui, -apple-system, "Segoe UI", sans-serif`.
- Tabular numerals on timers and time columns: `font-variant-numeric: tabular-nums;`.
- Sizes are fluid: `clamp(min, base + vw, max)` so layouts respond to laptop vs phone without explicit breakpoints.
- Base scale:
  - `--text-xs: clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem)` — meta, labels
  - `--text-sm: clamp(0.875rem, 0.8rem + 0.25vw, 1rem)` — body, table rows
  - `--text-md: clamp(1rem, 0.9rem + 0.3vw, 1.125rem)` — buttons, primary
  - `--text-lg: clamp(1.25rem, 1.1rem + 0.5vw, 1.5rem)` — section headers
  - `--text-xl: clamp(1.5rem, 1.3rem + 1vw, 2.25rem)` — page titles, big counts
  - `--text-2xl: clamp(2rem, 1.6rem + 2vw, 3.5rem)` — staff view status counts, on-course timer

### Spacing

- Scale (4px base): 4, 8, 12, 16, 24, 32, 48, 64.
- Row padding: 8px top/bottom, 12px left/right.
- Section gap: 24px.
- Page padding (operator): 16px; (mobile): 12px.

### Status badges and row tints

- Checked In: `--good` background at 12% opacity, `--good` text.
- Late: `--warn` background at 12% opacity, `--warn` text.
- No Show: `--text-faint` background at 12% opacity, `--text-faint` text.
- On Course (row tint): `--accent` background at 6% opacity.
- Finished (row tint): `--good` background at 6% opacity.
- DNF (row tint): `--alert` background at 8% opacity.
- Change badge `!`: solid `--warn`, white text, 16×16 px round.

### Live timer color thresholds

- `<1:30` → `--text`
- `≥1:30 && <2:00` → `--warn`
- `≥2:00` → `--alert`

### Connection dot states

- Online → solid `--sync-online`, no animation.
- Syncing → solid `--sync-syncing`, 1s pulse.
- Offline / signed out / error → solid `--sync-offline`, no animation.

### Touch targets (mobile views)

- Minimum 44×44 px (Apple HIG floor).
- Row tap target = full row height, never just the text.

---

## 16. CDN Dependencies

```html
<link href="https://api.fontshare.com/v2/css?f[]=satoshi@400,500,700&display=swap" rel="stylesheet">
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-database-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>
```

Same set in all three files. Mobile views can skip `firebase-auth-compat.js` only if they remain anonymous-only — but loading it doesn't cost much, and keeping the set uniform simplifies maintenance.

---

## What you should do next

1. **Test Block 3 on Netlify preview.** Open the `block3-staff-view-polish` preview URL with `staff.html?event=rng-event` on your iPhone 16 Pro. Walk the Block 3 checklist items in §13.
2. **Confirm theme parity** — the staff page should match the operator dark palette exactly.
3. **Then Block 4 — Competitor View + PII Lockdown.** This is the bigger block: creates the operator email/password account, splits PII into `events/<id>/private/`, adds passphrase-gated competitor view, and tightens DB rules.
4. **Operator email still locked.** Block 4 will create `rng.ops.operator@gmail.com` in Firebase Console. No action needed now; do not create the account until Block 4.

---

*Spec v2.5 revised 2026‑05‑25 — Block 3 shipped: staff view (`staff.html`) with 3 collapsed sections (On Deck / Running / Done), wake-lock + visibility re-acquire, reconnect banner with Retry now button, event picker, theme parity; shared `rng-fb-config.js` is now the single Firebase config source.*
