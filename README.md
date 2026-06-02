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

## About the Design Files
The bundled `index.html` is **NOT just a static mock — it is a complete, working single-file
application**: React (via in-browser Babel), all state, persistence, charts, i18n and interactions are
implemented and functional. It runs today by opening the file in a browser; data persists to
`localStorage`.

Treat it as **both a high-fidelity design reference and a reference implementation**. The task in
Claude Code is to **bring it into a real codebase / deployment** — recreate it using the target stack's
established patterns (React + Vite, Next.js, Expo/React Native, SwiftUI, etc.), or, if starting fresh,
pick the most appropriate framework and port the design + logic there. Do **not** assume the in-browser
Babel + CDN approach is the production target; it's a prototype delivery mechanism.

## Fidelity
**High-fidelity AND functional.** Colors, typography, spacing, layout, charts and interactions are all
final. Recreate the UI pixel-accurately and preserve the behaviors described below.

## Tech (as prototyped)
- Single `index.html`. React 18 + ReactDOM + Babel Standalone loaded from unpkg CDN; code in
  `<script type="text/babel">` blocks.
- Fonts: **Hanken Grotesk** (Latin) and **Noto Sans TC** (Chinese) from Google Fonts.
- No build step, no backend. All data in `localStorage`.
- Designed canvas width **402px** (iPhone 17 Pro logical width); the app frame is centered and capped at
  `--maxw: 402px`, full viewport height, with a bottom tab bar.

## Data Model
All persisted to `localStorage` as JSON. Keys:

| Key | Shape | Notes |
|---|---|---|
| `tt_entries_v1` | `Entry[]` | time entries (see below) |
| `tt_activities_v2` | `Activity[]` | user-editable activities |
| `tt_days_v1` | `{ [dateStr]: { wake, bed } }` | per-day start/end overrides (minutes from midnight) |
| `tt_settings_v2` | `{ wake, bed, lang }` | defaults + language (`'en'`/`'zh'`) |
| `tt_vision_v1` | `{ vision, mission, values, strategies, goals }` | free text |

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
- All percentages are computed against `dayLen` (the waking/tracked window), so **A + B + C + Unlogged = 100% of the day**.

(An earlier version modeled Sleep explicitly; it was removed. There is no sleep category.)

## Screens / Views
Bottom tab bar, 4 tabs: **Today · Analysis · Vision · Settings**.

### 1) Today (calendar)
- **Top bar:** ‹ prev day · centered date title ("Today"/"Yesterday"/weekday+date) with sub-label · next day ›. Tapping the title jumps to today.
- **Day-window pill** (centered, below top bar): shows `7:00 AM – 11:00 PM` with a pencil; opens the start/end-of-day editor sheet.
- **Summary strip:** a thin stacked proportional bar (A green / B amber / C red / Unlogged gray) over the day, then 4 centered cells showing each category's **% of day**.
- **Calendar body:** vertical hour grid (Google-Calendar style). Hour labels in the left 46px gutter; 1px vertical rule. Entries render as colored blocks (category tint background, 4px category-color left rail, name + time range, overlapping events split into columns). A red "now" line shows current time on today.
- **Add via long-press:** press-and-hold (~380ms) an empty slot → a dashed ghost block appears at the snapped 15-min time → release opens the New Entry sheet pre-filled at that time. There is also a **+ FAB** (bottom-right) and the sheet.
- **Edit:** tap any entry block to open the same sheet in edit mode (with Delete).

### 2) New / Edit Entry (bottom sheet)
- **Name** text field (with datalist of previously used names).
- **Start** and **End** in one compact card: each is an inline row (label left, big time right) above a full-width 15-min slider. Below: duration + "snaps to 15 min" (or "End must be after start" in red).
  - **Behavior:** changing **Start** auto-sets **End = Start + 1 hour**.
  - Default for a new entry: Start = previous entry's end (or day start); End = Start + 1h.
- **Category:** 3 large buttons A / B / C (letter badge + label), selected one tinted + colored border.
- **Activity:** wrap of pill chips; selected chip filled dark.
- **Footer:** Delete (edit mode only) + primary Add/Save.

### 3) Start & End of Day (bottom sheet)
Two compact 15-min sliders (Start of day / End of day) + a "Day length" readout. Saving updates that
specific day (`tt_days_v1`); the same UI in Settings updates the global default.

