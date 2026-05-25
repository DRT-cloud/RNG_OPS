# RNG OPS — Build Specification (v3.0)

**Last updated**: 2026-05-25
**Operator**: Cody Allenbaugh (`codyallenbaugh@gmail.com`)
**Operator Firebase account**: `rng.ops.operator@gmail.com` · UID `Te9HFmyA1cREPARflULb17jstNI3`
**Production URL**: <https://rng-ops.netlify.app/>
**Repo**: <https://github.com/DRT-cloud/RNG_OPS>

---

## 0. Reading this doc

This spec is the **target state and contract** for what the app does and how. For "what we built last week and what's broken right now," see `status.md`.

The spec is organized so the most stable, reference-y material is at the top and the planning material is at the bottom:

- §1 Identity & deployment
- §2 Surfaces (operator app, staff view; competitor view planned)
- §3 Data model (Firebase tree, record shapes, PII split)
- §4 Auth & DB rules (current, published)
- §5 Event lifecycle (create / switch / delete / restore)
- §6 Scoring & timing rules
- §7 Screen map (per-surface UX contract)
- §8 Design system (color tokens, type scale, components)
- §9 File structure & dependencies
- §10 Working agreement (how Cody + the assistant collaborate)
- §11 Block roadmap (what shipped, what's next, what's deferred)
- §12 Decision log (irreversible design calls and why)

---

## 1. Identity & deployment

**Product**: RNG OPS — operations admin app for Run and Gun (RNG) biathlon events. Optimised for one operator at a timing station plus a handful of staff phones reading live state.

**Hosting**: Netlify, auto-deploy on `main`. `netlify.toml` redirects `/` → `/rng-ops-firebase.html`.

**Backend**: Firebase Realtime Database (project `rng-ops`). No server-side code, no functions, no auth flows beyond Firebase Auth itself.

**Surfaces shipped**:
- `rng-ops-firebase.html` — operator app (single laptop at start/finish line)
- `staff.html` — read-only mobile staff view

**Surfaces planned**:
- `competitor.html` — competitor-facing leaderboard / "find my bib" (deferred until 1–2 events run)

**Auth model in production today**:
- Operator: email/password, persistent local session, single account.
- Staff: anonymous Firebase auth, no human-facing login, open URL.
- Competitor surface (future): anonymous auth + per-event passphrase.

---

## 2. Surfaces

### 2.1 Operator app — `rng-ops-firebase.html`

One file, one laptop, single source of truth for the event. Sign-in modal gates all DB access. Tabs:

- **Check-In** — mark competitors Pending / Checked In / Late / No Show; edit PII fields.
- **Run Scoring** — start releases, capture finishes, mark DNF, count obstacles.
- **Results** — overall and by-division rankings; raw vs penalised time.
- **Event Config** — Account (sign in/out), Event Settings (ID, name, day times, interval, obstacle penalty), Event Library (switch / new / delete), Firebase Connection, Import Competitors (CSV), Session backups.

Read-only mode toggle (set on Event Config) hides all write actions across all tabs without disconnecting Firebase.

### 2.2 Staff view — `staff.html`

Phone-first, read-only, no PII. Anonymous auth (no human login). Reads `eventIndex/`, `events/<id>/competitors/`, `events/<id>/config/` only.

**Sections** (collapsed by default, tap to expand):
- **On Deck** — `ciStatus` is "Checked In" or "Late" AND no `startMs`. Sorted by bib.
- **Running** — has `startMs`, no `finishMs`. Sorted by start time. Live elapsed timer when expanded.
- **Done** — has `finishMs` OR `dnf=true`. Sorted by finish time. DNF gets badge.

**Always-visible disclaimer banner** at top: "NOT FINAL — LIVE TRACKING ONLY" (yellow, 11px, one-line on iPhone SE).

