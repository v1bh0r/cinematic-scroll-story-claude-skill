# Engine

`assets/template.html` is one file, with no build step and no dependencies. It has eleven numbered
sections. Keep 1's *mechanism* and 4–11, and rewrite the world in 2–3 and the events in 6 (including
`evBusy`, which tells the paint governor a clock event is playing; see `performance.md`).

## How it fits together

```
scroll ─▶ u (0 … beats-1+TAIL) ─▶ camParam(u): hold 45%, ease 55% ─▶ cpT
cpT ─▶ critically damped spring ─▶ cp (the camera's beat position; everything keys off cp, not scroll)
cp  ─▶ camAt(cp): interpolate keys K[i]→K[i+1] (linear / quad via / cubic via+via2 / tvia for target)
     + push-in on arrival (S.rest) + opening push (S.open) + breathing ─▶ camera p, t, fov
cp, clocks S ─▶ uniforms ─▶ one fragment shader draws the world
cp ─▶ groups g (union of the two keys mid-move) ─▶ uGrp: what the shader traces at all
cp, S, scroll ─▶ pace(): move | event | rest ─▶ how often to paint, and what fraction of the canvas to fill
     ─▶ at the scale floor and still over budget: the quality step (uLow), then 30 fps while moving
     ─▶ drawScene into an off-screen texture ─▶ upscale pass (bilinear + sharpen) ─▶ canvas
cp ─▶ beat opacity (words arrive with the shot) ─▶ active beat ─▶ callouts projected from 3-D
```

- **Beats** are `.beat` elements inside a sticky stage. Without JS they are an ordinary column.
- **`K`** has one camera key per beat: `p`, `t`, `f` (vertical fov), and world params (`n` night,
  `l` lights). `pm`/`tm` give portrait framings, `lens:1` marks the hero shot, and
  `via`/`via2`/`tvia` bend the move *into* that key.
- **Clocks `S`** are events that run on time once you arrive and rewind when you leave. They advance
  with `toward(x, target, up, down, dt)`, or accumulate while `|cp - beat| < .12` and reset when
  `> .45`. One-shot events (the outage) fire `delay` ms after the beat becomes active, once per
  visit.
- **Shared tables.** World data needed by both JS (for callouts) and GLSL is kept in one JS array
  and interpolated into the shader source (`${TW.map(...)}`), so the two cannot drift. Anything
  animated that a callout must follow (`heldAt`, a reject on a chute) is written twice, once in
  GLSL and once in JS, with a comment that they move together.
- **Callouts (`pins`)** are `{el|node, beat, at() → [x,y,z], when?(), delay?}`. They're projected
  with the same camera math as the shader, including `cam.s`, the hero lens shift. Each is hidden
  under the beat's text box, off-screen or behind the camera. Its label is placed right, then left,
  then below, then above, taking the first spot clear of labels already placed.

## Pitfalls already paid for (don't relearn them)

