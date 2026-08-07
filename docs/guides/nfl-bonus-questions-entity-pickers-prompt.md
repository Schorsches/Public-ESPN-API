# Bonus Questions — API-Backed Team & Player Pickers

> A standalone, paste-ready prompt for Lovable's plan mode. It assumes the bonus-question
> feature already works and changes only how teams and players are selected, imported and
> stored. Everything below the line is the prompt.

---

## ROLE

You are a senior full-stack architect extending a working feature. Produce a phased
implementation plan before writing any code. Ask every clarifying question that would
materially change the approach, then wait for approval. Do not begin building until the plan
is approved.

This is an increment on working code. **Grading behaviour does not change.** Existing bonus
questions must continue to work identically. A plan that rewrites the grading engine, or that
requires re-authoring last season's questions, is a failed plan.

## WHAT EXISTS TODAY

A private, invite-only NFL prediction game for 10–50 friends. Bragging rights only. German UI.
Stack: Lovable + Supabase (Postgres with RLS, Edge Functions in Deno, `pg_cron` + `pg_net`),
React/TypeScript frontend. Regular season, playoffs and game predictions are implemented and
ingest NFL data from ESPN.

Bonus questions are authored by the admin, answered by players, and **graded manually by the
admin**. They are imported from a semicolon-delimited CSV:

```
title;description;type;options;points;bonus_matchday;deadline_at;visible;scoring
```

Six question types exist: `single_choice`, `player_pick`, `multi_select`, `ranking`,
`numeric`, `free_text`. Options are pipe-delimited display strings, and the `scoring` column
carries a small DSL:

```
Wer gewinnt den Super Bowl?;Ein Team auswählen;single_choice;KC|SF|BUF|PHI;10;1;…
Wer wird MVP der Regular Season?;Ein Spieler auswählen;player_pick;Patrick Mahomes|Josh Allen|…;8;1;…
Welche Teams erreichen die Playoffs?;…;multi_select;KC|SF|BUF|PHI|DAL|BAL;12;…;per_correct_points=2,wrong_penalty=1
Endstand NFC Ost (Platz 1-4);…;ranking;PHI|DAL|WAS|NYG;12;…;per_position_points=2,all_correct_bonus=4
Gesamtpunkte im Super Bowl;…;numeric;;5;…;tolerance=3,tolerance_points=2
Welches Team holt die meisten Siege?;…;free_text;;6;…;alternatives=Kansas City Chiefs|Chiefs
```

Before planning, read the current schema and confirm what exists: the bonus question and
answer tables, the grading flow, the CSV import path, and whether an `nfl_teams` reference
table is already populated from ESPN (the game-prediction feature should have created one).
Report anything missing — it changes the plan.

## THE PROBLEM

Options are free-typed display strings, which fails in three ways.

**They do not match ESPN.** The template writes Washington as `WAS`; ESPN uses `WSH`. Verified
against the live API. A question authored with `WAS` cannot be matched to ESPN team data at all.

**They are fragile to grade.** Last season's recorded answers used a surname-first format —
`Jackson, Lamar`, `Jeanty, Ashton` — while options were authored as `Patrick Mahomes`. Team
answers appear variously as `Buffalo Bills`, `Bills` and `Eagles`. The `free_text` type papers
over this with an `alternatives=Kansas City Chiefs|Chiefs` list that has to be maintained by
hand and silently marks a correct answer wrong when a spelling is missing.

**They make players work too hard.** A player answering "who wins the AFC East?" should get the
four teams in that division, with logos, not a typed list the admin had to look up.

## WHAT THIS CHANGES

1. Teams and players become **stored entities with stable ESPN IDs**, cached locally from the
   API.
2. Question options **reference those entities** instead of holding display strings.
3. The CSV keeps its current shape, and **resolves** abbreviations and names to entity IDs at
   import, with a review step for anything ambiguous.
4. Players pick from **entity pickers**: team tiles with logos, and a searchable player picker
   with optional team, position and rookie filters.
