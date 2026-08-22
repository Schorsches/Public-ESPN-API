# Team Season Records (W-L-D) — Implementation Prompt

> A standalone, paste-ready prompt for Lovable's plan mode. Everything below the line is the
> prompt.

---

## ROLE

You are a senior full-stack architect extending a working application. Produce a phased
implementation plan before writing any code. Ask every clarifying question that would
materially change the approach, then wait for approval. Do not begin building until the plan
is approved.

This is a small, additive increment on a working app. **Nothing about prediction entry,
deadlines, scoring or the existing views may change behaviour.** A plan that refactors the
ingest, the scoring service or the match components beyond what this feature needs is a failed
plan.

## WHAT EXISTS TODAY

"Gridiron Tippzone" — a private, invite-only NFL prediction game for a small group of friends.
Bragging rights only. German UI. Stack: Lovable + Supabase (Postgres with RLS, Edge Functions,
`pg_cron`), React/TypeScript. NFL schedule, live scores and results are already ingested from
the ESPN API, and prediction results are already evaluated from finished matches.

Three views show matches, and all three need this feature:

1. **Tippen** — the weekly prediction grid. One card per match: team abbreviations with logos,
   kickoff countdown, and the outcome buttons (Heim / Draw / Auswärts).
2. **Spieltag** — live scores and the live prediction standing.
3. **Übersicht** — the dashboard, whose "Nächste Deadline" card shows the next match.

## WHAT TO ADD

Show each team's **season record as Siege–Niederlagen–Unentschieden (W-L-D)** next to its
abbreviation in all three views, plus a small legend explaining the format, exactly as in the
mockup:

```
  NE          @        SEA
[ 3-2-0 ]           [ 4-1-0 ]

           ⓘ Saisonbilanz (W-L-D)
   Bilanz: Siege-Niederlagen-Unentschieden
```

Before week 1 every team is `0-0-0`. After each week's games finish, records update.

## THE CENTRAL FINDING — THIS COSTS ZERO ADDITIONAL API CALLS

**Do not build a new fetcher for this, and do not call the standings endpoint on a schedule.**

Team records are already embedded in the scoreboard payload the app fetches for the schedule
and for live scores. Every competitor carries a `records` array:

```json
"competitors": [
  { "homeAway": "home",
    "team": { "abbreviation": "HOU" },
    "records": [
      { "name": "overall", "abbreviation": "Game", "type": "total", "summary": "6-5" },
      { "name": "Home",  "type": "home", "summary": "4-2" },
      { "name": "Road",  "type": "road", "summary": "2-3" }
    ] }
]
```

Read `records[]` where `type === "total"`. The other two entries are home/road splits and are
not needed here.

So the entire feature is: **parse a field already arriving in a payload you already request,
persist it, and render it.** The API-call budget for this feature is zero.

## VERIFIED ESPN BEHAVIOUR — trust this over your own assumptions

I traced a single team across all 18 weeks of a completed season and checked the edge cases
directly. Do not re-derive these.

### 1. The record is POST-game for a finished match

Houston's week 1 game was a 9–14 loss, and the payload for that game reports `0-1` — not the
`0-0` they carried into it. By week 18 the same trace reads `12-5`, matching the final
standings. **The summary always includes the result of the game it is attached to.**

### 2. For a scheduled match, it is therefore the current record

An unplayed game has no result to include, so its `records` entry is the team's standing right
now. Verified: every match in the upcoming season's week 1 reports `0-0` for both sides.

This is why the feature works out naturally — an upcoming match shows today's record, and a
finished match shows the record as it stood after that match.

### 3. Ties make the string variable-length — this WILL break a naive parser

The summary is `W-L` when a team has no ties and `W-L-T` when it does. Dallas's 2025 trace:

```
0-1, 1-1, 1-2, 1-2-1, 2-2-1, 2-3-1, … 7-9-1
```

The string grows a third segment mid-season, the moment they tie. Houston, who never tied,
stayed two-segment all year: `0-1 … 12-5`.

**Parse defensively:** split on `-`, accept 2 or 3 numeric segments, default ties to `0`, and
reject anything else rather than guessing. The UI always renders three parts, so `12-5`
displays as `12-5-0`.

### 4. Preseason records accumulate and must be suppressed

Preseason is a separate ESPN record. Teams really do accumulate a preseason W-L: verified
`NYG=3-0`, `ATL=0-3`, and `SEA=0-0-1` from a preseason tie.

**Display `0-0-0` for every preseason match anyway.** A preseason W-L is not a season standing
and would mislead. Because the payload carries a real value, this is an **active override, not
a pass-through** — you must explicitly ignore what ESPN says when `season_type = 1`.

Put this behind a single config flag (default: suppress) so it can be flipped later without a
code change.

## DATA MODEL

Persist the record **per game, per side**, not only as a current-per-team value. That is what
makes an old matchday historically accurate for free, and it is the same payload either way.

Add to the existing games table:

```sql
alter table nfl_games
  add column if not exists home_wins    smallint,
  add column if not exists home_losses  smallint,
  add column if not exists home_ties    smallint,
  add column if not exists away_wins    smallint,
  add column if not exists away_losses  smallint,
  add column if not exists away_ties    smallint,
  add column if not exists records_frozen_at timestamptz;
```

All nullable — a match whose record could not be parsed shows no badge rather than a wrong one.

Store parsed integers, not the raw string. Parsing once at ingest beats re-parsing on every
render, and it lets you query and sort. Keep the raw `summary` only if your existing ingest
already retains the raw payload, in which case it is already there.

