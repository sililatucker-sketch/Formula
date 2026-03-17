# Formula App — 14-Item Update Implementation Plan

## Context

The Formula volleyball analytics app is a monolithic single-file React application at `/Users/sililatucker/Formula/index.html` (~6000 lines) wrapped with Capacitor for iOS/Android. It uses React hooks for state management, Firebase Realtime Database for persistence, and a custom stack-based navigation system. All 14 changes target this single file unless noted otherwise.

---

## Item 1: Editable Scoreboard

### Current State
- **Lines 4299–4332**: Scoreboard renders `set?.ourScore` and `set?.theirScore` as read-only `<span>` elements styled with `S.bigScore`.
- Scores only change via `scorePoint(forUs)` (line 1322) which increments by 1 and handles rotation/serve changes.
- Undo stack entries store `_undoScore` snapshots (lines 1421–1465) for rally-ending events.

### Changes Needed
1. **New state** (after line 654):
   ```js
   const [showScoreEdit, setShowScoreEdit] = useState(false);
   const [editScoreUs, setEditScoreUs] = useState(0);
   const [editScoreThem, setEditScoreThem] = useState(0);
   ```
2. **Make score tappable** (lines 4311, 4325): Add `onClick` to `S.bigScore` spans:
   ```js
   onClick={() => {
     setEditScoreUs(set?.ourScore || 0);
     setEditScoreThem(set?.theirScore || 0);
     setShowScoreEdit(true);
   }}
   ```
3. **Score edit modal** (render after line 4332): Full-screen overlay using the same pattern as the timeout overlay (line 4152). Contains:
   - Team labels ("US" / opponent name)
   - Two stepper controls: each has a `−` button, score display, and `+` button
   - SAVE button: pushes an undo entry with `_undoScore` snapshot of current state, then calls `updateCurrentSet()` to set new scores directly. Does NOT trigger rotation or serve logic — this is a manual correction only.
   - CANCEL button: closes modal without changes
   - On SAVE: also reset `rallyPhase` to `"idle"` and `currentRally` to `null` to prevent mid-rally confusion
4. **Gate Item 14 pulse**: Score edit tap must NOT trigger the score pulse effect. The pulse is gated behind `!showScoreEdit` (see Item 14).

### Files Affected
- `index.html` lines 654 (new state), 4311/4325 (onClick), after 4332 (modal overlay)

### Dependencies/Risks
- Manual edits bypass rotation logic intentionally. Users should use the rotation fix tools or Item 2's Rotate button separately if needed.
- Mid-rally score edits reset rally state to prevent inconsistency.

---

## Item 2: Rotation Button on Subs Page

### Current State
- **Lines 5753–5850**: `FixLineup` component shows a 6-zone grid (front: Z4/Z3/Z2, back: Z5/Z6/Z1), swap-two-zones functionality, and a "RESET TO ROTATION" section.
- **Lines 1100–1113**: `rotate(lineup)` — pure function that performs clockwise rotation (Z1←Z2←Z3←Z4←Z5←Z6←Z1).
- Parent passes `rotate` as prop at line 5670.

### Changes Needed
1. **Add `onRotate` callback** in the parent component where FixLineup is rendered (lines 5644–5671):
   ```js
   onRotate={() => {
     updateCurrentSet((s) => ({
       ...s,
       currentLineup: rotate(s.currentLineup),
       rotation: (s.rotation % 6) + 1,
       liberoSwaps: {},
     }));
     setRallyPhase("idle");
     setCurrentRally(null);
     showToast("Rotated clockwise");
   }}
   ```
2. **Add ROTATE button** inside `FixLineup` (around line 5825, between zone grid and RESET TO ROTATION section):
   - Full-width green button labeled "ROTATE →"
   - Calls `onRotate()` prop
3. **Update FixLineup signature** (line 5753) to accept `onRotate` prop.

### Files Affected
- `index.html` lines 5644–5671 (parent prop), 5753 (component signature), ~5825 (new button)

### Dependencies/Risks
- Must reset `liberoSwaps` on rotation since libero/MB back-row mapping depends on who is in back row after rotation.
- Resets rally state to prevent mid-rally rotation issues.

---

## Item 3: Full Data Persistence

### Current State
**Persisted** (via `updateCurrentSet` → `save()` → Firebase `rooms/${roomCode}/matches`):
- `currentMatch` with all `sets[]` (scores, lineups, events, rallies, subs, rotation, weServe, liberoSwaps)

**NOT persisted** (React `useState` only — lost on page reload):
- `tab` (line 635) — active tab ("court"/"stats"/"formula"/"scout")
- `rallyPhase` (line 636) — "idle"/"offense"/"defense"
- `prevRallyPhase` (line 637) — undo helper
- `currentRally` (line 638) — in-progress rally data
- `prevRally` (line 639) — undo helper
- `selectedPlayer` (line 640) — tapped player ID
- `undoStack` (line 641) — undo entries array
- `assistPrompt` (line 642) — assist attribution state
- `currentSetIdx` (line 607) — active set index
- In-match scout tab UI state (`scoutPending`, view mode) — not persisted, but `set.shots` IS persisted

### Changes Needed

**IMPORTANT**: This item is implemented LAST, after all other items are complete, so all new state fields (from Items 1, 4, 5, 11, 14) are finalized before building the persistence layer.

1. **New Firebase path**: `rooms/${roomCode}/liveSession/${matchId}`
2. **Live session object**:
   ```js
   {
     tab, rallyPhase, currentRally, selectedPlayer, undoStack,
     assistPrompt, currentSetIdx, prevRallyPhase, prevRally,
     // Plus any new state added by other items (showScoreEdit, showMatchSettings, etc.)
   }
   ```
3. **Atomic batched writes**: All state changes from a single rally resolution (score + rotation + event + rally phase) are grouped into ONE write, not four separate debounced writes. Implement by collecting state changes into a batch ref and flushing on a 500ms debounce:
   ```js
   const sessionBatchRef = useRef({});
   const flushTimerRef = useRef(null);

   const queueSessionWrite = (partialState) => {
     Object.assign(sessionBatchRef.current, partialState);
     clearTimeout(flushTimerRef.current);
     flushTimerRef.current = setTimeout(() => {
       fbDb.ref(`rooms/${roomCode}/liveSession/${currentMatch.id}`).update(sessionBatchRef.current);
       sessionBatchRef.current = {};
     }, 500);
   };
   ```
