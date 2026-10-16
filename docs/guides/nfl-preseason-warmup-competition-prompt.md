# Preseason Warm-Up Competition — Implementation Prompt

> A standalone, paste-ready prompt for Lovable's plan mode. It assumes the regular-season and
> playoff implementation already exists and describes only the preseason increment.
> Everything below the line is the prompt.

---

## ROLE

You are a senior full-stack architect extending a working application. Produce a phased
implementation plan before writing any code. Ask every clarifying question that would
materially change the approach, then wait for approval. Do not begin building until the plan
is approved.

This is an **increment on top of working code**, not a greenfield feature. Your first
obligation is to reuse what exists. A plan that rebuilds ingest, scoring or deadline logic
for the preseason is a failed plan.

## WHAT THIS BUILDS ON — VERIFY BEFORE PLANNING

The app is a private, invite-only NFL prediction game for 10–50 friends. Bragging rights
only: no money, no payments. German UI. Stack: Lovable + Supabase (Postgres with RLS, Edge
Functions in Deno, `pg_cron` + `pg_net` for scheduled work), React/TypeScript frontend.

The regular season and playoffs are **already implemented**. Before you plan anything, read
the current schema and code and confirm each of the following exists. Report anything that
does not — it changes the plan.

| Assumed to exist | What this increment needs from it |
|---|---|
| `nfl_games` keyed on a unique `espn_event_id`, upsert-only ingest with no DELETE path | Works unchanged for preseason games |
| `matchdays` keyed `(season_year, season_type, week_number)` with `is_predictable` and calendar-sourced labels | Preseason weeks slot in as new rows; `is_predictable` is reused |
| `sync-season-schedule` Edge Function walking the ESPN calendar | Extend its season-type loop to include type 1 |
| Live-score poll and a finalise-and-score job on a 5-minute tick | Work unchanged |
| A deadline ratchet with a set-once `locked_at` enforced by a DB trigger | Works unchanged |
| An idempotent, pure scoring function over (prediction, final result, league config, multiplier) | Reused as-is; only its scope changes |
| A tie-break chain and split weekly-win shares as `NUMERIC` | Reused as-is |
| `game_revisions`, `game_ingest_anomalies`, `job_runs` | Work unchanged |
| Predictions writable only by their owner, only before lock; no automated process writes them | **Must remain true.** This increment adds no exception. |

If the current implementation scopes standings and weekly results to a **league** rather than
to a competition, say so explicitly in your plan — that is the one real refactor here and the
riskiest part of this work. See the migration section.

## WHAT THIS ADDS

The NFL preseason as a **self-contained warm-up competition with its own champion**.

Four ESPN matchdays (season type 1) run as their own contest with their own standings, their
own weekly winners and a permanently recorded champion — entirely separate from the
regular-season table. A player can win the warm-up and finish last in the real season. That
is the point.

There is a second reason to build this, and it is the larger one. **The preseason is the only
phase of the NFL year where every game arrives with a valid kickoff time and fully resolved
teams, nothing flexes, and getting it wrong costs a warm-up title instead of the season that
counts.** It is a live shakedown of the entire ingest → live-score → finalise → score chain,
on real data with real players, before Week 1. Treat it as risk reduction that happens to be
a feature.

## INVARIANTS THAT CARRY OVER UNCHANGED

Restated because they constrain this work, not for re-litigation:

1. **No automated process may INSERT, UPDATE or DELETE a row in `predictions`.** Not the
   competition finaliser, not the champion-award job, not a trigger. Only the owning player,
   only before lock. Assert it with a test that runs every job against a seeded database and
   confirms `predictions` is unchanged.
2. **No game row is ever deleted.** Cancelled, never removed.
3. **Only `status.type.state === 'post' && status.type.completed === true` may feed scoring.**
   This one is load-bearing here for a specific reason — see the 2020 fixture below.
4. **Deadlines ratchet earlier, never later.** `locked_at` is set once.

## VERIFIED ESPN PRESEASON REFERENCE

