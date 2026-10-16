# Briefing: Overtime Flag & Quarter Scores (Linescores)

**Purpose.** ESPN-data half of a feature spec for the Claude chat that holds the prediction
game's details and writes the Lovable plan-mode prompt. Verified against the live API on
2026-10-02 across the complete 2025 season (272 regular-season + 14 postseason games) and the
current 2026 week 4.

**Headline: zero additional API requests.** Both the overtime information and the per-quarter
scores are already inside the scoreboard payload the app fetches for schedule, live scores and
results. This feature is parse → store → render.

---

## A. Verified facts

### A1. Where the data is

In each event of the existing scoreboard response:

```json
"status": { "period": 5,
            "type": { "name": "STATUS_FINAL", "state": "post", "completed": true,
                      "detail": "Final/OT", "shortDetail": "Final/OT", "altDetail": "OT" } },
"competitions": [{ "competitors": [
  { "homeAway": "home", "team": {"abbreviation": "DAL"}, "score": "40",
    "linescores": [ {"period":1,"value":0.0,"displayValue":"0"},
                    {"period":2,"value":10.0,"displayValue":"10"},
                    {"period":3,"value":7.0,"displayValue":"7"},
                    {"period":4,"value":20.0,"displayValue":"20"},
                    {"period":5,"value":3.0,"displayValue":"3"} ] },
  { "homeAway": "away", "team": {"abbreviation": "NYG"}, "score": "37",
    "linescores": [ … 6, 7, 3, 21, 0 ] } ] }]
```

(Real data: 2025 week 2, NYG @ DAL, 40–37 in overtime.)

### A2. Overtime detection

| Observation (2025, all 286 games) | Count |
|---|---|
| `period: 4`, `detail: "Final"`, 4 linescore entries | 258 + postseason |
| `period: 5`, `detail: "Final/OT"`, `altDetail: "OT"`, 5 linescore entries | **14 regular season + postseason OT** |

- **`status.type.name` stays `STATUS_FINAL` for overtime games.** There is no separate
  "final overtime" status, so a check on the status name alone will never find an OT game.
- **Rule: `overtime = status.period > 4`** (equivalently, more than 4 linescore entries).
  Prefer the numeric period over parsing the `"Final/OT"` text.
- Only single overtime periods (`period: 5`) were observed, including in the postseason.
  Multiple overtimes are possible in the playoffs (where games cannot end tied) and would
  presumably appear as `period: 6+`; the exact ESPN text for that ("2OT"?) was **not observed**,
  so derive the label from the period count, not from `detail`.

### A3. Linescores

- Present on **every competitor of every completed game** — 0 missing out of 572.
- **Sum of linescores equals the final score in all 572 cases (0 mismatches).** Safe to store
  and display; also usable as an integrity check on the stored final score.
- `period` is given on each entry — key by `period`, do not rely on array position.
- Values are floats (`3.0`); store as integers.
- **Scheduled games carry an empty array** (`linescores: []`) — verified on 2026 week 4.
- Live behaviour (array growing quarter by quarter during a game) is expected but was **not
  observed**: no game was in progress during verification. Treat partial arrays as normal
  while `state === "in"`.

### A4. Overtime ties exist

The 2025 season's only tie, **GB @ DAL 40–40**, was `Final/OT`: both teams scored 3 in
overtime. So the display must handle "OT" and "Unentschieden" together. Regular-season ties
are only possible after overtime, so a tie in the regular season always carries the OT flag.

### A5. Other sources — not needed

- `summary?event={id}` has a full boxscore and play-by-play, but costs one request per game and
  is ~550 KB. Not justified for quarter scores.
- Do **not** fetch quarter scores on click from ESPN. The data is already in hand.

---

## B. Recommended design

### B1. Ingest (existing job, no new requests)

In the same pass that already writes final scores:

- `overtime boolean` (or `periods smallint`, from which overtime and the number of OT periods
  derive) — set from `status.period`.
- Per team, the quarter scores: either a small JSONB array on the game row
  (`home_linescores`, `away_linescores`, e.g. `[0,10,7,20,3]`) or a child table
  `game_period_scores (game_id, period, home, away)`. For a display-only feature the JSONB
  array on the game row is simpler and needs no join.
- Write while the game is live (if the app shows live quarters) and **freeze with the final
  result**, following the same freeze/revision rules the app already applies to final scores.
- **Integrity check:** if the linescore sum ≠ stored final score, keep the score, drop the
  linescores for that game and log it — never show a quarter breakdown that contradicts the
  result.
- **Backfill** for already-final games of the current season: ESPN keeps linescores on final
  games (unlike odds), so a one-off re-read of past weeks' scoreboards fills them. That is one
  request per past week (≈ 4 now), not per game.

### B2. UI

- **Result line:** `27 : 24` becomes `27 : 24 n.V.` for overtime (German *nach Verlängerung*),
  or a small `OT` badge if the app keeps English sports terms (it already uses W-L-D). The
  marker must be text, not colour alone, and in the accessible label: "Endstand 40 zu 37 nach
  Verlängerung".
- **Tie after overtime:** `40 : 40 n.V.` together with the existing draw handling.
- **Tap on the result → expandable linescore** (inline accordion or bottom sheet, matching how
  the app already expands content):

  ```
            1   2   3   4   OT  | Ges.
  NYG       6   7   3  21    0  |  37
  DAL       0  10   7  20    3  |  40
  ```

  OT column only when present; with multiple overtimes label `OT1`, `OT2`. Semantic `<table>`
  with caption, tabular numerals, fits 360 px. The result element is a `button` with
  `aria-expanded`. Show nothing to expand for scheduled games; for live games show completed
  quarters only (if live linescores are stored).
- No layout shift: the data comes with the match row, no fetch on tap.

### B3. Prediction scoring

Confirm with the user how predictions treat overtime. The ESPN final score **includes**
overtime; if the game's rules intend the result after regulation (common in some prediction
games), the stored per-quarter data allows computing the 60-minute score. **Do not change
scoring as part of this feature unless the user explicitly decides so** — it would alter past
results.

---

## C. Edge cases for tests

1. Regulation final → `overtime = false`, 4 periods, no OT marker
2. OT final (`period 5`) → marker shown, OT column shown
3. Tie after OT (GB @ DAL 40–40) → `n.V.` and draw display together
4. Status name `STATUS_FINAL` with `period 5` → still detected as OT
5. `period 6` (double OT, synthetic fixture) → `OT1`, `OT2` columns, no crash
6. Scheduled game, `linescores: []` → nothing to expand
7. Linescore sum ≠ score (synthetic) → linescores dropped, logged, score unaffected
8. Linescore entries out of order (synthetic) → sorted by `period`
9. Backfill run twice → idempotent
10. Existing prediction/scoring tests → unchanged

## D. Questions for the user

1. Overtime label: `n.V.` (German) or `OT`?
2. Quarter view: inline expand or bottom sheet? Also during live games, or finals only?
3. Should predictions be judged on the final result including overtime (ESPN default) — confirm
   the current rule is unaffected.
4. Where to show it: Spieltag only, or also past results in Tippen and Übersicht?
