# Verification

Evidence, not description: take screenshots at every beat and of every event mid-way, and show them to
the user.

## Preview safety first

- If the page posts to a live endpoint (analytics beacon, lead webhook, pixel), **never open the real
  file** in a browser or preview pane. Copy it to a scratch directory and replace the endpoint URL
  with a dead one, using `sed` or Python. Serve the copy, then delete the copy and any temporary
  launch entry afterwards.
- Some desktop harnesses auto-open an HTML file right after it is written with the Write or Edit
  tools. For a page with a live beacon, edit it with a script instead.
- Serve over http (`python3 -m http.server --bind 127.0.0.1 --directory <dir>` via the harness's
  preview/launch mechanism). `file://` pages often can't be scripted.

## The `?debug` harness

Open `page.html?debug`. A hidden or background preview pane pauses `requestAnimationFrame` and
freezes CSS transitions, so drive frames by hand:

```js
__cine.manual();            // stop rAF; frames advance only when you tick
__cine.park(4);             // hold the camera at beat 4 (fractions allowed: 5.5 = mid-move)
__cine.tick(33, 120);       // 120 frames of 33 ms = ~4 s of clock time; the LAST frame always paints
__cine.tick(16.7, 120, true); // free=true: let the pace decide, so you can count paints
__cine.state();             // { cp, active, ready, canvas:[w,h], pace, scale, scene:[w,h], S:{...clocks} }
__cine.stats();             // paintsPerSec, shadedMpxPerSec, estGpuBusyPct, msPerMpx, scale, slopes, overload...
__cine.totals();            // cumulative { frames, px, estMs }: diff two reads to cost a window
__cine.gpu(20);             // ms per frame through the whole paint path at the current scale (one sync)
__cine.q;                   // the live quality settings (Q); change them and call __cine.refit()
__cine.setBurn(n);          // ?bench only meaningful: synthetic per-pixel load, to emulate a heavier world
__cine.slope();             // ms per Mpx, measured now (run it a few times: the spread is your noise)
__cine.lag(90, 40);         // feed the overrun detector fake slow frames (tests moveFloorFps)
__cine.grid();              // a coarse picture of the next full paint, to compare two builds of one shot
__cine.noUpscaler();        // rehearse the no-offscreen-target fallback
__cine.cull();              // at the current shot: pixels that change without SET / CITY, and with the shot's own groups
__cine.loseContext(600);    // lose the WebGL context, restore it 600 ms later
```

`stats()` also returns `frames` (p50 / p95 / p99 / worst frame interval and frames over 50 ms, real
frames only), `calibrationSteps` (ms each per-shot calibration step blocked the page), `boot` (first
text, scene ready, scene on screen, ms since navigation) and `qualityStep`.

The manual harness runs on its own clock. A background pane also throttles timers and, it seems, the
GPU: timings taken there (stalls, frame tails) are not what a visitor gets. Take those in a visible tab.

Screenshots in a hidden pane may lag one step. Tick 1–2 more frames and take a throwaway screenshot
before the one you keep. `?debug` also disables the opening push and CSS transitions.

## Shot list

1. **Beats.** Park at each integer beat, tick about 2 s, and screenshot. Check that the subject is
   clear of the text, that callouts are placed and not overlapping, and that the text is legible.
2. **Events.** For each clock, screenshot before, mid-way, at the peak and after. For example: the
   outage dark, then the label swap, then recovery; the held item waiting, then signed off.
3. **Moves.** Park at `i + 0.35/0.5/0.65/0.8` for every move with a `via`, or that enters or exits
   a set. Look for no clipping, no blank frames and no skimming.
4. **Opening.** Without `?debug`, scrolled to the top: the frame at 0, 3 and 7 s.
5. **Phone.** At a portrait viewport (375×812), reload and check each beat that has `pm`/`tm`, plus
   the hero.
6. **Desktop.** At 1440×860 (and a 1512×769 laptop in Chrome), check that `state().canvas` is near
   the full device resolution (it is fixed now: resolution is governed through `state().scale`), that
   `stats().slopes` holds plausible, similar numbers, and that a rest shot reaches `scale` 1 on a fast
   machine.
7. **Rewind.** Park forward, then back. Every clock returns to its state for that beat.
8. **Reduced motion.** Emulate `prefers-reduced-motion`: static frames, no push, events at a
   representative state.
9. **No WebGL.** Force `fail()` or block WebGL. The CSS backdrop and text work.
10. **Console.** No errors. The shader info log (`console.warn`) is empty.
11. **Pace and budget.** `__cine.park(n)`, then `tick(16.7, 150, true)` and read `stats()`: a rest shot
    reports state `rest` and ~20 paints/s, a shot with a playing clock event ~30, a scroll (`scrollBy`
    between ticks) `move` and every frame. If an event paints at 20, its clock is missing from `evBusy`.
    Then open `?bench&burn=N` with N chosen so the full canvas costs 15-30 ms, and compare Stock engine
    with Governed (or press Run A/B tour, tab in front). Look at the picture at the rest scale and at the
    move floor: edges should not stair-step at rest. Report the numbers *and the machine*.
12. **Culling.** At every integer beat, at `i + 0.5` and through each event, `__cine.cull()` must report
    `asAuthored.visible` 0. Then look at the `SET`/`CITY` numbers: a group with `visible` 0 at a shot is
    a saving you have not taken yet.
13. **The tail.** In a visible tab, under a load that makes the full canvas cost 15-30 ms, scroll the whole
    story and read `stats().frames`: p95 within about 1.5× the paint interval for the pace, worst under
    ~50 ms, and no calibration step over ~12 ms. A mean that looks fine with a bad p99 is a hitch.
14. **Sustained.** Three minutes on a real phone, resting and scrolling: phones throttle when warm.
    Compare the first and last minute's p95.
15. **Context loss.** `__cine.loseContext()`: text and the painted backdrop stay, the scene fades back,
    `cull()` afterwards still reports `asAuthored.visible` 0.
16. **Boot.** `stats().boot`: the text must be readable well before the scene is ready; record both.
17. **Sound.** No `AudioContext` exists before the click. Each cue fires once; the ambience follows
    its rule and fades when the stage scrolls away. If a real device isn't available, run the paths
    against an `OfflineAudioContext`, and say that it was not heard.

Report what was verified and what was not, for example "reduced motion is handled in code but not
checked in a browser".
