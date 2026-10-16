# Briefing: Division Table (Regular Season) from the ESPN API

**Purpose.** This is the ESPN-data half of a feature spec. Hand it to the Claude chat that holds
the prediction game's own details (schema, views, design system, existing ingest). That
instance should combine the two and produce the Lovable plan-mode prompt.

**Everything in Part A was verified against the live API on 2026-10-01**, after regular-season
week 3 (all 48 games final, week 4 not yet started). A parsed test fixture ships alongside this
file: `nfl-division-standings-fixture-2026-wk3.json`.

---

# PART A — VERIFIED ESPN FACTS

## A1. The one endpoint

```
GET https://site.api.espn.com/apis/v2/sports/football/nfl/standings?level=3&season={year}
```

- **`/apis/v2/`, not `/apis/site/v2/`.** The `site` path returns only a stub for standings.
- **`level=3` is required for divisions.** Without it (or with `level=2`) the response groups by
  *conference* — 2 groups of 16 — and there are no four-team groups at all. Verified for
  `level=3`, `level=3&season=2026`, `level=2&season=2026`, `season=2026` and no parameters.
- `season` is optional (defaults to the current season) but pass it explicitly — the same shape
  returns any past season, which makes it testable against completed years.
- **One request returns all 8 divisions and all 32 teams.** There is no per-division or per-team
  call to make.

## A2. Cost and fair use

| Measure | Value |
|---|---|
| Requests to refresh the whole table | **1** |
| Payload, uncompressed | 160 KB |
| Payload, gzip | **7.9 KB** (verified; ask for `Accept-Encoding: gzip`) |
| `Cache-Control` sent by ESPN | `max-age=1` — effectively none |

**ESPN gives no usable caching guidance, so the app owns the cache policy entirely.** Do not
call this endpoint from the browser and do not call it per page view. Fetch server-side, store
the parsed result in the database, and let every client read the stored copy.

## A3. Response structure

```
root                      (the league)
└─ children[]             2 conferences: AFC, NFC            ← abbreviation
   └─ children[]          4 divisions each                   ← id, name
      └─ standings.entries[]   4 teams each
            ├─ team   { id, abbreviation, displayName, location, name, … }
            └─ stats[]      array of { name, displayValue, value, type, abbreviation }
```

Walk the tree recursively and collect every node that has `standings.entries` — do not hard-code
depth, because the shape differs by `level`.

**`team.id` equals the app's existing `nfl_teams.espn_team_id`** (verified: Buffalo is `"2"`),
and `team.abbreviation` is the ESPN abbreviation already used everywhere (`WSH`, `LAR`). A
standings row therefore joins straight to the team reference data, the logo sprite and the
stadium data with no mapping table.

Division node names match the existing `nfl_teams.division` values exactly (`AFC East`, …), so
that join works too. Stable division IDs:

| Division | ID | Division | ID |
|---|---|---|---|
| AFC East | 4 | NFC East | 1 |
| AFC North | 12 | NFC North | 10 |
| AFC South | 13 | NFC South | 11 |
| AFC West | 6 | NFC West | 3 |

## A4. Which fields to read — and which to avoid

The `stats[]` array is keyed by `name`. Index it into a map first; never rely on position.

**Parse these (numeric `value` is reliable):**

| `name` | Meaning | Notes |
|---|---|---|
| `wins`, `losses`, `ties` | Overall record | Read `value`. Always present, `ties` is `0` when none |
| `winPercent` | Win percentage | Already accounts for ties (verified: DAL 7-9-1 → .441) |
| `playoffSeed` | **The ordering key** — see A5 | 1–16 per conference |
| `pointsFor`, `pointsAgainst`, `pointDifferential` | Scoring | `differential` is a duplicate of `pointDifferential` |
| `streak` | e.g. `W3`, `L1` | `value` is signed: `+3` for a win streak, `-1` for a loss streak |
| `divisionWins`, `divisionLosses`, `divisionTies` | Record vs own division | Read `value`, not `displayValue` (see below) |
| `gamesBehind` | Games behind the division leader | `"-"` for the leader |

