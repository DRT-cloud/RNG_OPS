# RNG OPS — Current Status

**Last updated**: 2026-05-25
**Companion to**: `rng-ops-spec.md` (v3.0)
**Operator**: Cody Allenbaugh

> The spec is the **target state**. This doc is **what is true right now** — what's live, what's deferred, what to do if something breaks on race day.

---

## 1. Production state (live today)

| Surface | URL | Auth | Status |
|---|---|---|---|
| Operator app | <https://rng-ops.netlify.app/> | Email/password (Firebase) | Live |
| Staff view | <https://rng-ops.netlify.app/staff.html?event=`<id>`> | Anonymous (URL is credential) | Live |
| Competitor view | — | — | Deferred (Block 4b) |

**What's running where**
- **Hosting**: Netlify, auto-deploys from `main`. `netlify.toml` redirects `/` → `/rng-ops-firebase.html`.
- **Backend**: Firebase Realtime Database (project owned by operator account `rng.ops.operator@gmail.com`).
- **Source of truth**: GitHub `DRT-cloud/RNG_OPS`, branch `main`.

**Auth model in production**
- Operator app: email/password, **LOCAL** persistence (stays signed in across browser restarts).
- Staff view: anonymous sign-in, no human login.
- DB rules published and tightened (see spec §4.3). Operator UID is hardcoded in the rules.

**PII migration status**
- All new writes go to `events/<id>/private/<id>` for PII fields (`phone`, `email`, `address`, `paid`, `pmm`, `shirt`).
- Auto-migration runs once per browser on first sign-in. Flag stored in `localStorage` as `rng_pii_migrated_v1_<uid>`.
- No known un-migrated records in production.

---

## 2. Recently shipped

| PR | Merge SHA | What |
|---|---|---|
| #7 | `4b2e9e2` | Block 4a — email/password auth, PII split, auto-migration, sign-in modal, sign-out confirm |
| #8 | `9135d13` | Staff field-name fix — staff.html was reading `c.fi`/`c.st`/`c.ci` keys the operator never wrote |
| #9 | `e6e9f56` | Staff yellow disclaimer banner — "not final scores, live tracking only" |
| #10 | `024bb6a` | Banner shrink — 11px, "NOT FINAL — LIVE TRACKING ONLY", single-line on iPhone SE |

**What's broken right now**: nothing known.

---

## 3. In-flight / next candidates

Nothing is currently in-flight after this docs PR merges. Below are the candidate next blocks, in rough priority order. None are committed — pick when ready.

**5a — Auto-snapshots (high value, low risk)**
- Background job in operator app: write a snapshot of `cx/` + `cfg` to `events/<id>/snapshots/<ts>` on a timer (e.g. every 5 min during a live event).
- Lets you roll back to a known-good state if data gets stomped mid-event.
- Storage cost: trivial. Implementation: ~1 evening.

**5b — Operator polish (low cognitive load)**
- Sort/filter UX on the roster table.
- Bib-number conflict warning when adding/editing.
- Squad-builder ergonomics (currently functional but clunky).
- Export filename includes event name + timestamp, not just `cfg.exportName`.

**6 — Competitor view (deferred until more events run)**
- Public-readable view at `competitor.html?event=<id>&bib=<n>` showing one competitor's start, finish, obstacles, penalty, net time.
- Blocked on: needing real race-day usage to learn what competitors actually want to see. Don't build it on assumptions.

**7 — Multi-operator (deferred)**
- Today the DB rules hardcode one operator UID. Adding a second operator means editing rules JSON.
- Not urgent — single-operator works for current event scale.

**Staff login (deferred — decision recorded)**
- User chose option (c): leave staff view anonymous. URL is the credential, PII is DB-rule-protected.
- Revisit only if a staff URL leaks to someone outside the event.

---

## 4. Race-day recovery checklist

Print this. Tape it to the timing tent. Order = try in sequence.

### A. Operator can't sign in
1. Confirm internet at the venue (load `https://google.com`).
2. Confirm credentials: `rng.ops.operator@gmail.com` + the saved password.
3. If wrong password: <https://rng-ops.netlify.app/> → "Forgot password?" link in sign-in modal sends reset email to the operator address. Need email access on phone.
4. If reset email doesn't arrive: open Firebase Console → Authentication → Users → find operator account → "Reset password" (sends a fresh link).
5. If Firebase Console is locked out: sign in to the Google account that owns the Firebase project (same `rng.ops.operator@gmail.com`).

