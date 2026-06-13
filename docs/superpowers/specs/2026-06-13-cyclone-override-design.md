# Cyclone Override — design (2026-06-13)

New feature: enter an extratropical cyclone's parameters and auto-generate every SPC product.

## Inputs (new "🌀 Cyclone Override" tab)
- **Central pressure** (mb) — primary intensity driver (`pressRank` 0–6). **No closed low** checkbox = open wave (rank 1, marginal).
- **Core temperature** (°F) — snow favorability (`tempFactor` 1.0 ≤10°F → 0.08 floor); products persist (reduced) through 32–60°F, never absolute zero.
- **Size** — effective radius (mi); sets footprint radius (`PXMI ≈ 0.31 px/mi`). **No determined edge** checkbox = diffuse system (~400 mi default).
- **Core city** — searchable datalist of **325 CONUS cities** (name + lat/lon ported from snow-simulator `RAW_CITIES`; includes western cities the simulator left region-less, e.g. LA/Seattle/Phoenix, plus Lufkin TX & Rhinelander WI). AK/HI excluded (CONUS-only projection).
- **Offset** toggle (ocean / another country) — nearest US city + **distance** + **8-point compass direction**; core placed by direction vector at `distance·PXMI`.

## Projection
lat/lon → SVG `960×600` via linear fit to basemap state centroids:
`x = 15.8463·lon + 2050.93`, `y = -23.1797·lat + 1176.93` (~22 px mean error; approximate by design).

## Physics → product mapping
- Per-hazard score = `(pressRank/6) · tempFactor` (blizzard wind less temp-sensitive).
- score → probability via `pick()` into each hazard's scale; → CIG via `cigOf()`.
- **Temperature falloff:** `tempFactor` 1.0 (≤10°F) → 0.6 (≤32) → 0.45 (≤40) → 0.32 (≤50) → 0.18 (≤60) → 0.08 floor. 32–50°F keeps solid coverage, 50–60°F much less, never absolute zero. Warm-core note above 32°F. Blizzard uses `tf^0.6`.
- **Severity is temperature-aware:** Watch/Warning levels and MD watch-prob derive from the computed tier / max hazard score, not raw pressure — so a warm deep low does not issue a blizzard warning.
- Extreme (snownado 80% + CIG4) only when pressure very low **and** core very cold.

## Generation
- **Outlook:** concentric **nested** prob contours per hazard (outer = lowest %, inner = peak) + nested CIG zones → passes the nesting check; categorical auto-computes; Day 1.
- **MD:** concerning = dominant hazard; watch prob = `rank/6·100`; meso-gamma if compact (≤120 mi) and intense (rank ≥ 4).
- **Watch:** Snownado/Blizzard; level Normal→DS→PDS by rank; probability table from scores.
- **Warning:** dominant-hazard type; Normal→DS→PDS→Emergency by rank (Emergency gated to Blizzard/Snownado).
- **HWO:** spotter level from categorical tier; Day-1 line names the cyclone.
- **Summary:** cyclone tab lists the issued suite + all cities within the footprint (haversine).

## Notes
- Inputs are module-local (transient); generated products live in `D.*` and persist through save/load.
- Placements are approximate (linear projection + circular footprint) — an auto-draft to refine.


## Tier ladder + score-driven categorical (update)
- Tiers extended below General: **ZERO · LO-SNO · General Snow · Marginal · Slight · Enhanced · Moderate · High · Extreme** (ranks 0–8); all rank-dependent code (probToRank, spotter map, severity, day-3 cap, preparedness) shifted accordingly.
- Cyclone categorical is now driven by a **snow-intensity score** `snowScore=(rank/6)·tempFactor` → `scoreToRank` (0–8), giving precise, gradual temperature control; very cold bomb (<960 mb, ≤12°F) → Extreme.
- Hazard peak prob/CIG are **derived from the categorical tier** (per-tier tables, dominant hazard full / others one notch down) so they never exceed it.
- 32–60°F now steps down through tiers (e.g. 962 mb: 28°F Slight → 32°F Marginal → 40°F General → 46°F Lo-Sno → 58°F+ Zero) instead of sitting high.
- Cyclone sets `outlook.catOverride`; the Outlook tab reflects it, and any manual draw/undo/clear in the Outlook clears the override and recomputes.


## Level / meso-gamma / snowfall rebalance (update)
- **Outlook levels raised:** snowScore now uses `min(1, rank/4.5)·tf` (stronger pressure response) and `scoreToRank` thresholds lowered (+ rank 8 at the top), so cold strong cyclones reach High/Extreme readily (e.g. 962 mb: 6°F Extreme, 14°F High). Warm/weak systems still step down low.
- **Meso-gamma:** trigger is now `catRank >= 6` (Moderate+) with **no size limit** — lower threshold, still gated high by the temp+pressure-driven catRank.
- **Expected snowfall is computed from individual hazard intensities, not the categorical:** synoptic base + snow-squall burst totals (by CIG) + snownado deposition that explodes with CIG (cig4 → 36–90″), while blizzard wind adds only ~2″ (drifting). So wind-only Marginal (~1–4″) barely beats General (~1–2″) and Extreme flies off the charts (45–114″+); a wind-only High still yields little snow.