4. **useEffect to auto-save** on state changes when `screen === "match"`. Each setter (setTab, setRallyPhase, etc.) calls `queueSessionWrite()` with its portion.
5. **Restore on match resume**: When navigating to match screen, load live session from Firebase and hydrate all state variables:
   - Clamp `currentSetIdx` to `0..sets.length-1`
   - Validate `selectedPlayer` still exists in current lineup, set to null if not
   - Validate `undoStack` entries reference existing event IDs
6. **Cleanup**: Delete live session when match ends (completes or ends early) at lines 1511–1527 and 1530.
7. **Cap undo stack** at ~50 entries to limit Firebase payload size.

**NOT persisted** (intentionally transient): `showTimeout`, `showRotationFix`, `showScoreEdit`, `showMatchSettings`, `toast`, `scorePulse` — modal/notification/animation states that would be confusing to restore.

### Files Affected
- `index.html` lines 607, 635–642, 977 (new persist logic), ~2362 (match resume), 1511–1527 (match end)

### Dependencies/Risks
- Undo stack serializes cleanly (plain objects with string/number values, no circular refs).
- Batched debounced writes prevent Firebase flooding while ensuring rally-resolution atomicity.
- Stale session: clamp and validate all restored state. If a live session references a match that is already `complete`, ignore/delete it.

---

## Item 4: In-Match Settings Icon

### Current State
- **Lines 4299–4332**: Score header has back button + T/O button (left side) and undo button (right side). No settings access during live play.
- **Lines 3183–3354**: Pre-match setup options (format, serve/receive, libero assignments). Not accessible once match starts.

### Changes Needed
1. **New state** (near line 654):
   ```js
   const [showMatchSettings, setShowMatchSettings] = useState(false);
   ```
2. **Gear icon** (line ~4329, right side of header alongside undo button):
   ```js
   <button style={S.undoBtn} onClick={() => setShowMatchSettings(true)}>⚙</button>
   ```
3. **Settings overlay** (full-screen, same pattern as timeout overlay at line 4152). Sections:

   **a. Serve/Receive Toggle**: Two buttons ("We Serve" / "They Serve"). On change:
   ```js
   updateCurrentSet(s => ({...s, weServe: newValue}));
   setRallyPhase("idle");
   setCurrentRally(null);
   ```

   **b. Play-to Score** (see Item 5): Quick buttons for 25 and 15, plus custom inline number input. Saves via `updateCurrentSet(s => ({...s, playTo: value}))`.

   **c. Libero-MB Assignment**: For each MB currently in lineup, show a dropdown to assign libero1/libero2/self. Updates `set.mbBackRowMap`.

   **d. Position Overrides**: List of on-court players with tappable position badges (OH/MB/OPP/S/DS/L). Updates `set.positionOverrides`.

   **e. Close button** at top.

### Files Affected
- `index.html` lines 654 (new state), 4299–4332 (gear icon in header), new overlay block after timeout overlay

### Dependencies/Risks
- Changing serve/receive mid-rally resets rally state to prevent inconsistency.
- Position overrides use existing `set.positionOverrides` mechanism — no new storage needed.
- Past events retain their original position data; overrides only affect future events.

---

## Item 5: Play-To Score Setting

### Current State
- **Lines 1680–1694**: `getSetTargetScore(setData)` hardcodes `baseTarget = 25` for regular sets and `15` for deciding sets. Win-by-2 applies when `maxScore >= baseTarget - 1`.
- AI suggestions (lines 4128–4131) use raw scores with hardcoded thresholds like `>= 20`. No awareness of play-to target.
- **Lines 3304–3344**: New set creation — `emptySet(setNum)` at line 3307 does NOT include any `playTo` field (line 542–567 shows `emptySet` definition).

### Changes Needed
1. **New field on set object**: `set.playTo` (optional number). NOT added to `emptySet()` — each new set defaults to standard rules unless manually overridden.

2. **Modify `getSetTargetScore()`** (line 1681):
   ```js
   const getSetTargetScore = (setData) => {
     const setNum = setData?.number || 1;
     const format = currentMatch?.format || 3;
     const decidingSetNum = format === 5 ? 5 : 3;
     // Use custom playTo if set, otherwise standard rules
     const baseTarget = setData?.playTo || (setNum >= decidingSetNum ? 15 : 25);
     // ... rest of win-by-2 logic unchanged
   };
   ```

3. **UI in settings panel** (Item 4): Three buttons — 25, 15, Custom. Custom opens inline number input (NOT `prompt()` — bad mobile UX). Validation: min 1, max 99. Saves via:
   ```js
   updateCurrentSet(s => ({...s, playTo: value}))
   ```

4. **CRITICAL — playTo must NOT carry over to new sets**: Verify the set creation code at lines 3304–3344. The `emptySet(setNum)` function (line 542) does NOT include `playTo`, and the set creation code (lines 3308–3326) copies specific fields but does NOT copy `playTo` from any previous set. This is already correct — no change needed here, just verify.

5. **Update AI suggestions** (lines 4128–4131):
   ```js
   const targetScore = getSetTargetScore(set);
   const ourRemaining = targetScore - (set?.ourScore || 0);
   const theirRemaining = targetScore - (set?.theirScore || 0);
   ```
   - Replace `scoreDiff >= 5 && (set?.ourScore||0) >= 20` with `scoreDiff >= 5 && ourRemaining <= 5`
   - Add: `if (ourRemaining <= 3) sug.push("${ourRemaining} points from the set — finish strong.");`
   - Add: `if (theirRemaining <= 3) sug.push("Opponent ${theirRemaining} from the set — tighten up now.");`

6. **Optional**: Show target score on scoreboard set chip (line 4314): `S${set?.number || 1} →${targetScore}`

