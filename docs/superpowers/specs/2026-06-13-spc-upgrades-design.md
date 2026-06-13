# Snow Prediction Center — Upgrade Batch Design (2026-06-13)

Eleven upgrades to the single-file app (`index.html`). Priority: **#8 (Outlook input lockup)** first; all eleven to ship.

## Decisions (locked)
- Expected snowfall: **computed** from risk (no new manual input).
- "DS" = **Dangerous Situation**.
- Outlook risk entry: **add numeric risk dropdowns** per hazard; categorical becomes computed-only.
- Output scope: **maximal everywhere** — every product becomes a multi-section bulletin.

## 1. Outlook — draw the three risks; categorical as computed output (items 7, 8, 11)
- The three hazards (Snownado, Blizzard Wind, Snow Squall) are **drawn like normal**: pick a probability or CIG level, draw a polygon, repeat at other levels to build nested contours (restored original workflow). Stored in `S.areas[h]` / `S.areas['cig_'+h]`.
- **Categorical is never hand-drawn.** It is a **computed OUTPUT view** ("Categorical (output)" panel) that recolors each drawn risk area to its implied tier and renders it on the map. `suggestTier()` reads the highest drawn prob/CIG per hazard, keeping the Extreme gate (CIG4 snownado + ≥80% prob, never Day 3).
- Removing the drawable categorical panel eliminates the shared draw-state collision → **#8 lockup fixed** (each hazard exposes its own independent level buttons; entering snownado no longer blocks blizzard/squall).
- Day 4–8 stays probability-only (Categorical output button hidden).

## 2. Expected snowfall output (item 1)
Computed `...EXPECTED SNOWFALL...` section. Tier→range: General 1–3″, Marginal 2–5″, Slight 4–8″, Enhanced 6–12″, Moderate 10–18″, High 15–30″, Extreme 24″+ (locally 36″+). **Every tier carries 3–7 base texture modifiers** (ratios, banding, rates, duration, drifting, loading, visibility), plus hazard-driven additions from the drawn squall/blizzard/snownado CIG levels.

## 3. Hazard overview output (item 9)
Generated `...HAZARD OVERVIEW / IMPACTS...` paragraph derived from active hazards' probability + CIG + Whiteout scale (`SPC.tax.WHITEOUT`, `CIG`). Describes specific impacts (e.g. W3 "Severe" roof scour, ≥65 mph whiteout/drifting, ≥4″/hr squall visibility loss).

## 4. Mesoscale Discussion — granular wording (item 3)
Replace 2-bucket logic with 6 buckets on Probability of Watch Issuance:
- 0–14: watch not anticipated at this time
- 15–34: a watch may eventually be needed
- 35–54: a watch may be needed in the next 1–3 hours
- 55–74: a watch is likely and increasingly probable
- 75–94: a watch is expected; issuance anticipated shortly
- 95–100: watch issuance is imminent / in progress

## 5. MD — meso-gamma distinct template (item 4)
Meso-gamma MDs use a separate text template (not just an added line): pinpoint/short-fused framing, immediate-action language for W2+ snownadoes or extreme wind, own summary structure. Both variants gain expanded technical reasoning (item 11).

## 6. Warning severity ladder + PDS (items 5, 6)
Warnings: **Normal → DS → PDS → Emergency** (replaces emergency checkbox with a 4-step selector; squalls capped — no Emergency, matching existing `canEmergency`). Each step: distinct banner + escalating impact/call-to-action.

## 7. Watch severity ladder + DS (item 6)
Watches: **Normal → DS → PDS** (replaces PDS checkbox with 3-step selector). Each step: distinct banner wording. Plus expanded "what to expect" + "preparedness actions" sections (item 11).

## 8. HWO — granular spotter scale (item 2)
Boolean → **6 levels**, each with distinct spotter-statement wording; suggested level auto-derived from Outlook tier, overridable:
0 Not Activated · 1 Standby/Situational Awareness · 2 Activated — Routine Reports · 3 Activated — Enhanced Reporting · 4 Full Activation — Priority Reports · 5 Emergency Activation — Continuous Reporting.

## 9. Storm Reports — 7-point verification (item 10)
Gap = needed CIG (from realized W) − forecast CIG. Scale:
≥+3 Critically Under-Warned · +2 Significantly Under-Warned · +1 Under-Warned · 0 Verified · −1 Slightly Over-Warned · −2 Over-Warned · ≤−3 Significantly Over-Warned. Per-report lines + net result use this scale.

## 10. Maximal outputs (item 11, cross-cutting)
Each product becomes multi-section: header → core call → details → impacts → action/preparedness → forecaster/office line. Applies to Outlook, MD, Watch, Warning, HWO, Verification.

## Non-goals
- No change to the map/SVG engine, save/load format compatibility beyond additive fields, or the basemap.
- No backend; remains static single-file app deployable to Pages/Vercel.

## Verification
Run via preview, exercise: (a) #8 — set categorical-driving risks for snownado then enter blizzard + squall risk without lockup; (b) each tab renders expanded output; (c) severity ladders and scales produce distinct wording at each level.
