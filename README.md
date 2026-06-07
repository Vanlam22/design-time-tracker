# Handoff: Time Tracker

## Overview
**Time Tracker** is a calm, mobile-first personal time-tracking app. The user logs their day on a
Google Calendar–style timeline, tags each entry as **A / B / C**, and reviews where their time goes
through daily / weekly / monthly analysis. The explicit goal of the app: **reduce "C" (waste) time and
grow "A" + "B" time** by making time allocation visible by category and by activity.

- **A** = Productive
- **B** = Indirectly Productive
- **C** = Waste of Time

There is also a **Vision** section (Vision / Mission / Core Values / Strategies / Goals) to anchor
long-term direction. Full **English / 繁體中文 (Traditional Chinese)** localization.

> **Status:** Live as an installable **PWA** on GitHub Pages with optional **Firebase cloud sync**.
> See [Deployment & Sync](#deployment--sync) below.
> **Live URL:** https://vanlam22.github.io/design-time-tracker/

## About the Design Files
The bundled `index.html` is **NOT just a static mock — it is a complete, working single-file
application**: React (via in-browser Babel), all state, persistence, charts, i18n and interactions are
implemented and functional. It runs by opening the file in a browser; data persists to `localStorage`
and, when signed in, syncs to Firebase.

The original handoff suggested porting this to a build-based stack. In practice the single-file
CDN + Babel approach was **kept and shipped as-is**, wrapped as a PWA (service worker + manifest) and
extended with Firebase Auth + Firestore for cross-device sync. Treat `index.html` as both the design
reference and the shipping implementation.

## Fidelity
**High-fidelity AND functional.** Colors, typography, spacing, layout, charts and interactions are all
final.

## Tech
- Single `index.html`. React 18 + ReactDOM + Babel Standalone loaded from unpkg CDN; code in
  `<script type="text/babel">` blocks.
- **PWA:** `manifest.webmanifest` + `sw.js` (service worker) cache the app shell and the CDN deps
  (React/ReactDOM/Babel, Google Fonts, Firebase SDK) for offline use and home-screen install.
- **Cloud sync (optional):** Firebase Auth (Google + email/password) + Firestore, configured in
  `firebase-config.js`. Runs local-only if no config is present.
- Fonts: **Hanken Grotesk** (Latin) and **Noto Sans TC** (Chinese) from Google Fonts.
- No build step. Designed canvas width **402px** (iPhone logical width); the app frame is centered and
  capped at `--maxw: 402px`, full viewport height, with a bottom tab bar.

## Data Model
All persisted to `localStorage` as JSON, and mirrored to Firestore (`users/{uid}`) when signed in. Keys:

| Key | Shape | Notes |
|---|---|---|
| `tt_entries_v1` | `Entry[]` | time entries (see below) |
| `tt_activities_v2` | `Activity[]` | user-editable activities |
| `tt_days_v1` | `{ [dateStr]: { wake, bed } }` | per-day start/end overrides (minutes from midnight) |
| `tt_settings_v2` | `{ wake, bed, lang }` | defaults + language (`'en'`/`'zh'`) |
| `tt_vision_v1` | `{ vision, mission, values, strategies, goals }` | free text |
| `tt_sync_v1` | `{ updatedAt }` | local change timestamp for cloud last-write-wins |

```
Entry = {
  id: string,            // unique
  date: string,          // "YYYY-MM-DD" (local)
  name: string,          // freeform label, may be empty
  start: number,         // minutes from midnight, 0..1440, snapped to 15
  end: number,           // minutes from midnight, > start
  cat: 'A' | 'B' | 'C',
  activityId: string     // references Activity.id
}

Activity = {
  id: string,
  name?: string,         // user-entered name (custom activities)
  nameKey?: string,      // i18n key for the 7 built-in defaults (e.g. 'act_work')
  color: string          // oklch string
}
```

Built-in default activities (each has a `nameKey`, so they localize): Work, Family, Health, Spiritual,
Learning, Social, Chores. User-added activities store a literal `name` and lose `nameKey` (don't
localize). Default day window: **wake 07:00 (420), bed 23:00 (1380)**.

### Core time math (no "sleep" concept)
A day = the window between **start of day (`wake`)** and **end of day (`bed`)**, both in minutes from
midnight. For a given day:
- `dayLen = bed - wake`
- `A`, `B`, `C` = summed minutes of entries in each category
- `logged = A + B + C`
- `blank` (Unlogged) = `max(0, dayLen - logged)`
- All percentages are computed against `dayLen`, so **A + B + C + Unlogged = 100% of the day**.

## Screens / Views
Bottom tab bar, 4 tabs: **Today · Analysis · Vision · Settings**.

### 1) Today (calendar)
- **Top bar:** ‹ prev day · centered date title · next day ›. Tapping the title jumps to today.
- **Summary strip:** a thin stacked proportional bar (A/B/C/Unlogged) + 4 cells showing each category's % of day. (The day window is set in Settings → Default day; there is no day-window editor on Today.)
- **Calendar body:** vertical hour grid; entries as colored blocks; red "now" line on today.
- **Add via long-press:** press-and-hold (~380ms) an empty slot → a dashed ghost block appears, then
  **drag up/down to position the start time** (the ghost follows your finger) → release opens the New
  Entry sheet pre-filled at the release time. There is also a **+ FAB** (bottom-right).
- **Edit:** tap any entry block to open the sheet in edit mode (with Delete).

### 2) New / Edit Entry (bottom sheet)
- **Name** text field (with datalist of previously used names).
- **Start** and **End** in one compact card: each is an inline row (label left) with a **scroll-wheel
  time picker** — three columns: **hour (1–12) · minute (00/15/30/45) · AM/PM**. Each column scrolls
  on touch and snaps; no slider. The **hour and minute columns loop** (…45 → 00 →…, …11 → 12 → 1…) so
  12 and 00 are always a short scroll away; AM/PM is a flat 2-option list.
  - **Default Start** for a new entry: the **latest logged end time** that day, or **now** if none.
  - Changing **Start** auto-sets **End = Start + 1h** (live); End can then be adjusted independently.
  - **No overlaps (auto-trim to fit):** times are clamped into the free gap around other entries — End
    springs back at the next entry's edge; a Start landing inside an entry jumps to the adjoining gap.
    Back-to-back entries (one ends as the next starts) are allowed.
- **Category:** 3 large buttons A / B / C.
- **Activity:** wrap of pill chips; selected chip filled dark.
- **Footer:** Delete (edit mode only) + primary Add/Save (disabled until End > Start).

### 3) Start & End of Day (bottom sheet)
Two scroll-wheel pickers (Start of day / End of day) + a "Day length" readout. Saving updates that
specific day (`tt_days_v1`); the same UI in Settings updates the global default.

### 4) Analysis
- **Segmented control:** Day / Week / Month. Period nav ‹ label ›; Day mode has a 14-day quick strip.
- **Overview card:** donut (A/B/C/Unlogged) with center A+B %; legend rows.
- **Where the hours went:** Day = single strip; Week = one strip per day; Month = stacked bar per day.
- **Activity breakdown:** ranked list with duration, % and progress bar. Empty state when no entries.

### 5) Vision
Five cards (Vision, Mission, Core Values, Strategies, Goals); one **Edit** button flips all into textareas.

### 6) Settings
- **Account (cloud sync):** shown when Firebase is configured. Signed out → email/password fields
  (Sign in / Create account) + "or" Google sign-in. Signed in → email, **Account ID**, sync status
  (Synced / Syncing / Sync error) + Sign out.
- **Language:** segmented English / 繁體中文.
- **Activities:** list with color dot, rename, delete; "Add activity".
- **Default day:** opens the start/end-of-day editor (writes the global default).
- **Data:** entry count + "Reset all data" (confirm).

## Interactions & Behavior
- **Long-press-to-add + drag:** 380ms press engages a ghost that tracks the finger (pointer capture +
  scroll-lock); the entry starts where you release. A move before engaging cancels (treated as scroll).
- Start auto-bumps End by +1h in the entry sheet (live as you scroll Start).
- All times snap to 15 minutes; the wheel picker enforces 00/15/30/45.
- Language toggle re-renders all strings, date/time formats, weekday/activity names; sets `lang-zh`.
- Bottom sheets animate up with a scrim; charts animate via CSS transitions.

## State Management
Top-level `App` holds: `tab`, `date`, `entries`, `activities`, `days`, `settings`, `vision`. Each
persists to `localStorage` via effects. `useFirebaseSync(data, applyRemote)` layers cloud sync on top:
whole-document last-write-wins to `users/{uid}` with a stable-stringify guard against echo loops, plus
Firestore offline persistence. An `actions` object exposes the data mutators; `getDay(dateStr)` returns
the per-day window or the settings default.

## Design Tokens
**Colors**
```
--page:#f0eee9  --bg:#fbfaf8  --card:#ffffff
--line:#e9e6df  --line-2:#f0ede7
--ink:#2b2924   --ink-2:#6f6a61  --ink-3:#a7a299
--a: oklch(0.64 0.115 150)   (Productive / green)
--b: oklch(0.77 0.115 78)    (Indirect / amber)
--c: oklch(0.605 0.145 27)   (Waste / red)
--blank:#d8d4cc              (Unlogged / gray)
--a-t / --b-t / --c-t        (category tints)
--now: oklch(0.62 0.2 25)    (now line)
```
Activity palette (8, cycled): `oklch(0.62 0.09 H)` for H ∈ {150,250,40,310,200,95,350,120}.

**Type:** Hanken Grotesk (Latin), Noto Sans TC (Chinese). Tabular numerals for clock times/durations.
**Radius:** `--r:16px`, `--r-sm:11px`; sheets 26px top corners; pills 999px. **Frame:** `--maxw:402px`.

## Assets
No image assets in the app UI (all icons are inline SVGs). PWA icons live in `icons/` (192/512 maskable,
180 apple-touch, 32 favicon). Fonts via Google Fonts.

## Deployment & Sync

### Hosting (GitHub Pages)
- Repo: https://github.com/Vanlam22/design-time-tracker · Live: https://vanlam22.github.io/design-time-tracker/
- Static site served from `main` / root. **To deploy: push to `main`** — Pages rebuilds in ~1 min.
- The PWA is installable (iOS: Share → Add to Home Screen) and works offline.
- **Service worker:** `sw.js` uses a `CACHE` version string — **bump it** (e.g. `tt-v4` → `tt-v5`)
  whenever the shell or pinned CDN deps change, so clients pull the new version.

### Cloud sync (Firebase)
- Project: `design-time-tracker-8fc76`. Config in `firebase-config.js` (the public `apiKey` is an
  identifier, not a secret — data is protected by security rules).
- **Auth:** Email/Password + Google enabled. Google redirect sign-in is unreliable inside an iOS
  standalone PWA, so **email/password is the recommended method** there.
- **Firestore:** one document per user at `users/{uid}` holding the whole dataset; multi-user (anyone
  can sign up and gets their own private space).
- **Security rules (per-user):**
  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /users/{uid} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }
    }
  }
  ```
- Runs **local-only** (sync hidden) if `firebase-config.js` has no `apiKey`.

### Files
- `index.html` — the complete working app (all screens, logic, i18n, sync).
- `manifest.webmanifest` — PWA manifest (standalone, maskable icons).
- `sw.js` — service worker (offline cache; bump `CACHE` on shell changes).
- `firebase-config.js` — Firebase project config (sync off until `apiKey` is filled).
- `icons/` — PWA icons (192, 512, apple-touch 180, favicon 32).
- `README.md` — this handoff.

## Possible next steps
1. **Export / import** of local data as a JSON backup (safety net independent of cloud).
2. Per-entry Firestore docs / pagination if a user's dataset ever approaches the 1MB document limit.
3. Optional account linking so Google and email/password resolve to one account.
4. Repackage as React Native / Expo for a native iOS app.

Preserve the data model and the A/B/C + day-window math exactly; they encode the product's core idea.

---

## Changelog

### 2026-06-07
- **Removed the day start/end editor from Today** (the time pill); the day window is still set in Settings → Default day.
- **Looping time wheels** — the hour and minute columns now wrap (…45 → 00 →…, …11 → 12 → 1…) so 12 / 00 are always a short scroll away; AM/PM stays a 2-option list.
- **No double-booking (auto-trim to fit)** — a new/edited entry's times are clamped into the free gap so they never overlap another entry; End springs back at the next entry's edge and a Start inside an entry moves to the adjoining gap. Back-to-back entries are allowed; existing overlaps are left untouched.

### 2026-06-03
- **Deployed as a PWA** on GitHub Pages (`manifest.webmanifest`, `sw.js`, generated icons); installable + offline-capable.
- **Added Firebase cloud sync** — Auth (Google + email/password) + Firestore (`users/{uid}`), whole-document last-write-wins; multi-user with per-user security rules. Local-only until configured.
- **Replaced the Start/End time sliders with a scroll-wheel picker** (hour / minute 00·15·30·45 / AM·PM), applied to the entry sheet and the day start/end sheet.
- **New-entry default Start** = latest logged end time that day, else the current time; **End auto-follows to Start + 1h** live.
- **Long-press on the calendar now drags to position** — the ghost follows the finger and the entry starts at the release point (fixes inaccurate starts from finger drift during the hold).
- Added **Account ID** display in Settings; surfaced sign-in errors in the Account UI.