### Files Affected
- `index.html` lines 542–567 (verify emptySet), 1680–1694 (getSetTargetScore), 3304–3344 (verify set creation), 4128–4131 (AI suggestions), settings panel (Item 4)

### Dependencies/Risks
- `playTo` stored on set object, auto-persisted via `updateCurrentSet()`.
- Win-by-2 still applies on top of custom `playTo`.
- New sets do NOT inherit `playTo` from previous sets — verified by code inspection.

---

## Item 6: Uniform Offense Buttons

### Current State
- **Lines 4705–4731**: Offense hitters get: KILL, ATT E, BLKD, CVR, DUG, ERR
- **Lines 4734–4763**: Offense setter gets: SET + KILL, ATT E, BLKD, CVR, DUG, ERR
- **Lines 4766–4776**: Offense non-hitter/non-setter gets: **only CVR and ERR** ← the problem
- **Lines 1407–1416**: `getHittersOnCourt()` excludes positions S, L, DS. A DS on court during offense only gets 2 buttons.

### Changes Needed
Replace the non-hitter/non-setter block (lines 4766–4776) with the full hitter button set. Copy the exact button structure and handlers from lines 4705–4731:

```js
{/* OFFENSE: non-hitter/non-setter — now gets FULL buttons */}
{phase === "offense" && !isHitter && !isSetter && (
  <>
    <button style={{ ...S.cb, background: "#166534" }} onClick={() => {
      recordEvent(pid, "kill"); showToast(`${firstName} KILL`);
      setAssistPrompt({ killerId: pid, rally: { ...currentRally } });
    }}>KILL</button>
    <button style={{ ...S.cb, background: "#7f1d1d" }} onClick={() => {
      recordEvent(pid, "attack_error"); showToast(`${firstName} E`);
      endRally(false, { ...currentRally, endStat: "attack_error", attacker: pid });
    }}>ATT E</button>
    <button style={{ ...S.cb, background: "#0a1929" }} onClick={() => {
      recordEvent(pid, "attack_blocked"); showToast(`${firstName} BLKD`);
      endRally(false, { ...currentRally, endStat: "attack_blocked", attacker: pid });
    }}>BLKD</button>
    <button style={{ ...S.cb, background: "#1e3a5f" }} onClick={() => {
      recordEvent(pid, "cvr"); showToast(`${firstName} CVR`);
    }}>CVR</button>
    <button style={{ ...S.cb, background: "#854d0e" }} onClick={() => {
      showToast("Dug → DEF");
      setRallyPhase("defense");
    }}>DUG</button>
    <button style={{ ...S.cb, background: "#991b1b" }} onClick={() => {
      recordEvent(pid, "ball_error"); showToast(`${firstName} ERR`);
      endRally(false, { ...currentRally, endStat: "ball_error" });
    }}>ERR</button>
  </>
)}
```

Rationale: Any player on court can attack (tip, dump, emergency swing). Restricting buttons causes data loss.

### Files Affected
- `index.html` lines 4766–4776 (replace entire block)

### Dependencies/Risks
- The assist prompt flow (`setAssistPrompt`) stores `killerId` with no hitter role check — works for non-hitters already.
- PIR calculations use event types, not role classification — correctly captures non-hitter attacks.
- Low risk, surgical change.

---

## Item 7: Player Names Not Numbers

### Current State — Locations Using Numbers Instead of/Alongside Names

| Location | Lines | Current Pattern |
|---|---|---|
| Subs list / rotation fix | 4404 | `#{number} firstName` (number-first) |
| PIR rankings (matchComplete) | 2844 | `#{number} firstName` (number-first) |
| Setup screens | 3155 | `#{number}` (number only) |
| Timeout PIR breakdown | 4267 | `#{num}` badge alongside name |
| AI suggestions | 4109–4126 | `player.name` (full name, not first) |

Most other locations already use `firstName || #number` fallback correctly.

### Changes Needed
1. **Add `displayName` helper** (near line 1150):
   ```js
   const displayName = (player) => {
     if (!player) return "???";
     const first = player.name?.split(" ")[0] || player.pName?.split(" ")[0];
     const num = player.number || player.num || player.pNum;
     if (first) return first;
     return num ? `#${num}` : "???";
   };
   ```
2. **Standardize all locations to name-first pattern**: `firstName #number` where space allows, `firstName` alone in tight spaces. Numbers become secondary context, not primary identity.
3. **Specific fixes**:
   - Line 4404: Change from `#{number} firstName` → `displayName(player)`
   - Line 2844: Change from `#{number} firstName` → `displayName(player)`
   - Line 3155: Add name alongside number using `displayName()`
   - Lines 4109–4126: Change `player.name` → `displayName(player)` for conciseness (first name only)
   - Line 4267: Keep number badge as secondary but ensure name is primary display

### Files Affected
- `index.html` — ~12 locations listed above, plus new helper at ~line 1150

### Dependencies/Risks
- Helper must handle both roster player shape (`{name, number}`) and derived stat shapes (`{pName, pNum}`).
- Low risk — display-only changes.

---

## Item 8: Softer AI Language

### Current State
All suggestion strings are at lines 4075–4149 (timeout overlay) and 1717–1730 (serve risk). Current tone issues: "struggling", "costly", "hurting", "reduce touches or sub", "clean up", "Too many errors".

### Changes Needed — Complete String Revision Table