5. The admin enters the correct answer **through the same picker**, so grading compares IDs.

**Grading stays manual.** No question is auto-graded from the API. The admin still decides
every answer.

That said, one correctness win comes for free and should be called out in your plan: when both
the players' picks and the admin's correct answer are entity IDs chosen from the same list,
grading becomes an **ID comparison instead of a string comparison**. `alternatives=` lists stop
being needed for team and player questions, and a pick can no longer be marked wrong over a
spelling or a name-order difference. Ranking and multi-select grading compare ID sets and
positions rather than text.

## VERIFIED ESPN API REFERENCE

All verified against the live API. No key required. **All calls happen server-side inside Edge
Functions, never from the browser.** ESPN publishes no rate limits — cache aggressively and be
conservative.

### Teams — 32 rows, one request

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams
```

Each entry under `sports[0].leagues[0].teams[].team`:

```json
{ "id": "22", "uid": "s:20~l:28~t:22", "slug": "arizona-cardinals",
  "abbreviation": "ARI", "displayName": "Arizona Cardinals",
  "shortDisplayName": "Cardinals", "name": "Cardinals", "location": "Arizona",
  "color": "a40227", "alternateColor": "ffffff", "isActive": true,
  "logos": [{ "href": "https://a.espncdn.com/i/teamlogos/nfl/500/ari.png" }] }
```

`id` is the stable key. Logos follow the predictable pattern
`https://a.espncdn.com/i/teamlogos/nfl/500/{abbreviation-lowercase}.png`, but prefer the
`logos[].href` value returned by the API over constructing it.

### Divisions — the exact four teams per division, one request

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/groups
```

Returns `groups[]` as two conferences, each with four divisions of exactly four teams:

```
AFC East   BUF MIA NE  NYJ      NFC East   DAL NYG PHI WSH
AFC North  CIN CLE PIT BAL      NFC North  CHI DET GB  MIN
AFC South  TEN IND JAX HOU      NFC South  ATL NO  TB  CAR
AFC West   DEN KC  LV  LAC      NFC West   LAR ARI SF  SEA
```

Note that division nodes in this response carry **no `id`** — key them on conference
abbreviation plus division name. If you need stable division IDs, the standings endpoint below
provides them (`AFC East = 4`, `AFC North = 12`, `AFC South = 13`, `AFC West = 6`,
`NFC East = 1`, `NFC North = 10`, `NFC South = 11`, `NFC West = 3`).

This endpoint is what makes a division-scoped question trivial: the four options are data, not
something the admin types.

### Standings — for division membership and stable division IDs

```
GET https://site.api.espn.com/apis/v2/sports/football/nfl/standings?level=3&season={year}
```

Note the path is `/apis/v2/`, **not** `/apis/site/v2/` — the latter returns only a stub for
standings.

**The entries are not returned in standings order.** Verified against the completed 2025
season: 2 of the 8 divisions came back out of order. AFC West returned `LAC, KC, LV, DEN` when
Denver (14-3) actually won it, and NFC West returned `LAR, SF, ARI, SEA` when Seattle (14-3)
won. Reading `entries[0]` as the division leader would be wrong for a quarter of the divisions.

**Always sort by the `playoffSeed` stat**, never by array position. Seeds 1–4 in each conference
are the four division winners, and seed 1 is the conference top seed. Do not compute
tiebreakers yourself — 2025's NFC South was won at 8-9 with two other teams also at 8-9, and
`playoffSeed` resolves that authoritatively.

You are not auto-grading, so this matters here only for correctly ordering a division's teams
when presenting them. But encode the sort rule anyway — it is the kind of thing that silently
becomes wrong later.

### Players — 32 roster requests, cached locally

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams/{teamId}/roster
```

Returns `athletes[]` grouped as `offense`, `defense`, `specialTeam`, `injuredReserveOrOut`,
`suspended`, `practiceSquad`. Each athlete:

