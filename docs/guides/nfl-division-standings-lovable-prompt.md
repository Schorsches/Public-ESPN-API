# NFL Division Standings — Lovable Plan-Mode Prompt

> Paste everything below the line into Lovable's plan mode. If Lovable accepts file attachments,
> also attach `nfl-division-standings-fixture-2026-wk3.json` — it is the expected parsed output
> referenced throughout.

---

## ROLE

You are a senior full-stack architect and product designer extending a working app. Produce a
phased implementation plan before writing any code. Ask every clarifying question that would
materially change the approach, then wait for approval. Do not build until the plan is approved.

This is an **additive feature**. Prediction entry, deadlines, scoring, the prediction
leaderboard and every existing view must behave exactly as before. A plan that refactors
existing ingest, scoring or navigation beyond what this feature strictly needs is a failed plan.

## CONTEXT

"Gridiron Tippzone" — a private, invite-only NFL prediction game for a small group of friends.
Bragging rights only, no money. German UI, dark theme, mobile-first. Stack: Lovable + Supabase
(Postgres with RLS, Edge Functions, `pg_cron`), React/TypeScript.

Already in place and to be **reused, not rebuilt**:

- ESPN ingest for schedule, live scores and results, with a job that detects games becoming final
- `nfl_teams` reference table (`espn_team_id`, `abbreviation`, `display_name`, `conference`,
  `division`, colours) — ESPN abbreviations everywhere (`WSH`, `LAR`)
- A team-logo sprite with classes `nfl-logo--{abbr}` keyed by the same abbreviations
- Team record badges (W-L-D) in the Tippen, Spieltag and Übersicht match cards
- A **Rangliste** view showing the prediction leaderboard
- Job telemetry (`job_runs`) and an admin health view

Before planning, inspect the code and report back: where the "game became final" detection
lives, how Rangliste is structured, how the record badge component is built, and whether a user
profile table exists for storing a favourite team.

## THE FEATURE

Players can view the **current NFL division standings** during the regular season, inside
**Rangliste**, with:

1. A segmented control at the top of Rangliste: **`Tippspiel` | `NFL-Divisionen`**. `Tippspiel`
   is the existing leaderboard, unchanged and the default.
2. In `NFL-Divisionen`: an **`AFC` | `NFC`** toggle showing that conference's **4 division
   cards** stacked. The last choice is remembered per device.
3. Each division card is a compact 4-row table. Tapping a row expands it for detail.
4. **Clinch badges** once ESPN starts reporting clinch status late in the season.
5. **Favourite team**: each player can mark one team, highlighted in every division table.
6. **Local cross-check** of ESPN's figures against the game results the app already stores,
   with an admin alert on any mismatch.

## DATA SOURCE — VERIFIED FACTS, DO NOT RE-DERIVE

All of the following was verified against the live API. Trust it over assumptions.

### The endpoint

```
GET https://site.api.espn.com/apis/v2/sports/football/nfl/standings?level=3&season={year}
```

- Path is **`/apis/v2/`** — `/apis/site/v2/` returns only a stub for standings.
- **`level=3` is mandatory.** Without it the response groups by conference and contains no
  divisions at all.
- **One request returns all 8 divisions and all 32 teams.** No per-team or per-division calls.
- Payload is 160 KB plain, **7.9 KB gzipped** — send `Accept-Encoding: gzip`.
- ESPN sends `Cache-Control: max-age=1`, i.e. no caching guidance. **The app owns the cache.**
- Call it **server-side only**, from an Edge Function. Never from the browser, never per page
  view. Clients read only the stored copy.

### Shape

```
root → children[] (AFC, NFC: abbreviation) → children[] (divisions: id, name)
     → standings.entries[] (4 per division) → { team: {id, abbreviation, …}, stats: [ … ] }
```

Walk the tree recursively and collect nodes with `standings.entries`; do not hard-code depth.
`team.id` equals `nfl_teams.espn_team_id` and division `name` equals `nfl_teams.division`
(`AFC East`, …), so rows join directly to existing reference data, the logo sprite and record
badges.

Stable division IDs: AFC East 4 · AFC North 12 · AFC South 13 · AFC West 6 · NFC East 1 ·
NFC North 10 · NFC South 11 · NFC West 3.

