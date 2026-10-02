# Briefing: Odds Ingest & Prediction Risk / Expertise Metric

**Purpose.** This document is the ESPN-data and methodology half of a feature spec. Hand it to
the Claude project that holds the prediction game's own details (schema, scoring rules,
Wetteinsatz mechanics, player count, view structure). That instance should combine this with
its project knowledge and produce the Lovable plan-mode prompt.

**Everything in Part A was verified against the live ESPN API on 2026-09-13.** Treat it as
established fact rather than re-deriving it. Part B is a design recommendation; Part D lists what
still has to be decided with the user.

---

## ⚠️ TIME-CRITICAL — READ FIRST

**ESPN strips the odds block from a game as soon as it goes FINAL.** Verified in regular season,
preseason and postseason alike. There is no historical odds endpoint in this data source.

Consequence: **odds cannot be backfilled.** The 2026 season has already started — two week-1
games (NE @ SEA, SF VS LAR) are final and their odds are already gone permanently. Every day
without capture, more games lose their market probability for good.

So the ingest change should ship **before** any of the metric work, even as a standalone
write-only step that nothing reads yet. Capture first, compute later.

---

# PART A — VERIFIED ESPN FACTS

## A1. Odds are already in a payload the app fetches — zero additional requests

Odds live at `competitions[].odds[]` in the standard scoreboard response the app already
requests for schedule and live scores:

```
GET https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard
      ?seasontype={1|2|3}&week={n}&dates={seasonYear}
```

**No new endpoint, no new job, no incremental request cost.** Do not add a separate odds fetch,
and do not use the core-API per-game odds endpoint — it costs one request per game for data
already in hand.

## A2. Structure

`odds` is an array of providers. Currently exactly one is returned — DraftKings
(`provider.id: "100"`, `priority: 1`). Code for an array of length ≥ 1, select by lowest
`priority`, and tolerate 0.

```json
{
  "provider": { "id": "100", "name": "DraftKings", "priority": 1 },
  "details": "CIN -3.5",
  "spread": -3.5,
  "overUnder": 50.5,
  "moneyline": {
    "home": { "open": { "odds": "-198" }, "close": { "odds": "-198" } },
    "away": { "open": { "odds": "+164" }, "close": { "odds": "+164" } }
  },
  "homeTeamOdds": { "favorite": true,  "underdog": false, "favoriteAtOpen": true,
                    "team": { "abbreviation": "CIN" } },
  "awayTeamOdds": { "favorite": false, "underdog": true,  "favoriteAtOpen": false,
                    "team": { "abbreviation": "TB" } }
}
```

Also present: `pointSpread`, `total`, `header`, `footer`, `link`. The `link` objects are
sportsbook affiliate deep links — **do not surface them in the UI**; this is a no-money game.

Prefer `moneyline.{side}.close.odds`, falling back to `.open.odds`. Both were present and
identical at the time of measurement; they diverge as a line moves.

The `favorite` / `underdog` booleans are convenient but are **not a substitute** for the
probability — they are binary and lose all magnitude.

## A3. Coverage

Verified across the 2026 season: **every scheduled game carries a moneyline**, including week 18
whose kickoff times are still TBD.

| Week checked | Games | With odds | With moneyline |
|---|---|---|---|
| 2 | 16 | 16 | 16 |
| 5 | 15 | 15 | 15 |
| 10 | 14 | 14 | 14 |
| 18 | 16 | 16 | 16 |

Odds exist months ahead, so there is no "too early to capture" problem — only the too-late one
in the warning above.

## A4. Converting American moneyline to probability

```
negative odds (favourite):  p_raw = |odds| / (|odds| + 100)
positive odds (underdog):   p_raw = 100 / (odds + 100)
```

**These do not sum to 1 — you must de-vig.** Measured overround across all 14 priced week-1
games was tightly clustered at **1.041 – 1.048** (~4.2% juice). Normalise:

```
p = p_raw / (p_raw_home + p_raw_away)
```

Worked example, verified: `-198 / +164` → raw `0.664 / 0.379`, sum `1.043` → **`0.637 / 0.363`**.

Measured spread of de-vigged probabilities in one week: **0.196 to 0.804**. Surprisal
`−log₂(p)` for picking the away side ranged 0.71 to 2.35 bits.

