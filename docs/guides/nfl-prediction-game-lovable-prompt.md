# NFL Prediction Game — Schedule, Live Scores & Flex Handling

> A paste-ready prompt for Lovable's plan mode. Everything below the line is the prompt.

---

## ROLE

You are a senior full-stack architect working on an **existing** application. Produce a
phased implementation plan before writing any code. Ask me every clarifying question that
would materially change the architecture, then wait for my approval. Do not begin building
until I approve the plan.

## THE APP AS IT EXISTS TODAY

"NFL Prediction Game" — a private, invite-only prediction game for a small circle of
friends (10–50 players). Bragging rights only: no money, no pot, no payments, no public
sign-up. German UI, i18n-ready structure.

Already built and working:

- Users, invite-only leagues, league membership with `admin` / `member` roles
- Two prediction modes, chosen by the admin at league creation and immutable afterwards:
  - `EXACT_SCORE` — players predict the exact score (home:away)
  - `OUTCOME_ONLY` — players predict only `1` = home win, `2` = away win, `0` = draw
- Scoring rules (`EXACT_SCORE`: exact 5 / correct point differential 3 / correct tendency 2;
  ties: exact 5 / correct tendency 3 · `OUTCOME_ONLY`: correct outcome 2, otherwise 0)
- Prediction entry, per-game deadlines, weekly and overall standings, weekly winners with
  split shares on a tie, bonus questions with manual admin grading, an admin area
- Stack: Lovable + Supabase — React/TypeScript frontend, Supabase Postgres with RLS,
  Supabase Edge Functions (Deno), `pg_cron` + `pg_net` for scheduled work

## WHAT IS MISSING — THE SCOPE OF THIS WORK

Everything that connects the app to real NFL data:

1. **Schedule ingest** — pull the full NFL schedule (regular season *and* playoffs) from
   ESPN and keep it current
2. **Live scores during games** — refresh scores while games are being played, visible to
   players in the app
3. **Result finalisation** — detect when a game ends, capture the final score, trigger
   scoring
4. **Flex-schedule handling** — absorb the NFL's in-season rescheduling ("flexing") of
   kickoff times, and the rarer movement of a game to a different week
5. **Weighted playoff rounds** — let the admin optionally make playoff rounds worth more
   points than regular-season games
6. **The preseason as a standalone warm-up competition** — the four preseason matchdays run
   as their own self-contained contest with their own standings and their own champion,
   entirely separate from the regular-season table

## THE ONE INVARIANT THAT OVERRIDES EVERYTHING ELSE

> **A player's prediction must never be lost, altered, invalidated or silently detached
> from its game — by anything.** Not by a flexed kickoff, not by a game moving to another
> week, not by a corrected score, not by a schedule re-sync, not by an ESPN outage, not by
> a partial or malformed API response, not by an admin correction, not by a re-run of any
> job.

Every design decision below exists to serve that sentence. When any other requirement in
this document appears to conflict with it, the invariant wins, and you should raise the
conflict with me rather than resolving it yourself.

Concretely, the following must be structurally impossible, not merely unlikely:

- **No automated process may INSERT, UPDATE or DELETE a row in `predictions`.** Ingest
  jobs, scoring jobs, cron functions and triggers all have zero write access to that
  table. The only writer is the owning player, through the normal prediction flow, before
  that game's deadline. Prove this with a test that runs every job against a seeded
  database and asserts the `predictions` table is byte-identical afterwards.
- **No game row may ever be deleted.** There is no DELETE path in the ingest code. A game
  that disappears upstream is marked cancelled, never removed. Predictions reference games
  with `ON DELETE RESTRICT`, so the database refuses the deletion even if code tries.
- **Ingest is upsert-only and diff-based.** It writes only the fields that actually
  changed, and records every change.

---

## VERIFIED ESPN API REFERENCE