### Fields

`stats[]` is a list of `{name, displayValue, value}` — index it by `name`, never by position.

| Use | Read | Notes |
|---|---|---|
| Overall record | `wins`, `losses`, `ties` → `value` | numeric, ties `0` when none |
| Win % | `winPercent` → `value` | already accounts for ties |
| **Ordering key** | `playoffSeed` → `value` | see ordering rule |
| Division record | `divisionWins`, `divisionLosses`, `divisionTies` → `value` | |
| Conference record | `vs. Conf.` → `displayValue` | string only, e.g. `"2-0"` / `"7-9-1"` |
| Points | `pointsFor`, `pointsAgainst`, `pointDifferential` → `value` | |
| Streak | `streak` → `value` | signed: `+3` = 3 wins in a row, `-1` = 1 loss |
| Clinch | `clincher` → `displayValue` + `description` | **optional**, absent most of the season |

Record strings are variable length — `W-L`, or `W-L-T` once a tie exists. Split on `-`,
accept 2 or 3 integers, default ties to 0, reject anything else.

**Traps — do not use:**
- `divisionRecord.value` — always `0`.
- `lockedDivRank` — `0` for every team, even after a completed season. Not a rank.
- `displayValue` of `divisionWins`/`divisionTies` — inconsistently formatted (`'1.000'`).

### ⚠️ The ordering rule — the single most important requirement

**Sort each division ascending by `playoffSeed`. Never trust the array order. Never sort by
win percentage or compute tiebreakers yourself.**

- The array order is intermittently wrong: at the end of 2025, 2 of 8 divisions came back out
  of order (AFC West `LAC, KC, LV, DEN` although Denver won it). Today all 8 happen to be in
  order — so testing only against current data would ship a latent bug.
- `playoffSeed` encodes the official NFL tiebreakers. In the attached fixture the AFC North has
  **four teams at 2-1**; CIN has the best point differential yet ranks **4th** because of its
  0-1 division record. A win-%-then-differential sort would put CIN 1st. The NFC South has a
  three-way 1-2 tie (CAR, NO, ATL) resolved only by seed.

The fixture `nfl-division-standings-fixture-2026-wk3.json` contains real data after week 3,
already parsed and in the correct order. **Use it as the golden test for parsing and ordering.**

### Clinch codes

ESPN's codes do **not** follow the common x/y/z convention. Map by the `description` string,
with a safe fallback:

| ESPN `description` | Code | German badge |
|---|---|---|
| Clinched Division and Bye | `*` | `Division + Freilos` |
| Clinched Division | `z` | `Division gesichert` |
| Clinched Wild Card | `y` | `Playoffs gesichert` |
| Eliminated from Playoff Contention | `e` | `Ausgeschieden` |

Unknown description → show nothing and log it. No `clincher` stat → no badge. The stat was
present on every team at the end of 2025 and on none yet in 2026.

### Outside the regular season

Before the first regular-season game is final, the data is not meaningful (seeds of `0`, and a
preseason result once leaked into point totals). Show an empty state instead of a table. Postseason
behaviour is out of scope.

### Not yet verified — design defensively

- Whether in-progress games are reflected before they are final.
- The lag between a game going final and the standings updating.

So: refresh only after games are final, and always display when the data was fetched.

## BACKEND

### Fetch policy — event-driven, minimal requests

| Trigger | Action |
|---|---|
| Existing finalize job marks a game `final` | Schedule a standings refresh ~5 min later; coalesce so at most **one request per 10 min** however many games finish |
| Same, second pass | One confirming refresh ~20 min after the first (covers unknown ESPN lag) |
| Daily, 06:00 UTC, regular season only | One safety refresh |
| No games that day / offseason / preseason | **No requests** |

Hook into the existing finalize detection; do not add a new polling loop. Reuse the existing
ESPN client conventions (timeouts, retry with backoff, advisory lock, `job_runs` row).

### Storage

New table, one row per `(season_year, espn_team_id)`: conference, division, division id,
rank (1–4, derived after sorting), playoff seed, wins, losses, ties, win %, division W/L/T,
conference W/L/T, points for/against/difference, streak value, clinch code and description
(nullable), `fetched_at`.

