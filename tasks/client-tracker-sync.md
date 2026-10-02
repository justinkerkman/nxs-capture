# Task: App data → client trackers

**Status (updated 2026-10-02, after the Krane Burke meeting):** the HealthKit→app leg is already built and confirmed working. The gap is one specific missing automation step downstream, on the Mac/Supabase side — not an app-code problem any more. Tracker display structure is still explicitly undecided; nothing below assumes a schema.

## Objective
Pipe data captured by the app into the client trackers, starting with food macros, then extending to the other HealthKit-sourced metrics over time.

## Phase 1 — MyFitnessPal macros → tracker: current state

**This section supersedes the original scoping below it.** The premise "MyFitnessPal export is dead, need another path" was wrong — that finding was about MyFitnessPal's Premium bulk CSV export, a different, unrelated feature. The live mechanism is MyFitnessPal's own free "sync to Apple Health" toggle, which writes the four macro types straight into HealthKit. Confirmed working end to end as far as `raw_payloads`.

### Confirmed working (checked against the real source, `~/nxs-capture-app/src/main.js` on Justin's Mac — not this repo)
- `HK_TYPES` (incremental "Sync Now"/"Connect") and `HK_AGG` (one-off "Import full history") both already carry `dietary_energy`/`protein`/`carbohydrates`/`total_fat`, tagged `nut:true`. This is "build 34/35" in the app's own comments — already shipped, not something to build.
- Nutrition HealthKit authorization is requested separately from the base scopes (`HK_NUTRITION_SCOPES`) and is allowed to fail without breaking the rest of the sync. It depends on a **local, unversioned patch** to `node_modules/@capgo/capacitor-health` (`patches/@capgo+capacitor-health+8.10.2.patch`, applied via `patch-package`'s `postinstall` hook). Confirmed present and correctly wired as of this check — but a plain `npm install` without that hook silently reverts it with no visible symptom, so this is the one thing that could silently break again.
- Confirmed live for Krane Burke 2026-10-02: `raw_payloads` holds real, varying (non-placeholder) macro values for 24 Sep–1 Oct, captured right around that morning's sync.
- Background delivery (sync while the app's closed) uses HealthKit's own background-delivery entitlement (`NxsHealthBgPlugin.swift` + `com.apple.developer.healthkit.background-delivery`), not the generic `UIBackgroundModes`/`BGTaskScheduler` path — correctly configured. That's *why* the app never shows up in Settings → General → Background App Refresh; expected, not a bug, though the in-app copy pointing there is a little misleading.

### The actual gap — confirmed 2026-10-02
`raw_payloads` gets the nutrition data fine. Nothing moves it from there into `daily_metrics`, which is what the client-facing Apple-Health macro panel (`GitHub/index.html`, ~line 6217+ — separate from the hand-logged Food Log) actually reads.
- The edge function `apple-health-import` is **shadow-only** — writes to `daily_metrics_shadow`, which nothing reads or displays (confirmed in the tracker's own `tracker-todo.md`, filed under "worth knowing, no action").
- The script that does it for real, `client-tracker/tools/import-apple-health.py --from-raw --uuid <client> --push`, is **not on any schedule** — checked every launchd plist in `SCRIPTS/`, none call it. It has to be run by hand, per client, every time.
- Exact command for Krane handed to Justin 2026-10-02 (uuid `2b9d0f1c-367c-4905-a832-2e514d7ff646`); not run as of this writing — his call when.

### Two nutrition surfaces exist and never merge — worth a deliberate decision, not a default
1. `daily_metrics` (`protein_g`/`carbs_g`/`fat_g`/`dietary_energy_kcal`) — the Apple-Health-derived panel above.
2. `tracker_state.state.nutrition` — the Food Log / Nutrition tab clients log into by hand or via barcode scan. Separate table, separate bug history (the day-type-target placeholder bug that was hitting several clients' displayed macros, fixed 2026-09-30 — unrelated to this HealthKit path, see `tracker-todo.md`).
Nobody has decided whether these should ever show as one number to a client or coach. Not decided here either.

### Sync window limits, for anyone's expectations
- "Sync Now"/"Connect" only reads a rolling 7-day window (`HK_OVERLAP_MS`) — that's why the 2026-10-02 pull only covered 24 Sep–1 Oct, regardless of how much longer MyFitnessPal has actually been writing to HealthKit.
- "Import full history" (the one-off baseline button, `HK_AGG`) reads ~3 years back and, since build 35, carries the four macro types too. Not yet run for Krane — unknown whether anything predates 24 Sep for him until it is.

### Still not touched, on purpose
Tracker/display structure for how these macros should actually be laid out for coaches or clients. Justin is giving that separately. Nothing here assumes a schema, a layout, or that the two nutrition surfaces above should be reconciled.

## Administrative, unrelated to the above
The local commit to this repo (this file) still can't push to GitHub — Claude's GitHub access to `justinkerkman/nxs-capture` isn't authorized for this org (the GitHub App isn't installed, or needs reconnecting). Doesn't block anything else here.

## Later phases (not started)
Extend the same pattern to other metrics clients need in the tracker beyond macros (steps, heart rate, sleep, weight, workouts are already read by the app for other purposes — sleep, for example, is read but only to prefill the local entry form, not pushed anywhere), and decide whether the `import-apple-health.py --from-raw` step should be scheduled, run by hand regularly, or replaced by turning the edge function live.

---

## Original scoping (2026-10-02 morning, before the Krane meeting — kept for the record, superseded above)

Original premise: thought MyFitnessPal's export was dead-ended on Krane's free tier, and that `HK_TYPES`/`HK_AGG`/`HK_SAMPLE` didn't read the four nutrition types at all. Both turned out wrong once checked against the real app source and a live sync — see above.