ESPN's API is undocumented and unofficial. No key is required. There are no published rate
limits — be conservative and cache. **All ESPN calls happen server-side, inside Edge
Functions, never from the browser** (this also avoids CORS problems and keeps players' IP
addresses out of ESPN's logs).

Every fact in this section was verified against the live API. Trust it over your training
data.

### The one endpoint that does most of the work

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard
      ?seasontype={1|2|3}
      &week={n}
      &dates={seasonYear}
```

| Parameter | Meaning |
|---|---|
| `seasontype` | `1` preseason · `2` regular season · `3` postseason · `4` off-season |
| `week` | week number **within that season type** |
| `dates` | the **season year**, not the calendar year |

**`dates` is the season year and this is a real trap.** The 2025 season's playoff games are
played in January and February 2026, but they are still `dates=2025`. Super Bowl LX was
played 2026-02-08 and lives under `?seasontype=3&week=5&dates=2025`. Store the season year
as its own column and never derive it from a kickoff timestamp.

Calling this endpoint with no parameters returns the *current* week, which is convenient
for a "what's on now" view but must not be used for schedule sync — always sync explicitly
by `(seasontype, week, dates)`.

### The season calendar is embedded in every scoreboard response

`leagues[0].calendar[]` contains the authoritative week structure with real UTC boundaries.
**Do not hardcode week dates anywhere.** Read them from here. Live response for the 2026
season:

```json
{ "label": "Regular Season", "value": "2",
  "startDate": "2026-09-09T07:00Z", "endDate": "2027-01-13T07:59Z",
  "entries": [
    { "value": "1",  "label": "Week 1",  "detail": "Sep 9-15",
      "startDate": "2026-09-09T07:00Z", "endDate": "2026-09-16T06:59Z" },
    { "value": "18", "label": "Week 18", "detail": "Jan 6-12",
      "startDate": "2027-01-06T08:00Z", "endDate": "2027-01-13T07:59Z" }
  ]}
```

Postseason week numbering, verified:

| `week` | `label` |
|---|---|
| 1 | Wild Card |
| 2 | Divisional Round |
| 3 | Conference Championship |
| **4** | **Pro Bowl** |
| 5 | Super Bowl |

**Regular-season week 1 and postseason week 1 are both `week.number = 1`.** A bare `week`
integer is therefore not a valid key for anything — see the data model section.

**Week 4 of the postseason is the Pro Bowl**, a skills exhibition with no meaningful score.
It must be excluded from prediction and scoring by default, with an admin toggle if I ever
want it.

### Event shape

Trimmed from a live response (2025 postseason, Wild Card):

```json
{
  "id": "401772979",
  "uid": "s:20~l:28~e:401772979~c:401772979",
  "date": "2026-01-10T21:30Z",
  "name": "Los Angeles Rams at Carolina Panthers",
  "shortName": "LAR @ CAR",
  "season": { "year": 2025, "type": 3, "slug": "post-season" },
  "week": { "number": 1 },
  "status": {
    "clock": 0.0, "displayClock": "0:00", "period": 4,
    "type": { "id": "3", "name": "STATUS_FINAL", "state": "post",
              "completed": true, "description": "Final", "detail": "Final" }
  },
  "competitions": [{
    "id": "401772979",
    "date": "2026-01-10T21:30Z",
    "timeValid": true,
    "neutralSite": false,
    "attendance": 73426,
    "broadcast": "FOX",
    "venue": { "id": "3628", "fullName": "Bank of America Stadium",
               "address": { "city": "Charlotte", "state": "NC" }, "indoor": false },
    "notes": [{ "type": "event", "headline": "NFC Wild Card Playoffs" }],
    "competitors": [
      { "id": "29", "homeAway": "home", "score": "31", "winner": false,
        "team": { "id": "29", "abbreviation": "CAR", "displayName": "Carolina Panthers" } },
      { "id": "14", "homeAway": "away", "score": "34", "winner": true,
        "team": { "id": "14", "abbreviation": "LAR", "displayName": "Los Angeles Rams" } }
    ]
  }]
}
```

Notes on this payload:

- `id` is the **ESPN event ID** — the stable anchor for a game. `competitions[0].id` is
  identical to it for the NFL.
- `competitors[].score` is a **string**, and is `"0"` (not absent) before kickoff. Parse
  defensively.
- `competitors[]` order is not guaranteed. Always select by `homeAway`, never by index.
- `notes[0].headline` gives a human round label — "NFC Wild Card Playoffs",
  "Super Bowl LX". Useful for display; not a key.
- `neutralSite: true` on the Super Bowl and international games. Home/away is nominal
  there — see the UX section.

### Game status: use `state` and `completed`, ignore the name

```json
"type": { "id": "3", "name": "STATUS_FINAL", "state": "post", "completed": true }
```

Derive your internal status from **`status.type.state`** (`pre` / `in` / `post`) and
**`status.type.completed`** (boolean) only. Treat `type.name` and `type.id` as advisory
display strings — ESPN adds and renames them (`STATUS_HALFTIME`, `STATUS_END_PERIOD`,
`STATUS_DELAYED`, `STATUS_SUSPENDED`, `STATUS_POSTPONED`, `STATUS_CANCELED`, …) and a
`switch` over them will eventually fall through on a live Sunday.

Mapping:

| `state` | `completed` | Internal status |
|---|---|---|
| `pre` | false | `scheduled` |
| `in` | false | `in_progress` |
| `post` | **true** | `final` — **the only state that may feed scoring** |
| `post` | false | `abnormal_end` — postponed, cancelled or suspended. Do **not** score. Flag for admin review and read `type.name` / `type.detail` to explain it. |

### `timeValid: false` — how flex games actually appear

This is the single most important detail for flex handling.

When the NFL has not yet fixed a kickoff time, ESPN still publishes the game with a
**placeholder timestamp** and marks it `timeValid: false`. Verified live: all 16 games of
2026 regular-season Week 18 currently return

```json
{ "shortName": "NYJ @ BUF", "date": "2027-01-10T05:00Z",
  "timeValid": false, "broadcast": "", "status": { "type": { "name": "STATUS_SCHEDULED" } } }
```

— every one of them carrying the identical fake kickoff `2027-01-10T05:00Z` and an empty
broadcast. **Treating that as a real kickoff time would compute a deadline from a fiction.**
Any game with `timeValid: false` has a provisional kickoff and must be handled as described
in the flex state machine below.

Real NFL flex timing, for context on how much notice players get: regular-season flex
decisions are announced at least 12 days ahead for most weeks, and as little as 6 days
ahead for the final weeks of the season.

### Playoff games exist before the teams are known

Verified live, in August 2026 — five months before the games:
`?seasontype=3&week=1&dates=2026` already returns 6 wild-card events with allocated IDs:

```json
{ "id": "401872910", "shortName": "TBD @ TBD", "date": "2027-01-16T05:00Z",
  "timeValid": false,
  "competitions": [{ "competitors": [
    { "homeAway": "home", "team": { "id": "-1", "abbreviation": "TBD", "displayName": "TBD" } },
    { "homeAway": "away", "team": { "id": "-2", "abbreviation": "TBD", "displayName": "TBD" } }
  ]}]}
```

So: **`team.id = "-1"` means home TBD and `team.id = "-2"` means away TBD.** A playoff game
row is created long before its participants are known, and the same event ID later gains
real teams as seeding resolves. This has a direct consequence for the anomaly detection
below — a team changing on an existing event ID is *expected* in exactly this one case and
suspicious in every other case.

### The preseason: fully available, and the easiest phase to ingest

`?seasontype=1&week={1..4}&dates=2026` returns **49 games** — 1 Hall of Fame Game plus three
weeks of 16. Verified before the first game of the 2026 preseason:

- **All 49 carry `timeValid: true`** — real kickoff times, no placeholders anywhere
- **All teams are resolved** — zero `-1`/`-2` entries, unlike the playoffs
- Broadcast information present on most games
- No flex exposure: the NFL does not flex preseason games, and windows are fixed at
  schedule release

So preseason ingest needs neither the TBD-resolution path nor the flex ratchet. That is why
it is the right phase to prove the pipeline on — see the phases section.

**Preseason week numbers are offset from their labels.** This will confuse players if you
render the raw number:

| ESPN `week.number` | `calendar` label | Window (2026) |
|---|---|---|
| 1 | Hall of Fame Weekend | Aug 6–12 |
| 2 | **Preseason Week 1** | Aug 13–19 |
| 3 | Preseason Week 2 | Aug 20–26 |
| 4 | Preseason Week 3 | Aug 27–Sep 8 |

Always take the display label from `leagues[0].calendar[].entries[].label`. Never render
`week.number` directly for preseason.

**There is no overtime in the preseason, so ties are common.** Measured across the 2025
season:

| Phase (2025) | Games | Ties | Rate | Went to OT |
|---|---|---|---|---|
| **Preseason** | 49 | **3** | **6.12%** | **0** |
| Regular season | 272 | 1 | 0.37% | 14 |
| Postseason | 13 | 0 | impossible | — |

Preseason ties are **16.7× more likely** than regular-season ties. The 2025 ties were
LV–SEA 23:23, MIA–CHI 24:24 and JAX–NO 17:17, each with `winner: false` on both competitors
and a final `period` of 4. Treat the draw path as a core case in the preseason, not an edge
case: in `OUTCOME_ONLY` the draw option matters constantly, and in `EXACT_SCORE` the
tie-scoring branch fires roughly sixteen times more often than it does in the regular season.

**The Hall of Fame Game is special in three ways.** It is a single-game matchday; it is
played at a neutral site (`neutralSite: true`, Tom Benson Hall of Fame Stadium, Canton OH);
and its `shortName` uses a different delimiter:

```json
{ "shortName": "CAR VS ARI", "name": "Carolina Panthers at Arizona Cardinals",
  "neutralSite": true, "notes": [{ "headline": "Hall of Fame Game" }] }
```

`shortName` says `VS` while `name` still says "at". They contradict each other, which is one
more reason never to parse either string for team identity — always read `competitors[]` and
select on `homeAway`.

**An entire preseason can be cancelled, and ESPN keeps the games.** The 2020 preseason was
cancelled in full. `?seasontype=1&dates=2020` still returns all 49 events, every single one:

```json
"status": { "id": "5", "name": "STATUS_CANCELED", "state": "post",
            "completed": false, "description": "Canceled" }
```

with `score` values of `"0"` and `"0"`.

This is the case that proves why scoring must require `completed === true` and not merely
`state === 'post'`. A check on `state` alone would have treated 49 games that were never
played as legitimate 0:0 draws and awarded points to every player who predicted a tie. **Use
2020 season type 1 as a regression fixture** — it is real data, not a hypothetical.

### Supporting endpoints

```bash
# All 32 teams — sync once per season into a local teams table
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams

# Single game, if you ever need to re-check one in isolation
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/summary?event={eventId}

# Ultra-light status only (~316 bytes) — a fallback, not the primary live path
GET https://sports.core.api.espn.com/v2/sports/football/leagues/nfl/events/{id}/competitions/{id}/status
```

**Do not build live polling on the per-game status endpoint.** NFL games are heavily
clustered — a dozen or more kick off simultaneously on a Sunday. One scoreboard call returns
the entire concurrent slate, so a single request per poll covers everything. Per-game polling
would multiply your request volume by ~15 for no benefit.

### Known quirks to code around

- **`.pvt` URLs.** Some `sports.core.api.espn.com` responses contain `$ref` links pointing at
  `sports.core.api.espn.pvt`, an internal hostname that does not resolve publicly. If you
  follow any `$ref`, rewrite `.pvt` → `.com` first.
- **Calendar dates vs. event availability.** The date ranges reported by the calendar do not
  always align perfectly with when events actually appear. Never assume a week is empty
  because the calendar says it started.
- **No stability guarantee.** Fields appear, disappear and get renamed without notice. This
  is why parsing must be permissive — see below.

### Proven client patterns

This repository contains a working reference implementation of an ESPN ingest layer that is
worth mirroring:

- `espn_service/clients/espn_client.py` — one centralised client, per-domain routing,
  explicit timeouts, retry with exponential backoff, typed exceptions for not-found and
  rate-limited
- `espn_service/apps/ingest/services.py` — upsert keyed on the external ESPN ID, per-record
  try/except so one bad record cannot fail a batch, raw payload retained, status derived from
  `state` + `completed` exactly as described above (see `_parse_event_status`)

---

## DATA MODEL

Adapt these to whatever already exists — do not blindly create parallel tables. See the
migration section first.

### `nfl_teams`

Synced once per season. `espn_team_id` (text) unique, abbreviation, display name, short
name, logo URL, primary/alternate colours, conference, division.

Never store `-1` / `-2` as teams. TBD is represented by `NULL` team references on the game,
not by a placeholder team row.

### `competitions` — the unit a champion is won in

One level above matchdays. Without this, a league has exactly one standings table per season
and the preseason cannot have its own champion.

```
id                uuid pk
league_id         uuid not null references leagues(id)
season_year       int  not null
kind              text not null   -- 'preseason' | 'regular_season' | 'postseason'
label             text not null   -- "Preseason 2026", "Hauptrunde 2026"
status            text not null default 'upcoming'
                        -- upcoming | active | provisional | complete | abandoned
champion_user_ids uuid[] not null default '{}'   -- plural: shared titles are allowed
completed_at      timestamptz null
UNIQUE (league_id, season_year, kind)
```

`champion_user_ids` is an array from the outset, not a nullable single reference. Shared
titles are an accepted outcome, and retrofitting the array later would mean migrating a table
that holds permanent records.

There is deliberately **no `prediction_mode` column** — every competition in a league uses
the league's mode. See the competitions section for why that matters more than it looks.

### `matchdays` — the unit of scoring, replacing any bare `week` integer

```
id                  uuid pk
competition_id      uuid         not null references competitions(id)
season_year         int          not null     -- 2026, the SEASON year
season_type         smallint     not null     -- 1 pre, 2 regular, 3 post
week_number         smallint     not null     -- within the season type
label               text         not null     -- "Week 12", "Wild Card", "Preseason Week 1"
starts_at           timestamptz  not null     -- from leagues[0].calendar
ends_at             timestamptz  not null
is_predictable      boolean      not null default true   -- false for Pro Bowl (3/4)
status              text         not null default 'upcoming'
                                 -- upcoming | in_progress | provisional | final
UNIQUE (season_year, season_type, week_number)
```

`label` always comes from the calendar, never from `week_number` — preseason week 2 is
labelled "Preseason Week 1" and rendering the number would be wrong.

Standings, weekly winners and multipliers all scope to `competition_id`. The same scoring
code computes the preseason table and the regular-season table over different scopes, which
is what keeps this additive rather than a second implementation.

Every game belongs to exactly one matchday. Every weekly result, weekly winner and
standings row keys on `matchday_id`, **never on a `week` integer** — regular week 1 and
Wild Card week 1 would otherwise collide and merge two unrelated sets of results.

Bonus questions get their own virtual matchdays, as they do today.

### `nfl_games`

```
id                    uuid pk
espn_event_id         text unique not null    -- THE stable anchor
matchday_id           uuid not null references matchdays(id)

-- identity: immutable once observed with real teams
season_year           int not null
season_type           smallint not null
home_team_id          uuid null references nfl_teams(id)   -- NULL while TBD
away_team_id          uuid null references nfl_teams(id)
matchup_resolved_at   timestamptz null   -- set when both teams first become real

-- schedule: mutable, this is what flexing changes
kickoff_at            timestamptz not null
kickoff_tbd           boolean not null default false   -- ESPN timeValid === false
venue_name            text
neutral_site          boolean not null default false
broadcast             text

-- deadline & lock
deadline_at           timestamptz not null
locked_at             timestamptz null    -- SET ONCE. Never cleared. Ever.

-- live state: display only, never scored
status                text not null default 'scheduled'
live_home_score       smallint
live_away_score       smallint
period                smallint
display_clock         text

-- final result: the only scoring input
final_home_score      smallint
final_away_score      smallint
finalized_at          timestamptz
result_source         text not null default 'espn'   -- espn | admin_override

raw_payload           jsonb          -- last successful ESPN event object
last_synced_at        timestamptz
created_at, updated_at
```

Indexes: `(matchday_id)`, `(kickoff_at)`, `(status)`, partial index on
`(kickoff_at) WHERE status IN ('scheduled','in_progress')`.

`predictions.game_id` references `nfl_games(id)` **`ON DELETE RESTRICT`**.

### `game_revisions` — append-only change log

```
id, game_id, field_name, old_value (text), new_value (text),
source text,          -- espn_sync | admin_override | system
observed_at timestamptz, actor_id uuid null
```

Written by the ingest diff for every changed field. This is what lets you show a player
"this game moved" and lets you reconstruct after the fact why a score changed.

### `game_ingest_anomalies` — the quarantine

```
id, espn_event_id, game_id null, anomaly_type text, detail jsonb,
detected_at, resolved_at null, resolved_by uuid null
```

Suspicious changes land here **instead of being applied**. See the anomaly rules below.

### `matchday_league_settings` — weighted playoff rounds

```
id, league_id, matchday_id,
points_multiplier   numeric(4,2) not null default 1.00,
multiplier_locked_at timestamptz null,
UNIQUE (league_id, matchday_id)
```

Per league, because two leagues in the same app may weight the playoffs differently.

### `prediction_scores` — store the arithmetic, not just the answer

```
prediction_id, game_id, matchday_id,
category        text,            -- exact | differential | tendency | outcome | miss
base_points     numeric(6,2) not null,
multiplier      numeric(4,2) not null default 1.00,
final_points    numeric(8,2) not null,   -- base_points * multiplier
computed_at     timestamptz not null,
UNIQUE (prediction_id)
```

Persisting all three columns makes every total reproducible and auditable, and makes it
obvious in the UI why a playoff hit was worth 7.5 instead of 5.

**Use `NUMERIC` for every points and share value, never `float`/`double precision`.** With
multipliers like 1.5 and 2.5 and weekly-win shares of 1/3, binary floating point will
produce totals that do not add up and standings that look wrong to the one person who
checks by hand.

### `job_runs`

`job_name, started_at, finished_at, duration_ms, status (success|partial|failed),
records_seen, records_changed, parse_failures, error_detail jsonb`. One row per run of every
job below.

---

## FLEX & DEADLINE STATE MACHINE

This is the heart of the work. Implement it as an explicit, unit-tested pure function —
`resolveSchedule(currentGameRow, incomingEspnEvent, leagueConfig, now) → { updates, revisions, anomalies, notifications }` — not as scattered `if` statements inside the sync loop.

### Rule 1 — `locked_at` is set once and never cleared

Enforced by a `BEFORE UPDATE` trigger: a transition from a non-NULL `locked_at` to NULL or
to a different value is rejected at the database level. Not a convention, a constraint.

### Rule 2 — deadlines ratchet earlier, never later

While `locked_at IS NULL`:

```
deadline_at = kickoff_at - league.lock_offset_minutes
```

recomputed on every sync, in both directions.

Once `locked_at IS NOT NULL`, `deadline_at` is **frozen**. A game that has been flexed to a
later slot after its deadline already passed stays locked.

The reason is not laziness, it is fairness: once a deadline passes, every player in the
league can see everyone else's prediction for that game. Re-opening it would let a player
edit their pick with full knowledge of the others'. Locked is locked.

### Rule 3 — flexing earlier moves the deadline earlier

If a game is flexed to an earlier kickoff and `locked_at IS NULL`, the deadline moves
earlier with it. If the newly computed deadline is already in the past at sync time, lock
the game immediately (`locked_at = now()`), write a revision, and notify the affected
players — those with a prediction, so they know it stands, and those without, so they know
why they can no longer enter one. Predictions already submitted remain valid and are scored
normally.

### Rule 4 — a game with `timeValid: false` is never auto-locked

When ESPN reports `timeValid: false`:

- Set `kickoff_tbd = true`
- Store the placeholder in `kickoff_at` but mark the derived deadline **provisional**
- **Never** set `locked_at` for a TBD game, regardless of what the placeholder timestamp
  says
- Show "Anstoßzeit noch nicht terminiert" in the UI, never a fake clock time

When `timeValid` flips to `true`, the real kickoff and a real deadline apply, a revision is
written, and every league member is notified that the game now has a time. If the real
kickoff arrives with less than `min_notice_minutes` (config, default 120) before the
deadline, still apply it, but flag the game and notify with high priority — players get
whatever window remains.

**Safety net:** if a TBD game's placeholder date passes and `timeValid` is still `false`,
do not lock and do not score. Raise an anomaly for admin review. This is the "ESPN forgot
about a game" case and it should page a human, not resolve itself.

### Rule 5 — a game moving to another week keeps its predictions

If the incoming `season_type` / `week.number` differs from the stored matchday, this is a
matchday **reassignment**, not a new game:

1. Update `matchday_id` on the same game row. The predictions ride along untouched —
   they reference the game, not the week.
2. Write a `game_revisions` entry.
3. Recompute standings for **both** the old and new matchday.
4. Notify the league.
5. If the old matchday was already `final`, downgrade it to `provisional` and recompute.

### Rule 6 — team changes: expected once, suspicious thereafter

| Transition | Action |
|---|---|
| `NULL` → real team (playoff seeding resolves, ESPN `-1`/`-2` → real) | **Apply.** Set `matchup_resolved_at`, open the game for prediction, notify players that the matchup is known. |
| real team → the *same* team | No-op. |
| real team → a *different* real team, on an existing event ID | **Do not apply.** Write a `game_ingest_anomalies` row of type `matchup_changed`, keep the stored teams, alert the admin. |

That last row is the guard that makes silent prediction corruption impossible. If ESPN
recycles an event ID or returns a corrupt record, the worst case is a stale game that an
admin must look at — never a set of predictions quietly pointing at a different fixture.

### Rule 7 — cancellation, not deletion

A game that vanishes from ESPN's response for its week across **two consecutive successful
syncs** is marked `status = 'cancelled'`, never deleted. Two syncs, not one, because a
single partial response must not be able to cancel anything.

A cancelled game is excluded from matchday finality and scores 0 for everyone, unless the
admin chooses to void it entirely (excluded from the matchday as if it never existed).
Present that as an explicit admin decision, logged, not a default.

### Rule 8 — an empty or failed response changes nothing

If a week's fetch fails, times out, or returns `events: []`, the job records the failure in
`job_runs` and **makes no writes for that week**. An empty array is never interpreted as
"all games for this week are gone".

---

## WEIGHTED PLAYOFF ROUNDS

Optional, off by default, configured per league by the admin.

- Each matchday carries a `points_multiplier` (`NUMERIC(4,2)`, default `1.00`) per league.
- Typical use: Wild Card ×1.5, Divisional ×2, Conference Championship ×2.5, Super Bowl ×3.
  Offer these as a one-click preset alongside free entry. Regular-season weeks default to
  ×1.00 but are settable too, in case I want a double-points final week.
- Scoring computes `base_points` from the existing rules, then
  `final_points = base_points × multiplier`. Both are persisted. A miss scores 0 regardless
  of the multiplier — multiplying zero must not become a special case that yields
  something else.
- **A multiplier freezes when the round's first game locks.** Set `multiplier_locked_at` at
  that moment. Before that, the admin can change it freely (audited). Afterwards, changing
  it requires an explicit "override and recalculate" admin action that writes to the audit
  log, triggers an idempotent rescore of that entire round, and shows a visible notice to
  all players. Nobody gets to quietly reweight a round after seeing the results.
- Weekly winners within a weighted round are computed from `final_points`. The overall
  standings sum `final_points`. Tie-break chains and split weekly-win shares work exactly
  as they do today, on `NUMERIC` values.
- The rules page must render the active multipliers, so the scoring scheme on screen is
  always the one actually in force.

---

## COMPETITIONS AND THE PRESEASON WARM-UP

The preseason runs as a **self-contained warm-up competition with its own champion**. It is
not a set of extra games appended to the regular season.

### Separation is absolute

- The preseason competition has its own standings, its own weekly winners and its own
  champion.
- **No preseason points reach the regular-season table.** Not weighted down, not partially —
  zero. Assert this with a query in the test suite, not just in review.
- A player can win the warm-up title and finish last in the real season. That is the point.

### The champion is set once, and having no champion is a valid outcome

When a competition's last matchday goes final, compute the standings, write
`champion_user_ids` and `completed_at`, and set `status = 'complete'`. Those fields follow the
same ratchet discipline as `locked_at`: written once, never silently rewritten. Changing them
requires an audited admin recalculation, whose only legitimate trigger is an ESPN score
correction.

**If no game in the competition was ever completed, the competition ends `abandoned` with an
empty `champion_user_ids`.** The 2020 preseason is exactly this case: 49 games, all cancelled,
nothing played. Crowning somebody off an empty table would be worse than crowning nobody.
Assert the empty-array outcome explicitly.

Ties for the title are expected — the preseason is only four matchdays — and are resolved with
the **existing tie-break chain** already used for the regular season (total points, then weekly
wins, then correct outcomes, then alphabetical). Where players remain level, they are **joint
champions**, all recorded in `champion_user_ids`, and the UI renders shared ranks the same way
it already does elsewhere. Do not invent a preseason-specific decider.

### The preseason inherits the league's prediction mode

A league's mode applies to every one of its competitions. Two consequences follow, and both
need to be surfaced in the UI rather than discovered later:

1. **Opting into the preseason moves the mode-lock deadline a month earlier.** The existing
   rule is that a league's prediction mode becomes immutable once the league has started,
   where started = the first game's deadline passing. If the preseason is enabled, the first
   deadline is the Hall of Fame Game in early August, not Week 1 in September. This is correct
   — the mode must be settled before any prediction is taken — but the league-creation and
   preseason-opt-in flows must both say so explicitly, because an admin who expected until
   September to decide will otherwise be locked in without warning.
2. **`EXACT_SCORE` on preseason games is close to a lottery.** Starters play little, so
   scorelines are near-noise. Leagues on exact-score mode will find the warm-up title
   substantially luck-driven. That is arguably fair — nobody has an edge — but the rules page
   must state plainly how the mode behaves here so it reads as a deliberate choice.

### Opt-in

Preseason participation is opt-in per league, defaulting to off. A league that opts in after
the preseason has started counts **only the matchdays whose deadlines have not yet passed**,
and the UI says so on the opt-in screen and in the preseason standings. Never retroactively
score a matchday nobody could predict.

### The Hall of Fame Game

It counts, as its own single-game matchday. Two things follow:

- With one game and 10–50 players, most of the league will tie for that matchday's win. The
  existing split-share logic handles it — `1/n` as `NUMERIC`, summing to exactly 1 — but test
  it at the extreme (30 of 50 tied), because a one-game matchday is the worst case for any
  rounding in the share calculation.
- It is a neutral-site game. Label the outcome options by team name, never "Heimsieg" /
  "Auswärtssieg".

**Reuse `matchdays.is_predictable` rather than adding a field.** It already exists for
excluding the Pro Bowl. If the Hall of Fame window is missed in a given year, the game is
still ingested and displayed — it simply is not scored. That makes a missed window cost one
matchday instead of the competition, with no code change either way.

## SCORING PIPELINE

**Live scores and final results are different things and live in different columns.**

- `live_home_score` / `live_away_score` / `period` / `display_clock` are display-only.
  Nothing reads them for scoring. Ever.
- `final_home_score` / `final_away_score` are written **only** when
  `status.type.state === 'post' && status.type.completed === true`, together with
  `finalized_at`.
- Scoring reads only the final columns.

**Scoring is a pure, idempotent function** of (prediction, final result, league config,
matchday multiplier). Running it twice produces identical rows. Running it on 200 games
produces the same result as running it on each game individually. This is what makes
recalculation safe.

**Retroactive corrections.** ESPN does occasionally amend a final score after the fact. If
a re-check finds `final_home_score` or `final_away_score` differ from what is stored:
write a `game_revisions` entry, rescore that game, recompute the affected matchday and
overall standings, and surface a visible notice to the league. Never apply a silent
change to a score that has already been used to award points.

**Admin overrides are sticky.** When an admin corrects a result manually, set
`result_source = 'admin_override'`. Subsequent ESPN syncs update the live/status fields but
**must not overwrite the final score** of an overridden game. Without this, the next
nightly sync silently reverts the admin's correction and nobody notices for a week. Clearing
the override is an explicit admin action.

**Matchday finality.** A matchday becomes `final` only when every non-cancelled game in it
is `final` and no revision has landed within the last 30 minutes. Until then it is
`provisional`, and that state is exposed in the API response and rendered in the UI — not
inferred by the client.

---

## SCHEDULED JOBS

Supabase Edge Functions invoked by `pg_cron` via `pg_net`. Each function requires a shared
secret in a header and verifies it before doing anything; the secret lives in Supabase
secrets and never reaches the client bundle. The functions are not publicly invocable.

| Job | Cadence | What it does |
|---|---|---|
| `sync-teams` | Weekly, plus manually | Upsert all 32 teams |
| `sync-season-schedule` | Daily 06:00 UTC, **plus 4×/day Tue–Thu** | Walk the calendar, sync every week of season types 1, 2 and 3. ~27 requests. Tue–Thu is when flex decisions are announced. Preseason weeks need no extra polling — they never flex — but sync them anyway so cancellations are caught. |
| `sync-upcoming-games` | Hourly | Re-sync only weeks containing a game kicking off within 36h. Usually 1 request. Catches late changes. |
| `poll-live-scores` | **Every 5 minutes**, only while at least one game is `in_progress` or kicks off within 15 minutes | **One** scoreboard call for the affected week covers the entire concurrent slate. Updates live columns. |
| `finalize-and-score` | Every 5 minutes | Pick up newly-final games, snapshot finals, run scoring, recompute matchday standings, set matchday finality. |
| `recheck-final-games` | Hourly for 24h after a game finalises | Catch retroactive ESPN score corrections. |

The live poll and the finaliser both run on a 5-minute tick and can share one invocation.

**Every job, without exception:**

- Takes `pg_advisory_xact_lock(hashtext('job:' || job_name))` at the start. Two overlapping
  cron ticks must never both execute the body — one takes the lock, the other exits cleanly
  and records a skip.
- Writes a `job_runs` row with counts and duration.
- Wraps each event in its own try/catch. **One malformed record never fails the batch** —
  increment `parse_failures`, log the raw record, continue.
- Uses a 10-second timeout, 3 retries with exponential backoff plus jitter, and a circuit
  breaker that stops calling after repeated failures, logs, and alerts rather than looping.
- Sets a descriptive `User-Agent`.
- Is safe to invoke manually from the admin health view, at any time, without side effects
  beyond the ones it always has.

**Idle is idle.** During the off-season, and on days with no games, the live poll must not
run at all. Gate it on a cheap query against `nfl_games`, not on wall-clock guesswork.

---

## PARSING EXTERNAL DATA

Validate ESPN payloads with Zod schemas using **`.passthrough()`**, not `.strict()`.

This is deliberate and it is the opposite of the right rule for inbound client requests.
Requests from the browser stay `.strict()` so unknown fields are rejected. ESPN payloads use
`.passthrough()` because ESPN adds fields constantly, and `.strict()` would convert a
cosmetic upstream addition into a total ingest outage in the middle of a game week.

- Validate the small set of fields you actually consume; ignore the rest.
- Every field you read must have a defined fallback. `competitors[].score` is a string;
  `broadcast` may be `""`; `venue` may be absent; `notes` may be empty.
- Select competitors by `homeAway`, never by array index.
- Retain the raw event object in `raw_payload` so any parsing bug can be diagnosed after
  the fact without replaying the season.
- Deduplicate by `espn_event_id` within a single response before writing — a payload
  containing the same event twice must not produce two rows or two conflicting updates.

---

## ROW LEVEL SECURITY

RLS is the security boundary here, not the UI. Hiding a control is presentation, not
protection.

| Table | `authenticated` SELECT | `authenticated` INSERT / UPDATE / DELETE |
|---|---|---|
| `nfl_teams`, `matchdays`, `nfl_games` | allowed | **denied to all** — written only by the service role inside Edge Functions |
| `game_revisions` | allowed (it is a feature: players see why a game moved) | denied |
| `game_ingest_anomalies`, `job_runs` | league admins only | denied |
| `matchday_league_settings` | league members | admin only, via a `SECURITY DEFINER` function that also enforces the `multiplier_locked_at` rule |
| `predictions` | **own rows always; other players' rows only where that game's `locked_at IS NOT NULL`** | INSERT/UPDATE own row only, `WITH CHECK (auth.uid() = user_id)` and the target game unlocked. No DELETE. |
| `prediction_scores`, standings | league members | **denied to all** — server-computed only |

The visibility rule on `predictions` must live **inside the policy**, so an unlocked
prediction belonging to another player cannot be selected at all — not fetched and then
hidden by the client. Test it by querying the table directly with another member's
credentials.

There is no route, function or policy anywhere that accepts points, ranks or game results
from a client. Results enter through ingest or an audited admin override, and nothing else.

Test matrix to run at the end of every phase, with three seeded users — A and B in league 1,
C in league 2:

- As B, read A's prediction for an unlocked game → **no rows**
- As B, read A's prediction for a locked game → allowed
- As B, write a prediction with A's `user_id` in the payload → **rejected**
- As B, write a prediction for a game whose `locked_at` is set → **rejected**
- As C, read league 1's games, predictions, scores or settings → **no rows**
- As B (a member, not an admin), change a matchday multiplier, override a result, or invoke
  a job endpoint → **rejected**
- With no session at all, any of the above → **rejected**
- Enumerate predictions, profiles and settings by sequential or guessed ID → **no rows**

---

## UX

**Live scoreboard.** During game windows, players see the live score, quarter and clock,
refreshing on the 5-minute cadence. Show the data's age ("aktualisiert vor 2 Min."), so a
stale number never looks live. Prefer Supabase realtime subscriptions on the games table
over client-side polling — the server already polls, the client should just receive.

**Points stay honest during live play.** Standings and points recompute only when games are
final. Show the matchday as `vorläufig` with a count of games still running, rather than
letting ranks flicker all Sunday afternoon. If you want a live "wie ich gerade stehe"
indicator, label it clearly as a projection and keep it visually distinct from the real
table.

**Schedule changes are announced, not silent.** When a game is flexed:

- A notice on the game card: *"Diese Partie wurde verlegt: Sonntag 19:00 → Montag 02:20.
  Deine Tipps bleiben erhalten."*
- A league-wide change feed listing all schedule changes for the current week
- The reassurance — "deine Tipps bleiben erhalten" — is not decoration. It is the single
  most valuable piece of copy in this feature, because the moment a player sees a game move
  is the moment they wonder whether their prediction survived.

**TBD games** show "Anstoßzeit noch nicht terminiert" with the week window, a provisional
deadline clearly marked as such, and an explanation that the time will be confirmed by the
NFL. Never render a placeholder timestamp as if it were real.

**Playoff games** open for prediction only once both teams are known. Until then, show the
round and the slot ("Wild Card, Sonntag") with a "Teilnehmer noch offen" state. Notify
players when a round's matchups resolve — the window between seeding and kickoff is short.

**Draw availability depends on the season type, and it is not a single rule.** Both ends of
this axis are real, so do not collapse it into "draws only happen in the regular season":

| `season_type` | Draw in `OUTCOME_ONLY` | Why | Enforcement |
|---|---|---|---|
| 1 preseason | **allowed, ~6% of games** | no overtime at all | — |
| 2 regular | allowed, ~0.4% of games | overtime can still end level | — |
| 3 postseason | **rejected** | play continues until someone wins | `CHECK` constraint |

In the postseason the draw option must be absent from the UI **and** rejected by a `CHECK`
constraint. In the preseason it is a core path that players will use often, so give it equal
visual weight to the two win options rather than tucking it away.

**Neutral-site games** — the Super Bowl, international games — carry `neutralSite: true`.
Label the outcome options by team name rather than "Heimsieg" / "Auswärtssieg", which are
meaningless when neither team is at home.

**The preseason must read as a separate contest, not as early-season games.** Give it its own
visibly distinct standings screen with its own title ("Preseason 2026") and its own trophy, and
state on both the preseason table and the regular-season table that no points move between
them. The failure mode to design against is a player accumulating 40 points in August and
expecting to start September in the lead. Announce the warm-up champion as its own moment when
the last preseason matchday goes final, and keep that record visible once the real season
starts.

**Timezones.** Store everything as `timestamptz` in UTC; display in the league timezone
(default Europe/Berlin). Note that a Sunday Night Football kickoff lands after midnight
Berlin time, so grouping games into "days" by Berlin local date splits Sunday's slate across
two days. Group by the ESPN matchday window, not by local calendar date.

**Accessibility** (target WCAG 2.2 AA, consistent with the rest of the app). Live score
updates go through a `polite` live region, announced on score changes only — not every
poll, and never a running clock. Respect `prefers-reduced-motion` for any score-change
animation. Never convey a game's state by colour alone: live, final, locked, postponed and
moved each need text or an icon.

---

## MIGRATING THE EXISTING APP

Predictions already exist. The migration must not endanger them.

1. **Inventory first.** Before proposing any schema change, read the current schema and
   report back: how games are represented today, how predictions reference them, how weeks
   are keyed, what already exists that these tables would duplicate. Adapt the model above
   to what is there — do not create a parallel `nfl_games` alongside an existing `games`.
2. **Additive migration.** Add new columns and tables. Do not drop or rename anything that
   predictions depend on, in this migration or the next one.
3. **Backfill `espn_event_id`** for existing games by matching on kickoff date plus both
   teams. Produce a report of matched, ambiguous and unmatched rows. **Ambiguous and
   unmatched rows are resolved by a human**, through an admin screen — never by a heuristic
   guess, because a wrong match silently reattaches a prediction to the wrong game.
4. **Verify before proceeding.** Row counts for `predictions` before and after the migration
   must be identical, and every prediction must still resolve to exactly one game. Assert
   this in the migration itself and fail loudly if not.
5. **Reversible.** Every migration ships with a tested down-migration.
6. **Back it up first.** Take a verified database snapshot before running any of this against
   real data, and confirm you can restore it into a scratch database. An untested backup is
   not a backup.

---

## PHASES

Each phase is independently runnable, testable and shippable, and ends with the RLS test
matrix for whatever it added.

**A note on sequencing.** The preseason is deliberately placed early, and not as a
convenience. It is the only phase of the NFL year where every game arrives with a valid
kickoff time and fully resolved teams, where nothing flexes, and where getting it wrong costs
a warm-up title rather than the season that counts. That makes it a live shakedown of the
whole ingest → live-score → finalise → score chain, on real data with real players, weeks
before Week 1. If the pipeline has a defect, it should surface in August. Do not defer the
preseason to the end because it looks like the smallest feature — its value is mostly in when
it runs.

**Phase 1 — Foundation.** Schema inventory, additive migration, backfill with the manual
resolution screen, `competitions` + `nfl_teams` + `matchdays` populated from the calendar,
ESPN client with timeouts/retry/circuit breaker, Zod schemas, `job_runs`, admin health view.
No player-facing change.

**Phase 2 — Schedule ingest, read-only.** `sync-season-schedule` for all three season types,
the `resolveSchedule` function with its full unit-test suite, `game_revisions`,
`game_ingest_anomalies`, schedule display with TBD and matchup-pending states, calendar-sourced
labels (so preseason week 2 renders as "Preseason Week 1"). Predictions not yet wired to the
new games.

**Phase 3 — Deadlines and flex.** The lock trigger, the deadline ratchet, prediction entry
against ingested games, the change feed and notifications, week-reassignment handling. This
phase carries the invariant — it gets the heaviest test coverage.

**Phase 4 — Live scores.** The 5-minute poll gated on active games, live columns, realtime
subscriptions, the live scoreboard UI with data-age display and provisional standings.

**Phase 5 — Finalisation and scoring.** Final snapshot, idempotent scoring, matchday
finality, retroactive correction handling, sticky admin overrides, recalculation.

**Phase 6 — The preseason warm-up competition.** Per-league opt-in with the mode-lock warning,
the preseason competition and its separate standings, the set-once champion with joint titles,
the `abandoned` path for a cancelled preseason, the Hall of Fame single-game matchday via
`is_predictable`, draw handling given the 6% tie rate, and the UI making unmistakably clear
that these points do not carry into the regular season.

This phase is the pipeline shakedown described above. Run it against live preseason data
before the regular season opens.

**Phase 7 — Weighted playoffs.** Per-league multipliers, presets, the freeze-on-first-lock
rule, override-and-recalculate, rules page rendering the active scheme, playoff draw
prohibition, neutral-site labelling.

**Phase 8 — Operations.** Anomaly review screen, manual job invocation, revision history per
game, league export including full schedule and revision history, alerting when a job fails
or a matchday stays provisional longer than expected.

---

## REQUIRED EDGE CASES

Every one of these gets a named test. For each, state the expected behaviour in the plan.

**Flex and deadlines**
1. Game flexed later, deadline already passed → stays locked, predictions intact, notice shown
2. Game flexed later, deadline not yet passed → deadline moves later, predictions intact
3. Game flexed earlier, deadline not yet passed → deadline moves earlier, predictions intact
4. Game flexed earlier, new deadline already in the past → locks immediately, notifies, predictions intact
5. `timeValid: false` → provisional deadline, never auto-locks
6. `timeValid` flips to `true` → real deadline applies, players notified
7. Real kickoff announced with less than the minimum notice → applied, flagged, high-priority notice
8. TBD placeholder date passes while still `timeValid: false` → anomaly raised, no lock, no scoring
9. Game moves from week 15 to week 16 → predictions ride along, both matchdays recomputed
10. Game moved out of a matchday that was already `final` → matchday reverts to provisional

**Data integrity**
11. Playoff game `-1`/`-2` → real teams → applied, matchup opens for prediction
12. Real team replaced by a different real team on an existing event ID → quarantined, not applied
13. Game disappears from one sync → no change; from two consecutive syncs → cancelled, not deleted
14. Cancelled game → 0 points for all, or admin-voided; matchday finality accounts for it
15. ESPN returns `events: []` for a week → nothing is written
16. ESPN returns 500 or times out mid-sync → partial run recorded, no corruption
17. Same event ID appears twice in one payload → deduplicated before writing
18. A field you parse is missing entirely → fallback applied, one `parse_failure` counted, batch continues
19. Two cron ticks overlap → advisory lock; second run exits cleanly and records a skip
20. Any job re-run twice over the same data → byte-identical result

**Scoring**
21. Final score corrected 3 days later → revision written, idempotent rescore, standings updated, players notified
22. Admin overrides a result, then the next sync runs → override survives
23. Regular-season overtime tie → scored as a draw in both prediction modes
24. Postseason draw submitted in `OUTCOME_ONLY` → rejected by the CHECK constraint
25. Regular week 1 and postseason week 1 both exist → separate matchdays, separate standings
26. Pro Bowl (season type 3, week 4) → excluded from prediction and scoring
27. Super Bowl → `neutralSite: true`, labelled by team name, weighted correctly
28. January games → filed under the previous season year
29. Weighted round: a hit scoring 5 base at ×2.5 → 12.50 stored as base 5.00 / multiplier 2.50 / final 12.50
30. Multiplier changed before the round's first lock → applied and audited; after → requires override, rescore and player notice
31. Three or more players tied for a weighted weekly win → shares split as `NUMERIC`, summing exactly to 1
32. A miss in a weighted round → 0, not a multiplied non-zero

**Predictions**
33. Prediction submitted exactly at the deadline second → server time decides, documented behaviour
34. Any job run against a seeded database → `predictions` table unchanged, asserted
35. Attempt to delete a game that has predictions → refused by `ON DELETE RESTRICT`

**Preseason and competitions**
36. **Whole preseason cancelled** (2020 fixture: 49 games, all `STATUS_CANCELED` with
    `state: 'post'`, `completed: false`, scores `"0"`) → nothing scored, competition ends
    `abandoned`, `champion_user_ids` empty, **no champion crowned**
37. Preseason tie in `OUTCOME_ONLY` → draw scores correctly; both `winner` flags are `false`
38. Preseason tie in `EXACT_SCORE` → tie branch applies (exact 5 / correct tendency 3)
39. Preseason game level at the end of period 4 → final immediately, not held open for overtime
40. ESPN preseason `week.number = 2` → displayed as "Preseason Week 1" from the calendar label
41. Hall of Fame Game → `neutralSite: true`, labelled by team name; `shortName` `"CAR VS ARI"`
    never parsed for team identity
42. Single-game matchday with 30 of 50 players tied → weekly-win shares of 1/30 as `NUMERIC`,
    summing to exactly 1
43. Preseason competition completes → `champion_user_ids` written once; a second finalisation
    pass changes nothing
44. Preseason champion is a tie → all tied players recorded as joint champions, shared ranks
    rendered
45. **Preseason points never appear in regular-season standings** → asserted by direct query
46. League opts in mid-preseason → only matchdays with unexpired deadlines count, stated in
    the UI
47. Preseason enabled → mode-lock fires at the Hall of Fame deadline, not at Week 1; admin was
    warned at opt-in
48. Preseason and regular-season week 1 both exist → separate matchdays in separate
    competitions, separate standings

---

## NON-GOALS

No real-money handling or payments. No public sign-up — invite only. No betting odds. No
live play-by-play ticker beyond score, quarter and clock. No mid-season prediction-mode
switching. No migration of predictions between modes. No fantasy scoring. No player-level
statistics beyond what bonus questions already need.

No separate prediction mode per competition — every competition uses the league's mode. No
combined year-long leaderboard spanning the preseason and the regular season. No team-based
standings derived from preseason results (note that the two Hall of Fame participants play four
preseason games while the other thirty play three, so such a table would not be comparable
anyway).

---

## PLAN OUTPUT FORMAT

For each phase: goal · concrete tasks · files and Edge Functions touched · database
migrations · RLS policies added or changed, with their exact rule · acceptance criteria ·
test cases · risks · open decisions.

Then, separately:

- The full `resolveSchedule` decision table, as a table, before you implement it
- The RLS policy matrix for every table you touch
- The cron schedule with expected daily request volume against ESPN
- How you will verify the core invariant — that no prediction can be lost — end to end

**Ask your clarifying questions now. Wait for my approval before building.**