- **Validate before writing:** exactly 8 divisions × 4 teams, 32 distinct team ids, all present
  in `nfl_teams`, every seed a positive integer. Any failure → **write nothing**, keep the last
  good copy, record the failure in `job_runs`.
- Upsert in a single transaction so a client never sees a half-updated table.
- Write only when something actually changed, to avoid pointless realtime events.
- RLS: authenticated members may read; nobody but the service role may write.

### Local cross-check

After each successful refresh, recompute from the app's own stored **final regular-season
games**: overall W-L-T, division W-L-T, conference W-L-T, points for and against, per team.
Compare with ESPN's values.

- Match → record success in telemetry.
- Mismatch → **still store ESPN's data** (it remains the source of truth for display and
  ordering), flag the affected teams, and surface the discrepancy in the admin health view.
- Ignore the comparison for a team while one of its games is in progress or a final is younger
  than the lag window, to avoid false alarms.

This was validated once by hand: 32 teams × 5 fields, 0 mismatches. A persistent mismatch means
a bug in the app's own result data or a correction ESPN made — both worth knowing.

### Favourite team

Add a nullable `favourite_team_id` (FK to `nfl_teams`) to the user profile. Writable only by the
owning user, via RLS. Set it from the profile screen and via a long-press/menu action on a row in
the division table ("Als Lieblingsteam markieren"). Clearing it is allowed.

## UI / UX

Design to the existing dark visual language: navy surfaces, yellow accent, rounded cards,
the type scale already used in Rangliste. Follow the existing component library; no new design
system.

### Structure

- **Rangliste header:** segmented control `Tippspiel | NFL-Divisionen`. Keyboard-accessible
  tabs (`role="tablist"`), URL-addressable (e.g. `?ansicht=divisionen`) so a deep link and the
  back button work. Default stays `Tippspiel`.
- **Conference toggle:** `AFC | NFC`, persisted locally. Default: the conference of the player's
  favourite team, else AFC.
- **Four division cards** per conference, each with the division name as heading.
- **Meta line** above the cards: `Stand: vor 12 Min. · nach Woche 4` and an info button
  explaining "Bei Gleichstand entscheiden die offiziellen NFL-Tiebreaker". This one line prevents
  "why is 2-1 behind 2-1?" confusion.

### Division card (360 px wide, no horizontal scroll)

| Column | Content | Notes |
|---|---|---|
| `#` | 1–4 | tabular numerals |
| Logo | sprite `nfl-logo--{abbr}`, 24 px | decorative, `aria-hidden` |
| Team | abbreviation, full name from 400 px up | |
| `Bilanz` | `3-0-0` | always three parts, same convention as the existing record badge |
| `Div.` | `1-0-0` | division record |
| `Serie` | `W3` / `L1` | from signed streak value |

- **Row tap expands** an inline detail panel (accordion, one open at a time per card):
  Punkte `101 : 78`, Differenz `+23`, Conference-Bilanz `2-0-0`, Playoff-Seed `2`, and the clinch
  status in words. Animate height with `prefers-reduced-motion` respected.
- **Division leader** gets a subtle accent bar on the row edge, plus the text "Führend" in the
  expanded panel and screen-reader label — not colour alone.
- **Clinch badge:** small pill after the team abbreviation with icon + short text; full German
  description in the expanded panel and `aria-label`.
- **Favourite team:** highlighted row background + star icon + visually hidden "Lieblingsteam".
- **Teams playing this week** (optional, cheap from existing schedule data): a small dot
  "spielt diese Woche" — only if it fits without crowding; note it as an open decision.
- **Touch targets** ≥ 44 px per row. **Tabular numerals** throughout so columns don't jitter
  when values reach two digits.

### Linking from match cards

Tapping a team's record badge in Tippen, Spieltag or Übersicht opens a **bottom sheet** with
that team's division card (same component), plus a link "Alle Divisionen" to the full view. This
reuses the card; it does not duplicate logic.

### States

| State | Display |
|---|---|
| Loading | skeleton of 4 cards × 4 rows, same height as real content (no layout shift) |
| Preseason / before week 1 final | friendly empty state "Die Tabelle startet mit dem ersten Spieltag" |
| Data older than 24 h during the season | yellow notice "Daten möglicherweise veraltet" with the timestamp |
| Fetch failed but old data exists | show old data + timestamp; never an error wall |
| No data at all | empty state with retry for admins only |

