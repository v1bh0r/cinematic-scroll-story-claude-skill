# cinematic-scroll-story

A Claude Code skill that turns a website or landing page into a cinematic, scroll-driven story: a live
WebGL2 ray-traced scene (no libraries, no video, no images) with a spring camera, callouts pinned to 3-D
points, events that play on their own clocks, and optional synthesised sound. It invents a new world for
each site from the site's own message.

The engine is built to be cheap: a resting shot is painted at 20 fps, a playing event at 30, and the scene
is drawn at a resolution chosen to fit a per-state GPU budget. See `references/performance.md`.

## Install

```bash
git clone https://github.com/v1bh0r/cinematic-scroll-story-claude-skill ~/.claude/skills/cinematic-scroll-story
```

Then ask Claude Code to "make this site cinematic" or "turn our landing page into a scroll story".

## Contents

| Path | What it is |
|---|---|
| `SKILL.md` | The workflow: read the site, find the story, invent the world, show a beat sheet, build, verify, direct |
| `references/` | Story method, world design, engine notes, verification, performance, one worked example |
| `assets/template.html` | The engine plus a placeholder scene. Open it with `?bench` to see the cost panel |

## Try the cost panel

```bash
cd assets && python3 -m http.server 4410 --bind 127.0.0.1
# open http://127.0.0.1:4410/template.html?bench&burn=1200 in a front tab
```

`burn` adds a synthetic per-pixel load; pick a value that makes your GPU take 15 to 30 ms for a full frame.
Measured savings (one machine, one browser, estimated GPU load) are in `references/performance.md`.