```json
{ "id": "3139477", "fullName": "Patrick Mahomes", "displayName": "Patrick Mahomes",
  "shortName": "P. Mahomes", "firstName": "Patrick", "lastName": "Mahomes",
  "jersey": "15", "age": 30,
  "position": { "abbreviation": "QB", "name": "Quarterback",
                "parent": { "abbreviation": "OFF" } },
  "experience": { "years": 0 },
  "status": { "name": "Active", "type": "active" },
  "college": { … }, "injuries": [ … ],
  "headshot": { "href": "https://a.espncdn.com/i/headshots/nfl/players/full/3139477.png" } }
```

**`experience.years === 0` identifies a rookie.** That is how a Rookie-of-the-Year question gets
a correct candidate list without the admin maintaining one. Verified: the Chiefs roster had 30
players at `experience.years === 0`.

Measured cost of a full refresh: **32 requests, roughly 16 seconds, about 14 MB of raw JSON,
2,968 players.** Storing only the fields you need is about **0.4 MB** — trivial. Run it once
daily.

Also verified: a name is **not** a unique key. Those 2,968 players contain 9 duplicate full
names, two of which duplicate position as well. Key on `espn_athlete_id`, always.

### Do not use the search endpoint for the player picker

```
GET https://site.web.api.espn.com/apis/search/v2?query=mahomes&sport=football
```

It works, but it is the wrong tool. Verified problems: a query for `mahomes` with
`sport=football` returned a **baseball** player (Pat Mahomes, Pittsburgh Pirates) alongside the
NFL one, and mixed players with news articles and video replays. Worse, the ESPN athlete ID is
not in the `id` field — that is an unrelated GUID — it is buried in `uid` as
`s:20~l:28~a:3139477` and must be parsed out.

**Search the local cache in Postgres instead.** It is faster, has no cross-sport pollution,
needs no ESPN round-trip per keystroke, and works when ESPN is down. The cached roster ID and
the search endpoint's `uid` athlete ID were verified to match (`3139477`), so nothing is lost.

## DATA MODEL

### Reuse `nfl_teams`

The game-prediction feature should already maintain this. Add `conference` and `division` from
the `groups` endpoint if absent. Do not create a second team table.

### New: `nfl_players`

```sql
create table nfl_players (
  id                uuid primary key default gen_random_uuid(),
  espn_athlete_id   text not null unique,          -- the stable anchor
  full_name         text not null,
  display_name      text not null,
  first_name        text,
  last_name         text,
  jersey            text,
  position_abbr     text,                           -- QB, WR, CB
  position_name     text,
  position_group    text,                           -- offense | defense | specialTeam
  team_id           uuid references nfl_teams(id),  -- nullable: free agents
  experience_years  int,
  is_rookie         boolean generated always as (experience_years = 0) stored,
  status            text,                           -- Active, Injured Reserve, …
  roster_group      text,                           -- offense … practiceSquad
  headshot_url      text,
  is_active         boolean not null default true,  -- false when off all rosters
  last_synced_at    timestamptz not null default now()
);
```

Index `(team_id)`, `(position_abbr)`, `(is_rookie) where is_rookie`, and the search index below.

### New: `bonus_question_options`

Replaces pipe-delimited strings for entity-backed questions, without removing the old column.

```sql
create table bonus_question_options (
  id            uuid primary key default gen_random_uuid(),
  question_id   uuid not null references bonus_questions(id) on delete cascade,
  entity_type   text not null check (entity_type in ('team','player','text')),
  team_id       uuid references nfl_teams(id)   on delete restrict,
  player_id     uuid references nfl_players(id) on delete restrict,
  text_value    text,
  label_snapshot text not null,   -- what it was called when authored
  sort_order    int  not null,
  unique (question_id, sort_order),
  constraint exactly_one_target check (
    (entity_type = 'team'   and team_id is not null and player_id is null and text_value is null) or
    (entity_type = 'player' and player_id is not null and team_id is null and text_value is null) or
    (entity_type = 'text'   and text_value is not null and team_id is null and player_id is null)
  )
);
```