**Reliability features** (Block 3):
- Screen Wake Lock with status pill near the connection dot.
- Reconnect banner with "Retry now" button when `.info/connected` flips to false.
- Event picker when `?event=` is missing — lists every event in `/eventIndex/` sorted by `updatedAt` desc.

### 2.3 Competitor view — `competitor.html` (deferred)

Not built. When built, will be passphrase-gated (per-event passphrase, SHA-256 hashed at `events/<id>/access/competitorPassphrase`), reads only `competitors/` and `config/`, never touches `private/`. Layout sketch: event header · "Next 5 starts" countdowns · "Find my bib" search · "All by squad" toggle · "Live leaderboard" toggle.

---

## 3. Data model

### 3.1 Firebase tree

```
/eventIndex/<eventId>            ← lightweight directory entries
  name, date, day1, day2, cxCount, mode, createdAt, updatedAt

/events/<eventId>/
  config/                        ← event-wide settings
  competitors/<cxId>             ← non-PII (public to any authed user)
  private/<cxId>                 ← PII (operator-only)
  access/                        ← passphrase hashes (operator-only) [planned]
  snapshots/                     ← versioned undo history (operator-only) [planned]
```

`<eventId>` is operator-chosen, validated against `^[a-z0-9-]{1,60}$`, no leading/trailing hyphen.

### 3.2 Competitor record (public — `competitors/<cxId>`)

| Field | Type | Notes |
|---|---|---|
| `id` | number | Stable, opaque |
| `bib` | number | Sortable as integer |
| `first` | string | Required |
| `last` | string | Required |
| `div` | string | "2-GUN" / "NV 2-GUN" / "PCC" / "NV PCC" |
| `day` | string | "fri" or "sat" |
| `squad` | number | 1+ |
| `slot` | number | Position within squad |
| `ciStatus` | string | "Pending" / "Checked In" / "Late" / "No Show" |
| `startMs` | number | epoch ms, null until released |
| `finishMs` | number | epoch ms, null until finish captured |
| `dnf` | boolean | true if competitor did not finish |
| `obstacles` | number | Count of skipped obstacles; each adds `cfg.penalty` seconds |
| `_changes` | array | Per-record audit trail (optional) |

### 3.3 Private record (PII — `private/<cxId>`)

| Field | Type |
|---|---|
| `phone` | string |
| `email` | string |
| `address` | string |
| `paid` | string ("Paid" / "Unpaid" / "Waived") |
| `pmm` | string (payment link) |
| `shirt` | string |

**Rule**: PII fields never appear at `competitors/<cxId>`. Operator app splits writes via `splitCxRecord(c)` → multi-path atomic update. Reads merge via `mergeCxRecord(pub, priv)` so the in-memory `cx[]` array is a single unified record per competitor and the UI code is unchanged.

### 3.4 Config record (`events/<id>/config/`)

| Field | Notes |
|---|---|
| `eventId` | mirrors the node key |
| `name` | display name |
| `eventDate` | event date (display) |
| `friStart` | "HH:MM" — first squad release time on Friday |
| `satStart` | "HH:MM" — first squad release time on Saturday |
| `interval` | "MM:SS" — default release interval between squads |
| `penalty` | seconds — obstacle penalty per skip |
| `mode` | "edit" or "readonly" |

### 3.5 Event index entry (`eventIndex/<id>`)

| Field | Notes |
|---|---|
| `name`, `date`, `day1`, `day2` | display strings |
| `cxCount` | last-known competitor count |
| `mode` | mirrors `config/mode` for UI badges |
| `createdAt`, `updatedAt` | epoch ms |

Written via `fbWriteEventIndex()` whenever competitors or config change. Used by staff view's no-`?event=` picker.

### 3.6 Session export envelope

JSON file produced by "Save Session to File" / consumed by "Load Session":

```
{ "version": 4, "exportedAt": <epoch>, "eventId": "<id>", "cfg": {...}, "cx": [...] }
```

`cx[]` carries the merged (PII-inclusive) records so a restore can rebuild both nodes.

---