| Line | Current | Proposed |
|---|---|---|
| 4100 | `"Attack efficiency critically low — run middles/quicks to open the pins."` | `"Attack efficiency below .100 — try working middles and quicks to create better pin angles."` |
| 4101 | `"Hitting below .200 — consider spreading offense to force defensive adjustments."` | `"Hitting below .200 — spreading the offense can pull their block and open lanes."` |
| 4102 | `"Serve errors exceed aces — dial back risk, prioritize placement."` | `"Serve errors exceed aces — shifting to placement-first serving could flip this margin."` |
| 4103 | `"X ball handling errors — clean up passing and setting contacts."` | `"X ball handling errors — focusing on clean first contacts will tighten the system."` |
| 4109 | `"🔥 name highest impact at +X — emphasize them."` | `"🔥 name leading the way at +X — look to keep getting them touches."` |
| 4114 | `"⚠️ name struggling at X PIR — reduce touches or sub."` | `"⚠️ name at X PIR — consider a role adjustment or a breather."` |
| 4119 | `"name serve errors costly — have them dial back."` | `"name serve margin is negative — a shorter toss or more placement could help."` |
| 4120 | `"name pass errors hurting — adjust positioning."` | `"name passing trending down — check court positioning and platform angle."` |
| 4124 | `"name hitting .XXX — keep feeding them."` | `"name hitting .XXX — keep running plays through them."` |
| 4126 | `"name negative hitting — reduce their swings."` | `"name at negative efficiency — consider redirecting sets to balance the attack."` |
| 4129 | `"Down 5+ — consider lineup change or aggressive serving to break their rhythm."` | `"Down 5+ — a strategic sub or a change in serve strategy could shift momentum."` |
| 4130 | `"Down 3 — stay disciplined, minimize unforced errors, make them earn it."` | `"Down 3 — staying disciplined on serve and pass will set up a run."` |
| 4135 | `"Low digs — adjust defensive base positions."` | `"Low dig count — reviewing defensive base positions could create more touches."` |
| 4136 | `"No stuff blocks — re-evaluate blocking scheme and assignments."` | `"No stuff blocks yet — consider adjusting block assignments or closing the seam."` |
| 1728 | `"Errors outpacing aces. Consider backing off."` | `"Errors outpacing aces. Shifting to placement could improve the margin."` |
| 1729 | `"Too many errors. Prioritize getting serves in."` | `"High error rate. Prioritizing serve accuracy will stabilize the rotation."` |

### Files Affected
- `index.html` lines 4100–4136, 1717–1730

### Dependencies/Risks
- Pure string literal changes — zero logic impact.
- Keep emoji (🔥/⚠️) for visual scanning.

---

## Item 9: Set Summary After Each Set

### Current State
- **Lines 1511–1527**: When a set completes, checks if match is over. If match over → `navigate("matchComplete")`. If match continues → `setSetupStep("lineup"); navigate("setup")`. **No intermediate summary.**
- **Lines 2591–2868**: `matchComplete` screen has full stats, MVP display, PIR rankings — the template to reuse.
- **Lines 4075–4149**: Timeout suggestion generation logic — needs to be extracted for reuse.
- Between-set formula data exists at lines 5268–5330 but only shows in the formula tab during live play.

### Changes Needed
1. **New screen**: `"setSummary"` — displayed between sets when match continues.

2. **Modify set completion flow** (line 1523–1526): Change:
   ```js
   // OLD:
   setSetupStep("lineup");
   navigate("setup");
   // NEW:
   navigate("setSummary");
   ```

3. **Extract suggestion logic**: Factor lines 4076–4149 into a reusable function:
   ```js
   const generateSuggestions = (set, match, formulaAnalysis) => {
     const sug = [];
     // ... all existing suggestion logic moved here ...
     return sug;
   };
   ```
   Called from both the timeout overlay and the set summary screen.

4. **setSummary screen render** (insert around line 2870, after matchComplete block):
   - **Set result banner**: WON/LOST with score, color-coded (green/red)
   - **Match score context**: "Leading 2-0", "Tied 1-1", etc.
   - **Key stats grid**: Kills, Errors, Hit%, Aces, Serve Errors, Digs, Blocks — reuse matchComplete pattern
   - **Top 3 PIR players**: Reuse PIR ranking pattern from lines 2836–2851
   - **AI suggestions for next set**: Call `generateSuggestions()` with the just-completed set data
   - **"Set Up Next Set →" button**: Calls `setSetupStep("lineup"); navigate("setup");`

5. **IMPORTANT — Hide back button**: The setSummary screen must NOT show a back button. The set is over — going back to the court makes no sense. Only show "Set Up Next Set →".

6. **Final set behavior unchanged**: The `if (setsWon >= setsToWin || setsLost >= setsToWin)` branch (line 1511) still goes to `matchComplete` as before.

### Files Affected
- `index.html` lines 1523–1526 (change navigation target), ~2870 (new screen block), 4076–4149 (extract to function)

### Dependencies/Risks
- `currentSetIdx` still points to the just-completed set on the setSummary screen — correct for displaying that set's stats.
- Only triggers when match continues (the `else` branch at line 1523) — final set still goes to matchComplete.

---

## Item 10: Back Button Fix

### Current State
- **Lines 586–604**: Custom navigation system:
  - `navigate(s)` (line 588) — pushes current screen to `navStackRef`, sets new screen (preserves history)
  - `setScreen(s)` (line 601) — **wipes entire navStackRef** to `[]`, then sets screen (destroys all history)
  - `goBack()` (line 592) — pops from stack, falls back to "home" if empty
- `setScreen()` called at 4 locations:
  - **Line 1032**: After saving team in editTeam → `setScreen("home")` — wipes stack unnecessarily
  - **Line 1876**: Room exit → `setScreen("home")` — **correct** (full state reset)
  - **Line 1937**: Team not found error → `setScreen("home")` — acceptable (error fallback)
  - **Line 2646**: matchComplete HOME button → `setScreen("teamHome")` — wipes stack

### Changes Needed
1. **Rename `setScreen` to `resetNavTo`** for clarity (lines 601–604). Update all 4 call sites to use the new name.

2. **Fix line 1032** (editTeam save): Replace `resetNavTo("home")` with `goBack()` — returns to wherever user came from (typically teamHome).

3. **Add `navigateBackTo(target)` helper** (after line 599):
   ```js
   const navigateBackTo = (target) => {
     const idx = navStackRef.current.lastIndexOf(target);
     if (idx >= 0) {
       navStackRef.current = navStackRef.current.slice(0, idx);
     } else {
       navStackRef.current = [];
     }
     setScreenRaw(target);
   };
   ```

4. **Fix line 2646** (matchComplete HOME): Replace `resetNavTo("teamHome")` with `navigateBackTo("teamHome")` — preserves ability to go back to home from teamHome.

