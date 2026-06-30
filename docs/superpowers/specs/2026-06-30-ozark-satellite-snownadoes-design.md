# 2026 Ozark Super Snownado — Satellite Snownadoes of the Outbreak

**Date:** 2026-06-30
**Scope:** Section K (`ozark` tab) of `snow-prediction-center/index.html` only.

## Goal

Enrich the archived 2026 Ozark Super Snownado case study with the *other* (satellite)
snownadoes that touched down during the same outbreak. Narrative premise: the
Super-Snownado ingested the arctic dryline and monopolized the available instability, so
every other snownado was **weak, brief, and narrow**. They appear in the Forecast
Evolution, the Meteorological History, and on the regional map.

## Data model

One new array `SATELLITES` (single source of truth), each entry:

```
{ d, t, p, la, lo, W, durMin, pathMi, widthYd, warn, n? }
```

- `d` calendar day (18 or 19), `t` time string "h:mm AM/PM" CST
- `p` place, `la`/`lo` lat/lon (for the map), `W` Whiteout rating (0–1 only)
- `durMin` minutes on the ground, `pathMi` path length (mi), `widthYd` width (yards)
- `warn` the warning issued for THIS snownado, `n` optional note

A small `parseTime(t)` helper converts to minutes-since-midnight so the pre-3:42 PM
satellites can be merged, by time, into the Forecast Evolution timeline. First main
touchdown = Dec 18, 3:42 PM.

## The 6 satellites (all W0–W1; brief, short-tracked, narrow)

| # | d | t | place | la | lo | W | durMin | pathMi | widthYd | warn |
|---|----|---------|--------------------|-------|--------|---|----|-----|-----|------|
| 1 | 18 | 2:57 PM | Okmulgee, OK | 35.62 | -95.96 | 1 | 2 | 0.3 | 30 | Snownado Warning — Okmulgee Co. |
| 2 | 18 | 3:19 PM | near Bristow, OK | 35.83 | -96.39 | 0 | 1 | 0.2 | 20 | radar-indicated Snownado Warning — Creek Co. |
| 3 | 18 | 4:46 PM | near Bartlesville, OK | 36.75 | -95.98 | 1 | 3 | 0.5 | 45 | Snownado Warning — Washington Co. |
| 4 | 18 | 6:34 PM | Neosho, MO | 36.87 | -94.37 | 1 | 2 | 0.3 | 30 | Snownado Warning — Newton Co. |
| 5 | 18 | 7:05 PM | near Pittsburg, KS | 37.41 | -94.70 | 1 | 2 | 0.3 | 25 | Snownado Warning — Crawford Co., KS |
| 6 | 19 | 3:40 AM | near Carlyle, IL | 38.61 | -89.37 | 0 | 1 | 0.2 | 20 | radar-indicated Snownado Warning — Clinton Co., IL |

Covers all four states the main storm crossed (OK, KS, MO, IL). The first two touch down
before the main 3:42 PM vortex. Two W0 events are radar-indicated only. Every one is down
1–3 minutes, ≤0.5 mi, ≤45 yd wide — none earn more than a base Snownado Warning, a
deliberate contrast with the Super-Snownado's DS/PDS/Emergency/Extreme-Emergency ladder.

## Rendering changes

### Forecast Evolution (`lines()` "...CONDITIONS LEADING UP TO THE STORM...")
1. Append an anticipation clause to the 1:15 PM EXTREME Day 1 Outlook `PRELUDE` entry:
   "A few additional, generally weak snownadoes are possible across the broader warm
   sector, though one dominant, long-track circulation is expected to consolidate most of
   the available instability."
2. Add a new `PRELUDE` real-time entry — Dec 18, 3:29 PM, "Mesoscale Discussion #216A
   (update)" — noting the two warm-sector touchdowns already down (Okmulgee & Creek Cos.)
   and that the consolidating main circulation will keep others weak and short-lived.
3. Merge satellites with `time < 3:42 PM` (#1, #2) into the timeline in time order,
   rendered as `🌀 SATELLITE SNOWNADO — <place>` with a detail line
   (W rating + name · lasts · path · width) and a `⚠ <warn>` line.

### Meteorological History (`lines()`)
New subsection after the main track note: `...OTHER SNOWNADOES OF THE OUTBREAK...`, framed
by one line that the dominant snownado starved them of fuel. Catalog all 6 in time order:
`time · place · W# Name · lasts N min · path X mi · width Y yd` plus a `⚠ <warn>` line and
optional note.

### Map (`render()`)
After the main track polys, plot each satellite as a small faint disc (`disc()` at its
projected `la`/`lo`, radius ~ a few px, colored by `WCOL[w.W]`, opacity ~0.5, no swath),
subordinate to the headline track.

## Out of scope
No changes to the Outlook/MD/Watch/Warning product builders or the snowfall formula. This
is additive to the archived `ozark` case study only.