## 4. Auth & DB rules

### 4.1 Operator app

- Firebase Auth email/password.
- Persistence: `firebase.auth.Auth.Persistence.LOCAL` — survives reload, browser close.
- Sign-in modal (`#mo-signin`) gates everything; appears on first load and after sign-out.
- Account section on Event Config shows signed-in email + Sign out (with `confirm()`).
- On sign-out: listeners detach, `fbDb`/`fbRef` null out, in-memory `cx` clears, modal reappears.
- Migration `runPiiMigrationOnce(uid)` runs once per device per UID on first sign-in; flag in `localStorage` as `rng_pii_migrated_v1_<uid>`.

### 4.2 Staff app

- Firebase Auth anonymous (silent, automatic on page load).
- No human login. URL is the credential.

### 4.3 Published DB rules (current)

```json
{
  "rules": {
    "eventIndex": {
      ".read":  "auth != null",
      ".write": "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'"
    },
    "events": {
      "$eventId": {
        "competitors": {
          ".read":  "auth != null",
          ".write": "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'"
        },
        "config": {
          ".read":  "auth != null",
          ".write": "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'"
        },
        "private": {
          ".read":  "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'",
          ".write": "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'"
        },
        "access": {
          ".read":  "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'",
          ".write": "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'"
        },
        "snapshots": {
          ".read":  "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'",
          ".write": "auth != null && auth.uid === 'Te9HFmyA1cREPARflULb17jstNI3'"
        }
      }
    }
  }
}
```

**Property**: anyone signed in (incl. staff anonymous) can read `competitors/`, `config/`, `eventIndex/`. Only the operator UID can write anywhere or read `private`/`access`/`snapshots`.

---

## 5. Event lifecycle

- **Create**: operator types `eventId` in Event Settings → `validateEventId()` checks regex → on first write, `events/<id>/config` and `eventIndex/<id>` are created.
- **Switch**: Event Library → "Switch Event" → picks from `/eventIndex/` → unbinds listeners, rebinds `fbRef` to new event ID.
- **New**: Event Library → "New Event" → opens modal to enter new ID + name → creates empty event in Firebase, current event left intact.
- **Delete** (paranoid 2-step): Event Config → "Delete This Event…" → modal warns + offers "Export Backup First" → typing `eventId` enables the red Delete button → 10-second undo bar after delete using an in-memory snapshot.
- **Restore session**: Event Config → "Load Session from File" → reviews count + timestamp → confirms → writes through to `competitors/` and `private/`, sets `config/`, writes `eventIndex/`.

---

## 6. Scoring & timing rules

- **Release**: operator clicks "Release" on the next squad row → captures `startMs = Date.now()` for every member of that squad simultaneously.
- **Finish**: operator clicks "Finish" on the competitor row → captures `finishMs = Date.now()`.
- **Raw time**: `finishMs - startMs`.
- **Penalised time**: `raw + (obstacles × cfg.penalty × 1000)`.
- **DNF**: marks `dnf = true`; competitor shown in Done section with DNF badge; excluded from ranked results.
- **No Show**: `ciStatus = "No Show"` — never released, never appears in On Deck/Running.
- **Late**: `ciStatus = "Late"` — still appears in On Deck (operator may release out-of-order).

---

## 7. Screen map

### 7.1 Operator app

| Tab | Purpose | Key actions |
|---|---|---|
| Check-In | Pre-event roster management | Edit PII, change `ciStatus`, mark No Show, search by bib/name |
| Run Scoring | Live timing | Release next squad, capture finishes, +obstacle, DNF |
| Results | Standings | Filter by day/division, sort by raw or penalised time |
| Event Config | Setup & admin | Sign in/out, event settings, event library, Firebase, CSV import, session backups |

### 7.2 Staff view sections

| Section | Inclusion rule | Sort |
|---|---|---|
| On Deck | `ciStatus ∈ {Checked In, Late}` AND no `startMs` | by bib asc |
| Running | has `startMs`, no `finishMs` | by `startMs` asc; live elapsed when expanded |
| Done | has `finishMs` OR `dnf` | by `finishMs` asc; DNF badge |