5. **Add duplicate guard** in `navigate()` (line 588):
   ```js
   const navigate = (s) => {
     if (s === screen) return; // prevent duplicate pushes
     navStackRef.current.push(screen);
     setScreenRaw(s);
   };
   ```

### Files Affected
- `index.html` lines 588–604 (navigation functions), 1032, 1876, 1937, 2646 (call sites)

### Dependencies/Risks
- Low risk. The `goBack()` fallback to "home" still provides a safety net if stack is empty.
- Test all screen transitions to verify back behavior after changes.

---

## Item 11: Scouting Tool Buildout

### Current State
- **Standalone Scout** (screen="scout", lines 3357–3678): Full-screen scout tool. Stores shots in `scoutSessions` state, persisted to Firebase `rooms/${roomCode}/teamScout/${activeTeamId}`. Session object: `{ id, name, shots: [...], created, teamId }`. No match/set association. No passing grade tracking. Shots rendered as colored SVG circles.
- **In-Match Scout Tab** (tab="scout", lines 5444–5601): Stores shots on `set.shots` via `updateCurrentSet()`. Separate `courtSvg` implementation (line 5471). Also renders dots.
- Shot data model: `{ id, originZone, outcome, x, y, timestamp }`.
- Scout session creation (teamHome, lines 2283–2342): Creates session with name, no match/set linking.

### Changes Needed

**(a) Auto-save with per-match/per-set keys:**

Add optional `matchId` and `setNumber` fields to standalone scout session creation (~line 2294):
```js
const session = {
  id: Date.now().toString(),
  name: scoutNameInput.trim(),
  shots: [],
  passingGrades: [],   // NEW for item 11d
  created: new Date().toISOString().split("T")[0],
  teamId: activeTeamId,
  matchId: selectedMatchId || null,   // NEW
  setNumber: selectedSetNumber || null // NEW
};
```

Add a match picker dropdown in teamHome scout section (~line 2286) before creating session:
```js
<select value={selectedMatchId} onChange={...}>
  <option value="">No match link</option>
  {teamMatches.map(m => <option key={m.id} value={m.id}>{m.opponent} - {m.date}</option>)}
</select>
```

In-match scout tab already persists per-set via `set.shots` — just ensure scout UI state (`scoutPending`, view mode) is included in Item 3's live session persistence.

**(b) Full-page scout view:**

Add a dedicated full-page view option for the standalone scout. Currently the standalone scout IS a full screen, but the court is constrained to `COURT_H = 200` (line 3417). Add a "Full Court" toggle that increases court dimensions:
```js
const [scoutFullView, setScoutFullView] = useState(false);
const COURT_H = scoutFullView ? 400 : 200;
```
Add a toggle button in the scout header: "⤢ Expand" / "⤡ Compact"

**(c) Shot chart with opponent jersey numbers instead of dots:**

The numbers on the shot chart must be the **opponent player's real jersey number** — NOT sequential counts. This lets coaches see which opposing hitter is attacking from where.

- **Extend shot data model**: Add `playerNumber` field:
  ```js
  { id, originZone, outcome, x, y, timestamp, playerNumber: string|null }
  ```

- **Player number input in scout UI** (standalone, ~line 3368): Add before the court area:
  - A small text input with `inputMode="numeric"` labeled "Hitter #"
  - The number is **sticky** — carries forward as default until changed (coaches chart multiple swings from same hitter in a row)
  - A row of recently-used opponent numbers as quick-tap buttons (derived from `shots.map(s => s.playerNumber).filter(Boolean)`, deduplicated, most recent first)

- **New state**:
  ```js
  const [scoutPlayerNum, setScoutPlayerNum] = useState("");  // sticky hitter number
  const [scoutNumbered, setScoutNumbered] = useState(false); // dot vs number toggle
  ```

- **Modify shot creation** (standalone line 3379, in-match line 5459): Include `playerNumber: scoutPlayerNum || null` in shot object.

- **Modify `courtSvg`** rendering (standalone line 3419, in-match line 5471): When `scoutNumbered` is true, render `<text>` elements showing `shot.playerNumber` (colored by outcome) instead of `<circle>` dots. Fall back to a dot if `playerNumber` is null/empty:
  ```js
  {inBounds.map((s, i) => (
    scoutNumbered && s.playerNumber ? (
      <text key={s.id || i} x={PAD + s.x * W / 100} y={PAD + s.y * H / 100 + 3}
        fill={outcomeColor[s.outcome]} fontSize="9" fontWeight="900"
        textAnchor="middle">{s.playerNumber}</text>
    ) : (
      <circle key={s.id || i} cx={PAD + s.x * W / 100} cy={PAD + s.y * H / 100}
        r={5} fill={outcomeColor[s.outcome]} opacity={0.9} stroke="#000" strokeWidth="0.5" />
    )
  ))}
  ```

- **Toggle button**: Add "# / ●" toggle in scout view controls for both standalone and in-match.

- **Must update BOTH courtSvg implementations** (standalone line 3419 and in-match line 5471).

**(d) 0–3 Passing Grade UI for opponent players in STANDALONE scout:**

This tracks opponent passing tied to **specific opponent jersey numbers**. This is DIFFERENT from:
1. Our team's in-match passing (already exists as per-player reception events with quality 0–3)
2. In-match broad opponent passing (not player-specific)

**Data model for passing entries:**
```js
{ id, grade: 0|1|2|3, playerNumber: string (REQUIRED), timestamp, zone: 1-6 (optional) }
```

**Uses the same "sticky number" UX** from the shot chart — coach enters opponent jersey number via the same `scoutPlayerNum` input, then taps grade buttons.

**Store on standalone scout session**: `session.passingGrades = [...]`
Do NOT add `opponentPassing` to the in-match set object — that is not part of this change.

**New "Passing" view tab** alongside Input/Heatmap in the standalone scout (line ~3500 where view tabs are rendered):

**UI layout:**
- Sticky "Player #" input (shared with shot chart, reuses `scoutPlayerNum`)
- 4 large grade buttons: 0 (red `#991b1b`), 1 (orange `#92400e`), 2 (yellow `#854d0e`), 3 (green `#166534`)
- Optional zone selector (which zone received)
- Running summary grouped by opponent player number:
  ```
  #7:  Avg 2.15  |  Tot 12  |  ███░░ (3s: 4, 2s: 5, 1s: 2, 0s: 1)
  #14: Avg 1.50  |  Tot 8   |  ██░░░ (3s: 1, 2s: 3, 1s: 3, 0s: 1)
  ```
