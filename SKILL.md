---
name: cinematic-scroll-story
description: Turn any website or landing page into a cinematic, scroll-driven story — a live, ray-traced WebGL2 scene (no libraries, no video, no images) that the visitor walks through beat by beat, Apple-product-page style, with a camera on a spring, callouts pinned to 3-D points, events that happen on screen, and optional synthesised sound. Invents a NEW world and visual rule for each site from its own message; never re-skins a previous one. Use when the user asks to "make this site cinematic", "animated hero", "scroll story", "Apple-style scroll animation", "WebGL hero", "turn our landing page into an experience", "direct it like Spielberg", or wants an emotional, story-led first screen for a product, service or brand.
---

# Cinematic scroll story

You are the director, the writer and the engineer. The output is one self-contained HTML page (or a
section dropped into an existing one): a sticky full-screen stage whose camera moves through a
fictional world as the visitor scrolls, while one short beat of copy at a time tells the site's story.

**Each site gets its own world.** The engine is reusable; the world is not. A bank is not a factory,
a clinic is not a city at dusk. The worked example in `references/case-study-twilight-factory.md`
shows how one world was derived from one message. Use it to learn how the derivation works, and don't
reuse its imagery: dusk, factory, skyline, "light means people". Do not open with the template's placeholder scene and
retheme it; design the world first, then build it.

## Files

| File | Read when |
|---|---|
| `references/story-method.md` | Always, before writing anything. Explains how to find the story, the world, the rule and the three acts, and how to write the beats |
| `references/world-design.md` | Always, before building. Covers how to invent a world for this site: archetypes, palette, motion, sound, and what a scene can show cheaply in a shader |
| `references/engine.md` | Before editing the template. Covers the architecture and the pitfalls already paid for: jitter, Chrome resolution, camera paths, portrait framing, callouts, clocks |
| `references/verification.md` | Before claiming it works. Covers the `?debug` harness, the shots to take, the `?bench` cost check, a11y, and preview safety |
| `references/performance.md` | Before changing the numbers or the shader, and whenever a page is too heavy. The paint governor, what it saves, what was tried and rejected, and what to cut from a world |
| `references/case-study-twilight-factory.md` | For calibration: one full example of message → world → beats, including every piece of director's feedback that changed it |
| `assets/template.html` | The engine plus a **placeholder** scene that exercises every mechanism. It is scaffolding, not a style |

## Workflow

1. **Read the site.** Fetch or read every page you can: hero, product, proof (case studies,
   numbers, logos, partner badges), process, CTA. Note the one promise, the steps of the method, the
   real proof with sources, the audience's words, and anything that must not be claimed.
   If the site says something is not public (a client name, internal figures), keep it off the page.

2. **Find the story** (`story-method.md`). Write down, in this order:
   - the audience's pain in their own words;
   - the moment that pain *happens* (an event you can show, not a sentence);
   - the method as a few stations;
   - where it can fail, and what it does with the failure;
   - the one decision that stays with a person;
   - the after, in the visitor's terms.
   Map these onto three acts.

3. **Invent the world** (`world-design.md`). Choose a metaphor world that is *native to this site's
   message and audience*.
   - Give it **one visual rule** that carries the meaning, for example what light, colour, water or
     motion stands for. Every effect on screen must obey it.
   - Decide the palette, time of day or weather, the camera's personality, and the sound's
     instrument.
   - Write the rule down in one sentence and check each act against it.
   - The picture is a metaphor. The words name the real things (systems, patients, invoices) and
     never the metaphor.

4. **Show the user before building.** Present:
   - the world;
   - the rule;
   - a beat sheet (one row per beat: shot, on-screen event, copy, callouts, what the viewer should
     feel);
   - two or three alternative worlds in one line each.

   Wait for a pick or adjustments. This is the cheapest place to change direction.

5. **Build** from `assets/template.html` (`engine.md`).
   - **Keep** the engine: scroll → beat param, spring camera, `camAt` paths, clocks, callout
     projection and placement, the paint governor (pace, scale, per-shot calibration, upscaler),
     settle, reduced motion, the no-WebGL fallback, sound plumbing, `?debug` and `?bench`.
   - **Replace** the world: the shader's `sky`, `trace`, `shade` and `glows`; the camera keys `K`;
     the data tables; the clocks' events (and add each clock to `evBusy`, or its event plays at the
     rest rate); the pins; and the copy.
   - Build in passes: static world and hero shot, then every shot framed, then events on clocks,
     then callouts, then sound, then copy polish.

6. **Verify** (`verification.md`).
   - Park the camera at every beat and screenshot it, and screenshot every event mid-way.
   - Check the moves between shots for clipping through geometry.
   - Check phone portrait, reduced motion, the no-WebGL fallback, console errors, and cost: open
     `?bench`, compare Stock with Governed under a load that makes the full canvas cost 15-30 ms.
   - Show the screenshots to the user; don't describe them.

7. **Direct it** (second pass, when asked or before shipping). Review it as a film:
   - a hook in the first 3 s;
   - agitation shown on screen, not stated;
   - tension in the middle: something fails, something waits for a person;
   - a payoff that obeys the rule;
   - shot variety;
   - one idea per beat.

   Then stop-slop the copy.

8. **Ship.** Follow the project's own publishing rules. Commit, push or publish only when asked.
   If the page posts analytics or leads, never load the real file in a preview; see
   `verification.md`.

## Non-negotiables

- **Story before shader.** No code until the world, rule and beat sheet are agreed.
- **A new world per site.** If the plan could be described as "the factory scene with a different
  logo", start again.
- **Show, don't tell.** Every beat has something that *happens* on screen, driven by a clock once
  the camera arrives, and that rewinds when you scroll away.
- **One rule.** If a visual effect contradicts the rule (lights going out where "light" meant good),
  fix the effect or the copy. A confused metaphor loses the heart.
- **Honest numbers.** Every number is measured, labelled as modelled, or quoted with a link to its
  source. The scene itself carries no numbers.
- **Accessible and cheap.**
  - Text stays live HTML.
  - Reduced motion shows static frames.
  - No WebGL means a painted CSS backdrop, and the page still reads.
  - Idle is cheap: the engine paints a resting shot at 20 fps and a playing event at 30, and fills only as
    much of the canvas as its budget affords (4 / 7 / 10 ms per painted frame while moving / in an event /
    at rest). Keep that. If the world is still too heavy, cut the world, not the governor
    (`references/performance.md`).
  - Sound is opt-in, off on every visit, with no files.
- **Camera discipline.**
  - It never passes through walls, never leaves a set mid-sequence without a motivated exit, and
    never parks between two shots.
  - Each shot keeps its subject clear of the text, including on phones.