Each row shows bib, last + first name, division, squad. Running adds `started HH:MM:SS` + elapsed. Done adds final time + obstacle count badge.

---

## 8. Design system

### 8.1 Tokens (`:root`)

```
--bg #0f1117    --sur #171a23   --sur2 #1e2130   --sur3 #252840
--tx #e8eaf0    --txm #8b90a0   --txf #4a5060
--pri #3fb8c8   --prih #2da0b0  --prid rgba(63,184,200,0.12)
--suc #4caf7d   --sucd rgba(76,175,125,0.15)
--wrn #f0a840   --wrnd rgba(240,168,64,0.15)
--dan #e05c6a   --dand rgba(224,92,106,0.15)
--yel #e8c84a   --yeld rgba(232,200,74,0.15)
```

Spacing scale `--sp1` (4px) through `--sp16` (64px). Radius `--r-sm`, `--r-md`, `--r-lg`. Type scale via `clamp()`: `--text-xs`, `--text-sm`, `--text-base`, `--text-lg`, `--text-xl`.

### 8.2 Components

- `.btn` variants: `.btn-pri`, `.btn-gho`, `.btn-red`, `.btn-sm`, `.btn-row`.
- `.mo` + `.md` for modals (centred overlay + card).
- `.f-row` + `.f-lbl` + `.f-inp` for form rows.
- `.sec` for staff view collapsible sections.
- `.disc-banner` for the yellow disclaimer.
- `.row`/`.row-bib`/`.row-mid`/`.row-r` for staff list rows.
- `#toast`, `.badge`, `.dot`, `.evlib-*` (event library card).

### 8.3 Font

Satoshi via Fontshare CDN (400/500/700), fallback `Inter`, then system sans.

---

## 9. File structure & dependencies

```
/
├── rng-ops-firebase.html    operator app (single file, ~2630 lines)
├── staff.html               read-only staff view (~625 lines)
├── rng-fb-config.js         shared Firebase config (16 lines)
├── netlify.toml             redirect / → /rng-ops-firebase.html
├── rng-ops-spec.md          this spec
└── status.md                current state, in-flight work, decision log
```

`rng-fb-config.js` is the single source of truth for Firebase config. Both HTML files load it via `<script src="rng-fb-config.js">` and read `window.FIREBASE_CONFIG`.

### CDN dependencies

- Firebase JS SDK 10.12.2 (compat APIs): `firebase-app-compat.js`, `firebase-database-compat.js`, `firebase-auth-compat.js`.
- Fontshare Satoshi (400/500/700).

No build step. No npm. No bundler.

---

## 10. Working agreement

- **Operator profile**: novice in web dev, expert in process engineering and operations. Treat technical depth on Firebase as something the assistant must explain when introducing; treat operations design as something Cody decides.
- **Cadence**: Friday is low-cognitive-load. No meetings before 10 AM. No work after 6 PM. Weekends are personal projects.
- **Prefer small refactors over rewrites.**
- Every code change ships as a **separate branch + PR**. The assistant must confirm a PR's `state` before pushing follow-up commits to the same branch — otherwise commits orphan when the PR has already been merged.
- Every plan from the assistant ends with a **"What you should do next"** section.
- Before code: the assistant summarises the relevant spec in 3–5 bullets and asks 1–4 focused questions to reach ≥99% confidence on screen, scope, and constraints.
- Confidence levels are stated when data is disputed or uncertain.

---

## 11. Block roadmap

### 11.1 Shipped