- Undo last entry button

### Files Affected
- `index.html` lines 2283–2342 (session creation + match picker), 3357–3678 (standalone scout screen + courtSvg), 5444–5601 (in-match scout tab + courtSvg), new state variables near line 654

### Dependencies/Risks
- Two separate `courtSvg` implementations must both get the numbered rendering update.
- The `scoutPlayerNum` state is shared between shot charting and passing grade entry in the standalone scout.
- Standalone scout passing data is stored in the scout session, NOT on the in-match set object.
- Passing data here is for opponent scouting. Our team's PASS column (Item 13) pulls from our reception events.

---

## Item 12: Tournament/Event Assignment

### Current State
- Match object (line 532–540) has an optional `tournament` string field. Line 2669 displays it as plain text on matchComplete.
- No tournament entity exists. No CRUD. No filtering.
- Matches listed on teamHome (lines 2345–2465), filtered by `teamId` only.
- Firebase path for matches: `rooms/${roomCode}/matches`.

### Changes Needed

**1. Tournament data model:**
```js
{
  id: string (Date.now().toString()),
  name: string,
  startDate: string (YYYY-MM-DD),
  endDate: string (YYYY-MM-DD),
  location: string (optional, "" if not set),
  teamId: string,
  created: string (ISO date)
}
```

**2. Firebase path**: `rooms/${roomCode}/tournaments/${activeTeamId}`

**3. New state** (near line 618):
```js
const [tournaments, setTournaments] = useState([]);
const [matchTournamentFilter, setMatchTournamentFilter] = useState(null); // tournament ID or null
```

**4. Firebase listener** (follow existing pattern at lines 932–964):
```js
useEffect(() => {
  if (!roomCode || !activeTeamId) { setTournaments([]); return; }
  const ref = fbDb.ref(`rooms/${roomCode}/tournaments/${activeTeamId}`);
  const handler = ref.on('value', (snap) => {
    setTournaments(fbArr(snap.val()));
  });
  return () => ref.off('value', handler);
}, [roomCode, activeTeamId]);

const saveTournaments = (t) => {
  setTournaments(t);
  if (!roomCode || !activeTeamId) return;
  fbDb.ref(`rooms/${roomCode}/tournaments/${activeTeamId}`).set(t);
};
```

**5. Tournament CRUD UI** (add collapsible section on teamHome BEFORE Match History, ~line 2345):
- "Tournaments" header with expand/collapse toggle
- "New Tournament" button → inline form (name required, start date, end date, location optional)
- List of tournaments with edit/delete buttons
- Each card shows: name, date range (if set), location (if set), match count (computed from `teamMatches.filter(m => m.tournamentId === t.id).length`)

**6. Match-to-tournament assignment** (on match creation setup options, ~line 3183):
- Add tournament picker dropdown before "Start Set" button:
  ```js
  <select value={currentMatch?.tournamentId || ""} onChange={(e) => {
    setCurrentMatch(p => ({...p, tournamentId: e.target.value || null}));
  }}>
    <option value="">No Tournament</option>
    {tournaments.map(t => <option key={t.id} value={t.id}>{t.name}</option>)}
  </select>
  ```
- Store as `match.tournamentId` (ID reference, not string name).
- Keep backwards-compatible display of legacy `match.tournament` string field.

**7. Filter on match list** (modify lines 2345–2465):
- Add filter bar above match list:
  ```js
  <select value={matchTournamentFilter || ""} onChange={(e) => setMatchTournamentFilter(e.target.value || null)}>
    <option value="">All Matches</option>
    {tournaments.map(t => <option key={t.id} value={t.id}>{t.name}</option>)}
  </select>
  ```
- Apply filter:
  ```js
  const displayMatches = matchTournamentFilter
    ? teamMatches.filter(m => m.tournamentId === matchTournamentFilter)
    : teamMatches;
  ```
- Use `displayMatches` instead of `teamMatches` in the list rendering.
- Show tournament name badge on each match card:
  ```js
  {m.tournamentId && (() => {
    const t = tournaments.find(x => x.id === m.tournamentId);
    return t ? <span style={{ fontSize: 9, opacity: 0.5 }}>{t.name}</span> : null;
  })()}
  ```

### Files Affected
- `index.html` lines 618 (new state), 932–964 (Firebase listener pattern), 2345–2465 (match list + filter), 3183–3354 (setup options + tournament picker), match object creation

### Dependencies/Risks
- Backwards compatibility: matches with old `tournament` string field still display it as fallback text.
- Tournament filter only affects display list — season stats computation (lines 1966–1984) uses `teamMatches` directly, so filtering is safe.
- `tournamentId` on match carries across sets (stored on match, not set).

---

## Item 13: Stats Table Changes — Remove CVR, Merge into DIG, Add PASS

### Current State

**Stats tables:**
| Table | Lines | CVR Column | DIG Column |
|---|---|---|---|
| Season Stats | 2216–2277 | `CV` at ~2228 | `DIG` at ~2227 |
| Match Complete Stats | 4992–5059 | `CV` at ~5005 | `D` at ~5004 |
| PIR Breakdown | 5078–5108 | `CVR` category at ~5084 | `DEF` category at ~5083 |

- `FORMULA_STATS` array (line 413): includes `{ key: "cvr", label: "Covers", scoring: false }`
- `IMPACT_WEIGHTS` (line 426): `cvr: 0.25`, `dig: 0.35` (separate weights)
- Passing Quality Table already exists (lines 5126–5158) showing detailed pass breakdown.
- Reception events already store `ev.quality` (0–3) at lines 4681–4700.
- `getViewPassingStats(playerId)` function (~line 4887) already computes `simpleAvg` from reception events.
- Season stats computation loop (lines 1966–1984) currently does NOT track reception quality.

### Changes Needed

**(a) Remove CVR column from all stats table headers and cells:**

