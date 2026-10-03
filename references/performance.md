# Performance: spend GPU where the eye is

The goal is a scene that looks as good as it can *when someone is looking at it* and costs very little
the rest of the time. A scroll story spends most of its life **resting** on a shot, or watching a
**clock event** play with the camera still. Only a fraction of it is the camera **moving**. The stock
engine used to draw every frame at one fixed resolution regardless, so a resting shot cost as much as
a fast scroll. The template now governs this. Read this before changing the numbers or the shader.

## What the template does (all in `assets/template.html`, tunable in `Q`)

Three decisions are made per frame, outside the scene, so a new world gets them for free.

| Decision | Rule | Why |
|---|---|---|
| **How often** a frame is drawn | Every display frame while the camera moves; `eventFps` (30) while a clock event plays; `restFps` (20) at rest | A resting shot changes by less than a pixel per frame. 30 and 20 divide a 60 Hz display evenly, so there is no judder; 24 does not. World and DOM still step every frame, so callouts stay glued to the picture |
| **How big** the scene is | The scene is drawn into an off-screen texture at a fraction (`scale`) of the canvas and stretched up with a light sharpen. `scale` is chosen so the frame costs about its budget: `moveMs` 4 / `eventMs` 7 / `restMs` 10 | Small while moving (motion hides it), large at rest (where the eye looks). A fixed canvas plus a smoothly varying scale never pops: shrinking is quick, growing is slow, like a rack focus |
| **How dear** a pixel is | Measured, not assumed: a synchronous slope between two small sizes, once at start and again the first time the camera rests on each shot (shots differ in cost). A display-side check (`overrun`) raises the estimate if frames come in under ~40 fps | Open loop, see "What was tried" below for why not a closed one |

Two more decisions sit beside these three:

| Decision | Rule | Why |
|---|---|---|
| **What is traced at all** | Each camera key says which groups it can see (`g`); the shader skips the rest; a move traces the union of its two keys | The cheapest pixel is the test never run. Verified, not assumed: `__cine.cull()` compares the authored groups with everything, pixel by pixel |
| **What gives way, in what order** | Resolution down to its floor; then the quality step (`uLow`: fewer cloud octaves, no quay reflection); then, only while moving, 30 fps | Spend where the eye is. The step removes decoration and never the beat's hero or event; each world decides what its step drops |

Safety rails:
- **Floors.** `minScale` is `[0.35, 0.5, 0.5]` for move / event / rest. Below ~0.5 edges stair-step;
  motion hides it, stillness does not. At the move floor, if frames still overrun, moving frames drop to
  `moveFloorFps` (30) rather than going blockier. It lifts again after 1.5 s without movement.
- **No upscaler** (the off-screen target failed): the canvas itself is sized to the rest budget, as it
  used to be.
- `saveData`: budgets × 0.6.
- Reduced motion: no loop; one paint per scroll event, at the rest budget.
- Hidden tab / off-screen stage: the loop stops (unchanged).

### The one thing a world must do: `evBusy`

`evBusy` is set in `step()` while any clock event is mid-flight (the outage playing, the held change
moving, the rebuild running). It is what lifts the paint rate from 20 to 30 fps. **When you add or
rename a clock in section 6, add it to that expression**, or its event will play at 20 fps.

## Measured (Chrome, Apple M4 Pro, synthetic load; harness-driven, not a human session)

Estimated GPU busy %, from the calibrated cost × pixels shaded per second. Load = ~24 ms to fill a
5 MP canvas (a mid-range laptop GPU, or a heavier world than the placeholder).

| State | Stock engine | Governed | Change | Picture (scene scale, stock → governed) |
|---|---|---|---|---|
| Rest | 35-40 % | 18-19 % | **−47 to −55 %** | 0.54-0.62 → 0.60-0.81: same or sharper |
| Event | 37-38 % | 23 % | −39 % | 0.52-0.62 → 0.56-0.73: sharper |
| Moving | 37-38 % | 26-27 % | −29 % | 0.52-0.62 → 0.42-0.56: softer, in motion |

