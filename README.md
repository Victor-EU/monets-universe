# Monet's Universe

A walkable painting. Eight works by Claude Monet laid out as one continuous landscape: you start on the Japanese bridge inside *The Water-Lily Pond* at Giverny, and by walking, flying or simply scrolling you pass through the flower garden to the hill of *Woman with a Parasol*, down to the Seine at Argenteuil, along the poplars of the Epte, across the fields to the haystacks, into Rouen under its cathedral, and out to the harbour of *Impression, Sunrise*. Behind the pond stand the two oval rooms of the Orangerie with the *Nymphéas*.

Each place is painted by its own canvas, projected from the spot where Monet stood, and carries the hours he painted it in: drag the hour and the haystacks go from end of summer to sunset to snow.

The whole piece is one HTML file with three.js loaded from a CDN. There is no build step.

## Run it

Serve the folder over HTTP (import maps do not work from `file://`) and open it in a browser with WebGL 2:

```bash
python3 -m http.server 8791
```

Then visit <http://127.0.0.1:8791/>. Any static host works the same way; GitHub Pages can serve the repository root as is.

## Controls

| Action | Desktop | Touch |
|---|---|---|
| Look | drag, or click for pointer lock | drag |
| Walk | W A S D or arrow keys | the pad, bottom left |
| Drift along the promenade | scroll wheel | swipe up or down |
| Fly | F to toggle; W flies where you look, Space or E up, Q down | the first dock button |
| Painting viewpoints | 1 to 8, and 9 for the whole universe from the air | the picture button |
| The hour | the sun button, or `[` and `]` to step | the sun button |
| Photo mode | P | the camera button |
| Canvas overlay | C shows the painting over the view at its true frame | |
| Sound | M | the speaker button |
| Return to the pond | R | the last dock button |

A flyer can land on roofs and walk along them, fall off their edges, and swim in the river, which has a current. Buildings and the cathedral are solid.

## URL parameters

These exist for testing and for reproducing a frame.

| Parameter | Effect |
|---|---|
| `?view=N` | start at viewpoint N (1 to 9), no flight |
| `&hour=T` | set the hour, 0 to 1 |
| `&quality=light` | `light`, `balanced` or `rich` |
| `&still` | freeze time-based motion so screenshots repeat |
| `&cam=x,y,z,yaw,pitch` | put a flyer at a position and look |
| `&s=metres` (with `&yaw=` `&pitch=`) | start at a distance along the promenade |
| `&fly=a,b,t` | a frame mid-flight from viewpoint a to b, at fraction t |
| `&photo` | start in photo mode |
| `&overlay[=alpha]` | the canvas overlay on |
| `&debug` | a live readout of patches, draw calls, triangles and frame time |
| `&bench` | GPU-synced frame times at every viewpoint |
| `&dbg=1..5` | shader debug views of the painting's projection |
| `&noproj` | the procedural paint alone, no canvases |

## Debug API

`window.monet` exposes the state and a few controls for headless checks: `monet.state` (camera, place, hour, fps, draw calls, triangles, patches), `monet.goView(i)`, `monet.setHour(t)`, `monet.setQuality(name)`, `monet.set(x, y, z, yaw, pitch)`, `monet.at(metres)`, `monet.tick(dt)` and `monet.snap(width, quality)`, which returns a JPEG data URL of the current frame.

## Repository layout

```
index.html            the whole piece: markup, CSS and one module script
paintings/            the eight canvases at 2048 px, the Orangerie's panels, and CREDITS.md
DESIGN.md             the design: thesis, geography, the hour system, the paint, the interface
BUILD.md              the build plan (M0 to M8) followed by the progress log, one entry per milestone
.claude/launch.json   the dev server for Claude Code's browser pane (python3 http.server on port 8791)
ref/                  not in the repository: full-resolution originals and web copies of the paintings
```

## Credits and licence

The code is released under the MIT licence, see `LICENSE`. The paintings are by Claude Monet (1840–1926) and are in the public domain; `paintings/CREDITS.md` lists every canvas, its collection and the reproduction it was made from. three.js is used under the MIT licence from jsDelivr.

The experience shape, a first-person world of paint with a dock and painting viewpoints, follows *A town made of paint* (van-goghs-town.surge.sh); see `DESIGN.md` for what is kept and what changed.