Season Stats Table:
- Remove `<th>CV</th>` header (~line 2228)
- Remove `<td>{cvr}</td>` cell (~line 2269)

Match Complete Stats Table:
- Remove `<th>CV</th>` header (~line 5005)
- Remove `<td>{cvr}</td>` per-player cell (~line 5032)
- Remove `<td>{vTotals.cvr}</td>` team totals cell (~line 5052)

**(b) Merge CVR into DIG for DISPLAY ONLY:**

Season Stats data computation (~line 2248):
```js
// Change:
dig: (stats.dig || 0) / d, cvr: (stats.cvr || 0) / d,
// To:
dig: ((stats.dig || 0) + (stats.cvr || 0)) / d,
```

Match Complete per-player (~line 5031):
```js
// Change:
<td>{ps.dig || 0}</td>
// To:
<td>{(ps.dig || 0) + (ps.cvr || 0)}</td>
```

Match Complete team totals (~line 5051):
```js
// Change:
<td>{vTotals.dig || 0}</td>
// To:
<td>{(vTotals.dig || 0) + (vTotals.cvr || 0)}</td>
```

**CRITICAL: Display-only merge.** Do NOT change:
- Event recording (`cvr` events still recorded as `"cvr"`)
- PIR weights (`cvr: 0.25` stays separate from `dig: 0.35`)
- Winning Formula variables (`cover_rate` remains independent)
- `FORMULA_STATS` array — do NOT remove `cvr` entry, it may be referenced by formula calculations

**(c) Add PASS column showing passing average (0–3.00 scale):**

This pulls from OUR TEAM'S existing reception event quality data (`ev.quality` 0–3). No new event type is needed.

**Season Stats** — add passing tracking to the computation loop (~lines 1966–1984):
```js
// New accumulator alongside existing stats tracking:
const playerSeasonPassing = {}; // { canonicalKey: { total: 0, qualitySum: 0 } }

// Inside the set event loop:
if (ev.stat === "reception" && ev.quality !== undefined) {
  const key = pidToCanonical[ev.player];
  if (key) {
    if (!playerSeasonPassing[key]) playerSeasonPassing[key] = { total: 0, qualitySum: 0 };
    playerSeasonPassing[key].total++;
    playerSeasonPassing[key].qualitySum += ev.quality;
  }
}
```

Add `<th>PASS</th>` header after DIG. Cell rendering:
```js
const passing = playerSeasonPassing[canonKey];
const passAvg = passing && passing.total > 0 ? passing.qualitySum / passing.total : null;

<td style={{
  ...S.std,
  color: passAvg !== null
    ? (passAvg >= 2.0 ? "#22c55e" : passAvg >= 1.5 ? "#007bff" : "#ef4444")
    : "#6b7280"
}}>
  {passAvg !== null ? passAvg.toFixed(2) : "—"}
</td>
```

**Match Complete Stats** — use existing `getViewPassingStats(playerId)`:
```js
// Add <th>PASS</th> after D header
// Per-player cell:
{(() => {
  const ps2 = getViewPassingStats(p.id);
  const avg = ps2?.simpleAvg;
  return <td style={{
    ...S.std,
    color: avg != null ? (avg >= 2.0 ? "#22c55e" : avg >= 1.5 ? "#007bff" : "#ef4444") : "#6b7280"
  }}>
    {avg != null ? avg.toFixed(2) : "—"}
  </td>;
})()}
```

Team totals row:
```js
{(() => {
  let totalPasses = 0, totalQuality = 0;
  for (const p of rosterPlayers) {
    const ps2 = getViewPassingStats(p.id);
    if (ps2 && ps2.total > 0) { totalPasses += ps2.total; totalQuality += ps2.total * ps2.simpleAvg; }
  }
  const teamAvg = totalPasses > 0 ? totalQuality / totalPasses : null;
  return <td style={{ ...S.std, fontWeight: 800 }}>{teamAvg !== null ? teamAvg.toFixed(2) : "—"}</td>;
})()}
```

**(d) PIR Breakdown — merge cover into defense display:**

In all PIR category display locations (lines 5078–5086, 2677–2685, and timeout overlay):
- Remove "CVR" category bar
- Add cover value to defense value for display:
  ```js
  const defVal = (breakdown.defense || 0) + (breakdown.cover || 0);
  ```
- Apply in all 3 PIR breakdown locations (match complete MVP, match complete player list, timeout overlay)

### Files Affected
- `index.html` lines 1966–1984 (season stats loop — add passing tracking), 2216–2277 (season stats table), 4992–5059 (match complete stats table), 5078–5108 (PIR breakdown), 2677–2685 (MVP breakdown)

### Dependencies/Risks
- **Critical**: Must NOT merge CVR/DIG at the recording/calculation level — only at display. They have different PIR weights and Winning Formula dependencies.
- Season stats loop needs new code to track passing quality (not currently computed there).
- The existing Passing Quality Table (lines 5126–5158) remains unchanged — it shows the detailed breakdown while the new PASS column shows the condensed average.
- `FORMULA_STATS` array: do NOT remove `cvr` entry.

---

## Item 14: Score Pulse Effect

### Current State
- **Lines 1322–1343**: `scorePoint(forUs)` increments score via `updateCurrentSet()`. No visual feedback beyond the number changing.
- **Lines 4311, 4325**: Score rendered as `<span style={S.bigScore}>` — static `fontSize: 30, fontWeight: 800` (line 5932). No animation.
- Scoring also happens implicitly through rally resolution (kills, aces, opponent errors call `scorePoint` internally).
- No CSS animations or keyframes exist in the app currently.

### Changes Needed

1. **New state and ref** (near line 654):
   ```js
   const [scorePulse, setScorePulse] = useState(null); // "us" | "them" | null
   const scorePulseTimerRef = useRef(null);
   ```

2. **Trigger pulse in `scorePoint()`** (line 1322):
   ```js
   const scorePoint = (forUs) => {
     updateCurrentSet((set) => { /* ...existing logic unchanged... */ });
     // Fire pulse (gated behind score edit modal)
     if (!showScoreEdit) {
       clearTimeout(scorePulseTimerRef.current);
       setScorePulse(forUs ? "us" : "them");
       scorePulseTimerRef.current = setTimeout(() => setScorePulse(null), 600);
     }
   };
   ```