## A5. There is no draw price — this is the main modelling gap

The moneyline is strictly two-way. No draw, tie or "3-way" market appears anywhere in the odds
object. But the game offers a **Draw** option in OUTCOME_ONLY mode, so a player can pick an
outcome the market does not price.

Empirical tie rates, measured from completed ESPN results earlier in this work:

| Season type | Tie rate | Why |
|---|---|---|
| Preseason (type 1) | **6.12%** (3 of 49) | no overtime is played |
| Regular season (type 2) | **0.37%** (1 of 272) | overtime, ties still possible |
| Postseason (type 3) | **0%** — impossible | play continues until a winner |

Recommended handling, to be confirmed with the user:

1. Inject `p_draw` as a **constant per season type** (≈0.004 regular, ≈0.06 preseason, 0 postseason),
   configurable rather than hard-coded.
2. Rescale the two-way probabilities so all three sum to 1: `p_home·(1−p_draw)`, `p_away·(1−p_draw)`.
3. **Cap the surprisal contribution of a correct draw pick.** At p≈0.004 an uncapped
   `−log₂(p)` is ~8 bits — one lucky tie would dwarf an entire season of skilled picking.

## A6. What is NOT worth fetching

- **`predictor` / FPI** — ESPN's model win probability. One request per game (+16/week) for a
  number the market already expresses better. Markets generally outperform public models.
- **`probabilities` endpoint** — play-by-play win probability. Very large, and irrelevant to a
  pre-kickoff risk measure.
- **Core-API odds endpoint** — one request per game for data already in the scoreboard payload.

---

# PART B — RECOMMENDED METRIC DESIGN

## B1. Separate boldness from skill

These are different questions and conflating them is the standard mistake:

| Measure | Question | Basis |
|---|---|---|
| **Mut-Index** (boldness) | How contrarian are your picks? | surprisal `−log₂(p)`, descriptive only |
| **Expertise-Index** (skill) | Are you actually good? | realised vs expected, this is the ranking |

Always picking longshots produces a high Mut-Index and a poor Expertise-Index. That separation
is the whole point — it prevents recklessness from reading as expertise.

## B2. Primary recommendation — "Siege über Erwartung"

```
Expertise = (number of correct picks) − Σ p_market(the outcome the player picked)
```

Why this one:

- **One sentence to explain.** "You got 11 right; the market said you should get 8.4; you are
  +2.6." In a friends' league, explicability is a feature, not a nicety.
- **Cannot be farmed by picking longshots.** Low expected value, but low realised value too, and
  you forfeit the easy wins.
- **Cannot be farmed by picking favourites either** — their expectation is already high.
- Needs only the de-vigged market probability, which is free.

## B3. Second axis — contrarian versus the league

This is the case the user described: the only player backing an underdog. It is a *crowd*
measure, not a market one, and belongs in its own number:

```
Einsamkeits-/Mut-Bonus = −log₂(p_crowd)
```

**Leave-one-out is mandatory.** Exclude the player's own pick when computing the crowd share, or
you partly measure the player against themselves. With ~7 players the difference between 1/7 and
0/6 is large.

**Smooth toward the market, not toward uniform.** Being "the only one" yields p = 1/7 in a
7-player league but 1/50 in a larger one — wildly different surprisal for identical behaviour:

```
p_crowd = (k + α · p_market) / (n + α)        α ≈ 2–3
```

where `k` = other players choosing that outcome, `n` = other players who have predicted. This
degrades gracefully when few picks exist yet, and needs no special-casing at n = 0.

## B4. The Wetteinsatz is confidence elicitation

The game already asks each player to flag one high-conviction match per week; it simply is not
scored as information today. Treat it as a **two-tier confidence signal** — unstaked = tier 1,
staked = tier 2 — which is how confidence-pool formats work without asking players to type
probabilities.

A natural third metric: **Wetteinsatz-Effizienz** — did stakes land on picks that were genuinely
bold *and* correct, versus stakes on 80% favourites.

## B5. Keep it out of the main standings

**Strong recommendation: run Expertise as a parallel leaderboard and leave the existing scoring
untouched.**

- The main table stays trivially explicable. Arguments about fairness are what kill friends'
  leagues.
