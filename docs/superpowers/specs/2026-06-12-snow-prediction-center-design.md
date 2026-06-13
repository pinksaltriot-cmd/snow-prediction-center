# Snow Prediction Center — Product Maker (Design Spec)

**Date:** 2026-06-12
**Status:** Approved design, pre-implementation
**References:**
- SPC: https://en.wikipedia.org/wiki/Storm_Prediction_Center
- NWS terminology: https://en.wikipedia.org/wiki/Severe_weather_terminology_(United_States)
- Conditional Intensity: https://www.spc.noaa.gov/exper/conditional-intensity-information/

---

## 1. Overview

A standalone, single-file web app — the **"Snow Prediction Center" (SPC) product maker** — that
replicates the U.S. Storm Prediction Center's product suite, mapped to snow. It is a
forecaster's-desk tool: the user draws click-polygon risk areas on a US map and issues the full
suite of snow forecast products, then logs actual "notable snownado" reports to verify the forecast.

This is **independent** of the existing `snow-simulator` app. It shares no code or data with it.

**Hazard mapping** (snow analogs of SPC's convective hazards):
- **Snownadoes** ← tornadoes ("dust devils, but snow")
- **Blizzard Wind** ← damaging wind
- **Snow Squalls** ← hail

---

## 2. Architecture

- **One standalone HTML file**, vanilla JS, **zero dependencies**, GitHub-Pages-ready.
  Matches the deploy style of the existing snow app.
- **Embedded Albers-USA SVG basemap**: state outlines as static inline SVG `<path>`s. No map
  library. The user clicks vertices in SVG coordinate space; polygons render as shaded SPC-style
  areas.
- **Drawing model — click polygon vertices**: click points to lay an outline, double-click to
  close. Polygons are stored as `{ points: [[x,y]...], layer, level }`. Nested areas auto-stack
  (e.g. HIGH drawn inside MDT inside ENH). Per-polygon edit/delete.
- **Layout**:
  - **Left rail** — drawing tools: product selector, hazard/panel selector, category/CIG level
    picker, draw / edit / delete, valid-time fields.
  - **Center** — the US map (active panel).
  - **Right rail** — live product preview: the rendered map panel + auto-generated SPC-style text.
- **Top tabs** — the product families: **Outlook · Mesoscale Discussion · Watch · Warning ·
  HWO / Hydro · Storm Reports**.
- **Export**: each product exportable as **PNG** (map) + **copyable text**. Optional session
  save/load as JSON.

---

## 3. Taxonomy

### 3.1 Categorical risk tiers (Outlook)

| Tier | Abbr | Color | Gate |
|---|---|---|---|
| General Snow | SNOW | light green (#c1e9c1) | ordinary snow, no severe hazards |
| Marginal | MRGL | dark green (#66a366) | — |
| Slight | SLGT | yellow (#ffe066) | — |
| Enhanced | ENH | orange (#ffa366) | — |
| Moderate | MDT | red (#e06666) | — |
| High | HIGH | magenta (#ee99ee) | — |
| **Extreme** | **EXTR** | black/white hatched | **only if ≥80% risk of CIG4 snownadoes** |

### 3.2 Hazard probabilities (% within 25 mi of a point)

- **Snownadoes**: 2 / 5 / 10 / 15 / 30 / 45 / 60 %
- **Blizzard Wind**: 5 / 15 / 30 / 45 / 60 / 75 / 90 %
- **Snow Squalls**: 5 / 15 / 30 / 45 / 60 %

### 3.3 Conditional Intensity Groups (CIG) — replace the SIG hatch

Each is a distinct *shaded zone* (not hatching) on the relevant hazard panel, expressing
**intensity if the hazard occurs**. The CIG count differs per hazard (4 / 3 / 2 gradient):

- **Snownadoes — CIG1–CIG4**
  - CIG1 = 20% chance of strong (W2+) snownado
  - CIG2 = 30% chance W2+
  - CIG3 = 40% chance W2+ (~19% W4+)
  - CIG4 = significant chance of **W5–W6** (violent). Gate for Extreme Risk at ≥80%.
- **Blizzard Wind — CIG1–CIG3**
  - CIG1 = gusts ≥35 mph (blizzard criterion)
  - CIG2 = ≥50 mph
  - CIG3 = ≥65 mph
- **Snow Squalls — CIG1–CIG2** (no CIG3, mirrors SPC's 2-level hail)
  - CIG1 = ≥2″/hr
  - CIG2 = ≥4″/hr with visibility ≤¼ mi (thundersnow-grade)

### 3.4 The Whiteout Scale (W0–W6) — snownado intensity

Snownado vortices are weaker than tornadoes; every tier sits **below its EF equivalent**, with
**W6 as a snow-only top tier** beyond the EF scale.

| Tier | Wind | Name | Damage |
|---|---|---|---|
| W0 | 25–40 mph | Flurry Spinner | Blowing-snow swirl; dusts cars; harmless |
| W1 | 41–55 mph | Squall Devil | Light drifting; trash cans over; visible snow devil |
| W2 | 56–70 mph | Strong | Drifts pile fast; signs down; brief whiteout *(strong threshold)* |
| W3 | 71–85 mph | Severe | Major drifting; roofs scoured; vehicles pushed |
| W4 | 86–100 mph | Devastating | Deep drift burial; structures damaged |
| W5 | 101–120 mph | Violent | Total whiteout vortex; severe structural damage |
| W6 | >120 mph | Cryonic | Snow-only extreme: complete burial; catastrophic |

---

## 4. Products

### 4.1 Snow Outlook (flagship)

- **Day tabs**: Day 1, Day 2, Day 3 (categorical + probabilistic), Day 4–8 (probabilistic-only %).
  - **Day 3 cannot issue Extreme** (mirrors SPC's "no Day 3 HIGH" rule).
- **Panel switcher** on one map: **Categorical · Snownado prob · Blizzard Wind prob · Snow Squall
  prob**. Each probabilistic panel shows its % areas plus the **CIG overlays** as distinct shaded
  zones.
- **Drawing**: pick panel + level (e.g. "SLGT", "Snownado 10%", "CIG2"), click vertices, double-click
  to close. Nested areas auto-stack. Per-polygon edit/delete.
- **Auto-derivation**: the categorical panel suggests a tier from the highest probabilities drawn
  (prob→categorical conversion). Flags **Extreme eligibility** when a CIG4 snownado area ≥80% is
  present. User can override.
- **Auto-text**: SPC-style narrative (`...AREAS AFFECTED...`, `...SUMMARY...`, `...DISCUSSION...`)
  derived from drawn areas.
- **Export**: PNG of active panel + copyable text.

### 4.2 Mesoscale Discussion (MD) + Meso-gamma

- **Snow MD**: draw one polygon; set "Concerning" (Snownado / Blizzard Wind / Snow Squalls),
  **probability of watch issuance %**, valid time. Auto-text in MD format. Precedes a watch.
  - No high wind floor — issued whenever a watch may be needed.
- **Meso-gamma MD**: high-confidence variant, distinct header/styling. Trigger (snow-appropriate,
  not derecho-level): confident **W2+ snownadoes** *or* **blizzard wind ≥60 mph**.

### 4.3 Watches, PDS & Emergency

- **Watches** (draw area + probability table): **Blizzard Watch**, **Snownado Watch**.
  Snow Squalls remain **warning-only** (as in reality — no squall watch).
- **PDS (Particularly Dangerous Situation)** variants for high-end events: *PDS Blizzard Watch*,
  *PDS Snownado Watch*.
- **Watch probability table**: snow analogs of SPC's table — e.g. prob of 2+ snownadoes, prob of
  one W2+, prob of blizzard wind ≥60 mph, etc.

### 4.4 Warnings & Emergencies

- **Warnings** (short-fuse polygons):
  - **Blizzard Warning** — sustained/gusting ≥35 mph + heavy snow + vis ≤¼ mi for ≥3 hr.
  - **Snow Squall Warning** — intense, brief, moderate-heavy snow; vis ≤¼ mi.
  - **Snownado Warning** — snownado occurring or imminent.
- **Emergency** wording (top tier): **Snownado Emergency**, **Blizzard Emergency** — reserved for
  catastrophic, confirmed threats to populated areas.

### 4.5 Hazardous Weather Outlook (HWO) + Hydrologic Outlook

- **HWO**: 7-day narrative — Day 1 detail + Days 2–7 — with a **snow-spotter activation** statement.
- **Hydrologic Outlook (ESF)**: snowmelt-flooding / snowpack outlook narrative.

### 4.6 Storm Reports & Verification

- **Manual logger** (no prebuilt list, no map-clicking): the user types a report —
  **city name (free text, any place from metropolis to village) + W-rating (W0–W6) + notability
  tier** (Notable / Super-Notable / **Super-Hyper-Notable**) + optional name/time.
- Reports appear in a **reports log / table** (SPC LSR style).
- **Verification panel** — **intensity-based** (cities are free-text without guaranteed
  coordinates, so verification compares realized intensity, not strict point-in-polygon):
  - Compares the realized **max W-rating & notability** against the day's forecast ceiling
    (categorical tier + snownado CIG / probability).
  - Labels the outcome: **under-warned / verified / over-warned**, e.g. *"Predicted CIG2 / 30% —
    observed W5 super-hyper-notable → under-warned."*
- **Notability filter**: surface only the tier of interest. On an **Extreme Risk** day the user
  logs only **Super-Hyper-Notable** snownadoes.

---

## 5. Implementation phases

1. **Foundation** — single-file shell, embedded Albers-USA SVG basemap, click-polygon draw engine,
   taxonomy constants (tiers, colors, probs, CIGs, Whiteout Scale).
2. **Snow Outlook** — day tabs, panel switcher, categorical + probabilistic + CIG drawing,
   auto-derivation, auto-text, PNG/text export.
3. **Mesoscale Discussion + meso-gamma**.
4. **Watches (+ PDS) & Warnings (+ Emergency)** with probability tables.
5. **HWO + Hydrologic Outlook**.
6. **Storm Reports & Verification**.

---

## 6. Non-goals / YAGNI

- No integration with the existing `snow-simulator` app.
- No geographic geocoding of typed report cities (verification is intensity-based).
- No real meteorological data feeds; all content is user-authored.
- No map-pin placement of storm reports (typed log only).