3. **Inject CSS keyframes** (add `<style>` tag in the HTML `<head>` section, before the React root):
   ```css
   @keyframes scorePulse {
     0%   { transform: scale(1);   color: inherit; }
     30%  { transform: scale(1.6); color: #22c55e; }
     100% { transform: scale(1);   color: inherit; }
   }
   @keyframes scorePulseRed {
     0%   { transform: scale(1);   color: inherit; }
     30%  { transform: scale(1.6); color: #ef4444; }
     100% { transform: scale(1);   color: inherit; }
   }
   @keyframes scorePulseGrey {
     0%   { transform: scale(1);   color: inherit; }
     30%  { transform: scale(1.3); color: #9ca3af; }
     100% { transform: scale(1);   color: inherit; }
   }
   ```
   Green pulse for us, red pulse for them, grey pulse for undo.

4. **Apply animation on score spans** (lines 4311, 4325):
   ```js
   // Line 4311 (our score):
   <span style={{
     ...S.bigScore,
     display: "inline-block", // required for transform to work on span
     animation: scorePulse === "us" ? "scorePulse 0.6s ease-out" : "none",
   }}>{set?.ourScore || 0}</span>

   // Line 4325 (their score):
   <span style={{
     ...S.bigScore,
     display: "inline-block",
     animation: scorePulse === "them" ? "scorePulseRed 0.6s ease-out" : "none",
   }}>{set?.theirScore || 0}</span>
   ```

5. **Trigger on undo** (in `undoLastAction()`, ~line 1422): If the undone event had `_undoScore`, fire a grey pulse on the affected side:
   ```js
   if (last._undoScore) {
     const wasOurPoint = last._undoScore.ourScore < (getCurrentSet()?.ourScore || 0);
     clearTimeout(scorePulseTimerRef.current);
     setScorePulse(wasOurPoint ? "us" : "them");
     // Use grey animation variant via a separate state or CSS class
     scorePulseTimerRef.current = setTimeout(() => setScorePulse(null), 600);
   }
   ```

6. **Use ref to clear previous setTimeout**: The `scorePulseTimerRef` prevents overlapping animations on rapid undo taps. `clearTimeout` before each new `setTimeout`.

### Files Affected
- `index.html` lines 654 (new state/ref), 1322–1343 (scorePoint), 4311/4325 (score spans), ~1422 (undo), HTML `<head>` (keyframes)

### Dependencies/Risks
- `display: inline-block` on the score span is required for CSS `transform: scale()` to work — verify it doesn't break the flex layout of `S.scoreBoard`. The parent is `S.teamCol` which uses flex — `inline-block` children work fine in flex.
- Pairs with Item 1: pulse is gated behind `!showScoreEdit` so tapping to edit doesn't fire the animation.
- `scorePulseTimerRef` prevents overlapping animations on rapid taps/undos.

---

## Implementation Order

Follow this sequence exactly:

1. **Item 10** (Back Button Fix) — smallest, most isolated, affects all navigation
2. **Item 2** (Rotate Button) — small, self-contained
3. **Item 14** (Score Pulse Effect) — small, pairs with Item 1
4. **Item 1** (Editable Scoreboard) — gates pulse behind edit mode
5. **Item 5** (Play-To Score) — foundation for Item 4's settings panel
6. **Item 4** (In-Match Settings Icon) — embeds play-to UI + other settings
7. **Item 6** (Uniform Offense Buttons) — surgical, one block replacement
8. **Item 7** (Player Names Not Numbers) — display-only sweep
9. **Item 8** (Softer AI Language) — string-only changes
10. **Item 13** (Stats Table Changes) — display changes + new PASS column from existing data
11. **Item 9** (Set Summary After Each Set) — new screen + refactor suggestion logic
12. **Item 12** (Tournament/Event Assignment) — new data model + CRUD + filtering
13. **Item 11** (Scouting Tool Buildout) — largest scope, multiple sub-features
14. **Item 3** (Full Data Persistence) — LAST, after all new state is finalized

---

## Verification Plan

1. **Navigation (Items 10, 9)**: Test all screen transitions — home → teamHome → setup → match → setSummary → setup → match → matchComplete → teamHome → home. Verify back button returns to previous screen at each step. Verify setSummary has no back button.
2. **Live Match Flow (Items 1, 2, 4, 5, 6, 14)**: Start a match. Verify score pulse (green for us, red for them) fires on every point. Tap score to open edit modal — verify no pulse fires. Edit score manually — verify no rotation change. Open settings gear — change play-to, serve/receive, libero assignment. Rotate from subs page. Verify all players get full offense buttons.
3. **Persistence (Item 3)**: Start match, record several rallies, force-close app, reopen. Verify all state restores: tab, rally phase, undo stack, set index, selected player. Verify rapid scoring produces a single batched write, not multiple partial writes.
4. **Display (Items 7, 8, 13)**: Check all stats tables for correct columns (no CVR, DIG includes covers, PASS shows averages with color coding). Verify "—" for players with no passes. Verify player names appear as primary identifiers everywhere. Trigger timeout overlay and verify softer language.
5. **Set Summary (Item 9)**: Complete a set in a best-of-3 match (not the final set). Verify summary screen appears with stats, PIR, and suggestions. Verify "Set Up Next Set →" button works. Verify final set still goes to matchComplete.
6. **Play-To (Item 5)**: Set play-to 15 for set 1. Complete the set. Verify the new set does NOT inherit play-to 15 — it defaults back to 25. Verify AI suggestions use relative point calculations.
7. **Tournament (Item 12)**: Create a tournament. Create matches assigned to it. Verify filter on match list shows only that tournament's matches. Verify legacy matches with string `tournament` field still display.
8. **Scout (Item 11)**: Enter opponent jersey numbers. Verify sticky number behavior. Toggle between dot and number view. Enter passing grades for multiple opponent players. Verify per-player summary displays correctly. Verify data persists in scout session.