Add an `answer_kind` discriminator to `bonus_questions` (`text | team | player | numeric`) so
the UI and the grader know which path a question uses. Existing questions keep
`answer_kind = 'text'` and behave exactly as before.

### Answers store the ID **and** a label snapshot

`bonus_answers` gains nullable `team_id` / `player_id` alongside the existing text column, plus
**`picked_label`** and **`picked_team_abbr`** captured at pick time.

This is not redundancy, it is the fix for the one way this feature could lose a player's answer.

## THE INVARIANT THAT CONSTRAINS THIS WORK

Consistent with the rest of the app: **a player's answer must never be lost, altered or
silently reinterpreted.** Rosters churn constantly — players are traded, cut, placed on IR,
retire mid-season. A pick made in August must still render and still grade in February.

Therefore:

- **The player sync is upsert-only. It has no DELETE path.** A player who leaves every roster
  gets `is_active = false` and `team_id = null`. The row stays forever.
- Foreign keys from options and answers use **`ON DELETE RESTRICT`**, so the database refuses a
  deletion even if code attempts one.
- **`picked_label` and `picked_team_abbr` are written at pick time and never updated.** If a
  player picks "Patrick Mahomes (KC)" and he is traded, the answer still displays what was
  actually picked, and the current team is shown separately if useful.
- A pick referencing an inactive player renders with a quiet badge — "nicht mehr im Kader" —
  rather than disappearing or erroring.
- No sync job may write to `bonus_answers`. Ever. Prove it with a test that runs the sync
  against a seeded database and asserts the answers table is unchanged.

## CSV IMPORT AND RESOLUTION

The CSV keeps its exact current columns. Options continue to be pipe-delimited display strings.
What changes is that the import **resolves** them.

### Resolution order

**Teams** — case-insensitive, first match wins: `abbreviation` → alias table → `shortDisplayName`
→ `displayName` → `location` → `slug`.

**Players** — normalise both sides (lowercase, strip accents, strip `.` `'` `-`, collapse
whitespace), then: exact normalised full name → **`Lastname, Firstname` reordered** → last name
plus first initial → trigram similarity above a threshold. More than one candidate above
threshold is **ambiguous, not a guess**.

### Real cases the resolver must handle

Drawn from the actual template and last season's recorded answers:

| Input | Issue | Expected |
|---|---|---|
| `WAS` | ESPN uses `WSH` | resolves via alias |
| `LA` | ambiguous — Rams or Chargers | **flagged for review, never guessed** |
| `JAC` | ESPN uses `JAX` | resolves via alias |
| `OAK`, `SD`, `STL`, `WFT`, `Redskins` | relocations and renames | resolve to `LV`, `LAC`, `LAR`, `WSH` |
| `Jackson, Lamar` | surname-first | resolves to Lamar Jackson |
| `Frank Gore Jr.` | suffix and period | resolves |
| `Ja'Mori Maclin` | apostrophe | resolves |
| `Sedrick Van Pran-Granger` | hyphen and space in surname | resolves |
| `Chiefs` | short name | resolves to Kansas City Chiefs |
| `Justin Jefferson` | **two active players** — MIN WR and CLE LB | **ambiguous → review** |
| `DeVonta Smith` | **two active players** — PHI WR and CAR CB | **ambiguous → review** |

Those last two rows are the ones most likely to be missed, and they are not hypothetical.
Measured across all 32 rosters: **2,968 rostered players contain 9 colliding full names.** The
dangerous shape is a famous player sharing a name with an obscure one — a question offering
"Justin Jefferson" plainly means the Vikings receiver to a human, while a resolver taking the
first match may silently attach every pick to a Browns linebacker.

