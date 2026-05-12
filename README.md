# Cycling Power Generator

A fully client-side web app that estimates **cycling power output** from a GPX or FIT activity file using a physics-based model. No server, no build step — open `cyclingPowerGenerator.html` in any modern browser.

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)

---

## Features

| | |
|---|---|
| **File formats** | `.gpx` (drag-and-drop or browse) and `.fit` (Garmin, COROS, Wahoo, Polar, …) |
| **Physics model** | Gravity + rolling resistance + aerodynamic drag, all configurable |
| **Wind & weather integration** | Fetches real historical wind, temperature, humidity and pressure from [Open-Meteo](https://open-meteo.com/); applies a bearing-adjusted headwind/tailwind component and computes per-point air density |
| **Power smoothing** | Centred time-aware rolling average (default 15 s, adjustable) |
| **Charts** | Power + elevation profile, speed, grade, wind component, power breakdown doughnut |
| **Power zones** | Z1–Z6 time-in-zone distribution relative to average power |
| **Map** | Route rendered on a dark Leaflet map, polyline coloured by power intensity |
| **FIT export** | Downloads the original FIT file with the calculated power channel added (field 7) |
| **Recalculate** | Change any parameter live — no need to re-upload |

---

## Usage

1. Open `cyclingPowerGenerator.html` in a browser (Chrome, Firefox, Safari, Edge).
2. Drop your `.gpx` or `.fit` file onto the upload zone.
3. Adjust the rider/equipment parameters on the left.
4. Optionally click **Fetch Wind Data** (requires internet; only available when the file contains timestamps).
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
| Kinetic energy | `F_k = m·a` (a = dv/dt, central difference) |
| Effective airspeed | `v_eff = v_ground + wind_speed·cos(wind_from − bearing)` |

- **m** = rider weight + bike weight (kg)
- **g** = 9.8067 m/s²
- **Crr** = rolling resistance coefficient (typically 0.003–0.006)
- **CdA** = drag coefficient × frontal area (m²; typically 0.25–0.40 for road bikes)
- **ρ** = air density (kg/m³; 1.225 at sea level, 15 °C)
- **η** = drivetrain efficiency (1 − loss%)
- **grade** = Δelevation / Δdistance
- **a** = acceleration (m/s²)

Wind, temperature, humidity and MSL pressure are fetched per-point at hourly resolution and linearly interpolated to each GPS timestamp. A positive `v_eff` component means headwind (more drag); negative means tailwind (less drag).

Air density is computed from the fetched temperature, humidity and pressure (corrected to the rider's actual elevation via the barometric formula), so a hot, humid mountain ride sees ρ ≈ 0.95 kg/m³ instead of the default 1.225. The static ρ input is used only as a fallback when weather has not been fetched.

The kinetic-energy term redistributes power into surges and out of coasting, so the model matches real power-meter traces on variable-pace rides. It integrates to ≈ 0 over a closed loop and does not change average power.

### Inputs — accuracy-relevant choices

- **Elevation is smoothed** (10 s centred window) before computing grade. GPS / barometric altitude is noisy, and grade is a derivative, so unsmoothed elevation produces spurious gravity-power spikes.
- **Speed source**: when the input is a FIT file with a wheel / fused-speed channel (field 6 or `enhanced_speed` field 73), that speed is used directly. Otherwise speed is derived from GPS deltas and lightly smoothed.
- **Acceleration** is computed by central difference on smoothed speed, then smoothed again to keep speed noise from amplifying through the kinetic term.

Power is finally smoothed with a centred time-aware rolling average (default ±7.5 s window).

### Normalised Power

Computed as the 4th-power mean of a 30-second rolling average, following the standard training-peaks methodology:

```
NP = (mean(rolling_30s_avg⁴))^(1/4)
```

---

## Adjustable parameters

| Parameter | Default | Description |
|---|---|---|
| Rider weight | 75 kg | Body mass |
| Bike weight | 8 kg | Complete bike mass |
| CdA | 0.32 m² | Aerodynamic drag area |
| Crr | 0.004 | Rolling resistance coefficient |
| Air density (ρ) | 1.225 kg/m³ | Fallback; overridden by fetched weather (computed from T, RH, P and rider elevation) |
| Drivetrain loss | 2 % | Mechanical efficiency loss |
| Power smoothing | 15 s | Rolling-average window; 0 = raw |

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
| [Open-Meteo](https://open-meteo.com/) | Free historical weather API (wind speed & direction) |
| [ANT+ FIT Protocol](https://developer.garmin.com/fit/protocol/) | Binary FIT format specification used for both decoding and export |
| [Leaflet.js](https://leafletjs.com/) | Interactive route map |
| [Chart.js](https://www.chartjs.org/) | Power, speed, grade, wind and breakdown charts |
| [CartoDB Dark Matter](https://carto.com/basemaps/) | Map tile layer |
| [Hunter Allen & Andrew Coggan — *Training and Racing with a Power Meter*](https://www.amazon.com/Training-Racing-Power-Meter-Third/dp/193403068X) | Normalised Power and power-zone methodology |

### Key references for the physics

- Martin, J. C. et al. (1998). *Validation of a mathematical model for road cycling power.* Journal of Applied Biomechanics, 14(3), 276–291.
- Wilson, D. G. (2004). *Bicycling Science* (3rd ed.). MIT Press.
- Analytic Cycling — [analyticcycling.com](https://www.analyticcycling.com/)

---

## Tech stack

- Vanilla HTML + CSS + JavaScript — zero build step, zero dependencies to install
- [Leaflet 1.9](https://leafletjs.com/) via CDN
- [Chart.js 4.4](https://www.chartjs.org/) via CDN
- FIT binary decoder written from scratch (no external library)

---

## License

[MIT](LICENSE) © 2025
