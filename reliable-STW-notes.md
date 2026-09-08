# Reliable STW — Project Notes

*Working notes on fixing speed-through-water (STW) reliability on a B&G H5000 / paddlewheel system, in a leeway-heavy, current-dominated, brackish sailing area.*

## 1. The core diagnosis

- The H5000's built-in STW compensation and the SignalK "speed and current" heel-table plugin are both **static, heel-only correction tables**. They remove an average bias but not the real scatter, because they assume a fixed heel → leeway relationship.
- On this boat (flat-bottomed, extreme, high-pointing), heel is a **weak proxy for leeway**. Leeway is driven by the lateral force balance (sail trim, sailset, mode, waves), which is largely decoupled from heel, which is driven mostly by righting moment. Same heel + same speed can produce very different leeway depending on conditions — confirmed by our own SignalK heel-table data.
- Reciprocal-course (double-run) calibration cancels **current** and isolates the paddlewheel's own sensor error (offset/linearity/fouling). It does **not** and cannot fix the separate, dynamic **leeway-driven flow-angle error** — that's not a sensor property, it's the water arriving at an angle. This is why the double-run work hasn't improved reliability under sail: it's solving a different problem than the one that's actually hurting us.
- GPS-derived "leeway" (COG vs heading) is contaminated by current in exactly the same way STW-vs-SOG calibration is — the two are inseparable from GPS + compass alone in a current-dominated area like ours.

## 2. The correct target architecture

1. **Measure the true 2-axis flow vector at the hull directly** (longitudinal + transverse/leeway component), not inferred from GPS.
2. **Use the real-time leeway angle to correct the raw paddlewheel/EM reading** for off-axis flow — not assumed to be a clean cosine relationship, since sensor response to off-axis flow is device-specific.
3. **Layer a multi-speed calibration on top**, but treat leeway-correction and speed-linearity as a **joint surface** (raw reading as a function of true STW *and* leeway angle, possibly heel too for housing effects) rather than two independent, sequentially-applied corrections — the off-axis response likely itself varies with speed (Reynolds-number/flow-separation effects).

Practical build sequence for the joint surface:
- Establish baseline speed linearity in the lowest-leeway condition sailable (flat water, ~0 heel, e.g. motoring or tight reach).
- Fit the residual error against measured leeway (and heel) using real sailing data.
- Sanity-check afterward for leftover speed/leeway interaction rather than assuming full separability.

## 3. Ground-truth leeway without dedicated hardware: the two-tack test

Direct analog of the reciprocal-run trick, applied to leeway instead of speed:

- Sail two tacks (port/starboard) at matched TWA, boat speed, heel, and trim mode, back to back so current is ~constant across both.
- If leeway were zero, the two ground tracks (referenced to true wind) would be symmetric.
- The asymmetry between them isolates leeway, without needing to know the current — same assumption as the reciprocal run (current roughly constant over a short window).
- Repeat across sail sets / trim modes / sea states to build a richer, condition-tagged leeway dataset than heel-alone ever could.

This is the no-new-hardware fallback / cross-check, and is worth doing regardless of whether we add a dedicated sensor.

## 4. Hardware options for direct 2-axis (leeway) sensing

Both are electromagnetic (Faraday induction), dual-axis (longitudinal + transverse), and both output leeway more or less directly:

### Airmar DX900+
- Dual-axis EM sensor, NMEA 0183 output including **NLA (nautical leeway angle)**, VBW, VHW, VLW, up to 10 Hz.
- Spec'd accuracy: ±0.1 kt under 10 kt, ±1% above 10 kt.
- Currently sold, but the underlying concept dates to ~2009 ("DX900-EM," delayed for years before shipping as DX900+ in 2017 — matches the "old AF" impression).
- Field reports (Panbo, forums): real reliability/calibration concerns — inconsistent readings across speed range despite claimed linearity, **suspected carbon-hull interference on at least one racing boat**, bottom-paint interference reported. Worth confirming directly with Airmar/other carbon-hull owners before committing, given our hull.
- Patent (US10852142B2, Airmar) explicitly includes **variable-gain preamps to compensate for salinity/conductivity differences** (fresh vs. salt water) — a real, documented engineering solution to the freshwater/brackish problem.

### Brickhouse Innovations (Larry Marsh, US10,416,187)
- Different company/patent from Airmar — not the same product family, despite the conceptual similarity.
- Differentiator: pulls sensing **away from the hull boundary layer** — electrodes offset ¼–½" minimum, larger annular-coil variants energize a much bigger water volume further from the hull, addressing the classic EM-log weakness of a tiny, hull-hugging, turbulence-contaminated sensing volume (worse at speed and in waves — exactly our failure mode).
- Removable electrodes (guide tubes + valves) for in-water cleaning/maintenance.
- Field-tested on Navy vessels 2014–2015; reads as small-scale/specialist rather than a mainstream retail product — confirm current availability, pricing, lead time directly.
- **Open question, unresolved from available docs**: the accessible patent text does not mention conductivity/salinity compensation at all (silent, not necessarily absent). Given our waters are genuinely brackish and conductivity-variable (islands, mixing zones), **ask Brickhouse directly** whether the sensor compensates for salinity and whether it's been validated outside open-ocean saltwater conditions before committing.

### General EM-log caveat (applies to any option, including DIY)
- EM sensors work via Faraday induction — induced signal scales with water conductivity. Freshwater is much lower conductivity than saltwater; brackish is variable, potentially shifting within a single outing near river mouths/estuaries.
- Uncompensated, this risks trading one poorly-understood systematic error (paddlewheel/leeway) for another (conductivity drift) — must confirm compensation exists and is validated for our conditions, not just for open ocean.

## 5. Open action items

- [ ] Confirm with Airmar / other carbon-hull DX900+ owners whether carbon-hull interference is a real risk for us.
- [ ] Ask Brickhouse directly: salinity/conductivity compensation — present? validated in brackish/fresh conditions?
- [ ] Run two-tack leeway tests across sail sets/trim modes this season regardless of hardware decision — builds ground truth either way.
- [ ] Decide buy (DX900+) vs. build (Brickhouse-style DIY, dual-axis, boundary-layer-offset design) vs. wait, based on above.
- [ ] Once real 2-axis data (from hardware or two-tack tests) exists, fit the joint (STW, leeway, heel) correction surface rather than a heel-only table.