Two of the nine collide on **name and position simultaneously** — `Christian Jones` (CIN OT and
ARI OT) and `Jaylon Jones` (CHI CB and IND CB). **Position filtering therefore cannot
disambiguate**; only the team or the ID can. This is why the picker must always display team,
position and jersey alongside the name, and why the resolver must refuse to guess.

Never resolve a duplicate name silently.

### Import is a two-step commit

1. **Dry run.** Parse the CSV, resolve everything, and produce a report: resolved (with what it
   resolved to), ambiguous (with candidates), unresolved. **Write nothing.**
2. **Review and commit.** The admin fixes ambiguities by picking from the candidates, then
   commits. Only then are questions and options written, in a single transaction.

An import that cannot fully resolve **does not partially commit.** A half-imported question set
is worse than a failed import.

The alias table is a real table, admin-editable, seeded with the known cases above. Fixing a
mismatch once must fix it for every future import — not require a CSV edit each time.

Keep the old text path working: a question whose options do not resolve to entities can still be
imported as `answer_kind = 'text'` exactly as today. **Nothing about the CSV becomes mandatory.**

## PLAYER SEARCH

Search the local cache, not ESPN.

Enable `pg_trgm` and `unaccent`. Add a generated, normalised search column that indexes both
name orders so `mahomes`, `patrick mahomes` and `mahomes, patrick` all hit:

```sql
alter table nfl_players add column search_text text
  generated always as (
    lower(unaccent(full_name)) || ' ' ||
    lower(unaccent(coalesce(last_name,''))) || ', ' || lower(unaccent(coalesce(first_name,'')))
  ) stored;

create index nfl_players_search_trgm on nfl_players using gin (search_text gin_trgm_ops);
```

Expose search as a Postgres function taking the query plus optional `team_id`, `position_abbr`,
`position_group` and `rookies_only` filters. Rank exact prefix above word prefix above trigram
similarity, and order active players ahead of inactive ones. Cap results (20 is plenty) and
debounce the client at ~200 ms.