| Symptom | Cause | What the template does |
|---|---|---|
| Jitter at start and while scrolling | The camera followed scroll directly; the canvas resized mid-flight; layout was read every frame | Spring-damped `cp`; canvas size fixed after calibration; layout read only in `measure()`; `hud` writes only what changed (`put`, `_o`, `_t`) |
| Jagged, low-res in Chrome only | Calibration timed a single frame; Chrome's GPU-process round trip (~1 ms) looked like GPU cost, so it chose ⅓ of the pixels | Time two sizes in batches, best of three, and take the **slope**; the fixed round trip cancels. Re-measured once per shot (`measureSlope`, `shotSlope`) |
| GPU and fans busy while nothing moves | The loop drew every display frame at one resolution, whether the camera moved or not | `pace()`: every frame while moving, 30 fps for a clock event, 20 fps at rest. The world still steps every frame, so callouts stay glued |
| Sharpness pops when the resolution changes | Resizing the canvas reallocates it | The canvas keeps one size; the scene is drawn into an off-screen texture at a smoothly varying `scale` and stretched up. Shrinks fast, grows slowly |
| A clock event plays choppy | Its clock is not in `evBusy`, so it runs at the rest rate | Add every clock to the `evBusy` expression in `step()` |
| A hitch the first time the camera rests on a shot | The per-shot calibration ran in one go (~150 ms under load) | `calStep()`: one ~2 ms draw per idle frame, never on the frame after a paint; check `stats().calibrationSteps` |
| An object vanishes, or a reflection loses something, after culling | A group was switched off for a shot (or a move) that can see it | Set `g` per key only from `__cine.cull()` (`asAuthored` visible 0), at the shot, mid-move and through its events. Glows have long tails: they count as visible |
| The page goes blank after a GPU reset | The lost-context handler gave up | `webglcontextrestored` rebuilds programs and targets, re-calibrates, fades back in; the text and backdrop stay meanwhile |
| Governor collapses to the minimum scale | A control loop on GPU timer queries (a floor of several ms in Chrome on Metal) or on live ms-per-pixel (fixed overhead grows per pixel as the scene shrinks) | Open loop on a measured *slope*. See `performance.md`, "What was tried" |
| First frame stalls ~250 ms, the page pops | The driver finishes compiling on the first draw | `KHR_parallel_shader_compile`, polled with `setTimeout` (rAF pauses in hidden tabs); first draw on a 64×40 canvas; fade in afterwards |
| Scroll stops between two shots and shows nothing | Scroll can rest anywhere | `settle()`: after 180 ms still (240 ms after touch), glide to the shot you were heading for |
| Camera passes through a wall or skims a roof on a move | Linear interpolation between two good shots | `via` (quadratic) or `via`+`via2` (cubic) bend the path; `tvia` bends the look. Check every move at 0.35 / 0.5 / 0.65 / 0.8 |
| An "exterior shot" mid-sequence felt like a detour | Framing one station from outside the set | Stay inside a set for its whole sequence. Leave only with a motivated exit (following the product out through a door or portal) that carries into the next act |
| Subject lost behind the text on phones | Desktop shots put the subject right of the text, but a phone puts the text full-width at the bottom | `pm`/`tm` portrait framings; `portrait` re-read in `measure()`; horizontal fov floor via `HFOV` |
| Hero title cut at the bottom | `background-clip:text` clips descenders | Padding-bottom and an equal negative margin on the heading |
| The hero subject collides with the title or CTA | A fixed hero framing on different aspect ratios | `lens()` computes a vertical lens shift so `SUBJECT_TOP` lands just under the CTA |
| Negative dt, clocks running backwards | Mixed timestamp sources | Clamp `dt` to [0, 0.1] |
| Events fired every frame (sounds, pins) | Level-triggered checks | Edge-detect on previous values (`prevHold`, `prevOut`, `towerDone`) |
| Shader compile error on a helper | GLSL needs definition before use | Put helpers (`decided()`, `heldAt()`) above their first caller |
| Low-contrast text on a changing backdrop | The scrim is tuned for one shot | A top and bottom scrim, a side gradient that fades in with step beats, `text-shadow`, checked at every beat |

## Reduced motion and fallbacks

- `prefers-reduced-motion` snaps `cp` to the nearest beat and puts each clock at its state for that
  beat. It also disables the push, breathing, opening push and the outage flicker, and freezes time
  at a fixed value.
- If WebGL2 is missing or the context is lost, `.nogl` hides the canvas and callouts. The CSS
  gradient backdrop and the text still work.
- With no JS, beats render as a normal column.

## Performance budget

Read `references/performance.md`: the paint governor (pace, scale, calibration, groups, the quality step), its `Q` knobs, the
measured savings, what was tried and rejected, and what to change in the world if it is still too heavy.
In short: budgets are per state (`moveMs` 4, `eventMs` 7, `restMs` 10 of GPU time per painted frame),
resolution follows them between a floor and the full canvas, and an idle shot is painted at 20 fps.
Cull arrays behind one AABB test (the `units()` pattern: walk only the cells the ray crosses), skip far
groups when the ray cannot reach them, and use one reflection bounce only where reflectivity k > 0.004.
