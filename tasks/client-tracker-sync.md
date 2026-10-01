# Task: App data → client trackers

**Status:** scoping only. No tracker structure has been touched or will be, until Justin confirms the format.

## Objective
Pipe data captured by the app into the client trackers, starting with food macros, then extending to the other HealthKit-sourced metrics over time.

## Phase 1 — MyFitPal macros → tracker (current focus)
Goal: get Dietary Energy, Protein, Carbohydrates, Total Fat from MyFitPal into the client tracker.

### Carried over from the HealthKit permission audit
- The app already requests HealthKit read access to all four nutrition types — that's why they show as granted on-device.
- But nothing in the app's sync code reads them. The three arrays that drive what actually gets pulled (`HK_TYPES` for ongoing daily sync, `HK_AGG`/`HK_SAMPLE` for the one-off baseline import) don't include any of the four, and there's no `Health.readSamples`/`Health.queryAggregated` call referencing dietary energy, protein, carbs, or fat anywhere in that file.
- So right now: permission granted, nothing downstream uses it. The plumbing to add is on the read side, not a permissions fix.

### Blockers before any code gets written
1. **Source repo not attached here.** That analysis was done against the app's `src/main.js` (HK_TYPES/HK_AGG/HK_SAMPLE) — that file isn't in `nxs-capture` (this repo is just the single-page Recovery Capture tool, `index.html` only, no `src/`). Need the actual nxs-capture-app repo attached before implementation starts.
2. **MyFitPal data path unconfirmed.** Need to confirm whether MyFitPal writes these four macro types into Apple Health itself (in which case this is the same HK_TYPES-array change as every other metric), or whether it needs a separate MyFitPal-side export/API instead.
3. **Tracker structure — explicitly not decided yet.** Justin has a structure he plans to implement. No schema, sheet, table, or field names for "the tracker" should be assumed or built against until he gives the go-ahead. This file intentionally stops short of that.

### Next steps once unblocked
1. Attach/clone the real nxs-capture-app source.
2. Confirm the MyFitPal → HealthKit (or MyFitPal → API) path for the four macro types.
3. Add the four types to `HK_TYPES` (daily sync) and `HK_AGG`/`HK_SAMPLE` (baseline import) as appropriate, with field mapping.
4. **Stop and confirm tracker structure with Justin** before writing anything that touches the tracker.
5. Build the write path into the tracker per the confirmed structure.

## Later phases (not started)
Extend the same pattern to any other metrics clients need in the tracker beyond macros (steps, heart rate, sleep, weight, workouts are already read by the app for other purposes — sleep, for example, is read but only to prefill the local entry form, not pushed anywhere).
