# Airbike Protocol

A single-file, offline-capable 52-week airbike training tracker built from
research-backed protocols (Helgerud 4×4, Seiler polarized, threshold, over-unders).
Vanilla JS, no dependencies, no build step, no network requests after first load.
Ships as one `index.html` file to GitHub Pages.

## Running it

Open `index.html` in any modern browser. That's the whole app.

- First load over the network caches the page in the browser.
- Subsequent loads work fully offline — no service worker needed; the browser
  cache carries the single file.
- On mobile, "Add to Home Screen" installs it as a PWA (a minimal manifest is
  embedded as a data URL).

## How the adaptive progression works

Each week has a **base prescription** generated from the 4-phase template
(Phase 1: Foundation / Phase 2: Aerobic Development / Phase 3: Polarized /
Phase 4: Peak). Those templates encode the week-by-week progression from the
original protocol (e.g. Z2 duration +2 min/week in Phase 1, 4×4 RPM +1/week in
Phase 2, etc.).

On top of the base, a small **progression engine** runs at page load. It looks
at the previous week's logs and may apply one of three adjustments to the
current week:

| Trigger | Action | Why |
|---|---|---|
| ≥ 3 sessions missed last week | Banner: "suggest repeating this week" | Insufficient stimulus — don't stack new load on incomplete work |
| Avg RPE ≥ 9 or > 1 session missed | Flatten: no RPM bump this week | Fatigue / under-recovery — hold the ceiling, let adaptation catch up |
| Z2 targets hit on ≥ 75% of Z2 sessions (avgRPM ≥ 90% of target, avgHR ≤ target) | Bump: Z2 ceiling +1 RPM, threshold/VO₂max ceiling +2 RPM | On-target performance → ready for more load |
| Otherwise | Normal built-in progression | No intervention needed |

**Bumps are cumulative.** Every successful bump permanently raises the RPM
ceiling for subsequent weeks on top of the base progression. So a user who
consistently hits targets will see their RPM targets climb faster than the
protocol's baked-in weekly increments.

**Repeat suggestions are advisory.** The tracker shows a banner but doesn't
shift the calendar — the user decides whether to actually redo the week.

**Tuning the engine.** All thresholds live in one constant at the top of the
script block in `index.html`:

```js
const TUNING = {
  MISSED_REPEAT_THRESHOLD: 3,
  MISSED_FLATTEN_THRESHOLD: 1,
  HIGH_RPE_THRESHOLD: 9,
  Z2_RPM_HIT_PCT: 0.90,
  Z2_HR_MAX_PCT: 1.02,
  ON_TARGET_FRACTION: 0.75,
  RPM_BUMP_Z2: 1,
  RPM_BUMP_THRESHOLD: 2
};
```

Change any value, commit, push — the engine uses the new thresholds on next
page load. The `computeAdjustment()` function right below it is ~40 lines and
easy to edit if you want to add new rules (e.g. monthly bodyweight-based
load adjustments).

## Deload weeks & time trials

The plan automatically schedules deload weeks at W6, W12, W21, W27, W28, W32,
W36, W40, W44, W48, W52. Deloads cut volume by 40%. On W6, W12, W21, W27, W32,
and W52, Saturday becomes a **10-minute airbike time trial** — a max-effort
test that feeds the VO₂max trend chart.

VO₂max estimate formula (documented on the TT form):

```
VO₂max ≈ (cals × 24.5) / bodyweight_kg
```

Derived from `cals → kJ → L O₂ → ml/kg/min`, assuming the 10-min max effort
averages ~85% of VO₂max. Accuracy ±15%. Users can override with a measured
value in settings or on the TT form itself.

## Where data is stored

Everything lives in **`localStorage`** under a single root key `airbike_v1`.
One JSON blob with four sections:

```js
{
  version: 1,
  settings: { startDate, hrMax, bodyweightLb, units, vo2maxManual },
  logs: { "YYYY-MM-DD": { weekIndex, dayIndex, completed, durationMin,
                          avgRpm, avgHr, peakHr, rpe, notes, ts } },
  bodyweightLog: { "YYYY-MM-DD": number },
  timeTrials: { "<weekIndex>": { cals, finalHr, bodyweight, vo2max,
                                  manualVo2, source, ts } }
}
```

- No backend. No analytics. No network requests. Everything stays on the device.
- Dates are ISO strings (`YYYY-MM-DD`). The current week is computed from
  `(today − startDate) / 7`, clamped 1..52.
- The plan itself is not stored — it's regenerated from the phase template +
  progression engine on every page load. Only user-contributed data persists.

## Backing up your data

**Export:** Open Settings (gear icon) or the Progress view and click
**Export JSON**. You'll get a file named `airbike-backup-YYYY-MM-DD.json`
with everything in `airbike_v1`. Save it anywhere safe.

**Import:** Open Settings → **Import backup** → pick a previously-exported
JSON file. You'll get a confirmation prompt; importing *replaces* all
current data. Useful for moving between devices, restoring after a browser
reset, or rolling back an accidental edit.

**How often to back up:** localStorage is ephemeral. Chrome's "Clear browsing
data" nukes it. Private/incognito tabs nuke it. "Storage almost full" nukes it.
**Export monthly, minimum.** Stick the file in Dropbox or iCloud Drive.

## HRmax, bodyweight, start date

Settings are editable any time:

- **HRmax** — defaults to 180 (220 − 40 for a 40yo). If you have a
  field-measured max from a real ramp test, put that in instead. All HR zones
  recalculate live.
- **Bodyweight** — weekly bodyweight prompt shows on Monday of each training
  week. Used for VO₂max calculation in time trials.
- **Start date** — defaults to "next Monday" on first load. Change this to
  retroactively start the program on any date. The plan shifts with it.
- **Units** — `lb` or `kg`. Stored internally as lb; displayed in the chosen
  unit.

## Deploying updates

This is a plain static file. GitHub Pages just serves whatever's at
`index.html` on the `main` branch of your repo.

```bash
# Edit index.html (e.g. tweak TUNING constants, add a new interval variant)
git add index.html
git commit -m "Tune adaptive progression thresholds"
git push
```

GitHub Pages picks up the change within a minute or two. Open the site on
your phone — a hard refresh (pull down on mobile Safari) will grab the new
version. Since there's no service worker, cache-busting is the browser's
normal refresh behavior.

**Editing the plan.** The phase template lives in `PHASES_INFO` + the four
`buildPhase1/2/3/4` functions. Each phase builder takes `(info, bump)` and
returns an array of 7 day objects. Change durations, RPM targets, interval
structures there — the rest of the app will consume the new shape
automatically.

**Editing the rules.** The `TUNING` constant and `computeAdjustment()` live
near the top of the script block. Rules are evaluated top-to-bottom and
return on the first match, so order matters for prioritization.

**Migrations.** If you change the data model, bump `VERSION` and add a branch
to the `migrate()` function near the top of the state layer. The app calls
`migrate()` on every load, so old backups stay importable.

## File layout

```
airbike-protocol/
├── index.html    # the entire app — HTML, CSS, JS in one file
└── README.md     # this file
```

That's it. No `package.json`, no `node_modules`, no bundler, no framework.

## Credits

Training protocols based on:

- Helgerud et al. (2007) — Norwegian 4×4 VO₂max intervals
- Seiler (2010) — Polarized training distribution
- Stöggl & Sperlich (2014) — Polarized vs. threshold comparison
- Bacon et al. (2013) — Interval training meta-analysis
- Wisløff et al. (2007) — Aerobic interval training in heart failure patients
