# NFL Reference Data — Teams, Players & Coaches Snapshot

A pre-fetched snapshot of NFL teams, players and head coaches from the ESPN public API, so an
application can seed its reference tables **without calling ESPN at all**.

Generated once from 66 API requests (1 teams + 1 groups + 32 rosters + 32 coaches). Importing
these files costs **zero** requests.

## Snapshot

| | |
|---|---|
| Source | ESPN public API (unofficial, no key) |
| Snapshot taken | 2026-08-09 |
| Season | 2026 preseason |
| Teams | 32 |
| Players | 2,968 |
| Head coaches | 32 (one per team) |
| Rookies (`experience.years = 0`) | 660 |
| Colliding full names | 9 names, 18 rows |

## Files

| File | Size | Use |
|---|---|---|
| `nfl_reference_seed.sql` | 518 KB | **Supabase / Postgres.** Creates tables, functions and indexes, then upserts everything. Idempotent. |
| `nfl_teams.csv` | 4.7 KB | Table-editor or `\copy` import |
| `nfl_players.csv` | 448 KB | Table-editor or `\copy` import |
| `nfl_coaches.csv` | 3.3 KB | Head coach per team, with team name |
| `nfl_reference.slim.json` | 144 KB | **Client-side bundle.** Column-array format, headshot URLs derived. |
| `nfl_reference.json` | 1.3 MB | Full object-per-row JSON for server-side processing |
| `nfl_stadiums.slim.json` | 8 KB | Stadium roof, surface, coordinates and venue, keyed by ESPN abbreviation |
| `manifest.json` | — | Counts and provenance for verification |

## Quickest path: SQL

Paste `nfl_reference_seed.sql` into the Supabase SQL editor and run it. It:

- enables `pg_trgm` and `unaccent`
- creates `nfl_teams`, `nfl_players` and `nfl_coaches` with `create table if not exists`
- creates the immutable normalisation functions and the trigram search index
- upserts on the ESPN ID, so re-running updates rather than duplicating
- ends with a verification query

Expected result:

```
 teams | players | coaches | coach_orphans | ambiguous | rookies | orphans
-------+---------+---------+---------------+-----------+---------+---------
    32 |    2968 |      32 |             0 |        18 |     660 |       0
```

**Verified**: executed on PostgreSQL 16, runs in ~0.2 s, and produces identical counts on a
second run.

If you already have an `nfl_teams` table with a different shape, load the CSVs into staging
tables and map the columns yourself rather than editing the seed.

## CSV import

```sql
\copy nfl_teams   from 'nfl_teams.csv'   with (format csv, header true)
\copy nfl_players from 'nfl_players.csv' with (format csv, header true)
\copy nfl_coaches from 'nfl_coaches.csv' with (format csv, header true)
```

Import teams first — players and coaches both reference them by `team_espn_id`. Verified: 0
unmatched team references.

## Columns

**`nfl_teams`** — `espn_team_id` (stable key), `abbreviation`, `display_name`,
`short_display_name`, `name`, `location`, `slug`, `conference`, `division`, `color`,
`alternate_color`, `logo_url`.

**`nfl_players`** — `espn_athlete_id` (stable key), `full_name`, `first_name`, `last_name`,
`jersey`, `position_abbr`, `position_name`, `position_group`, `team_espn_id`, `team_abbr`,
`experience_years`, `is_rookie`, `status`, `headshot_url`, `name_is_ambiguous`.

**`nfl_coaches`** — `espn_coach_id` (stable key), `first_name`, `last_name`, `full_name`,
`team_espn_id`, `team_abbr`, `team_display_name`, `conference`, `division`,
`experience_years`, `headshot_url`, `name_is_ambiguous`.

Conference and division come from the `groups` endpoint, not from the teams endpoint. Coach
identity comes from the roster payloads, which carry a `coach` block; the headshot and
experience come from the core-API coach record.

## Things worth knowing

**`name_is_ambiguous` is not decoration.** Nine full names are shared by two active players
each, and in two cases they share a position as well, so position cannot disambiguate them —
only the team or the ID can:

| Name | Players |
|---|---|
| Justin Jefferson | MIN WR, CLE LB |
| DeVonta Smith | PHI WR, CAR CB |
| Christian Jones | CIN OT, ARI OT |
| Jaylon Jones | CHI CB, IND CB |
| Brandon Johnson | LV WR, SEA CB |
| Byron Young | LAR LB, PHI DT |
| Cam Miller | MIA QB, CAR CB |
| Devin Neal | NO RB, JAX S |
| Marcus Harris | KC DT, TEN CB |