**Parse from the display string, because there is no numeric field:**

| `name` | Example | Notes |
|---|---|---|
| `vs. Conf.` | `"2-0"` | Conference record. `displayValue` only; `value` is null |
| `vs. Div.` | `"1-0"` | Redundant with the three division fields above |

Record strings are **variable-length**: `W-L` when a team has no ties and `W-L-T` once it has
one (verified: `7-9-1`). Split on `-`, accept 2 or 3 integer segments, default ties to `0`, and
reject anything else rather than guessing.

**Do not use:**

- **`divisionRecord`** — its `value` is always `0.0`; only the display string carries data.
- **`lockedDivRank`** — `0` for every team, including at the end of a completed season. It looks
  like a division rank and is not one.
- **`displayValue` of `divisionWins` / `divisionTies`** — formatted inconsistently (`'1.000'`,
  `'0.000'`) while `divisionLosses` is `'0'`. Use `value`.
- **The raw `overall` string for parsing** — fine for display, but the numeric `wins`/`losses`/
  `ties` are safer.

## A5. ⚠️ ORDERING — the most important rule in this document

**Sort each division ascending by `playoffSeed`. Never trust the array order, and never sort by
win percentage.**

Both mistakes were observed directly:

**The array order is intermittently wrong.** In the completed 2025 season, 2 of 8 divisions came
back out of order: the AFC West as `LAC, KC, LV, DEN` although Denver (14-3) won it, and the NFC
West as `LAR, SF, ARI, SEA` although Seattle (14-3) won it. In the current 2026 payload all 8
happen to be in order. It cannot be relied on, so a developer who tests only against today's
data would ship a latent bug that surfaces in December.

**Win percentage is the wrong sort.** `playoffSeed` encodes the **official NFL tiebreakers**
(head-to-head, division record, common games, …). In 4 of today's 8 divisions the seed order
differs from a win-% order. The clearest case is the AFC North, where all four teams are 2-1:

| Rank | Team | W-L | Division record | Seed |
|---|---|---|---|---|
| 1 | PIT | 2-1 | 1-0 | 4 |
| 2 | BAL | 2-1 | 0-0 | 6 |
| 3 | CLE | 2-1 | 0-0 | 8 |
| 4 | CIN | 2-1 | **0-1** | 9 |

CIN has the best point differential in the group (+17) yet ranks last, because of its division
record. A win-% sort with a point-differential tiebreak would put CIN **first**. The NFC South is
a three-way tie at 1-2 (CAR, NO, ATL) resolved only by seed.

**Do not reimplement NFL tiebreakers.** They are long, order-dependent and change between
two-team and multi-team cases. Take ESPN's resolution as given.

What was and was not checked about this: in both the completed 2025 season and the current 2026
data, every division's leader holds the lowest `playoffSeed` in that division, and the order
within a division follows from it. The ordering of the *non-leading* teams is ESPN's own and was
**not** independently checked against the NFL rulebook, so treat it as ESPN's authority.

## A6. Data integrity — independently verified

I rebuilt every team's overall record, division record, conference record, points-for and
points-against from the 48 raw final game results (weeks 1–3) and diffed them against the
standings endpoint:

> **32 teams × 5 fields = 160 comparisons, 0 mismatches.**

Consequences:

- The standings figures are exact and there is no preseason contamination in the regular-season
  numbers.
- **The app can already compute W-L-T, division record, conference record and points itself from
  the final games it stores**, at zero API cost. What it *cannot* sensibly compute is the
  *order*, because of the tiebreakers. This suggests a hybrid: take `playoffSeed` from ESPN for
  ordering, and optionally cross-check the rest locally (see Part C).

## A7. Playoff clinch indicators — exist, but not yet this season