(Two independent runs; the spread is the calibration's run-to-run noise.) A third run a day later,
with this GPU measuring heavier (~7 ms/Mpx under the same burn, against 4.5-5: the same machine varies
by session), gave rest −49 % (and sharper), event −23 %, moving −20 %. Event and moving sat at the
scale floor with the quality step on, and the estimate does not credit the step's own saving, so those
two are conservative.

The template as shipped is far cheaper than any real world (~0.07 ms for the whole 5 MP frame on this
GPU), so there the saving is in pixels shaded per second: rest 297 → 98 Mpx/s (−67 %), events −50 %,
moving unchanged (a fast GPU affords full resolution; the governor never goes above 1).

Caveats, stated plainly: one machine, one browser; GPU load is *estimated* from the calibrated slope,
not measured per frame; a real thermal or battery effect was not observed. Run the bench on the
hardware that matters (below) before quoting a number.

### Groups and the quality step, measured

At full scale on the same GPU (median of many frames; the placeholder is near the limit of what can be
timed here):

| Shot | Everything traced | Shot's groups | + quality step |
|---|---|---|---|
| 1 (skyline; the quay set culled) | 0.0140 ms | 0.0149 | 0.0152 |
| 3 (close-up; the city culled) | 0.0142 | 0.0115 (−19 %) | 0.0096 (−32 %) |
| 4 (the decision; the city culled) | 0.0141 | 0.0118 (−16 %) | 0.0113 (−20 %) |

Culling five boxes and a few glows (shot 1) is lost in the noise; culling eight towers is not. The
saving grows with what a world can hide: an interior that hides a whole city is worth far more.

### The tail, not the mean

A per-shot calibration done in one go measured 144-169 ms of frozen page under load: three dropped
frames the first time the camera rested on each shot. Spread over idle frames, one ~2 ms draw each,
never on the frame after a paint, the steps measured 1.7-10.5 ms. Averages hid it; a worst-frame gate
found it. `stats().frames` reports p50 / p95 / p99 / worst; judge those.

## Verify it: `?bench`

Open `page.html?bench` (add `&burn=N` to start under load, and `&debug` for the manual harness). A
panel shows pace, scene vs canvas pixels, shaded Mpx/s and the estimated GPU load, with a **burn**
slider (a synthetic per-pixel load, so you can see what a heavy world or a weak GPU does), toggles for
sharpen and adapt, a **Stock engine / Governed** switch, and **Run A/B tour**, which scrolls through
every shot under each preset and reports Mpx/s, fps and estimated busy. Keep the tab in front for the
tour (a hidden tab pauses `requestAnimationFrame`).

Pick a burn that makes the full canvas cost 15–30 ms (watch "est. GPU ms/frame" at scale ×1.00). That
is where the governor has something to do; at 0 it correctly does nothing about resolution.

## What was tried and rejected (do not retry without new evidence)

1. **GPU timer queries (`EXT_disjoint_timer_query_webgl2`) as a closed loop.** Available in Chrome, but
   in Chrome on Metal each timed frame cost the driver a stall (~2 ms on a frame that otherwise took
   ~0.07 ms) and the result had a floor of several ms regardless of pixel count. A controller steering
   by it drove every state to the minimum scale. Sampling one frame in eight reduced the stall but not
   the floor. Dropped; Safari and Firefox do not offer it anyway.
2. **A running per-pixel cost model from live frame times.** Cost per pixel rises as the scene
   shrinks (fixed overhead), which shrinks it further. A positive feedback loop. The calibration is a
   *slope between two sizes*, so fixed overhead cancels; keep it that way.
3. **Baking the sky into a texture** (the "static layer" idea). The sky was ~30–50 % of the placeholder's
   tiny cost, so it looked promising. Baked into a 512×128 half-float texture and looked up per pixel it
   was **slower** (0.02 → 0.14 ms/frame at 5 MP on this GPU: arithmetic is cheap, dependent texture
   reads are not) and **less faithful** (mean 1.3–4 levels off, up to 58 in cloud cells). Not worth it
   here. If a world's sky is genuinely expensive (many octaves of fbm, volumetrics), measure first: stub
   the sky with a gradient and read `__cine.gpu(400)` before and after; only bake if the saving is
   large *and* the real GPU you target is ALU-bound.
4. **A 24 fps rest rate.** Cinematic, but judders on a 60 Hz display (uneven 2/3-frame cadence). Use 20 or 30.
5. **Measuring with one timed frame, or at tiny sizes.** On a fast GPU a tiny batch is all noise; a
   single sync'd frame carries ~1 ms of round trip. `measureSlope` sizes its big batch to take a few ms,
   takes the best of three (contention only adds time), caps the slope by `ms ÷ pixels` (it can only
   overstate) and asks for a second opinion on an outlier.

## If it is still too heavy: look at the world, not the governor

The governor trades resolution and rate. If the *floor* of quality is still too costly, the scene itself
has to get cheaper. In rough order of payoff:

1. **Early-outs.** Skip the trace for rays that cannot hit anything (above the skyline, past the far
   plane); skip the reflection bounce unless `k > 0.004`; bound each group of primitives with one AABB.
2. **Distance LOD.** Window grids, hash-per-cell lights and fbm octaves are invisible past a few hundred
   pixels of apparent size: fade to an average colour, drop an octave.
3. **Fewer rays, not fewer pixels.** One reflection bounce at most; no soft shadows; analytic glows
   instead of volumetric marching.
4. **Rasterise the geometry, ray-trace only what needs it.** A skyline is instanced boxes. A raster
   pass with emissive window textures, fog, bloom and a planar reflection for the water gives most of the
   look at a fraction of the cost. This is a rewrite of the scene, not a tweak; do it only if the above
   is not enough, and keep the governor (it works on any scene drawn through `drawScene`).
5. **Layered 2.5D parallax.** Pre-render 3–5 depth layers and composite with parallax. Cheapest of all,
   convincing only within a limited camera range, and push-ins need care. Suits a story with a gentle
   rail.
6. **Pre-rendered video / frames.** Cheapest at runtime and can be the best looking, but scrubbing needs
   all-intra frames (large), the live camera and the clocks are lost, and the "no files" rule ends. A
   fallback tier at most.

## Tuning checklist

- Start from the defaults. Change `restMs` / `eventMs` / `moveMs` first; they are the only knobs most
  worlds need.
- Raise `minScale[1..2]` only if the world's edges are soft at its floor and you accept the cost.
- Lower `restFps` to 15 for an almost still world; never below ~12 (water and cloud drift start to tick).
- A world with fast ambient motion at rest (rain, a spinning rotor) needs `evBusy` held true, or a
  higher `restFps`.
- Re-run the bench at every beat after adding a heavy feature. Report what was measured and on what.