Strip punctuation from the user's input the same way the column does, or `O'Cyrus` will not match.

## SYNC JOB

One new Edge Function, following the existing ESPN client patterns — shared secret header,
timeouts, retry with backoff, circuit breaker, a `job_runs` row, per-record try/catch so one bad
athlete never fails the batch.

| Job | Cadence | Cost |
|---|---|---|
| `sync-nfl-teams-and-divisions` | Weekly, plus manual | 2 requests |
| `sync-nfl-players` | Daily | 32 requests, ~16 s |

Both take the existing per-job advisory lock so overlapping ticks cannot both run.

Sequence the 32 roster calls with a small delay rather than firing them in parallel — this is
an undocumented API being used as a guest.

**Reconciliation, not deletion:** a player absent from all 32 rosters in a **fully successful**
run gets `is_active = false`. A partially failed run must never deactivate anybody, or one
timeout would mark a third of the league inactive.

## UI

**Team picker.** Logo tiles with name and abbreviation. Grouped by conference and division when
the question spans the league; just the four tiles when the question is division-scoped. A
radio group for single choice, checkboxes for multi-select — real form controls, so keyboard
and screen-reader behaviour comes from tested primitives rather than styled `div`s.

**Player picker.** A combobox over the local search, with an optional team filter to narrow
first — which is the fastest path when a player already knows the team. Position and rookie
filters are pinned by the question, not chosen by the player, so a Rookie-of-the-Year question
cannot return a ten-year veteran. Each result shows headshot, name, jersey, position and team.
Use the existing shadcn/Radix combobox for correct ARIA and focus management; do not hand-roll
it.

**Ranking questions.** Drag-and-drop needs a keyboard equivalent — up/down buttons or
arrow-key reordering with announced position changes. Drag-only ordering is inaccessible.

**Admin correct-answer entry uses the identical picker.** This is what turns grading into an ID
comparison, and it also means the admin cannot enter an answer that was never an option.

**Converting `free_text` questions.** Where a question is answerable by a team or player, offer
the admin a one-click conversion that proposes the entity mapping and asks for confirmation.
Keep `free_text` for genuinely open questions — Gatorade colour, and similar. Converting is
**opt-in per question and never retroactive to answers already given**: if answers exist, keep
the question on its original path and say so, rather than reinterpreting stored text as an
entity.

**Accessibility.** WCAG 2.2 AA, consistent with the rest of the app. Full keyboard operability
end to end, visible focus, touch targets ≥ 44 px, never state by colour alone, errors linked via
`aria-describedby`, result counts announced politely. Include an `axe-core` pass over the picker
screens.

**German copy throughout**, i18n-ready.

## ROW LEVEL SECURITY

Extend the existing pattern; do not invent a new one.

| Table | `authenticated` SELECT | Writes |
|---|---|---|
| `nfl_teams`, `nfl_players` | allowed — public reference data | **denied to all**; service role only, inside Edge Functions |
| team alias table | allowed | league admins only, via `SECURITY DEFINER` |
| `bonus_question_options` | league members | admins only, via `SECURITY DEFINER` |
| `bonus_answers` | **own rows always; others' only after that question's deadline** | INSERT/UPDATE own row only, before the deadline |

The deadline-based visibility rule must live **inside the policy**, so another player's answer
cannot be selected before the deadline — not fetched and hidden by the client.

Run the existing authorization matrix over everything added, with three seeded users — A and B
in league 1, C in league 2:

- As B, read A's bonus answer before the deadline → **no rows**; after → allowed
- As B, submit an answer carrying A's user ID → **rejected**
- As B, submit an answer for a question whose deadline has passed → **rejected**
- As B, submit an option ID that does not belong to that question → **rejected**
- As B (member, not admin), edit options, aliases, or run an import → **rejected**
- As C, read league 1's questions, options or answers → **no rows**
- With no session, any of the above → **rejected**

## MIGRATION

Expand → migrate → contract, in separate deployable steps, behind a feature flag.

**Step 1 — expand.** Create `nfl_players`, the alias table, `bonus_question_options`. Add
nullable `answer_kind` to `bonus_questions` and nullable entity columns plus `picked_label` to
`bonus_answers`. Ship. Nothing reads them yet.

**Step 2 — backfill.** Populate teams, divisions and players from the API. Set
`answer_kind = 'text'` on every existing question so current behaviour is explicit rather than
implied. Assert: `bonus_questions` and `bonus_answers` row counts unchanged, every existing
question still renders, every existing answer still grades to the same points as before.

**Step 3 — contract.** `answer_kind` becomes `NOT NULL`. Switch the UI to the entity path for
questions that have `bonus_question_options`. Ship.

**Step 4 — adoption.** Enable the new pickers for new questions. Offer per-question conversion
of `free_text`, opt-in, never for questions that already have answers.

Every migration ships with a tested down-migration. Snapshot the database first and confirm it
restores into a scratch database.

**The acceptance criterion for steps 1–3 is that last season's questions and answers produce
byte-identical grading output.** Capture that as a fixture before you start.

## TESTING

**Unit — the resolver.** Every row of the resolution table above, including the three that must
fail: `LA`, `Justin Jefferson` and `Christian Jones` must all return "ambiguous", not a guess —
the last one proving that matching on position too is still not enough. Plus normalisation of
`O'Cyrus`, `Frank Gore Jr.`, `Van Pran-Granger`.

**Unit — search.** `mahomes`, `Mahomes, Patrick`, `patrick mahomes`, `mahome` (prefix),
`ocyrus` (punctuation stripped) all return the expected player. Rookie filter excludes veterans.
Team filter excludes other rosters.

**Integration — roster churn.** Player picked → next sync marks them inactive → the answer still
renders with its snapshot label and still grades. Player traded → answer shows the pick as made.

**Integration — import.** A CSV with one ambiguous row commits **nothing**. The same CSV
imported twice produces one question set, not two. A CSV with an unknown team fails cleanly with
a usable message.