### 4) Analysis
- **Segmented control:** Day / Week / Month. Period nav ‹ label ›. In **Day** mode, a horizontal scroll
  strip of the last 14 days (weekday + date, dot if data) for quick jumping.
- **Overview card:** donut (A/B/C/Unlogged) with center showing **A+B %** "of day"; legend rows for
  Productive / Indirect / Waste / Unlogged (duration + %). *(An insight-tiles row was intentionally removed.)*
- **Where the hours went:** Day = single proportional strip across the day window (start/mid/end ticks).
  Week = one strip per day (Mon–Sun) with that day's A+B % at the right.
- **Month:** "A / B / C balance by day" — one stacked vertical bar per day of the month.
- **Activity breakdown:** ranked list of activities with duration, % of logged time, and a progress bar.
- Empty state when no entries.

### 5) Vision
Header is just the title "Vision". Five cards (Vision, Mission, Core Values, Strategies, Goals), each with
a small colored icon + label and the user's text (or an italic guiding prompt placeholder). A single
**Edit** button pinned at the bottom flips all five into textareas at once; **Save** / cancel.

### 6) Settings
- **Language:** segmented English / 繁體中文 — switches the entire app instantly (and persists).
- **Activities:** list with color dot, name, rename (pencil) and delete (trash); "Add activity" button + sheet.
- **Default day:** opens the start/end-of-day editor (writes the global default).
- **Data:** entry count + "Reset all data" (confirm) which clears entries/days/vision and restores default activities & settings (language preserved).

## Interactions & Behavior
- Long-press-to-add on the calendar (380ms, cancels if finger moves >8px; haptic `navigator.vibrate(12)` if available).
- Start auto-bumps End by +1h in the entry sheet.
- All times snap to 15 minutes.
- Language toggle re-renders all strings, date/time formats (12-hour `6:30 AM` vs `上午6:30`), weekday names, and the default-activity names; sets `document.body.class = 'lang-zh'` and `lang` attribute.
- Bottom sheets animate up from the bottom with a scrim; closing animates down.
- Charts animate (donut stroke, bar widths) via CSS transitions.

## State Management
Top-level `App` holds: `tab`, `date` (current day on Today), `entries`, `activities`, `days`,
`settings`, `vision`. Each persists to `localStorage` via effects. An `actions` object exposes
add/update/delete entry, setDay, setSettings, add/rename/delete activity, setVision, clearAll.
`getDay(dateStr)` returns the per-day window or the settings default.

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
--a-t: oklch(0.95 0.035 150) --b-t: oklch(0.955 0.04 85) --c-t: oklch(0.955 0.04 30)  (tints)
--now: oklch(0.62 0.2 25)    (now line)
(--s / --s-t indigo tokens remain defined but are unused after sleep removal)
```
Activity palette (8, cycled): `oklch(0.62 0.09 H)` for H ∈ {150,250,40,310,200,95,350,120}.

**Type:** Hanken Grotesk (Latin), Noto Sans TC (Chinese). Tabular numerals (`font-variant-numeric: tabular-nums`) for all clock times/durations.

**Radius:** `--r:16px`, `--r-sm:11px`; sheets 26px top corners; pills 999px.
**Shadow:** `--shadow-sm: 0 1px 2px rgba(40,36,28,.05)`; `--shadow: 0 10px 34px -10px rgba(40,36,28,.22)`.
**Frame:** `--maxw:402px`, full-height column; on desktop it floats with a 42px-radius device-like card.

## Assets
No image assets. All icons are inline SVGs (defined in an `Icon` component). Fonts via Google Fonts.

## Suggested next steps in Claude Code
The biggest real limitation today is **data lives in one browser**. Likely priorities:
1. **Deploy** the app to a permanent URL (and/or wrap as a PWA so it's installable on iOS home screen, offline-capable).
2. **Cloud sync + accounts** — move the data model to a backend (or a service like Supabase/Firebase) so entries follow the user across devices.
3. **Backup / export-import** of the local data as a safety net.
4. Optionally repackage as **React Native / Expo** for a true iOS app.
Preserve the data model and the A/B/C + day-window math exactly; they encode the product's core idea.

## Files
- `index.html` — the complete working app (single file; all screens, logic, i18n).
- `README.md` (repo-level, in the bundle) — quick start + GitHub Pages hosting notes.