All verified against the live API. Trust this over training data.

### Endpoint and shape

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard
      ?seasontype=1&week={1..4}&dates={seasonYear}
```

`dates` is the **season year**, not the calendar year. `seasontype=1` is the preseason.

Returns **49 games**: 1 Hall of Fame Game plus three weeks of 16. Verified for 2026 before
the season began:

- **All 49 carry `timeValid: true`** — real kickoff times, no placeholders anywhere
- **All teams resolved** — zero `-1`/`-2` TBD entries, unlike the playoffs
- Broadcast information present on most games
- **No flex exposure** — the NFL does not flex preseason games; windows are fixed at schedule
  release

This is why preseason ingest needs neither TBD resolution nor the flex ratchet. The existing
`resolveSchedule` logic runs against it harmlessly and should be left in place rather than
bypassed — a code path that is exercised but never triggered is exactly what you want before
the season that matters.

### Week numbers are offset from their labels

| ESPN `week.number` | `calendar` label | Window (2026) |
|---|---|---|
| 1 | Hall of Fame Weekend | Aug 6–12 |
| 2 | **Preseason Week 1** | Aug 13–19 |
| 3 | Preseason Week 2 | Aug 20–26 |
| 4 | Preseason Week 3 | Aug 27–Sep 8 |

Take the display label from `leagues[0].calendar[].entries[].label`. **Never render
`week.number` directly** — players would see "Woche 2" for what the NFL calls Preseason
Week 1.

### There is no overtime, so ties are common

Measured across 2025:

| Phase (2025) | Games | Ties | Rate | Went to OT |
|---|---|---|---|---|
| **Preseason** | 49 | **3** | **6.12%** | **0** |
| Regular season | 272 | 1 | 0.37% | 14 |
| Postseason | 13 | 0 | impossible | — |

Preseason ties are **16.7× more likely** than regular-season ties. The 2025 ties were
LV–SEA 23:23, MIA–CHI 24:24 and JAX–NO 17:17 — each with `winner: false` on both competitors
and a final `period` of 4.

The draw is a **core path** here, not an edge case. In `OUTCOME_ONLY` players will use it
constantly; in `EXACT_SCORE` the tie-scoring branch fires roughly seventeen times more often
than in the regular season and needs real test coverage.

### The Hall of Fame Game is special in three ways

```json
{ "shortName": "CAR VS ARI", "name": "Carolina Panthers at Arizona Cardinals",
  "neutralSite": true, "notes": [{ "headline": "Hall of Fame Game" }],
  "venue": { "fullName": "Tom Benson Hall of Fame Stadium",
             "address": { "city": "Canton", "state": "OH" } } }
```

- It is a **single-game matchday**, which is the worst case for weekly-win share arithmetic.
- It is **neutral-site**, so "Heimsieg" / "Auswärtssieg" are meaningless labels.
- Its `shortName` uses **`VS`** while `name` still says **"at"**. They contradict each other.
  Never parse either string for team identity — read `competitors[]` and select on `homeAway`.

### An entire preseason can be cancelled, and ESPN keeps the games

The 2020 preseason was cancelled in full. `?seasontype=1&dates=2020` **still returns all 49
events**, every one:

```json
"status": { "id": "5", "name": "STATUS_CANCELED", "state": "post",
            "completed": false, "description": "Canceled" }
