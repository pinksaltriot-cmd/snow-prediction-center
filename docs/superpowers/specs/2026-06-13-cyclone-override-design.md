# Cyclone Override — design (2026-06-13)

New feature: enter an extratropical cyclone's parameters and auto-generate every SPC product.

## Inputs (new "🌀 Cyclone Override" tab)
- **Central pressure** (mb) — primary intensity driver (`pressRank` 0–6).
- **Core temperature** (°F) — snow favorability (`tempFactor` 1.0 cold → 0.15 warm).
- **Size** — effective radius (mi); sets footprint radius (`PXMI ≈ 0.31 px/mi`).
- **Core city** — searchable datalist of **325 CONUS cities** (name + lat/lon ported from snow-simulator `RAW_CITIES`; includes western cities the simulator left region-less, e.g. LA/Seattle/Phoenix, plus Lufkin TX & Rhinelander WI). AK/HI excluded (CONUS-only projection).
- **Offset** toggle (ocean / another country) — nearest US city + **distance** + **8-point compass direction**; core placed by direction vector at `distance·PXMI`.

## Projection
lat/lon → SVG `960×600` via linear fit to basemap state centroids:
`x = 15.8463·lon + 2050.93`, `y = -23.1797·lat + 1176.93` (~22 px mean error; approximate by design).

## Physics → product mapping
- Per-hazard score = `(pressRank/6) · tempFactor` (blizzard wind less temp-sensitive).
- score → probability via `pick()` into each hazard's scale; → CIG via `cigOf()`.
- **Temperature falloff:** `tempFactor` grades 1.0 (≤10°F) → 0 (≥38°F); above freezing trends to rain. ≥38°F issues nothing; a warm-core note appears above freezing. Blizzard uses `tf^0.6` (robust in cold, zero in rain).
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