### Realtime and performance

- Render from the stored table only; one query per view (all 32 rows), filtered client-side by
  conference.
- Subscribe to changes on the standings table so an open view updates after a refresh, with a
  short, polite "Tabelle aktualisiert" announcement (`aria-live="polite"`), never on every poll.
- Cache with the existing data-fetching layer; switching AFC/NFC must be instant (no refetch).

### Accessibility (WCAG 2.2 AA)

- Each card is a real `<table>` with `<caption>` (division name) and `scope` on headers.
- Screen-reader text per row, e.g. "Platz 1, Buffalo Bills, 3 Siege, 0 Niederlagen,
  0 Unentschieden, Division 1 zu 0, Serie 3 Siege".
- Expandable rows use a button with `aria-expanded` and `aria-controls`.
- Never encode rank, clinch, leader or favourite by colour alone. Contrast ≥ 4.5:1 for text,
  3:1 for UI elements.
- Works at 200 % zoom without horizontal scroll.

### Copy (German)

`NFL-Divisionen`, `Bilanz`, `Div.`, `Serie`, `Punkte`, `Differenz`, `Conference-Bilanz`,
`Playoff-Seed`, `Führend`, `Lieblingsteam`, `Stand: vor X Min.`,
`Bei Gleichstand entscheiden die offiziellen NFL-Tiebreaker`.

## PHASES

1. **Data:** table, Edge Function refresh, validation, finalize-job hook, daily safety run,
   telemetry. Golden test against the fixture. No UI. Existing tests pass unchanged.
2. **Cross-check:** local recomputation, mismatch flagging, admin health view entry.
3. **UI:** Rangliste segmented control, conference toggle, division cards, expandable rows,
   states, accessibility.
4. **Favourite team & linking:** profile field, highlight, bottom sheet from match-card badges.
5. **Clinch badges:** description-driven mapping, hidden until data exists. Test with the 2025
   season (`season=2025`), which has clinch data for every team.

## TESTS AND EDGE CASES

Each is a named test.

1. Fixture: parsing and ordering produce exactly the fixture's order and values
2. Array delivered out of seed order (shuffle the fixture input) → output order unchanged
3. AFC North four-way 2-1 → PIT, BAL, CLE, CIN; CIN not first despite best differential
4. NFC South three-way 1-2 → CAR, NO, ATL
5. Record string `7-9-1` and `12-5` both parse; garbage string rejected
6. `divisionRecord`, `lockedDivRank` never read (assert in code review / unit test on the mapper)
7. Payload with 31 teams, a duplicate team, an unknown team id, or seed `0` → nothing written,
   previous data kept, failure logged
8. Response without `level=3` shape (conference groups only) → rejected
9. Ten games final within an hour → at most one request per 10-minute window
10. No games today → zero requests
11. Clincher absent → no badge; known descriptions → correct German label; unknown → no badge, logged
12. `season=2025` → clinch badges render for all 32 teams
13. Cross-check: matching data → no alert; tampered local result → alert for that team only
14. Favourite team set/cleared; other users cannot change it (RLS test)
15. 360 px, 200 % zoom, keyboard-only, screen reader: table readable, no horizontal scroll
16. Preseason → empty state, no table
17. Existing Rangliste leaderboard, prediction entry, deadlines and scoring → unchanged

## NON-GOALS

No postseason bracket or playoff picture. No conference-wide (1–16) seeding table. No
reimplementation of NFL tiebreakers. No client-side ESPN calls. No betting odds or sportsbook
links. No change to prediction scoring or the existing leaderboard.

## PLAN OUTPUT FORMAT

First, your findings from inspecting the existing code (finalize detection, Rangliste,
record badge, profile table). Then per phase: goal, tasks, files and Edge Functions touched,
migrations, RLS, acceptance criteria, tests, risks. Then separately:

- the exact parsing/mapping function signature and its rejection cases
- how the coalescing of refreshes is implemented
- a wireframe description of the division card at 360 px and at desktop width

**Ask your clarifying questions now. Wait for approval before building.**