```

with `score` values of `"0"` and `"0"`.

This is the case that makes invariant 3 load-bearing. A check on `state === 'post'` alone
would treat 49 games that were never played as legitimate 0:0 draws and award points to every
player who predicted a tie. **Adopt 2020 season type 1 as a golden regression fixture** —
real data, not a hypothetical. Any competition built on top of scoring must survive it.

## WHAT TO REUSE RATHER THAN REBUILD

Be explicit in your plan about which existing module each item comes from. If your plan
introduces a new implementation of anything in this table, justify it or drop it.

| Need | Reuse |
|---|---|
| Ingesting 49 preseason games | Existing `nfl_games` upsert path, unchanged |
| Preseason matchdays | Existing `matchdays` table; add rows for season type 1 |
| Excluding a matchday from scoring | **Existing `matchdays.is_predictable`** — already built for the Pro Bowl |
| Kickoff times, deadlines, locking | Existing deadline ratchet and `locked_at` trigger, unchanged |
| Live scores during preseason games | Existing 5-minute poll, unchanged |
| Detecting finals and scoring them | Existing finalise-and-score job and pure scoring function |
| Per-matchday weighting, if ever wanted | Existing `matchday_league_settings.points_multiplier` |
| Ranking, tie-breaks, shared ranks | Existing tie-break chain, unchanged |
| Splitting a weekly win among tied players | Existing `NUMERIC` share logic |
| Change history and job telemetry | Existing `game_revisions`, `job_runs` |

The only ingest change is **extending the season-type loop in `sync-season-schedule` to
include type 1**. That is roughly four extra requests per run.

## WHAT IS GENUINELY NEW

### 1. `competitions` — the unit a champion is won in

The current model gives a league one standings table per season. A separate warm-up title
needs one level above matchdays.

```sql
create table competitions (
  id                uuid primary key default gen_random_uuid(),
  league_id         uuid not null references leagues(id) on delete cascade,
  season_year       int  not null,
  kind              text not null check (kind in ('preseason','main')),
  label             text not null,                  -- "Preseason 2026", "Hauptrunde 2026"
  status            text not null default 'upcoming'
                      check (status in ('upcoming','active','provisional','complete','abandoned')),
  champion_user_ids uuid[] not null default '{}',
  completed_at      timestamptz,
  created_at        timestamptz not null default now(),
  unique (league_id, season_year, kind),

  -- an abandoned competition must never carry a champion
  constraint abandoned_has_no_champion
    check (status <> 'abandoned' or champion_user_ids = '{}'),
  -- a complete competition must have been stamped
  constraint complete_is_stamped
    check (status <> 'complete' or completed_at is not null)
);
```

`kind` is intentionally two values. **Playoffs stay inside `main`** as weighted matchdays,
which is how the existing multiplier design already works — do not split them out as part of
this increment. If the existing implementation already models postseason separately, leave it
alone; preseason is orthogonal to that choice.

`champion_user_ids` is an **array from the outset**, not a nullable single reference. Shared
titles are an accepted outcome, and retrofitting the array later would mean migrating a table
holding permanent records.

There is deliberately **no `prediction_mode` column**. Every competition in a league uses the
league's mode — see the mode-lock consequence below.

### 2. `matchdays.competition_id`

Every matchday belongs to exactly one competition. Standings, weekly winners and multipliers
scope to `competition_id`. The same scoring code computes the preseason table and the main
table over different scopes — which is what keeps this additive rather than a parallel
implementation.

Index `matchdays (competition_id)` and whatever standings tables you scope.

### 3. Champion award: atomic, exactly-once, reversible only by audit

When a competition's last predictable matchday reaches `final`, compute the standings and
award the title. Three properties matter.

**Atomic and idempotent.** The guard belongs in the `WHERE` clause, so a double-fired job or
two concurrent workers cannot double-award:

```sql
update competitions
   set champion_user_ids = $2,
       completed_at      = now(),
       status            = 'complete'
 where id = $1
   and completed_at is null;   -- the entire concurrency guard