Adding a separate current-record-per-team table is **not** required. Every view renders a
match, and every match carries its own snapshot. Do not build a second source of truth that can
disagree with the first.

## SNAPSHOT AND FREEZE RULES

Write these into the existing ingest, in the same pass that already updates scores and status.
No new job, no new cron entry.

**While the match is `scheduled` or `in_progress`:** refresh the snapshot on every sync. The
value tracks the teams' current standing as other games finish, which is exactly what an
upcoming match should display. Leave `records_frozen_at` null.

**When the match becomes `final`:** capture the record from that same payload — which now
includes this game's result — and set `records_frozen_at`. From then on the snapshot is
**frozen** and later syncs must not overwrite it. This is what makes reviewing week 3 in
December show week 3's records.

**On a retroactive result correction:** if your existing revision handling changes a final
score, the frozen record may now be wrong. Allow a re-snapshot in that path only, and write it
to the existing revision/audit log like any other corrected field. Never re-snapshot silently
outside that path.

**If `records` is missing or unparseable:** leave the columns null, count it in the existing
job telemetry, and continue. **Never substitute `0-0-0` as a fallback** — outside the preseason
that is a factual claim, and a missing badge is better than a false one.

## UI

**Placement.** A compact badge directly beneath (Tippen) or beside (Spieltag, Übersicht) the
team abbreviation, in the same visual language as the mockup: subdued pill, tabular numerals,
never competing with the score or the countdown.

**Legend.** The `ⓘ Saisonbilanz (W-L-D)` line with `Bilanz: Siege-Niederlagen-Unentschieden`
appears **once per view**, not per card.

**Format.** Always three parts, `W-L-D`, ties rendered as `0` when absent.

**Do not break the existing layout.** These cards are already dense on a narrow phone. The
badge must not cause the team row to wrap or the outcome buttons to reflow at 360 px width, and
must survive 200% zoom. Use tabular figures so a two-digit win total does not shift the layout
mid-season. Verify at 360 px before and after.

**States.**
- record known → badge
- record null (unparseable or not yet ingested) → **no badge**, no placeholder, no dash that
  could be mistaken for a zero
- preseason → `0-0-0`

**Accessibility.** `3-2-0` is meaningless to a screen reader. Give each badge an
`aria-label` in the style of `Bilanz: 3 Siege, 2 Niederlagen, 0 Unentschieden`, and mark the
visual string `aria-hidden`. The badge is informational only — never the sole carrier of any
state, and never distinguished by colour alone.

**No layout shift.** The record arrives with the match data in the same query. Do not fetch it
separately or the badges will pop in after render.

## NON-BREAKING REQUIREMENTS

- Columns are **additive and nullable**. Ship the migration first, with nothing reading the
  columns, and confirm the existing views are untouched.
- The ingest change is **write-only additional fields**. It must not alter how status, scores,
  kickoff or deadlines are written.
- **No change to prediction entry, deadline enforcement, scoring or standings.** Assert this
  with the existing test suite passing unchanged — that is the acceptance criterion for phase 1.
- No new RLS policies needed: the columns live on a table members can already read. Confirm
  that the existing games policy covers them and that nothing new is writable by a client.

## PHASES

**Phase 1 — Persistence.** Migration plus the ingest parse, snapshot and freeze logic. No UI
change. Backfill from whatever raw payloads you already retain; where none exist, leave null
and let them fill in on the next sync. Verify against a completed week by hand.

**Phase 2 — Rendering.** The badge component, the legend, and wiring into all three views.
Layout checked at 360 px, 200% zoom and with a screen reader.

**Phase 3 — Correction path.** Re-snapshot on retroactive result corrections, wired into the
existing revision handling.

## EDGE CASES

Each gets a named test.

1. Summary `12-5` (no ties) → parsed as 12-5-0, rendered `12-5-0`
2. Summary `7-9-1` → parsed as 7-9-1
3. A team that ties mid-season: `1-2` on one matchday and `1-2-1` on the next → both parse
4. `records` array absent, or no entry with `type === "total"` → columns null, no badge, job
   telemetry incremented, sync continues
5. Malformed summary such as an empty string or `"—"` → treated as unparseable, not as zero
6. Scheduled match → snapshot refreshes on each sync
7. Match goes final → snapshot captured **including** that game's result, then frozen
8. A later sync touches a frozen match → record unchanged
9. Retroactive score correction → re-snapshot permitted, audit entry written
10. Preseason match → `0-0-0` displayed even though ESPN reports e.g. `3-0`
11. Season rolls over → records restart from `0-0-0`; no leakage from the previous season
12. Week 1 before any game → `0-0-0` for all 32
13. Badge at 360 px with two-digit values (`12-5-0`) → no wrap, no reflow of outcome buttons
14. Existing prediction, deadline and scoring tests → **pass unchanged**

## NON-GOALS

No stadium details, map or weather — that is a separate piece of work, not this one. No
division or conference standings table. No home/road split records, even though the payload
carries them. No playoff seeding. No streaks, form guides or power rankings. No new scheduled
job, and no use of the standings endpoint.

## PLAN OUTPUT FORMAT

First, what you found in the existing code: where the scoreboard payload is parsed today, where
match data reaches each of the three views, and whether raw payloads are retained for backfill.

Then per phase: goal, tasks, files and Edge Functions touched, the migration, acceptance
criteria, test cases, risks.

Then separately:

- The parser's decision table for the summary string, including every rejection case
- Where exactly the freeze is enforced, and how a later sync is prevented from overwriting
- Proof that no existing behaviour changed — which tests cover that claim

**Ask your clarifying questions now. Wait for approval before building.**
