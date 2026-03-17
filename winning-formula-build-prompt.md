# Winning Formula — Full Rebuild Prompt for Claude Code

Paste this entire prompt into Claude Code. Work through each phase in order and confirm completion before moving to the next.

---

## PHASE 1 — PRISMA SCHEMA

Update `formulaWinningFormula` to store full model state:
- Add `betaCoefficients` JSON (elastic net betas keyed by feature name)
- Add `intercept` Float
- Add `setCount` Int
- Add `lambdaSelected` Float
- Add `aucScore` Float
- Add `version` Int (increment on each recompute, default 1)
- Keep existing `targets`, `zones`, `narrative` fields

Create a new `formulaWinningFormulaHistory` table that snapshots every version:
- `id`, `teamId`, `version`, `betaCoefficients`, `intercept`, `setCount`, `aucScore`, `computedAt`
- This enables trend detection later — every recompute saves a row here

Run migrations after schema changes.

---

## PHASE 2 — PORT ELASTIC NET TO SERVER SIDE

Move the following functions from `formula/src/lib/formula-engine.ts` into `api/src/services/formula-ai.ts`:
- `_fitElasticNet`
- `_cvSelectLambda`
- `_sigmoid`, `_dot` helpers
- `computeFormulaRow()` — the feature matrix builder

Then rebuild `computeWinningFormula(teamId)` as the single server-side pipeline:

1. Pull all set-level events for the team from the database
2. Build the feature matrix using `computeFormulaRow()` — 10 features: `hitting_eff`, `kill_rate`, `serve_pressure`, `ace_error_ratio`, `reception_eff`, `reception_error_rate`, `stuff_block_rate`, `block_touch_rate`, `dig_rate`, `cover_rate`
3. **Apply temporal weighting** — rows should not be equally weighted. Apply exponential decay so recent sets carry more weight than early season sets. Use a decay factor of 0.95 per set going backwards from most recent. Implement this by multiplying each row's gradient contribution by its weight during the elastic net fitting step.
4. Run `_cvSelectLambda` to select best lambda
5. Fit `_fitElasticNet` with selected lambda, `alpha = 0.5`
6. Compute ROC threshold per feature
7. Build `betaMap` — absolute value of each feature's fitted coefficient, keyed by feature name
8. Build `gainsMap` — distance of team's current average from ROC threshold per feature
9. **Fix the ranking bug** — when sorting `belowThreshold` to assign `high_impact` zones, sort by `Math.abs(betaMap[name]) * gainsMap[name]` not `gainsMap[name]` alone. This ensures team-specific predictiveness drives what surfaces, not a universal benchmark.
10. Classify priority zones and generate narrative using existing domain phrase logic
11. **Compute win probability lift** — for each `high_impact` variable, calculate projected win rate lift using the following fixed conservative increments (Phase 1 values, to be replaced with data-derived values in a future update):

```
hitting_eff:          +0.030
dig_rate:             +0.5
serve_pressure:       +0.05
reception_eff:        +0.03
stuff_block_rate:     +0.3
kill_rate:            +0.04
ace_error_ratio:      +0.05
reception_error_rate: -0.02
block_touch_rate:     +0.3
cover_rate:           +0.4
```

For each stat: take current team averages → compute win probability via sigmoid of `intercept + dot(currentStats, beta)` → apply increment to that stat only → recompute probability → difference is the projected lift. **Only compute and return lift if `setCount >= 12`.** Cap any single stat lift at 25%. Round to nearest whole number percent.

12. Upsert `formulaWinningFormula` with full model state including betas, intercept, setCount, lambdaSelected, aucScore, version incremented
13. Insert a row into `formulaWinningFormulaHistory` with the same snapshot
14. Return the full result

---

## PHASE 3 — UPDATE API ROUTES

In `formula.ts`:

**GET `/teams/:id/winning-formula`** — return the full stored model state including `betaCoefficients`, `setCount`, `aucScore`, `version`, `targets`, `zones`, `narrative`, and `projectedLift` per high impact variable (only if setCount >= 12)

**POST `/teams/:id/winning-formula`** — trigger `computeWinningFormula(teamId)`, return the result immediately

---