**Integration — sync safety.** A partially failed roster run deactivates nobody. Every job run
leaves `bonus_answers` unchanged, asserted.

**Regression.** The full existing bonus-question suite passes unchanged, and the graded output of
last season's fixture is identical before and after.

**End to end (Playwright).** Author a division question → the four correct teams appear as
options → a player answers by keyboard only → the deadline passes → other answers become visible
and not before → the admin grades using the same picker → points match expectations.

Plus `axe-core` over the picker screens, TypeScript strict, no `any`.

## PHASES

**Phase 1 — Reference data.** `nfl_players`, division data on `nfl_teams`, both sync jobs, the
Postgres search function with its indexes. No user-facing change. Verify player counts and
rookie counts against the API by hand.

**Phase 2 — Pickers.** Team picker and player picker components, wired for new questions behind
the feature flag. `answer_kind`, `bonus_question_options`, entity-aware answer storage with label
snapshots. Existing questions untouched.

**Phase 3 — Import resolution.** The alias table, the resolver, the dry-run report and the
two-step commit. Old CSVs still import.

**Phase 4 — Grading and conversion.** Admin correct-answer entry through the pickers, ID-based
comparison for single/multi/ranking, and opt-in `free_text` conversion for questions without
answers.

## EDGE CASES

Each gets a named test.

1. `WAS` in a CSV → resolves to `WSH`
2. `LA` in a CSV → **ambiguous, import blocks**
3. `Justin Jefferson` → **ambiguous (MIN WR / CLE LB), import blocks**
4. `Christian Jones` → **ambiguous even with position known (CIN OT / ARI OT), import blocks**
5. `Jackson, Lamar` → resolves to Lamar Jackson
6. `O'Cyrus Torrence`, `Frank Gore Jr.` → resolve; searchable without punctuation
7. A CSV with one unresolvable row → **nothing is committed**
8. The same CSV imported twice → one question set
9. Picked player is cut before the deadline → answer intact, renders with snapshot label and an
   inactive badge
10. Picked player is traded → answer shows the pick as made; current team available separately
11. Partially failed roster sync → nobody deactivated
12. Any sync run → `bonus_answers` unchanged, asserted
13. Division question → exactly the four current teams of that division as options
14. Division standings read for ordering → sorted by `playoffSeed`, **not** array order (AFC West
    and NFC West are the cases that expose this)
15. Rookie-scoped question → only `experience.years = 0` players selectable
16. Answer submitted with an option ID belonging to a different question → **rejected**
17. Answer submitted after the deadline → **rejected**
18. Existing `free_text` question with answers → conversion refused, question keeps working
19. Existing questions after the full migration → identical grading output
20. A team relocates or renames mid-project → alias resolves old and new; existing picks keep
    their snapshot labels
21. ESPN roster endpoint returns an empty `athletes` array for one team → that team skipped,
    logged, no deactivations

## NON-GOALS

**No automatic grading of bonus questions.** The admin grades every question, as today. Entity
IDs merely make that comparison exact.

No live player statistics, no fantasy scoring, no player news or injury display beyond what the
picker needs to disambiguate. No historical import of previous seasons' questions. No changes to
the scoring DSL or point values. No changes to game-prediction behaviour. No real money, no
public sign-up.

## PLAN OUTPUT FORMAT

First, the results of your inspection of the existing bonus-question schema, grading flow and
import path — what exists, what does not, and what that changes.

Then, for each phase: goal · concrete tasks · files and Edge Functions touched · migrations with
their expand/migrate/contract step · RLS policies with their exact rule · acceptance criteria ·
test cases · risks · open decisions.

Then, separately:

- The full resolution order for teams and players, as a decision table, before you implement it
- The exact ambiguity rules — when the importer refuses to guess
- How you will prove last season's questions grade identically after the migration
- How you will prove no sync job can modify a stored answer

**Ask your clarifying questions now. Wait for approval before building.**