```

Take the existing per-job advisory lock as well, but do not rely on it alone — the predicate
is what makes this correct.

**Set-once.** A `BEFORE UPDATE` trigger rejects any change to `champion_user_ids` or
`completed_at` once `completed_at` is non-null, mirroring the existing `locked_at` trigger.
Reuse that trigger's shape rather than inventing a new pattern.

**Correctable only through audit.** The one legitimate reason to re-award is an ESPN score
correction landing after the fact. Expose that as a `SECURITY DEFINER` admin function that
writes to the audit log, clears the stamp, rescores the competition and re-awards. Never as
a plain `UPDATE`.

**No champion is a valid outcome.** If no game in the competition ever completed, the
competition ends `abandoned` with an empty array — enforced by the `abandoned_has_no_champion`
constraint above. The 2020 preseason is exactly this case. Crowning somebody off an empty
table would be worse than crowning nobody.

**Ties produce joint champions.** Resolve with the existing tie-break chain; where players
remain level, record every one of them in `champion_user_ids` and render shared ranks the way
the app already does. Do not invent a preseason-specific decider.

### 4. Status transitions

`upcoming → active → provisional → complete`, or `→ abandoned` from any state. Enforce valid
transitions in a trigger rather than trusting callers. `provisional` means all matchdays are
played but a revision landed recently — reuse whatever provisional-window rule the existing
matchday finality logic already applies.

### 5. Opt-in, and the mode-lock consequence

Preseason participation is **opt-in per league, defaulting to off**. Creating the preseason
competition is the opt-in.

Two consequences must be surfaced in the UI, not discovered later:

1. **Opting in moves the prediction-mode lock a month earlier.** The existing rule is that a
   league's mode becomes immutable once the league has started, where started = the first
   game's deadline passing. With the preseason enabled, that first deadline is in August, not
   September. This is correct — the mode must be settled before any prediction is taken — but
   an admin expecting until September to decide will otherwise be locked in without warning.
   Say so on the opt-in screen explicitly.
2. **`EXACT_SCORE` on preseason games is close to a lottery.** Starters play little, so
   scorelines are near-noise. State plainly on the rules page how the league's mode behaves
   in the preseason, so it reads as a deliberate choice rather than a defect.

A league opting in **after** the preseason has started counts only matchdays whose deadlines
have not yet passed, and the UI says so on both the opt-in screen and the preseason standings.
Never retroactively score a matchday nobody could predict.

### 6. Draw availability is a matrix, not a single rule

Both ends of this axis are real. Do not let it collapse into "draws only happen in the
regular season":

| `season_type` | Draw in `OUTCOME_ONLY` | Why | Enforcement |
|---|---|---|---|
| 1 preseason | **allowed, ~6% of games** | no overtime at all | — |
| 2 regular | allowed, ~0.4% | overtime can still end level | — |
| 3 postseason | **rejected** | play continues until someone wins | `CHECK` constraint |

Whatever constraint currently rejects postseason draws must not be widened to cover the
preseason. In the preseason UI, give the draw option equal visual weight to the two win
options rather than tucking it away.

## MIGRATION — THE RISKIEST PART OF THIS WORK

Adding `competition_id` touches tables that already hold real predictions and real standings
for a working season. Use **expand → migrate → contract**, in separate deployable steps, and
never combine them.

**Step 1 — expand.** Create `competitions`. Add `matchdays.competition_id` as **nullable**.
Add nullable `competition_id` to standings and weekly-results tables. Ship. Nothing reads the
new columns yet.

**Step 2 — backfill.** For every existing `(league_id, season_year)`, create the `main`
competition and link all existing matchdays and results to it. Idempotent, re-runnable, and
it asserts its own success:

- every `matchdays` row has a non-null `competition_id`
- every league-season with matchdays has exactly one `main` competition
- `predictions` row count is **identical** before and after
- every prediction still resolves to exactly one game, and every game to one matchday

Fail the migration loudly if any assertion misses. Do not proceed on a warning.

**Step 3 — contract.** Set the columns `NOT NULL` and add foreign keys. Switch reads to the
competition-scoped path. Ship.

**Step 4 — preseason.** Only now create preseason competitions and matchdays for opted-in
leagues.

Every migration ships with a tested down-migration. Take a verified database snapshot first
and confirm it restores into a scratch database — an untested backup is not a backup.

**Ship it dark.** Put the preseason behind a feature flag so steps 1–3 can go out and be
observed against the live regular season before any player sees a warm-up competition.

## ROW LEVEL SECURITY

Extend the existing policy pattern; do not invent a new one.

| Table | `authenticated` SELECT | Writes |
|---|---|---|
| `competitions` | members of that league | **denied to all** — service role and `SECURITY DEFINER` admin functions only |
| preseason standings / weekly results | members of that league | **denied to all** — server-computed |
| `matchdays` (incl. preseason rows) | allowed | denied |

No route or function accepts points, ranks or a champion from a client. The champion is
computed server-side from stored final scores and nothing else.

Run the existing authorization matrix against everything this adds, with three seeded users —
A and B in league 1, C in league 2:

- As B, read A's preseason prediction for an unlocked game → **no rows**
- As B, read A's preseason prediction for a locked game → allowed
- As C, read league 1's preseason competition, standings or champion → **no rows**
- As B (member, not admin), opt a league into the preseason, set a champion, or invoke the
  award function → **rejected**
- With no session, any of the above → **rejected**

## TESTING

**Unit.** The scoring function against fixed preseason fixtures including all three 2025 tie
games, in both prediction modes, with expected totals. The champion selection against
hand-built standings: clear winner, two-way tie, five-way tie, nobody scoring at all.

**Integration.** The 2020 golden fixture end to end: 49 cancelled games ingested, nothing
scored, competition ends `abandoned`, `champion_user_ids` empty. A concurrency test firing the
award path twice in parallel and asserting a single award. A test asserting preseason points
are absent from main-competition standings, by direct query.

**End to end (Playwright).** Opt a league in → submit preseason predictions → deadline passes
→ other players' predictions become visible and not before → games finalise → preseason
standings populate → last matchday finalises → champion announced → main-competition standings
remain empty.

**Regression.** The existing regular-season and playoff suites must pass unchanged after the
migration. That is the acceptance criterion for steps 1–3 — the refactor is only safe if it is
invisible to the season already in progress.

Plus an `axe-core` pass on the new screens and TypeScript strict mode with no `any`.

## UX

**The preseason must read as a separate contest, not as early-season games.** Its own visibly
distinct standings screen, its own title ("Preseason 2026"), its own trophy. State on both the
preseason table and the main table that no points move between them. The failure mode to
design against is a player banking 40 points in August and expecting to lead in September.

**Announce the champion as a moment** when the last preseason matchday goes final, and keep the
record visible once the real season starts.

**Preseason week labels** come from the calendar. **Neutral-site games** are labelled by team
name. **Draws** get equal weight to the win options. **A cancelled preseason** shows an
explicit "abgesagt" state with no champion, never an empty table implying one.

Accessibility carries over: WCAG 2.2 AA, keyboard-operable prediction grid, never state by
colour alone, `polite` live regions for score updates announced on change rather than per poll.

## PHASES

Each independently shippable, each ending with the authorization matrix for what it added.

**Phase 1 — Competition scaffolding (no player-facing change).** `competitions` table with its
constraints and transition trigger, the expand and backfill migrations with assertions, the
standings refactor to competition scoping, the feature flag. The existing regular-season suite
must pass unchanged.

**Phase 2 — Preseason ingest, read-only.** Extend `sync-season-schedule` to season type 1.
Calendar-sourced labels. All 49 games visible as a schedule, none predictable yet. Verify
against the 2020 fixture.

**Phase 3 — The warm-up competition.** Per-league opt-in with the mode-lock warning, preseason
prediction entry reusing the existing flow and deadline ratchet, preseason standings and weekly
winners, the draw path given the 6% tie rate.

**Phase 4 — Champion and polish.** Atomic set-once award, joint titles, the `abandoned` path,
the audited re-award function, the champion announcement UI, admin recalculation.

## EDGE CASES

Each gets a named test. State the expected behaviour in the plan.

1. **Whole preseason cancelled** (2020 fixture: 49 × `STATUS_CANCELED`, `state: 'post'`,
   `completed: false`, scores `"0"`) → nothing scored, competition `abandoned`, champion array
   empty, no champion crowned
2. Preseason tie in `OUTCOME_ONLY` → draw scores correctly; both `winner` flags `false`
3. Preseason tie in `EXACT_SCORE` → tie branch applies, not the differential branch
4. Preseason game level at the end of period 4 → final immediately, not held open for overtime
5. ESPN preseason `week.number = 2` → displayed as "Preseason Week 1"
6. Hall of Fame Game → neutral-site labelling; `shortName` `"CAR VS ARI"` never parsed for
   teams
7. Single-game matchday with 30 of 50 players tied → shares of 1/30 as `NUMERIC`, summing to
   exactly 1
8. Champion award fires twice concurrently → exactly one award, second is a no-op
9. Preseason champion is a tie → all tied players recorded, shared ranks rendered
10. **Preseason points never appear in main-competition standings** → asserted by direct query
11. League opts in mid-preseason → only unexpired matchdays count, stated in the UI
12. Preseason enabled → mode lock fires at the first preseason deadline, not at Week 1; admin
    was warned
13. Preseason and regular-season week 1 both exist → separate matchdays, separate
    competitions, separate standings
14. ESPN corrects a preseason score after the champion is crowned → audited re-award, players
    notified
15. Backfill migration run twice → identical result, no duplicate competitions
16. Any job run against a seeded database → `predictions` unchanged, asserted
17. A preseason game is cancelled individually while others are played → excluded from
    finality, competition still completes
18. Preseason competition completes → main competition unaffected and still empty

## TIMING — 2026 SEASON

Concrete, because the window is narrow:

All kickoffs in UTC, taken from the live schedule:

| | First kickoff | Last kickoff |
|---|---|---|
| Hall of Fame Game | 2026-08-07 00:00 — **already played** | — |
| Preseason Week 1 | **2026-08-13 23:00** | 2026-08-16 00:00 |
| Preseason Week 2 | 2026-08-21 00:00 | 2026-08-24 00:00 |
| Preseason Week 3 | 2026-08-27 23:00 | 2026-08-29 22:00 |
| Regular season Week 1 | 2026-09-10 00:20 | 2026-09-15 00:15 |

- The **Hall of Fame Game has already been played** (CAR 33 – ARI 30, final). It cannot be
  predicted this year. Ingest it for schedule completeness and as the neutral-site and
  `VS`-delimiter test fixture, with `is_predictable = false`.
- **Preseason Week 1 is the first realistic matchday**, and its first kickoff is the effective
  deadline for shipping.
- There are **11 days between the last preseason game and the first regular-season game** — a
  clean window to award the warm-up title and reset for the real season.

So the 2026 warm-up competition is **3 matchdays and 48 games**. Reaching Preseason Week 1
means shipping phases 1–3 within roughly six days; if that slips, phase 2 alone still leaves
the schedule visible and correct, and the competition can start at Preseason Week 2 under the
mid-preseason opt-in rule. Plan the phases so slipping costs a matchday, not the season.

## NON-GOALS

No separate prediction mode per competition — every competition uses the league's mode. No
combined year-long leaderboard spanning preseason and regular season. No splitting the playoffs
into their own competition as part of this work. No team-based standings from preseason results
(the two Hall of Fame participants play four preseason games while the other thirty play three,
so such a table would not be comparable). No real money, no public sign-up, no betting odds.

## PLAN OUTPUT FORMAT

First, the results of your assumption check against the existing codebase — what exists, what
does not, and what that changes.

Then, for each phase: goal · concrete tasks · files and Edge Functions touched · migrations
with their expand/migrate/contract step · RLS policies added or changed with their exact rule ·
acceptance criteria · test cases · risks · open decisions.

Then, separately:

- A table mapping every item in the reuse list above to the existing module you will reuse
- The migration assertion list, with the exact queries that verify prediction integrity
- The champion-award concurrency argument: why exactly-once holds under a double-fired job
- How you will verify that no preseason point can reach the main standings

**Ask your clarifying questions now. Wait for approval before building.**