Any importer resolving a player by name must treat these as ambiguous and refuse to guess.

**Coach headshots are only ~34% covered.** 11 of 32 coaches have a `headshot_url`; the rest
are empty, mostly newly promoted coaches with thin profiles. The URL contains a non-derivable
path segment (`/coaches/65/17553.jpg`), so it cannot be constructed from the ID the way player
headshots can. Design the coach picker with an initials or logo fallback, not a broken image.
Date of birth, birthplace and college are similarly sparse in the source and were excluded.

**No two coaches share a full name, but two surnames collide** — Jim Harbaugh (LAC) and John
Harbaugh (NYG), Matt LaFleur (GB) and Mike LaFleur (ARI). A search on `harbaugh` or `lafleur`
returns two people, so a coach picker must display the team alongside the name.

**ESPN uses `WSH` for Washington**, not `WAS`. Also `JAX` not `JAC`, and `LA` is ambiguous
between `LAR` and `LAC`.

**Names need normalisation for search.** The dataset contains apostrophes (`Ja'Mori Maclin`),
periods (`Frank Gore Jr.`) and hyphens (`Sedrick Van Pran-Granger`). The seed indexes four
forms — spaced and squashed, in both `First Last` and `Last First` order — so `ocyrus`,
`O'Cyrus`, `jackson, lamar` and `vanpran` all match.

**Missing values are normal.** 37 players have no jersey number and 9 have no headshot URL.
Both are nullable. Headshots follow
`https://a.espncdn.com/i/headshots/nfl/players/full/{espn_athlete_id}.png`, so the slim JSON
omits the column — construct it client-side with an `onerror` fallback.

**This snapshot is preseason.** Rosters are at their largest (90-man limits) and will be cut
to 53 before Week 1. Treat `is_rookie` and `status` as accurate for the snapshot date only.

## Stadiums

`nfl_stadiums.slim.json` maps each ESPN team abbreviation to its home venue: `venue`, `city`,
`state`, `roof`, `surface`, `espn_indoor`, `lat`, `long`. Roof, surface and coordinates come
from a supplied dataset; venue, city, state and `espn_indoor` come from ESPN venue records.

Two source abbreviations were remapped to ESPN's: **`LA` → `LAR`** and **`WAS` → `WSH`**.

**ESPN venue records carry no coordinates** — only `address`, `grass` and `indoor` — so the
supplied dataset is the only lat/long source, cross-checked against ESPN's reported state.

Corrections applied, each recorded in a `note` on the affected entry:

- **PHI** — latitude was `36.900833`, placing Lincoln Financial Field 334 km south in the
  Atlantic. Longitude was already exact, so this was a single leading digit. Corrected to
  `39.900833`.
- **BUF** — Buffalo moved into the new Highmark Stadium for 2026 (ESPN venue 11938). Surface
  corrected from turf to grass, and coordinates moved 441 m west from the old Ralph Wilson
  Stadium site to the new one, geocoded from OpenStreetMap.
- **LAR / LAC** — not an error. `roof` is `yes` while ESPN reports `indoor: false`, because
  SoFi's canopy covers the field but the sides are open. Both readings are kept, so use
  `roof` for shelter and `espn_indoor` for climate control.

`LAR`/`LAC` and `NYG`/`NYJ` share a venue and therefore share coordinates.

After correction, all 32 coordinates fall inside the state ESPN reports for that venue, and
roof/surface agree with ESPN's `indoor`/`grass` for 30 of 32 — the two exceptions being the
SoFi definitional difference.

## Refreshing

Re-fetch and regenerate rather than editing by hand:

```bash
curl "https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams"
curl "https://site.api.espn.com/apis/site/v2/sports/football/nfl/groups"
# then, for each of the 32 team ids:
curl "https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams/{id}/roster"
# coaches: id comes from each roster's coach block, then per coach:
curl "https://sports.core.api.espn.com/v2/sports/football/leagues/nfl/seasons/{year}/coaches/{coachId}"
```

Roughly 66 requests in total (2 + 32 rosters + 32 coaches), about 25 seconds and ~14 MB of raw
JSON for a full refresh. Sequence the calls with a small delay — this is an undocumented API
being used as a guest.

**Refresh must be upsert-only.** A player who leaves every roster should be marked inactive,
never deleted, or any stored pick referencing them breaks. A partially failed refresh must not
deactivate anybody.