A `clincher` stat appears on every team in the *completed* 2025 payload and on **none** of the 32
teams in the current 2026 payload. It evidently appears once clinching becomes possible, so it
must be treated as **optional**.

ESPN's own `description` field for each code:

| `displayValue` | `description` | 2025 final seeds |
|---|---|---|
| `*` | Clinched Division and Bye | 1 |
| `z` | Clinched Division | 2–4 |
| `y` | Clinched Wild Card | 5–7 |
| `e` | Eliminated from Playoff Contention | 8–16 |

**These do not follow the common x/y/z convention** (where `x` is a berth and `z` the bye), and
`*` is a character rather than a letter. Drive any label from the `description` string rather
than hard-coding a meaning for the single character. Handle absence gracefully — before the first
clinch there is simply no badge.

## A8. State outside the regular season

- **Before week 1 / during the preseason, the table is not meaningful.** Observed in August: AFC
  teams carried `playoffSeed = 0`, and a preseason result had already leaked into point stats
  (CAR showed `pointDifferential +3`, `pointsAgainst 30` after the Hall of Fame game). Show
  nothing, or a neutral "Saison startet bald" state, until a regular-season game is final.
- **Bye weeks** need no special handling for a W-L-T table, but the table must tolerate teams
  having played different numbers of games. This could not be observed yet: all 32 teams have
  played exactly 3. The 2026 schedule shows the first byes in week 5 (15 games that week,
  14 in week 10), so unequal counts begin there.
- **Postseason seeding is out of scope.** The meaning of `playoffSeed` after week 18 was not
  investigated here.

---

# PART B — WHAT WAS *NOT* VERIFIED

Be explicit with the user about these rather than assuming.

1. **Whether in-progress games affect the standings.** Verification ran with no game live. It is
   unknown whether a game in progress is reflected mid-play or only once final. Recommended
   conservative design: refresh on final only, and treat any difference as a bug to investigate.
2. **Lag between a game going final and the standings updating.** Not measured. The first
   opportunity to measure it is the first week-4 kickoff (Thursday, 2026-10-02 00:15 UTC).
   Recommended: log `final_detected_at` and the first fetch whose figures include that game, and
   derive the real lag from a few weeks of data before tightening the refresh policy.
3. **Whether ESPN's tiebreak order is final-correct in edge cases** such as a multi-team tie
   involving a bye or a tie game. Treat as ESPN's authority; flag for review if wrong.
4. **Postseason behaviour** of seeds and the standings endpoint.

---

# PART C — RECOMMENDED ARCHITECTURE (for the receiving Claude to refine)

## C1. Fetch policy

This data only changes when a game finishes, so poll on events, not on a clock.

| Trigger | Action |
|---|---|
| The existing finalize job detects a game newly `final` | Request the standings **once**, coalescing: at most one request per ~10 minutes no matter how many games finish together |
| Daily safety refresh, 06:00 UTC, **only during the regular season** | One request, catches anything missed |
| A matchday has no games | **No request** |

Rough budget, labelled as an estimate: with games on ~3–4 days a week and coalescing, about
10–15 requests a week and a few hundred over the season. Negligible, and it reuses the existing
job rather than adding a cron entry.

Do not poll during games. Add a short delay after a final is detected before fetching — the lag
in B2 is unmeasured, so a first re-fetch a few minutes after the final plus one confirming
re-fetch is a safe pattern until it is measured.

## C2. Storage

Store the parsed table in the database; clients read only the stored copy. Suggested shape:
one row per `(season_year, espn_team_id)` holding rank, seed, W/L/T, division and conference
records, points for/against, streak, and `fetched_at`, plus an optional `clincher_code` and
`clincher_description`.

- **Upsert, never delete.** A fetch that returns fewer than 32 teams, or an unparseable payload,
  must **write nothing** and leave the previous copy in place.
