# World design

Every site gets a new world. Derive it from the message; don't pick it from a menu. The archetypes
below are prompts for thinking, not templates. Combine them, invert them, or discard them.

## Derive, don't decorate

Answer these from the site, then let the answers choose the world:

1. **What is the audience's world physically like?** Hospitals, ports, trading floors, farms, labs,
   shops. A literal world is often stronger than an abstract one.
2. **What does the product *transform*?** Something broken to something whole, chaos to order, hidden
   to visible, slow to flowing, or a burden carried by people to a burden taken off them.
3. **What is the human role that stays?** It needs a visible, lit, warm place in the world: a
   person, a booth, a lighthouse keeper, a conductor.
4. **What emotional temperature fits the brand?** Calm and precise, urgent and heroic, warm and
   local, or playful.

## Archetypes to provoke ideas

| Transformation | World ideas | A possible rule |
|---|---|---|
| Hidden risk → visible | Ice field at dawn and its cracks; reef under clear water; old bridge under inspection light | "What you can see, you can fix" |
| Congestion → flow | Canal locks; rail yard; river delta; airport apron at night | "Water/traffic moves when the work is done right" |
| Fragmented → connected | Archipelago and ferries; night-time power grid; mountain relay stations | "A line lights when two places can reach each other" |
| Fragile → resilient | Terraced farm through a storm; lighthouse coast; greenhouse in frost | "Warm colour means sheltered" |
| Manual → automated, human in charge | Harbour with cranes and one control tower; observatory; signal box | "The one lit window is where decisions are made" |
| Growth, compounding | Orchard through seasons; coral growing; city blocks rising | "Green means it pays back" |
| Precision, compliance | Clockwork; a watchmaker's bench; a surveyed landscape with markers | "Aligned means certified" |

Change the time of day, weather and season too. Dusk has been used; so have dawn, fog, snow, monsoon,
underwater, orbit and midday heat haze. Choose what the rule needs. If the rule is about light, use
darkness; if it is about water, choose a reflective surface and low sun.

## What a single fragment shader can show cheaply

The template traces **analytic primitives** in one pass: boxes (`bx`), spheres, ellipsoids, planes,
with one reflection bounce. That covers a surprising amount:
- **Architecture and machines:** boxes, slabs, arrays of units on a line (`units()`-style
  loops walked cell by cell).
- **Terrain far away:** a height function on the sky side, silhouettes, and fbm mountains as
  sky-colour layers.
- **Water:** a plane with a wave normal and Fresnel reflection.
- **Lights:** analytic point glows (`gPt`), segment glows (`gSeg`) for beams and cables, emissive
  window grids from a hash per cell, and flicker via value noise on time.
- **Atmosphere:** height fog, haze toward the horizon colour, clouds as fbm in sky space, stars as
  hashed cells.
- **Transformation:** a building's height, width, damage and lit fraction as `mix(old, new, rebuilt)`
  keyed to a per-element `turn`. Movement is keyed to a clock uniform.
- **People:** small ellipsoids in a lit window read as a person at distance. Don't show faces.

Avoid organic detail up close (faces, foliage, fabric). Frame such things from afar, or imply them
with light and silhouette. Budget about 30 primitives per ray after AABB culling. Group arrays behind
one bounding-box test.

## Palette and type

- The near-black canvas plus the brand's **one** accent works for UI. The sky and world can carry
  natural colour, but nothing clickable uses it.
- Warm versus cool is the easiest emotional lever. Keep warm for the human element if the rule
  involves people.
- Keep text as HTML over a scrim. Use a side gradient behind step copy, and a bottom gradient on
  phones. Check contrast at every beat, because the backdrop changes under the text.

## Sound (opt-in)

Pick one instrument family that fits the world, and synthesise it; no files:
- **Ambience:** filtered detuned oscillators with a slow LFO. It could be a hum, wind, surf or a
  drone.
- **Failure:** a filtered-noise crackle.
- **Decision:** two warm notes.
- **Each restored element:** one pentatonic note stepping up, through a feedback delay.

Master level is about −18 dB into a compressor. Fade out when the stage leaves the viewport or the
tab hides. Sound follows the rule: ambience lives while the thing it represents lives.