## PHASE 4 — AUTO-TRIGGER ON MATCH COMPLETE

Find where matches are finalized in the API — where the last set is saved or match status is set to complete. After the match save resolves, call `computeWinningFormula(teamId)` asynchronously — do not await it in the request, fire it in the background so it does not block the coach's match save response.

---

## PHASE 5 — STRIP CLIENT ENGINE

In `formula/src/lib/formula-engine.ts`:
- Remove `_fitElasticNet`, `_cvSelectLambda`, `_sigmoid`, `_dot`, `computeFormulaRow`, and the full `computeWinningFormula` client-side pipeline
- The client engine should now only contain rendering helpers — zone color mapping, narrative formatting, display utilities
- No regression logic should remain client-side

Update `api-client.ts` so `formulaApi.computeWinningFormula()` calls `POST /teams/:id/winning-formula` and `formulaApi.getWinningFormula()` calls `GET /teams/:id/winning-formula`

Update hooks in `useFormula.ts` to consume the new response shape including `setCount`, `aucScore`, and `projectedLift`

---

## PHASE 6 — UI ADDITIONS

On the Winning Formula screen add:

### 1. Set count + confidence indicator

Display near the top of the results. Use the following plain-language labels based on setCount and aucScore. Never show the AUC number itself. Never use the words "confidence," "model," "accuracy," "regression," or any stats terminology in the UI.

| Condition | Display |
|---|---|
| setCount < 8 | `"Add more match data to unlock your Winning Formula."` |
| setCount 8–11, AUC < 0.60 | `"Your Formula is live — these early insights are a starting point that gets more accurate with every match you add."` |
| setCount 8–11, AUC 0.60–0.75 | `"Your Formula is working — use these insights to guide practice now, and they'll get sharper as the season goes on."` |
| setCount 8–11, AUC > 0.75 | `"Your Formula is already telling you something real — act on these now and they'll get even more precise with more data."` |
| setCount 12+, AUC < 0.60 | `"These insights reflect your team's current trends — the more matches you add, the more specific your Formula gets."` |
| setCount 12+, AUC 0.60–0.75 | `"Your Formula has a strong read on your team — these insights are reliable now and get sharper with every match."` |
| setCount 12+, AUC > 0.75 | `"Your Formula knows your team — these insights are highly reliable and ready to drive your practice planning."` |

Below the label, always show: `"Based on {setCount} sets"`

### 2. Win probability lift

Display on each `high_impact` variable card only when `setCount >= 12`:

- Primary line: `"Could win {X}% more sets"`
- Secondary line in smaller text: `"If your {displayName} improves by {plain increment description}"`

Plain increment descriptions per stat:
```
hitting_eff:          "a small but consistent margin"
dig_rate:             "about half a dig per set"
serve_pressure:       "a small but consistent margin"
reception_eff:        "a small but consistent margin"
stuff_block_rate:     "roughly one extra stuff block every 3 sets"
kill_rate:            "a small but consistent margin"
ace_error_ratio:      "a small but consistent margin"
reception_error_rate: "reducing errors by a small margin"
block_touch_rate:     "roughly one extra block touch every 3 sets"
cover_rate:           "roughly one extra cover every 3 sets"
```

If setCount is between 8 and 11, show a locked state on the card:
`"Win projections available after {12 - setCount} more sets of data"`

Never use the words "projected," "probability," "lift," "regression," or "increment" anywhere in the UI.

### 3. Tagline

Display prominently on the Winning Formula screen:
`"Your Winning Formula gets stronger every match."`

### 4. Variable card zone labels

Replace all priority zone labels with plain language a coach immediately understands:

| Current | Replace with |
|---|---|
| `high_impact` | `"Focus Here"` |
| `moderate_impact` | `"Worth Watching"` |
| `on_target` | `"On Track"` |
| `low_relationship` | `"Not a Factor Right Now"` |

---

## VERIFICATION

After all phases are complete, confirm:
- Two different teams with different stat profiles produce different top predictors
- A fresh GET after a match save returns an updated `setCount`
- Win probability lift only appears at 12+ sets
- The history table is receiving a new row on each recompute
- The client bundle no longer contains any elastic net or sigmoid logic
- All UI labels use plain language with no stats terminology
