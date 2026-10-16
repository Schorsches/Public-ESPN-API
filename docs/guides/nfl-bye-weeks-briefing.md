# Briefing: Teams on Bye ("Spielfrei") per Spieltag

**Purpose.** ESPN-data half of a feature spec for the Claude chat that holds the prediction
game's details and writes the Lovable plan-mode prompt. Verified against the live API on
2026-10-08 (2026 regular season, all 18 weeks, plus spot checks of 2025).

**Headline: zero additional API requests.** The list of teams on bye is already part of the
scoreboard payload the app fetches for each Spieltag. This feature is parse → store → render.

---

## A. Verified facts

### A1. Where the data is

In the existing scoreboard response:

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard?seasontype=2&week={n}&dates={seasonYear}
```

```json
"week": {
  "number": 5,
  "teamsOnBye": [
    { "id": "12", "abbreviation": "KC",  "displayName": "Kansas City Chiefs", "logo": "…", … },
    { "id": "29", "abbreviation": "CAR", "displayName": "Carolina Panthers",  "logo": "…", … }
  ]
}
```

- `id` equals `nfl_teams.espn_team_id`; `abbreviation` is the ESPN abbreviation already used for
  the logo sprite (`nfl-logo--{abbr}`) and all other team data. No mapping needed.
- The same block is present on the parameterless "current week" call (verified: week 5, KC and
  CAR).

### A2. ⚠️ The field is ABSENT, not empty, in weeks without byes

| Week | `week` keys | Byes |
|---|---|---|
| 1–4, 12, 15–18 | `number` only | none — **`teamsOnBye` missing entirely** |
| 5–11, 13–14 | `number`, `teamsOnBye` | 2–6 teams |

Parse with a default: `week.teamsOnBye ?? []`. Code that assumes the key always exists will
crash or log errors in 9 of 18 weeks.

### A3. Accuracy — independently verified

For 8 weeks across 2025 and 2026, `teamsOnBye` was compared with
`all 32 teams − teams appearing in that week's games`: **identical in every week**.

Full 2026 season: **all 32 teams appear exactly once**; no team twice, none missing.

2026 byes per week (reference / test data):

| Week | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 13 | 14 |
|---|---|---|---|---|---|---|---|---|---|
| Teams on bye | 2 | 4 | 4 | 4 | 2 | 4 | 6 | 4 | 2 |

Examples: week 5 `CAR, KC` · week 6 `CIN, DET, MIA, MIN` · week 10 `CHI, DEN, PHI, TB` ·
week 14 `ARI, DAL`. 2025 week 5 `ATL, CHI, GB, PIT`; week 14 `CAR, NE, NYG, SF`.

### A4. Scope

Byes exist only in the regular season (`seasontype=2`). Preseason and postseason have no bye
concept in this sense (postseason "byes" for top seeds are a different thing and out of scope).

---

## B. Recommended design

### B1. Ingest (existing sync, no new requests)

- In the same pass that already processes a week's scoreboard, read `week.teamsOnBye ?? []` and
  store it against the matchday: either an array of team ids on the matchday row or a small
  table `matchday_byes (matchday_id, team_id)` with a unique constraint.
- **Replace the set for that matchday on each sync** (upsert/diff), so a schedule change is
  reflected. The list is known for the whole season in advance; it rarely changes.
- **Validate:** every id must exist in `nfl_teams`; no team may be both on bye and in a game of
  that week. On violation, keep the previous set and log it in the existing job telemetry.
- **Fallback / integrity check:** compute `all teams − teams in this week's games`. If the field
  is missing in a week where the computation yields teams, use the computation and log it. If
  both exist and disagree, prefer ESPN's field and log it.
  - Caveat: a game postponed into another week makes the computation wrongly list those teams
    as spielfrei. ESPN's field reflects the actual schedule — another reason it is primary.
- **Backfill:** one re-read per already-synced week fills past matchdays, or nothing at all if
  the app already stores the full season's schedule (byes can then be derived immediately for
  all 18 weeks).

### B2. UI

- **Spieltag view:** one compact line below (or above) the game list:
  **`Spielfrei: [logo] KC · [logo] CAR`** — logos from the existing sprite, abbreviations as
  text.
- **Hidden entirely when there are no byes** — no empty heading, no "0 Teams".
- Optional, to decide with the user:
  - the same line in **Tippen** for the selected week, and/or on the **Übersicht** card for the
    next deadline;
  - in the **division table**, a small "spielfrei" marker next to teams resting this week;
  - tapping a team opens its division card (reuse the existing bottom sheet if it exists).
- **Accessibility:** a list with a visible label ("Spielfrei"), logos `aria-hidden`, full team
  names available to screen readers ("Spielfrei in Woche 5: Kansas City Chiefs, Carolina
  Panthers"). Not conveyed by colour or icon alone.
- **No layout shift:** renders from stored data together with the week's games; up to 6 teams
  must fit at 360 px (wrap to a second line is fine, no horizontal scroll).

### B3. Non-breaking

Additive storage only. No effect on predictions, deadlines, scoring, standings or game counts.
The "X von Y getippt" progress must keep counting **games**, not teams — bye teams never create
prediction slots. Existing tests pass unchanged.

---

## C. Edge cases for tests

1. Week with `teamsOnBye` (week 5: CAR, KC) → stored and displayed
2. Week without the key (weeks 1–4) → empty set, line hidden, no error logged
3. Week 11 with 6 byes → all displayed, fits 360 px
4. Unknown team id in the list → set rejected, previous kept, logged
5. Team listed on bye **and** in a game → rejected, logged
6. Field missing but computation yields teams → fallback used, logged
7. Field and computation disagree (synthetic postponed game) → ESPN field wins, logged
8. Season check: across weeks 1–18 every team on bye exactly once (fixture from A3)
9. Re-sync of the same week → idempotent, no duplicate rows
10. Preseason/postseason weeks → no bye line
11. Prediction progress counts ("16 von 16 getippt" → "14 von 14" in a bye week) unaffected in
    logic, correct in display

## D. Questions for the user

1. Where to show it: Spieltag only, or also Tippen and the Übersicht deadline card?
2. Mark resting teams in the division table?
3. Placement within Spieltag: above or below the game list?
4. Should tapping a bye team open its division card?
