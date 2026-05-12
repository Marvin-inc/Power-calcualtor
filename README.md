# Cycling Power Generator

A fully client-side web app that estimates **cycling power output** from a GPX or FIT activity file using a physics-based model. No server, no build step — open `cyclingPowerGenerator.html` in any modern browser.

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)

---

## Features

| | |
|---|---|
| **File formats** | `.gpx` (drag-and-drop or browse) and `.fit` (Garmin, COROS, Wahoo, Polar, …) |
| **Physics model** | Gravity + rolling resistance + aerodynamic drag + kinetic energy, all configurable |
| **Wind & weather** | Fetches historical wind, temperature, humidity and pressure from [Open-Meteo](https://open-meteo.com/); applies a bearing-adjusted headwind/tailwind component and computes per-point air density |
| **Power smoothing** | Centred time-aware rolling average (default 15 s, adjustable) |
| **Charts** | Power + elevation profile, speed, grade, wind component, power breakdown doughnut |
| **Power zones** | Z1–Z6 time-in-zone distribution relative to average power |
| **Map** | Route rendered on a dark Leaflet map, polyline coloured by power intensity |
| **FIT export** | Downloads the original FIT file with the calculated power channel added (field 7), all original data preserved |
| **CdA estimator** | Estimates CdA from height, mass, shoulder width, seat-tube angle and torso angle using five published equations; position presets and a bike-drag component give realistic total CdA values |
| **Recalculate** | Change any parameter live — no need to re-upload |

---

## Usage

1. Open `cyclingPowerGenerator.html` in a browser (Chrome, Firefox, Safari, Edge).
2. Drop your `.gpx` or `.fit` file onto the upload zone.
3. Adjust the rider/equipment parameters. Click **👤 Estimate** next to CdA to open the CdA estimator.
4. Optionally click **Fetch Wind & Weather** (requires internet; only available when the file contains timestamps).
5. Inspect the stats, charts and map.
6. For FIT files, click **Export FIT with Power** to download a copy of the original file with the power channel embedded.

---

## Physics model

Total power at the wheel:

```
P = (F_gravity + F_rolling + F_aero + F_kinetic) × v_ground / η
```

| Component | Formula |
|---|---|
| Gravity | `F_g = m·g·sin(arctan(grade))` |
| Rolling resistance | `F_r = m·g·cos(arctan(grade))·Crr` |
| Aerodynamic drag | `F_a = ½·ρ·CdA·v_eff·\|v_eff\|` |
| Kinetic energy | `F_k = m·a` (a = dv/dt, central difference on smoothed speed) |
| Effective airspeed | `v_eff = v_ground + wind_speed·cos(wind_from − bearing)` |

- **m** = rider weight + bike weight (kg)
- **g** = 9.8067 m/s²
- **Crr** = rolling resistance coefficient (typically 0.003–0.006)
- **CdA** = drag coefficient × frontal area (m²)
- **ρ** = air density (kg/m³)
- **η** = drivetrain efficiency (1 − loss%)
- **grade** = Δelevation / Δdistance (from smoothed elevation)
- **a** = acceleration (m/s²)

### Accuracy improvements over a basic steady-state model

**Kinetic energy term.** `P_k = m·v·a` is added so the model reflects real power-meter behaviour: power rises on surges and drops to zero when coasting. The term integrates to ≈ 0 over a closed loop and does not bias average power.

**Elevation smoothed before computing grade.** GPS and barometric altitude are noisy. Because grade is a derivative, raw elevation noise produces large spurious gravity-power spikes. Elevation is smoothed with a 10 s centred time-aware window before grade is computed.

**FIT speed channel used when available.** When the device recorded a wheel-speed or GPS/Kalman-fused speed (FIT field 6 / `enhanced_speed` field 73), that channel is used in preference to GPS-delta-derived speed, which reduces noise in both speed and the kinetic-energy term.

**Per-point air density.** Wind & weather fetch also retrieves temperature, relative humidity and sea-level pressure from Open-Meteo. Air density is computed per point using the barometric formula to correct MSL pressure to the rider's actual elevation, plus a Tetens saturation vapour pressure term for humid air:

```
ρ = P_d / (R_d · T) + P_v / (R_v · T)
```

A hot, humid mountain ride (e.g. 25 °C, 40 % RH, 1800 m) sees ρ ≈ 0.96 kg/m³ instead of the default 1.225 — a 22 % reduction in aerodynamic drag at the same speed. The static ρ input is used only as a fallback when weather has not been fetched.

### Normalised Power

Computed as the 4th-power mean of a 30-second rolling average, following the standard methodology:

```
NP = (mean(rolling_30s_avg⁴))^(1/4)
```

---

## Adjustable parameters

| Parameter | Default | Description |
|---|---|---|
| Rider weight | 75 kg | Body mass |
| Bike weight | 8 kg | Complete bike mass |
| CdA | 0.32 m² | Total aerodynamic drag area (rider + bike); use the estimator to derive a value |
| Crr | 0.004 | Rolling resistance coefficient |
| Air density (ρ) | 1.225 kg/m³ | Fallback value; overridden per-point by fetched weather |
| Drivetrain loss | 2 % | Mechanical efficiency loss |
| Power smoothing | 15 s | Centred rolling-average window applied to final power; 0 = raw |

---

## CdA estimator

Click **👤 Estimate** next to the CdA field to open the estimator modal.

### Inputs

| Input | Notes |
|---|---|
| Height (m) | Rider standing height |
| Body mass (kg) | Pre-populated from the rider weight field |
| Shoulder width (m) | Biacromial width, required for Heil full equation |
| Seat tube angle (°) | STA from horizontal |
| Torso angle (°) | TA from horizontal; aero tuck ≈ 5–15°, road hoods ≈ 35–45° |
| Position preset | Fills STA, TA, Cd and bike CdA from typical values for the chosen position |
| Cd | Drag coefficient; editable, defaults set by preset |
| Bike + accessories CdA | Added to all estimates; typical values below |

### Frontal area equations

| # | Source | R² | S.E.E. | Inputs needed |
|---|---|---|---|---|
| 1 | AIS / Sam Callan, USA Cycling | — | — | h, m |
| 2 | Bassett et al., *Med Sci Sports Exerc* 1999 | 0.76 | 0.009 m² | h, m |
| 3 | Heil 2001 (mass + angles) | 0.54 | 0.017 m² | m, STA, TA |
| 4 | Heil 2001 (mass + height + angles) | 0.56 | 0.014 m² | h, m, STA, TA |
| 5 | Heil 2001 (full, incl. shoulder width) | 0.69 | 0.013 m² | h, m, STA, TA, SW |

Each row shows **Rider A → Rider CdA → + Bike → Total CdA**. Any row (or the mean) can be applied to the CdA field with one click.

### Position presets

| Position | STA | TA | Cd | Bike CdA | Typical total CdA |
|---|---|---|---|---|---|
| TT / aero tuck | 78° | 8° | 0.65 | 0.030 m² | 0.25–0.30 m² |
| Time trial | 76° | 12° | 0.70 | 0.035 m² | 0.28–0.33 m² |
| Road, drops | 73° | 25° | 0.82 | 0.040 m² | 0.35–0.39 m² |
| Road, hoods | 73° | 40° | 0.92 | 0.045 m² | 0.40–0.43 m² |
| Upright / touring | 73° | 55° | 1.05 | 0.055 m² | 0.46–0.50 m² |

> **Note.** Heil's equations were fit exclusively on TT cyclists (TA 5–25°) and their TA exponent is only ~0.1, so they under-respond at road or upright torso angles. Position-appropriate Cd values are the main mechanism for capturing the large drag increase when sitting up. Total uncertainty is approximately ±0.016–0.019 m² rider CdA (5–10%).

---

## FIT export details

The exporter patches the original binary in a single pass:

- Every message-20 **definition** gets field 7 (`power`, uint16, no scale/offset) appended.
- Every message-20 **data record** with valid GPS gets 2 bytes of calculated power appended.
- Records before GPS acquisition get the FIT invalid marker `0xFFFF`.
- All other messages (session, lap, HR, cadence, developer data, …) are preserved verbatim.
- Header CRC and file CRC are recomputed (ANT+ FIT CRC-16 algorithm).

The output is a standards-compliant FIT activity file importable into Strava, Garmin Connect, TrainingPeaks, etc.

---

## Sources & acknowledgements

| Resource | Role |
|---|---|
| [royceschultz/Cycling-Power-Calculator](https://github.com/royceschultz/Cycling-Power-Calculator) | Original inspiration and physics model reference |
| [Open-Meteo](https://open-meteo.com/) | Free historical weather API (wind, temperature, humidity, pressure) |
| [ANT+ FIT Protocol](https://developer.garmin.com/fit/protocol/) | Binary FIT format specification used for both decoding and export |
| [Leaflet.js](https://leafletjs.com/) | Interactive route map |
| [Chart.js](https://www.chartjs.org/) | Power, speed, grade, wind and breakdown charts |
| [CartoDB Dark Matter](https://carto.com/basemaps/) | Map tile layer |
| [Hunter Allen & Andrew Coggan — *Training and Racing with a Power Meter*](https://www.amazon.com/Training-Racing-Power-Meter-Third/dp/193403068X) | Normalised Power and power-zone methodology |

### Key references for the physics

- Martin, J. C. et al. (1998). *Validation of a mathematical model for road cycling power.* Journal of Applied Biomechanics, 14(3), 276–291.
- Wilson, D. G. (2004). *Bicycling Science* (3rd ed.). MIT Press.
- Analytic Cycling — [analyticcycling.com](https://www.analyticcycling.com/)

### Key references for the CdA estimator

- Bassett, D. R. et al. (1999). *Comparing cycling world hour records, 1967–1996.* Medicine & Science in Sports & Exercise, 31(11), 1665–1676.
- Heil, D. P. (2001). *Body mass scaling of projected frontal area in competitive cyclists.* European Journal of Applied Physiology, 85(4), 358–366.
- Kyle, C. R. (1991). *Wind tunnel tests of bicycle wheels and helmets.* Cycling Science, Sept/Dec, 51–56.

---

## Tech stack

- Vanilla HTML + CSS + JavaScript — zero build step, zero dependencies to install
- [Leaflet 1.9](https://leafletjs.com/) via CDN
- [Chart.js 4.4](https://www.chartjs.org/) via CDN
- FIT binary decoder written from scratch (no external library)

---

## License

[MIT](LICENSE) © 2025