| Block | Scope | Status |
|---|---|---|
| Phase 1 / 1.5 | Local-storage MVP, fixed seed data → CSV import, scoring math, results | Done |
| Block 2 | Multi-event library, read-only mode, paranoid delete + undo, JSON session v4 | Done (PR #5) |
| Block 3 | Staff view + wake-lock + reconnect banner + 3 sections + event picker | Done (PR #6, field-name fix in PR #8) |
| Block 4a | Operator email/password auth + PII split + auto-migration + tightened DB rules | Done (PR #7) |
| Staff banner | "NOT FINAL — LIVE TRACKING ONLY" disclaimer | Done (PR #9 + PR #10) |

### 11.2 Deferred (decision-pending)

| Item | Why deferred | When to revisit |
|---|---|---|
| **Block 4b — Competitor view** | Want real event experience before designing the public surface | After 1–2 events run |
| **Multi-operator support** | Single-UID rules work fine for now | When a second operator is needed |
| **Staff login** | Decided to leave staff view open; URL is the credential | If a sponsor or PII concern forces it |

### 11.3 Candidate next blocks (no commitments)

- **Block 5a — Auto-snapshots** to `events/<id>/snapshots/` every N minutes. Versioned undo for the whole event. Race-day insurance.
- **Block 5b — Operator UI polish** based on what bites during the first live event. Open scope until that event happens.
- **Block 6 — Competitor view** (formerly 4b). Designed against real operational data, not guesses. Re-uses access/passphrase plumbing.

---

## 12. Decision log

- **2026-05-22** — Pick explicit event IDs over auto-derived slugs. Operator types the ID; legacy slug derivation kept as fallback for old data only.
- **2026-05-22** — Read-only mode is a `cfg.mode` field, not a DB rule. Cheaper to toggle, safer per-device.
- **2026-05-23** — Block 3 staff view: 3 strict sections (On Deck / Running / Done). DNF lives in Done with badge, not its own section.
- **2026-05-24** — Sign-in modal is full-screen overlay, not a separate route. Avoids extra file + redirect hop.
- **2026-05-25** — PII migration auto-runs on first operator login per device. Idempotent. Flag keyed by UID in `localStorage`.
- **2026-05-25** — Sign-out requires `confirm()` to prevent accidental sign-out mid-event.
- **2026-05-25** — Persistence is LOCAL (survives reload). Operator stays signed in on race-day reloads.
- **2026-05-25** — Block 4b (competitor view) deferred. Run live events first, design against real data.
- **2026-05-25** — Staff view stays open (no login, no passphrase). PII is already DB-rule-protected; URL obscurity is sufficient for spectator-grade access control.
- **2026-05-25** — Staff disclaimer banner is always-visible (not dismissible). Text: "NOT FINAL — LIVE TRACKING ONLY". Yellow tokens, 11px, single-line guarantee via `nowrap + ellipsis`.

---

## 13. Verification checklist (per change)

Before merging any PR:

- [ ] Local Node syntax check on every modified script block (`node --check`).
- [ ] Visual review of any UI change on iPhone-SE width (320px CSS) on the deploy preview.
- [ ] If touching Firebase reads/writes: confirm staff view still loads in incognito on the preview.
- [ ] If touching PII paths: confirm in Firebase Console that PII still resolves to `private/`, not `competitors/`.
- [ ] If touching DB rules: rules published only after every active device has run migration.
- [ ] PR description includes a test plan.

---

## 14. Known risks

| Risk | Mitigation |
|---|---|
| Operator forgets password mid-event | Firebase password reset via Gmail recovery on `rng.ops.operator@gmail.com` |
| Single-device dependency on race day | JSON session export ("Save Session to File"); Firebase JSON export from console as periodic backup |
| Stale staff view if reconnect fails silently | Wake-lock + `.info/connected` watcher + Retry button (Block 3) |
| Sloppy follow-up commits orphaning after PR merge | Assistant must check PR `state` before pushing to a previously-PR'd branch (see §10) |
| Public staff URL surface | DB rules prevent PII reads; non-PII data is intentionally world-readable to any anonymous Firebase user with the event ID |

---

**End of spec.** For day-to-day "what's the state of the build right now" see `status.md`.