### B. Operator app loads but data is wrong / missing
1. Don't panic. **Do not delete anything.**
2. Open Account section → **Export JSON** immediately. This captures the current (possibly broken) state as a backup before you do anything else.
3. Check the snapshots node if 5a is shipped: `events/<id>/snapshots/<ts>` in Firebase Console. Pick the most recent good one.
4. If no snapshots exist: open the last known-good local JSON export (downloads folder) → Account section → **Import JSON** → confirm event ID matches.
5. JSON envelope is v4: `{version:4, exportedAt, eventId, cfg, cx}`. Import replaces `cfg` + `cx` for that event ID only — other events are untouched.

### C. Staff view shows wrong data / nothing
1. Confirm the URL has `?event=<id>` and the ID matches the operator's active event.
2. Confirm the device is online (load any other website).
3. Hard reload (iOS Safari: tap and hold reload, "Reload Without Cache").
4. If still wrong: the operator's writes may not have replicated yet. Wait 10 seconds.
5. If still wrong after 30 seconds: open operator app, confirm the data is correct **there**. If operator shows correct, staff is a display bug — file it post-event, don't fight it during the race.

### D. Firebase is down (rare but possible)
1. Check <https://status.firebase.google.com/>.
2. Operator app: switch to **Offline mode** by closing it. The local export JSON from earlier is your source of truth.
3. Score manually on paper. Re-enter via JSON import after Firebase recovers.

### E. Netlify is down (very rare)
1. The operator app is a single HTML file. You can run it locally:
   - Clone repo or grab `rng-ops-firebase.html` from a previous device.
   - Open the file directly in Chrome (works — Firebase config is in `rng-fb-config.js`, same folder).
2. Staff view won't work without hosting. Use operator app as the source of truth.

---

## 5. What to test before the next live event

Run through this list at least 24 hours before race day. Treat any failure as a bug to fix before the event.

- [ ] Operator sign-in works on the race-day laptop with the saved password.
- [ ] Create a test event, add 3 competitors, run them through Check-In → Start → Finish → score.
- [ ] Export JSON, delete the test event, re-import JSON, confirm data restores.
- [ ] Open staff view on an iPhone (smallest screen you'll have on-site). Confirm banner is one line.
- [ ] Confirm staff view updates within 5 seconds of operator changes.
- [ ] Confirm staff view does **not** show PII fields (phone, email, address, paid, pmm, shirt).
- [ ] Sign out of operator, sign back in, confirm session persists after closing the browser.
- [ ] Forgot-password flow: trigger reset email, confirm it arrives.

---

## 6. Decisions made this session (recap)

These are also in spec §12 (decision log). Listed here so you don't have to cross-reference during a tired post-event review.

1. **Email/password over passphrase** — Firebase Auth handles reset, lockout, persistence. Don't roll our own.
2. **LOCAL persistence over SESSION** — operator wants to stay signed in across laptop restarts on race day.
3. **Full-screen sign-in modal** — gate everything. No partial UI before auth.
4. **Sign-out requires confirmation** — single tap shouldn't dump a race in progress.
5. **PII split into `events/<id>/private/<id>`** — staff view reads `events/<id>/cx/<id>` (no PII), operator reads both. Enforced by DB rules.
6. **Staff view stays anonymous** — URL is the credential. Revisit only on leak.
7. **Yellow banner, 11px, single line** — "NOT FINAL — LIVE TRACKING ONLY". Set via `white-space:nowrap` + `text-overflow:ellipsis`.
8. **Competitor view deferred** — wait for real race-day signal before building.
9. **PRs always branch off `main`, never push to a merged branch** — codified after PR #7/#8 and PR #9/#10 both orphaned commits on already-merged branches.

---

## 7. Open questions for Cody

None blocking. Things to consider before the next event:

- Do you want 5a (auto-snapshots) before the next live event, or are local JSON exports enough?
- Is there a second operator (helper, backup) who needs sign-in on race day? If yes, plan Block 7 (multi-operator) now.
- Any staff feedback from the last event about what the staff view is missing?

---

## 8. File inventory (current `main`)

| File | Purpose | Lines (approx) |
|---|---|---|
| `rng-ops-firebase.html` | Operator app — single HTML, all logic inline | ~2628 |
| `staff.html` | Staff view — read-only, anonymous | ~625 |
| `rng-fb-config.js` | Shared Firebase config (operator + staff load this) | 16 |
| `rng-ops-spec.md` | Build specification (target state) | ~413 |
| `status.md` | This file (current state) | — |
| `netlify.toml` | Hosting config — root redirect to operator app | small |

---

*End of status. For target state and design contract, see `rng-ops-spec.md`.*