- A risk-weighted main score is hard to explain and feels arbitrary when you lose.
- The risk formula can then be tuned mid-season without invalidating the real competition.

**Do not multiply main points by a risk factor.** With ~16 games a week, one lucky longshot
decides the season. If the user insists on risk affecting real points, use a small **additive,
capped** bonus on correct picks — never a multiplier.

## B6. Be honest about sample size

272 predictions a season sounds ample, but the standard error on a hit-rate difference is roughly
±3 games. Across ~7 players, the gap between 1st and 3rd on any expertise ranking will frequently
be noise. Render a range or an explicit "indikativ" label rather than a hard rank. Building this
in now avoids the December argument.

---

# PART C — IMPLEMENTATION CONSTRAINTS

1. **Capture in the existing ingest pass.** Same loop that already writes status, scores and
   kickoff. No new job, no new cron entry.
2. **Freeze at the prediction deadline.** Snapshot the closing line at kickoff − offset and never
   overwrite it. Two reasons: the data is destroyed once the game ends, and everyone's boldness
   must be measured against the same number regardless of when they predicted.
3. **Store parsed values, not the raw string** — both American odds and the de-vigged probability,
   so the conversion is auditable and not repeated per render.
4. **Nullable everywhere.** A game with no odds must produce no risk score, never a fabricated
   0.5. Exclude it from aggregates rather than imputing.
5. **Additive, non-breaking.** No change to prediction entry, deadlines, scoring or standings.
   The existing test suite passing unchanged is the acceptance criterion for the ingest phase.
6. **No sportsbook links, no odds displayed as betting inducements.** This is a no-money game
   among friends; surface derived risk/expertise, not a price to bet at.

---

# PART D — DECISIONS THE RECEIVING CLAUDE MUST SETTLE WITH THE USER

Ask these before producing the prompt; each materially changes the design.

1. **Parallel leaderboard, or does Expertise feed the main table?** (Recommendation: parallel.)
2. **Which metrics to expose** — Mut-Index, Expertise-Index, Wetteinsatz-Effizienz, or a single
   headline number?
3. **Draw probability constants** — use the measured empirical rates, or a configurable value?
   And what cap on a correct-draw surprisal?
4. **Preseason** — include preseason games in Expertise at all? Odds exist, but preseason results
   are close to noise, and the game already treats preseason as a separate competition.
5. **Backfill policy** — odds for already-final games are unrecoverable. Does the metric start
   from the next unplayed week, or does the season's first partial week get excluded explicitly?
6. **Crowd axis timing** — other players' picks are hidden until the deadline. Confirm the crowd
   probability is computed only after the deadline, from the final set of picks, so nothing leaks
   early.

---

# PART E — WHAT THE LOVABLE PROMPT SHOULD COVER

- **Phase 1, ship immediately:** odds snapshot in the existing ingest, frozen at deadline,
  nothing reading it. Migration additive and nullable.
- **Phase 2:** metric computation, running after a matchday is final, reusing the existing
  result-evaluation path rather than duplicating it.
- **Phase 3:** UI — leaderboard and per-prediction display, German copy, accessible (a bare
  number like `+2.6` needs a label; never colour alone).
- **Edge cases to require tests for:** missing odds; only one of the two moneylines present;
  malformed American odds string; overround far from ~1.04 (suspect data); a correct draw pick;
  preseason; postseason where draws are impossible; leave-one-out with n = 0 other predictions;
  a player who predicted after the deadline is excluded; retroactive result correction changing a
  metric; re-running the computation is idempotent.
- **Explicitly out of scope:** displaying betting odds or sportsbook links to players; changing
  existing scoring; any new ESPN endpoint.

---

## Appendix — reference values for testing

| Check | Verified value |
|---|---|
| Overround range, one week, 14 games | 1.041 – 1.048 |
| `-198` / `+164` de-vigged | 0.637 / 0.363 |
| Widest priced game observed | ARI @ LAC, `-520` / `+390` → 0.804 / 0.196 |
| Closest to a coin flip | BUF @ HOU, `-108` / `-112` → 0.496 / 0.504 |
| De-vigged probability range observed | 0.196 – 0.804 |
| Tie rate, regular season 2025 | 0.37% (1 / 272) |
| Tie rate, preseason 2025 | 6.12% (3 / 49) |