- **Validate before writing:** 8 divisions of exactly 4 teams, 32 distinct `team.id` values, all
  present in `nfl_teams`. Anything else is a failed run, logged in the existing job telemetry.
- Keep the last good copy and show a visible "Stand: vor X Min." so a stale table never looks
  live.

## C3. Cross-check against games already stored

Because W-L-T, division and conference record and points can be recomputed from stored final
games, the ingest can compare the two and raise an alert on any disagreement. This was run once
by hand (A6) and found zero mismatches; running it continuously turns the standings endpoint into
a self-checking data source rather than a trusted black box.

## C4. Presentation constraints

- **Mobile first.** The existing cards are dense; a four-row table per division must fit a 360 px
  width without horizontal scroll. Columns that fit: rank, logo, abbreviation, W-L-T. Secondary
  columns (division record, point differential, streak) belong behind an expand control or a
  wider breakpoint, not squeezed in.
- **Reuse existing assets:** the team logo sprite (class `nfl-logo--{abbr}`, keyed by the same
  ESPN abbreviation) and the `nfl_teams` conference/division fields.
- **Semantic table** with a `caption` per division and `scope` on headers. Never convey rank,
  clinch status or streak by colour alone.
- **Tiebreak transparency.** Because rank can differ from raw win total, add one line of help
  text ("Bei Gleichstand entscheiden die offiziellen NFL-Tiebreaker") so a 2-1 team ranked
  below a 2-1 team does not look like a bug.
- **Ties** render as `W-L-T`, always three parts, matching the team-record badge elsewhere in
  the app.
- **No layout shift:** the table renders from stored data, never from a client-side fetch.
- German copy throughout.

## C5. Non-breaking requirements

- Additive tables and columns only; the ingest change must not alter how status, scores or
  deadlines are written.
- No effect on prediction entry, deadlines or scoring. The existing test suite passing unchanged
  is the acceptance criterion for the first phase.
- Reads need no new write policies; the table is derived, read-only reference data.

---

# PART D — DECISIONS THE RECEIVING CLAUDE SHOULD SETTLE WITH THE USER

1. **Where does the table live?** A new tab, a section of Rangliste, the Übersicht, or a
   modal reachable by tapping a team in a match card?
2. **All 8 divisions at once, or one at a time?** A conference filter, a division switcher, or
   highlight "the divisions of the teams playing this week"?
3. **Which columns on mobile**, and what sits behind the expand control?
4. **Local cross-check (C3):** build it now, or later?
5. **Clinch badges (A7):** include from the start, or add once they actually appear?
6. **Refresh timing:** accept the conservative policy in C1 now and tune after measuring the
   lag, or measure first?
7. **Scope boundary with the team-record badges** already built: should the badge in a match card
   link to that team's division table?

---

# APPENDIX — Reference values for tests

| Check | Verified value |
|---|---|
| Requests to refresh everything | 1 |
| Payload, gzip / plain | 7.9 KB / 160 KB |
| Divisions × teams | 8 × 4 = 32 |
| Independent recomputation | 160 comparisons, 0 mismatches |
| AFC North, all 2-1 | PIT 1st (div 1-0, seed 4) … CIN 4th (div 0-1, seed 9) |
| NFC South, three-way 1-2 tie | CAR 1st (seed 4), NO 2nd (11), ATL 3rd (13) |
| 2025 final, array out of order | AFC West `LAC,KC,LV,DEN` → correct `DEN,LAC,KC,LV` |
| 2025 final, array out of order | NFC West `LAR,SF,ARI,SEA` → correct `SEA,LAR,SF,ARI` |
| Tie record string | DAL 2025: `7-9-1`, `ties` value `1`, `winPercent` `.441` |
| Streak encoding | `W3` → value `3`; `L1` → value `-1` |
| Clincher, completed 2025 | 2 `*`, 6 `z`, 6 `y`, 18 `e`; none present in 2026 yet |

The full parsed expected output is in `nfl-division-standings-fixture-2026-wk3.json`.
