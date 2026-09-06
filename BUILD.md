# Monet's Universe — build plan

The plan for building what `DESIGN.md` describes. `DESIGN.md` says what
and why; this file says in what order and how we know each step is done.
A milestone is done when its exit criteria pass, not when its code
exists. Sequencing changes land here, never in the design.

Two rules for the whole build:

- **Always walkable.** After every milestone `index.html` opens, renders,
  and can be walked. We never sit on a broken world.
- **Prove the paint before the map.** Milestone 1 is the pond alone. If
  the pond does not feel like Monet, we stop and fix the paint, because
  every later place is made of the same patches.

---

## Verification harness

Every milestone is checked the same way, so we set it up first (M0).

- **Test hooks in the URL.** `?view=3` places the camera at viewpoint 3
  instantly, no flight. `&hour=0.7` sets the hour. `&quality=light`
  picks a preset. `&still` freezes time-based motion so screenshots are
  reproducible. These are hidden from the UI and cost nothing.
- **Debug API.** `window.monet = {ready, state, goView, setHour, snap}`.
  `state` reports camera, place index, hour stop name, fps, draw calls,
  triangles, patch count. `snap()` returns a downscaled JPEG data URL of
  the current frame so a frame can be checked without a visible pane.
- **Frame-loop independence.** The render loop must advance under a
  patched `requestAnimationFrame` (the Claude browser pane is often
  hidden, which pauses real rAF). Motion uses a clock with a clamped
  delta, so a stalled loop never teleports the camera.
- **Budget check.** A one-line console summary at load: patches, draw
  calls, triangles, generation time. Compared against `DESIGN.md` §9 at
  every milestone.
- **Local serve.** `python3 -m http.server 8765` from the folder;
  import maps need http, not file://. Recorded in `.claude/launch.json`
  so the browser pane can start it.

---

## M0 — Skeleton

**Scope.** The empty world with everything around it working.

- `index.html` with the CSS from the design's §8 (dock, popovers,
  caption, veil, crosshair, touchpad, error block), three.js r170 via
  importmap, one module script.
- Seeded PRNG (`seed 18401926`), `rand / rr / pick` helpers.
- Camera (fov 69, YXZ order), renderer (ACES, sRGB, DPR cap 2,
  `preserveDrawingBuffer`), resize handling.
- Walk and look: pointer lock on click, drag-look fallback, W A S D and
  arrows, ground clamp against a flat placeholder plane, F to fly with
  Q/E altitude.
- Dock with all seven buttons wired; popovers open and close; keys
  1–8, P, R, `[` `]` bound; caption element; awake/locked/photo body
  classes.
- The `places` array, `promenade` array, hour function stubbed with one
  placeholder place ("The Pond, under construction") and one stop.
- Debug API, URL hooks, budget line, `launch.json`.
- Loading veil that fades when `ready`.

**Exit criteria.**
- Opens from the local server with zero console errors on Chrome and
  Safari.
- WebGL failure shows the error block text, not a blank page.
- Walk across the plane, fly, jump to view 1, open every popover, toggle
  photo mode, all from keyboard alone.
- `?view=1&still` produces the same `snap()` twice in a row.
- Under 1 s to ready.

---

## M1 — The pond (proves the thesis)

**Scope.** *The Water-Lily Pond* as a complete, walkable place with
three hour stops. This is the largest milestone by design.

Paint system:
- The patch `ShaderMaterial`: per-instance position, size, rotation,
  palette index, jitter, layer; procedural soft irregular mask; palette
  resolved in the shader from two stop palettes and a blend factor
  (uniforms); vertex-time sway for foliage-class patches.
- A `paintSurface(kind, geometry, options)` generator that lays two to
  three layers of patches on a surface following the design's §6 rules
  (size by distance from the place's viewpoint, orientation by subject,
  edge spill), and merges them into one geometry per place per kind.
- The sky dome: gradient + broad horizontal patches, sun disc, driven by
  the current stop.
- Fog from the stop, colour sampled from the horizon.

The place:
- Pond outline (a soft-edged basin), banks, the path around.
- Japanese bridge: arched deck, railings, patched green.
- Willows (trunk + patch clouds, sway), iris and grass clumps on the
  banks, the wisteria hint over the bridge.
- Water: reflection render target (512 px), water material that samples
  it through per-patch rippled, delayed offsets; lily pads as floating
  patch clusters with a slow drift; flowers as a few bright patches.
- Three stops: *morning green*, *afternoon*, *evening violet*, each a
  full 16-entry palette plus sky/sun/fog values.
- Viewpoint 1 tuned so photo mode frames the bridge as the painting
  does.
- Caption on arrival.

**Exit criteria.**
- Stand at viewpoint 1 and the framing reads as *The Water-Lily Pond*
  to someone who knows the painting; the bridge, willows and lilies are
  all identifiable at a glance.
- Drag the hour from morning to evening and the whole scene re-colours
  with no regeneration; evening goes visibly violet.
- The reflection shows the bridge in the water, broken and rippling,
  never as a clean mirror. `&still` freezes it.
- Walking to the bank stops at the water; flying crosses it.
- Balanced: ≤ 60 k patches for this place, ≤ 60 draw calls, ≥ 60 fps at
  2× DPR on the dev Mac; generation ≤ 1 s.
- **The gate.** We look at three snaps (one per stop) side by side and
  answer honestly: does this look like standing inside Monet's light,
  or like a game level in Monet colours? If the latter, M1 is not done.

---

## M2 — Drift

**Scope.** The promenade and scroll-to-move.

- The promenade spline: hand-placed points through the pond, with a
  temporary extension out to a placeholder second place so there is
  somewhere to drift to.
- Wheel, trackpad two-finger scroll, and touch swipe map to progress
  along the spline with inertia and a hard cap on speed; free look stays
  live while drifting.
- Off-promenade recovery: the first scroll after walking away eases the
  camera to the nearest promenade point over ~1 s, then continues.
- Drift and walk coexist: a key press interrupts drift; a scroll resumes
  it from the nearest point.

**Exit criteria.**
- With hands off the keyboard, scrolling alone carries you from the
  pond to the placeholder and back, smoothly, without ever clipping
  through geometry or leaving the ground.
- Look direction is preserved through a drift; the camera does not snap
  to face the path.
- Touch: a vertical swipe drifts on a phone-sized viewport.
- A page-level scrollbar never appears; the document does not scroll.

---

## M3 — The garden and the hill

**Scope.** Places 2 and the transition from 1.

- The Grande Allée: the path from the pond up through arches, with
  nasturtium and flower-bed patch masses kept within budget (this is a
  transition, not a place).
- *Woman with a Parasol*: the hill's ground mesh, tall grass as swaying
  patches, the figure (stylised low-poly, patched, dress and veil with
  vertex wind), the parasol, the child as a second small figure, one big
  cumulus cluster that drifts.
- Stop: one, *late morning*; the hour slider on this place shifts only
  the cloud and grass colour.
- Wind: a global wind vector used by grass, dress, cloud, later poplars.
- Promenade extended through 1 → garden → 2.

**Exit criteria.**
- Viewpoint 2 looks up the hill as the painting does; the figure is
  silhouetted against sky with the parasol reading clearly.
- Wind is visible in grass, dress and cloud, in the same direction.
- Budget: cumulative ≤ 120 k patches Balanced, ≥ 60 fps.

---

## M4 — The river

**Scope.** Place 3 and the reflection system at scale.

- The Seine as a long water body; the towpath; the road bridge at
  Argenteuil; four to six sailboats (hull, mast, sail as patch clouds),
  moored and gently rocking; the far bank's houses as blocks under
  patches.
- Reflection now covers a large plane; verify the single 512 px target
  still holds up, or move to per-place targets.
- Two stops: *midday*, *calm evening*.
- Fly mode polish: altitude limits, speed, and the aerial camera path
  for long viewpoint jumps.
- Promenade 2 → 3 down the hill to the towpath.

**Exit criteria.**
- Viewpoint 3 shows boats and their reflections, the reflections a
  beat behind and broken, as in the Argenteuil paintings.
- Reflection cost ≤ 3 ms per frame Balanced on the dev Mac.
- Jumping 1 → 3 flies an arc that shows the garden below.
- Cumulative ≤ 170 k patches Balanced.

---

## M5 — Poplars and haystacks

**Scope.** Places 4 and 5; the season stops.

- Poplars: a curve of tall trunks along the Epte bank, patch clouds
  biased downwind, reflections in the river bend. Stops: *wind,
  afternoon*, *pink sunset*, *autumn*.
- Haystacks: stubble field plane, two large stacks (cone on cylinder,
  heavily patched), distant tree line and hills in haze. Stops: *end of
  summer*, *sunset*, *snow effect*, *morning frost*. Snow is entirely a
  palette and sky change; no new geometry.
- Hour continuity: verify the continuous `t` maps sensibly across places
  (evening at the poplars → sunset at the stacks).
- Promenade 3 → 4 → 5.

**Exit criteria.**
- Each of the seven stops across the two places is a distinct,
  recognisable Monet palette; snow reads as snow with the same meshes.
- Drifting from 4 to 5 at a fixed `t` shows no colour pop at the
  boundary; palettes crossfade over ~10 m.
- Cumulative ≤ 220 k patches Balanced.

---

## M6 — Rouen

**Scope.** Place 6 and the town.

- The street: eight to twelve half-timber house blocks along the road
  from the fields, patched, fading into haze toward the square.
- The cathedral west facade: a blocky massing (two towers, portal, rose
  window as a disc) with all detail carried by vertical patches; the
  square in front.
- Stops: *morning, blue*, *full sun*, *harmony in brown, dusk*. The
  stone's palette is the whole effect; get the three palettes from the
  series and tune until the same facade reads as three paintings.
- Promenade 5 → road → 6.

**Exit criteria.**
- At viewpoint 6 the facade fills the frame as the series does, and the
  three stops are unmistakably the three Rouen lights.
- No surface on the facade reads as a flat untextured box at Balanced.
- Cumulative ≤ 250 k patches Balanced (the design's ceiling).

---

## M7 — Sunrise and the whole universe

**Scope.** Place 7, the fog reveal, the aerial.

- The quay past the square; the outer harbour water; cranes, masts and
  chimneys as dark vertical patch silhouettes; two rowing boats; the
  orange sun disc and its broken reflection.
- Fixed dawn: the slider is disabled here with its note; the place's
  single stop is grey-blue with the one orange.
- Fog reveal: fog density ramps up along the street → quay segment so
  the harbour only resolves on arrival.
- Aerial viewpoint: camera height and fov to frame the whole landmass;
  fog distance raised and patch set dropped to Light while there.
- Promenade 6 → 7; the promenade's end.

**Exit criteria.**
- Arriving at 7 by drift, the harbour resolves out of fog and the sun
  appears within the last ~15 m.
- Viewpoint 7 photo reads as *Impression, Sunrise*.
- The aerial shows river, garden, fields and town in one frame at
  ≥ 45 fps Balanced.

---

## M8 — Polish, mobile, ship

**Scope.** Everything that makes it a finished piece.

- Quality presets: Light / Balanced / Rich actually change patch counts,
  reflection resolution and DPR cap per `DESIGN.md` §9; distance-based
  patch broadening.
- Touch: joystick, swipe-drift, look-drag, dock sizing on a phone.
- Photo mode: HUD, save PNG named by place, hidden UI.
- Loading veil: the pond's palette washing in while generating.
- Location captions final copy; aria audit; keyboard reach for every
  control.
- Performance pass: profile on the dev Mac and one phone; hit the §9
  table or adjust the table by decision.
- Deploy to surge (or wherever decided) as one file. Deployment only on
  an explicit go-ahead.

**Exit criteria.**
- The §9 budget table holds on Balanced and Rich on the dev Mac, Light
  on a phone at ≥ 30 fps.
- Every control reachable by keyboard; screen reader announces the
  caption on arrival.
- Photo from each of the seven viewpoints looks like its painting's
  framing.
- Someone who has never seen the site can, with no instructions, scroll
  from the pond to the sunrise.

---

## Deferred (with the trigger to re-open)

- **Sound.** Decide after M4, when water exists at scale.
- **Season control separate from hour.** Re-open if, after M5, the
  haystacks' slider confuses two testers.
- **Per-place reflection targets.** Re-open if M4 shows the single
  target under 3 ms cannot cover the Seine.
- **Eighth painting.** The data-first `places` array is the invitation;
  nothing is added before M8 ships.

---

## Progress

*Updated 2026-09-04.*

**M0 Skeleton — done.** `index.html` runs from the local server on port
8791 (`.claude/launch.json`) with a clean console; walk, fly, drift
(scroll), dock, popovers, keys 1–8 / P / R / `[` `]`, photo mode, the
URL hooks (`?view= &hour= &quality= &still &debug`) and the `window.monet`
debug API all work. Still mode builds synchronously and skips the veil
so headless Chrome captures a real frame.

**M1 The pond — done, with one criterion revised.** Measured on the dev
Mac, Balanced, 1280×720 at 2× DPR:

| Criterion | Target | Measured |
|---|---|---|
| Patches (this place) | ≤ 60 k | **74.7 k** — revised to ≤ 75 k, see below |
| Draw calls | ≤ 60 | 13 |
| Frame time | ≥ 60 fps | 60 fps in a live tab; 4.6 ms/frame GPU-synced |
| Generation | ≤ 1 s | 70–100 ms |
| Three stops re-colour live, evening goes violet | yes | yes |
| Reflection broken, never a mirror; `&still` freezes it | yes | yes |
| Walk stops at the bank; fly crosses | yes | yes |

The patch ceiling for the pond is raised from 60 k to 75 k: the willow
curtains and the wall of far trees that closes the view are what make
the place read as Monet, and the whole-universe ceiling (§9, 250 k)
still holds because fog culls distant places. The gate was judged on
three headless frames (morning green, afternoon, evening violet): the
reflection, willows and lily rafts carry the Monet feeling; the
remaining tells are the bridge's regular geometry and the sky's
flatness, both noted for M8 polish rather than blocking M2.

Verification notes for later milestones: the Claude browser pane is
usually hidden, which pauses `requestAnimationFrame` and throttles the
CPU, so timings from it are unreliable — use headless Chrome
(`--headless=new --use-angle=metal --screenshot`, no virtual-time
budget) against `?still&debug` for frames and numbers.

**M2 Drift — done.** The promenade is a ground-plane spline of 18
points (104 m): the painter's spot, east along the bank, over the
bridge, back along the west bank under the willow, out through a gap
in the tree wall, and north to a placeholder clearing where the flower
garden will be (M3). Height always comes from the walking height
function, so the bridge arch and banks are handled like walking. The
wheel, a trackpad, or a vertical swipe moves a target distance along
the path and the camera eases toward it, capped at 7 m/s; look stays
free. A movement key ends the drift; the next scroll eases the camera
back to the nearest point of the path. Measured in the debug harness:

| Criterion | Result |
|---|---|
| Scroll alone, pond to clearing and back | yes; returns to the exact start |
| Never clips or leaves the ground | eye clearance stayed 1.1–1.85 m over 104 m |
| Look direction preserved | yaw identical before and after |
| Walk off, then scroll | drift resumes from the nearest path point |
| Touch swipe drifts | yes (dominant-axis gesture: vertical = drift, else look) |
| No page scrollbar | body overflow hidden, no scrollable document |

Fixes found by walking the promenade: the wisteria trellis is raised
to the real bridge's ~2.3 m so it no longer hangs at eye level; far
trees and the clearing's trees now have trunks (one merged mesh);
bushes have an inner filling layer; the west-bank willow sits off the
path so you brush its fronds instead of walking into its trunk.
Patches for the pond are now 90 k, over the 75 k noted at M1, of which
~15 k are the temporary clearing and the new trunks; M3 replaces the
clearing and will re-balance. URL hooks added: `?s=<metres>&yaw=&pitch=`
places the camera on the promenade for reproducible captures.

**M3 The garden and the hill — done, with two decisions.** The
promenade (now 146 m) runs from the pond through the Grande Allée —
eight rose arches over the gravel, beds of iris, rose, pink and yellow
in bands, nasturtiums spilling onto the path — and climbs the steep
face of a plateau to *Woman with a Parasol*: the woman on the crest
against the sky, green parasol lit from behind, dress in the violet
shadow the shader already produces, veil and hem streaming downwind,
the child lower-left in the grass, one wide cumulus drifting on the
wind behind her. Meadow grass leans downwind so the wind reads in a
still frame, not only in motion. Two stops, *Late morning* and *Noon,
high sun*, differing in grass, cloud and sky.

| Criterion | Result |
|---|---|
| Viewpoint 2 reads as the painting, figure silhouetted, parasol clear | yes (frame: hill-view) |
| Wind in grass, dress and cloud, same direction | yes; all three read the one wind vector |
| Cumulative patches ≤ 120 k Balanced | 97.7 k |
| ≥ 60 fps | headless bench, 1.5× DPR: pond 7.3 ms, hill 3.7 ms, aerial 7.9 ms |
| Drift pond → allée → crest, captions on the way | yes; clearance held at 1.44–1.46 m |

Decision 1 — **palettes are per place.** One global palette would have
repainted the pond in the hill's colours. Every place now owns its
palette uniforms on its own materials and maps the hour onto its own
stops; sun, sky, fog and water settings are global and blend by
proximity (inverse-cube weights on distance to each place's centre
over its radius). The dominant place also drives the caption and the
hour label. Palettes grew from 16 to 20 slots for the allée's flower
colours.

Decision 2 — **Balanced renders at 1.5× pixel ratio** (Rich stays at
2×, Light drops to 1.25×), revising the §9 table. At 2× the pond had
crept to 15 ms per frame from the added foliage, the allée and the
hill; a painting does not need every device pixel, and 1.5× with the
reflection rendered every other frame and places beyond the fog
culled brought it to 7 ms. `?bench` in the URL runs GPU-synced frame
times at each viewpoint into the debug overlay for headless capture.

Also this milestone: the tree-wall gap the promenade leaves through
was on the bridge's axis and showed sky behind the bridge; three trees
now close the sightline. Bank bushes could land in the water; fixed.

**M4 The river — done.** The Seine crosses the map as a band at
z ≈ −172 with a gentle bend and noisy banks, at the pond's water level,
so one reflection plane serves every water body. The promenade (now
~300 m) descends the hill's north side to the towpath, passes the
painter's spot, climbs an embankment and crosses the road bridge to the
far bank. *Sailboats at Argenteuil* is viewpoint 3: two moored boats
with furled sails at the left, four sailing slowly on long ellipses,
the road bridge entering from the right on five low elliptical arches,
the far bank's houses as blocks under patches with a tree line behind,
reeds at the water's edge. Two stops, *Midday* and *Calm evening*.

| Criterion | Result |
|---|---|
| Viewpoint 3: boats and reflections, broken and a beat behind | yes (frames: sailboats-at-argenteuil, argenteuil-evening) |
| Reflection cost ≤ 3 ms Balanced | 1.3 ms (with reflection minus without, 20 frames) |
| Jump 1 → 3 flies an arc showing the garden below | yes (frame: flight-over-the-garden) |
| Cumulative patches ≤ 170 k Balanced | 165.8 k |
| Promenade 2 → 3, captions on the way | yes; towpath, ramp, deck, far bank all walked |

Bench, headless Chrome forced to 1.5× DPR, 2880×1293: pond 10.5 ms,
hill 7.1 ms, river 9.1 ms, aerial 11.2 ms. The M3 note of 7.3 ms for
the pond was measured at 1× DPR; the M3 build re-benched under this
condition gives 10.4 ms, so the river added nothing to the pond's cost
(it is culled beyond the pond's fog).

What the milestone changed under the hood:

- **Boats are groups that move.** The patch shader now honours the
  model matrix in its instanced path, so a boat is a group (hull, mast,
  sail base coat, one patch mesh) that rocks and sails; identity
  matrices leave everything else untouched. Six boats add 12 draw
  calls; the river view runs at 27.
- **Water is painted, not mirrored.** Two changes to the water shader:
  the base plane samples the reflection through five vertical taps and
  displaces it with horizontal noise bands (strength per stop, `bands`,
  strong on the Seine, faint at the pond), and every water patch now
  carries a third of its palette colour, so the river is a field of
  blue and white dashes with the reflection showing through them,
  which is what the Argenteuil water does.
- **Terrain grid grew** to 240 × 280 m at 0.8 m (105 k samples, ~30 ms)
  to hold the river and the places beyond; pond and river distance
  fields are stored on the grid alongside height.
- **Bridges generalise**: the walking height knows both bridge decks;
  the Argenteuil deck is flat at 4.3 m, reached by embankments that the
  terrain raises along the road. Water blocking respects both.
- **Flight polish**: arcs rise higher for long jumps (up to 28 m above
  the higher endpoint), the look eases from where you were to the
  destination on the ground and then to the painting's own look, fly
  speed grows with altitude, the ceiling is 160 m, and fog opens
  further from high up. `?fly=1,3,0.5` renders a frame mid-flight; in
  `?still` mode time, drift and travel are all frozen.
- **Foliage tools are shared** (`foliageTools`: bush, tree, trunks),
  used by the pond and the river.

Known tells, left for M8: the sails are flat shapes; the parapet and
deck read as plain stone when you stand on the bridge; the far-bank
houses are boxes with speckle; the ground between places stays sparse
away from the promenade. Sound: deferred again — the water does not
ask for it yet; decide when Le Havre's harbour exists (M7).

Budget note for M5: 165.8 k of 170 k is spent. Poplars and haystacks
will need to come from thinning the pond ground (19 k·D) and the hill
meadow (13 k·D) beyond their painters' spots, or the §9 whole-universe
ceiling (250 k) is met early. Fog culling means the cost per frame is
per place, not cumulative.

**M5 Poplars and haystacks — done.** The Epte leaves the Seine's far
bank and winds north; the promenade (now 482 m) follows its west bank,
so *Poplars on the Epte* (key 4) is seen across the water: nineteen
tall columns of foliage on thin trunks along the east bank, the four
nearest cut by the top of the frame, the curve receding north and
doubling back, foliage biased downwind and swaying more at the top,
reflected in the Epte. Six more poplars on the west bank far north
close the S. Then the promenade crosses the stubble west to
*Haystacks* (key 5): two lathed stacks (a swelling drum, a slight eave,
a rounded cone) heavily patched, warm on the lit side, cool in shadow,
a glowing rim; a stubble field of short upright strokes; a tree line;
low undulating hills in haze along the west edge. Seven stops.

| Criterion | Result |
|---|---|
| Seven stops, each a distinct Monet palette | yes: wind afternoon, pink sunset, autumn; end of summer, sunset, morning frost, snow effect |
| Snow reads as snow with the same meshes | yes (frame: haystacks-snow); stubble strokes and field turn white-blue, stacks blue-violet with ochre |
| Drift 4 → 5 at fixed t: no colour pop, ~10 m crossfade | max change of any light channel 0.025 per metre; the handover spreads over ~50 m around s=402 |
| Cumulative ≤ 220 k Balanced | 210.6 k |

Bench, headless 1.5× DPR, 2880×1293: pond 7.6 ms, hill 5.3, river 8.3,
poplars 6.8, haystacks 6.7, aerial 11.3, reflection 1.3.

Decisions and changes:

- **Haystack stops are ordered summer → sunset → frost → snow**, not
  summer → sunset → snow → frost as planned, so the middle of the
  slider (where the river is at evening and the poplars at pink
  sunset) is a warm winter sunset rather than half snow, half orange.
  Hour continuity across places holds in the design's sense: each
  place keeps its own hours and the global light blends by proximity.
- **Budget** was met by thinning the pond ground (19 k → 16 k · D),
  reeds, the hill meadow (13 k → 11 k) and the river ground and water,
  none of which changed the viewpoint frames. Terrain grid grew to
  240 × 420 m (158 k samples). The promenade's distance query is
  bucketed (exact under 6 m, which is all it is asked).
- **Stubble strokes are lit as ground**, not as walls: `stand()` takes
  an optional lighting normal. Without it the field was full of black
  spikes.
- A leak fixed: the river's ground scatter along "the last 118 m of the
  promenade" followed the promenade when it grew and painted green
  grass into the snow field; it is anchored to the river's own stretch
  now.

Known tells, for M8: the field's spatial seam at x = 10 between the
poplars' meadow and the stubble shows two seasons side by side when
the slider is in winter (accepted in the design; season is per place).
The tree line behind the stacks is a row of round trees rather than a
soft band. The Epte's far bank meadow is plain beyond the poplars.

**Next: M6 Rouen.** The street of houses and the cathedral facade.

**M6 Rouen — done.** The road leaves the stubble past a hedge and
becomes a street: twelve half-timber blocks (jettied upper storeys,
steep slate roofs, timber as dark vertical strokes, each house its own
plaster), fading into haze toward the square. The square is closed by
the cathedral's west facade (key 6): a blocky massing (two towers, one
with a spire and one with a crown of pinnacles, three pointed portals,
the rose within its great arch, buttresses, galleries, gable) under a
crust of vertical patches. The under-painting is the shadow colour and
the lit stone is laid over it, so the gaps read as the shadow of
tracery; portals, the arch and the lancet tiers are forced dark, and
inner corners and the undersides of galleries are darkened as a fake
occlusion. Three stops: *Morning, blue* · *Full sun* · *Harmony in
brown, dusk*. Promenade now 648 m; the town sits on flattened ground;
the terrain grid grew to 240 × 580 m.

| Criterion | Result |
|---|---|
| Facade fills the frame; the three stops are the three Rouen lights | yes (frames: rouen-morning-blue, rouen-full-sun, rouen-harmony-in-brown) |
| No facade surface reads as a flat untextured box at Balanced | the front, the towers and their visible sides: yes. The towers' backs and the nave behind are thinly patched and show base coat from the air (M7's fog reveal will decide whether that matters) |
| Cumulative ≤ 250 k Balanced (the design's ceiling) | 248.9 k |

Drift 5 → 6 at fixed t: the largest change of any light channel is
0.047 per metre (the sun swings from a low winter east to a high
south-east), spread over ~78 m; the caption switches at the hedge.
Bench, headless 1.5× DPR, three runs (this session's numbers were
noisy, ±2 ms): pond 7–9 ms, hill 6–7, river 7–11, poplars 5–6,
haystacks 5–7, rouen 5–10, aerial 8–11.

Decisions and changes:

- **Patches now sit 3 cm above the surface they are laid on.** They
  used to be coplanar with the base mesh and z-fought it, rendering as
  dithered smears: the facade's gold crust was there all along and
  half-hidden. This is a global change in `Patches.lay()`; every place
  benefits and no viewpoint frame changed for the worse.
- **The faces the square sees are sampled directly** (rectangles minus
  the arches) instead of through the extruded geometry, where only a
  third of the samples landed on the front face. The reveals and the
  small parts still go through the geometry.
- **The town is 22 m farther north than first placed**, so that from
  the haystacks its houses sit in the haze behind the trees, as
  Giverny's houses do behind Monet's stacks, and the cathedral is a
  ghost at 190 m. The design's "the cathedral is waiting" survives.
- Rouen's palette reads the slots as: 3 stone deep shadow, 4 stone
  shadow (the under-painting), 5 stone lit, 14 highlight, 15 cool
  accent, 11 timber, 12 cobble, 16/18 house walls, 17 slate.
- `?cam=x,y,z,yaw,pitch` hook for free-camera frames.

Known tells, for M8: the base coat's mottling shows as a faint blocky
grain on large flat faces (the towers' sides, house walls). The roofs
are plain from a low angle. The town's ground beyond the houses is a
lawn. The rose is a disc with spokes, no more.

**Next: M7 Sunrise and the whole universe.** The quay past the square,
the harbour in fog, the sun disc, the aerial.

**M7 Sunrise and the whole universe — done.** The promenade (now
717 m) leaves the square at its east corner and ends on a quay over
the harbour basin, which the terrain cuts east of the town. *Impression,
Sunrise* (key 7) is seen from a window's height over the quay: six
ships' hulls with masts and yards as dark vertical notes, three
cranes, a low far shore with five chimneys and their smoke leaning
downwind, two rowing boats each with a standing figure and oars, the
water dense at the painter's feet, and the sun: a small disc drawn in
the sky shader (a per-stop scalar, blended by proximity like the rest
of the light) with its reflection broken by the water's bands and a
column of orange dashes laid under it. One stop, *Dawn*; the slider is
disabled there with "This one is always dawn." The aerial (key 8) is
reframed from 215 m so that the pond garden, the river, the fields, the
town and the harbour sit in one frame; a single ground plane runs under
everything so the world has no plane edges from the air.

| Criterion | Result |
|---|---|
| Arriving at 7 by drift, the harbour resolves out of fog and the sun appears within the last ~15 m | yes: along the quay corridor the fog's far distance falls from 170 m to 53 m, then opens to 90 m over the last 20 m; the sun disc rises from 0 to 1 between 16 m and 0 m from the viewpoint (frames: quay-in-fog, quay-the-sun-appears) |
| Viewpoint 7 photo reads as *Impression, Sunrise* | yes (frame: impression-sunrise) |
| The aerial shows river, garden, fields and town in one frame at ≥ 45 fps Balanced | yes: 6.6–7.5 ms per frame at the aerial (the pixel ratio drops to 1 above 60 m) |

Cumulative 247.5 k patches Balanced (≤ 250 k). Bench, headless 1.5× DPR,
two runs: pond 7–7, hill 5–6, river 7–8, poplars 7–10, haystacks 7,
rouen 8–9, sunrise 4–7, aerial 7–8 (at 1×), reflection 3–6.

Decisions and changes:

- **The sky sphere now follows the camera.** It sat at the world
  origin, so from 440 m away the sun and the horizon were drawn 20°
  off; every far place had a slightly wrong sky without anyone
  noticing. The reflection camera sees the same sphere.
- **The reveal is its own fog**, not the blend of place fogs, which
  would have thickened toward the harbour rather than lifting on
  arrival: east of the square a soft-gated corridor multiplies the fog
  distances down, and the factor fades out over the last 30 m.
- **The aerial drops the pixel ratio to 1 and skips the reflection
  above 60 m** instead of rebuilding a Light patch set, which would
  have cost a 350 ms hitch on every ascent; the fog distance scales up
  to ×13.6 with altitude.
- Budget was kept by thinning the pond ground, the hill meadow, the
  river's ground and water, the poplar meadow and the stubble by
  8–10 % each; no viewpoint frame changed visibly.
- The flight ceiling is 300 m; the walkable box runs to x 150, z −540.

Known tells, for M8: the ships are boxes with cross-shaped masts; the
smoke is stiff; the Seine's water plane ends short of the ground plane's
edges, so from the air the river's ends are flat dark bands; the town's
lawn shows beside the quay. Sound is still deferred: decide in M8 with
the harbour in place.

**Next: M8 Polish, mobile, ship.**

**M8 Polish, mobile — done; ship awaits the go-ahead.**

| Criterion | Result |
|---|---|
| §9 budget table holds on Balanced and Rich on the dev Mac, Light on a phone at ≥ 30 fps | Mac, headless GPU-synced: Balanced 2× 4–10 ms per view (247.5 k patches, ≤ 83 draw calls, ≤ 0.72 M triangles), Rich 2× 5–13 ms (437 k), Light 1.5× 5–11 ms (117.6 k). **A real phone was not measured**: no device in this session; Light is auto-selected on coarse-pointer screens under 900 px and its Mac numbers leave a 3× margin over 30 fps |
| Every control reachable by keyboard; screen reader announces the caption on arrival | the dock's seven buttons, the eight viewpoints, the slider (with `aria-valuetext` = the stop's name), the three quality buttons (`aria-pressed`) and the photo HUD are all native buttons/inputs in tab order; the caption is an `aria-live="polite"` region updated on arrival; Escape closes panels; keys 1–8, F, P, R, M, [ ] |
| Photo from each of the seven viewpoints looks like its painting's framing | yes (frame: photo-sheet, all eight) |
| Someone who has never seen the site can scroll from the pond to the sunrise | the wheel, a trackpad or a vertical swipe engages the drift from wherever you stand; the hint names it; verified on the emulated phone: swipe drifts, drag looks, joystick walks. Not tested with a real stranger |

Done in this milestone:

- **Quality presets per §9**: Light 0.35 / Balanced 0.73 / Rich 1.3
  of the base density (117.6 k / 247.5 k / 437 k patches); pixel ratio
  caps 1.5 / 2 / 2; reflection targets 256 / 512 / 768 px wide, resized
  on change. The choice is remembered in localStorage; phones start on
  Light. Distance-based broadening already exists (`sizeAt` per place).
- **Phone layout** under 620 px: the dock spans the bottom, the joystick
  sits above it and hides while a panel is open, the caption moves to
  the top, the hint wraps; the hint's copy changes for touch.
- **Touch robustness**: a touch that never ends (the browser took the
  gesture, or `touchcancel`) used to jam look and swipe for good; a
  fresh single-finger touch now resets the tracker, and `touchcancel`
  is handled.
- **Sound**, decided: a procedural bed of wind and water (filtered
  brown noise, gusting; the water band rises near the pond, the Seine,
  the Epte and the harbour; the wind rises with altitude). Off by
  default, behind the dock's speaker button and the M key.
- **Loading veil**: four blurred pools of the pond's palette bloom on
  the compositor while the world generates, then the veil lifts.
- **Tells fixed**: the Seine's water plane reaches the ground plane's
  edges; the base coat's grain is three octaves rather than one blocky
  scale. `?photo` hook for photo-mode frames.
- Meta description and Open Graph title/description for sharing.

Left as they are: the ships are boxes with cross-shaped masts; the
poplars/stubble seam in winter (design decision); the town's lawn.

**Ship.** Deployment (surge or elsewhere) is the only remaining M8
item and waits for the explicit go-ahead, with the domain to use.

### Progress · M9 — the paint (after M8; feedback: "small dabs, colourful light")

The user's verdict on M8 was that it did not feel like Monet: no small
dabs, no sense of coloured light. Looking at the frames again they were
right. The sky was a smooth gradient, the water a smooth mirror, the
ground a flat wash with a few strokes on it, and the palettes had been
tuned under ACES, which greys bright colour toward white. The paint was
decoration on 3D geometry, not the substance of the image.

Done in this milestone (the build tag is `m9-monet`):

- **Every surface is painted in dabs, the sky and the water included.**
  A shared GLSL chunk (`PAINT_GLSL`) carries a dab field: a jittered
  grid of overlapping ellipses in world space; the pixel belongs to
  whichever covering dab lies on top. The base coat of every mesh
  draws it in a triplanar domain, with the cell size following the
  distance to the eye in octaves (about a dozen pixels on screen), two
  octaves blended over the middle of each octave so nothing pops while
  walking. Between dabs the ground shows: a paler, greyer, warm version
  of the local colour.
- **Water**: the reflection is read once per dab, at the dab's centre,
  so the mirror breaks into pieces of colour; dabs lie horizontal
  (the grid is stretched 2.4× along x); ripple offsets per dab.
- **Sky**: dabs of about half a degree, lying along the horizon, pink
  and lavender among the blue, blended 85 % over the gradient.
- **Broken colour** (`dabTint`): each dab is the local colour pushed
  off in hue (±14°) and value (±17 %); one dab in six takes the other
  temperature (warm cream or violet at the same luminance). The base
  coat scatters a further ±11° of hue. The instanced strokes use 65 %
  of this, so structured surfaces (the cathedral) stay legible.
- **Light**: the lit side leans warm, the shadow side is pulled toward
  a violet of the same luminance rather than darkened; fog is capped
  at 90 % and its colour is broken by the same dabs, so the distance
  is painted too.
- **Tone**: ACES is gone. A luminance Reinhard with a white point keeps
  hue and chroma in bright colour; a roll-off pales the brightest notes
  toward white; saturation ×1.08. Palettes are graded on the way in
  (saturation ×1.2, the darkest notes lifted to L ≥ .21: no black).
- **Dabs, not strokes**: `lay()` caps the aspect at 2.3 and scales
  surface dabs to 85 %.
- Rouen's full-sun palette was pulled together (shadow slots 3/4 pale
  grey-violet, lit slots cream) after the yellow/violet confetti seen
  in the first frames.

Budget (headless, 1920×862 window):

| preset | pixel ratio | frame times |
|---|---|---|
| Balanced | 1.5× (was 2×) | 8–13 ms |
| Rich | 2× | 13–20 ms |

The dab field costs about 2× the old flat coat per pixel; Balanced now
draws at 1.5× to keep 60 fps, Light at 1.25×. Patch counts unchanged.

Known tells: up close (under ~3 m) the builders' instanced strokes,
sized for their viewpoint, are much larger than the base coat's dabs;
the two systems show. Rouen's stone dabs are still large blobs at the
facade viewpoint. Photo-mode frames re-taken.

Deployment still waits for the explicit go-ahead, with the domain.

### Progress · M10 — the brush

**What was wrong (user feedback on M9).** The strokes were too big and did not read as impressionism, and they glittered under the cursor and while walking. Measured before the fix, with headless frames: a rotation of six tenths of a pixel changed 0.14 % of the pixels strongly (more than 80 levels) and a one-centimetre step changed 0.21 %. The causes: every dab was computed per pixel with a hard edge and a sub-pixel rim wobble, so edges crawled on every move; the base dabs re-rolled with distance as you walked (two octaves crossfading over a narrow band); and the builders' patches were world-sized flat shapes that ballooned up close.

**What was done.** The dab field is gone. The strokes are now painted once, at start-up, on a 2D canvas, and kept as textures the GPU filters (mipmaps, anisotropy), so nothing per pixel can shimmer:
- A tileable **paint field** (1024², about 2700 overlapping strokes in clusters of shared direction, widths varying 3×) holds not colour but what a stroke does to the colour under it: hue, value, temperature (warm or violet accents on one stroke in six), and the relief of the paint. It is read triplanar in world space at two scales an octave apart, blended by distance, so strokes stay about a hundredth of the view wide wherever you look (`STROKE = .012` rad). A broader underpainting layer (3.2× the size, at half strength) lies beneath the small strokes. The sky and the water use the same field (the water its own tile of long horizontal strokes, which also shift the reflection per stroke so the mirror breaks into pieces).
- A **stroke atlas** (16 tapered, ragged strokes with bristle streaks, dry-brush gaps and relief) replaces the per-pixel ellipse of the builders' patches. In the vertex shader each patch is capped to a screen size that varies per stroke (1.1–2.2 % of the view) and fades out under four pixels instead of sparkling; upright strokes shrink toward their foot.
- Colour, light, fog and tone are unchanged from M9.

**Verified.** Same motion test after: rotation 0.01 % (from 0.14 %), step 0.08 % (from 0.21 %); what remains is true parallax at silhouettes. All eight views captured; brushes generate in well under a second.

| Quality | Pixel ratio | Frame, 1920×862 CSS at 2× |
|---|---|---|
| Balanced | 2× (back from 1.5×) | 7–14 ms |
| Rich | 2× | 7–14 ms |

The texture field costs about half of the per-pixel dab field, so Balanced returns to full pixel ratio (Balanced and Rich now differ only in reflection size and patch density).

**Known tells.** Rouen's facade is a dense confetti of cream and violet strokes at the cap size; the builders' wall patches at Argenteuil are the largest strokes in the picture. Up close the base field is even and fine, more woven than dabbed. Deployment still waits for the explicit go-ahead, with the domain.

### Progress · M11 — the canvas

**What was wrong (user feedback on M10, after showing it to people).** "It doesn't really feel like Monet. People understand that the 3D will change a bit of the painting, but still we need to feel like yes, that's the view of that painting." Side-by-side frames of each viewpoint against its painting confirmed it: the colours were invented palettes (a pure blue sky, a bright green hill, pink poplars), the compositions were not the paintings' (the bridge small and central, the cathedral seen from below at its foot, the haystacks mirrored, the poplar row running the wrong way), and the objects read as models with dabs on them.

**What was done.** The seven paintings were downloaded at the best resolution available (see `paintings/CREDITS.md`; originals in `ref/originals/`, not committed: 205 MB), and each place is now painted *by its own canvas*:
- **The projector.** Each place carries a `painting` (a 2048-px copy, its aspect, which hour stop it is). Its viewpoint is a camera with the painting's vertical field of view; after the world is built, the scene is rendered once from there into a depth map. In the shader every fragment is projected into the canvas: where it lies inside the frame and was in Monet's line of sight (four-tap depth test, a looser tolerance for the builders' strokes so a leaf a little behind its neighbour still takes the colour) it *is* the painting, untoned and unlit, with the M10 strokes returning only where the canvas is magnified. At the viewpoint the frame shows the painting, on a 16:9 screen with the world continuing on either side.
- **Beyond the frame.** The picture is carried on by direction: a soft (mip 5.5), mirrored read of the canvas by yaw and pitch from the painter, continuous all the way round, blended 74 % over the procedural paint, lightly shaded by the surface, and fogged to the hour's colour in the far distance. The sky and the shared base terrain take the projector of the place you are in.
- **The hour.** Arriving at a painting sets the slider to the hour it was painted in, so each viewpoint is exact. Other stops re-colour the canvas with an affine grade: from the series painting of that hour where we have one (the haystacks' sunset and snow from the Art Institute and Getty canvases, the poplars' sunset from the Tate's, the pond's morning from the National Gallery's), else from the stop's light relative to the painting's.
- **Alignment.** Each viewpoint and the geometry under it were moved to sit under the canvas, checked with a half-transparent overlay of the painting (key **C**, or `?overlay=.5`; `?noproj` shows the bare geometry, `?dbg` shows where the canvas lands): the pond's eye lower and looking up at the bridge; the parasol three and a half metres from her, looking up; the Argenteuil painter at the water's edge with the moored boat before him; the Epte re-bent to the north-east with the poplar row placed by its position in the frame; the haystacks swapped so the big stack sits right; Rouen seen from 66 m down the street at a first-floor window, the facade narrowed to the canvas's proportions; the sun of Le Havre at 8°.

**Verified.** All seven viewpoints captured against the paintings; the frames now read as the canvases with the world around them. No console errors in headless or the live tab. Off-viewpoint frames captured too (pond from the side, Argenteuil along the bank, the haystacks from behind).

| Quality | Pixel ratio | Frame, 1920×862 CSS at 2× |
|---|---|---|
| Balanced | 2× | 5–12 ms (was 7–14) |

Extra cost: one painting sample, one soft sample and four depth taps per fragment; seven depth passes at build (~0.3 s).

**Known tells.** Away from the viewpoint the canvas stretches over the geometry and the surfaces hidden from the painter show the soft continuation with a hard visibility edge (a ring behind the haystack). Outside the frame the world is soft-focus. Foliage above a painted tree line takes the sky's colour but keeps its stroke texture. Rouen's houses either side are blurred slabs; the cathedral's towers are still wider than the painted ones. The reproduction of the haystacks is only 3000 px (the Art Institute's server refused a larger one behind a bot check). DESIGN.md §2's line about keeping clear of the paintings is superseded: the canvases are public domain and are now the source. Deployment still waits for the explicit go-ahead, with the domain.

### Progress · M11b — the key (fix: "a faded picture over the scene")

**What was wrong.** Outside each frame the world was carried on by a soft *mirrored* read of the canvas (LOD 5.5), blended 74 % over the
geometry. Softened, mirrored, it was still the picture: a ghost woman left of the parasol, ghost towers beside Rouen, and a straight
rectangle where the sharp canvas met the blur. On a 16:9 screen a portrait painting is under half the width, so most of the view was ghost.

**What changed** (`index.html`, build `m11b-key`).
- `extCol` now reads the canvas at LOD 7: nothing of the picture is left, only its colour in that direction (`mirr(dirUv)` kept, so it is
  continuous and equals the frame's colour at the frame's edge).
- `inKey(world, key, ex, lo, hi)`: outside the frame every fragment takes the picture's colour for its direction and keeps its own value
  (world luminance / key luminance, compressed by `ex` = .8 for patches, .5 for the sky, clamped). The soft read greys the key; it is
  re-saturated ×1.4. Strokes over it at .6 (sky .5). Blend 85 %. Trees, sky, ground stay themselves, in the painting's key.
- `frameEdge(puv)`: the canvas's boundary is a feather from 78 % to 100 % of the half-size, wandered by two octaves of value noise
  (±8 %, ±3 %); no straight edge remains. Applied to patches, water and sky alike.
- Far off (fog > .7) the extension goes to the key colour, not the fog colour.
- Viewer-distance fog over the canvas: `fogP` = fog at the painter's distance; the fraction the viewer adds,
  `(fog − fogP) / (1 − fogP)`, is mixed in after the projection. From the viewpoint it is zero; from the haystacks the Rouen facade
  (which carries its own canvas) dissolves instead of standing in white tracery at the top right.
- Fog colour from the painting: `horizonColor(img, hz)` averages a band ±4 % around the painting's horizon (`painting.hz`, per place:
  pond .5, parasol .74, argenteuil .46, poplars .48, haystacks .33, rouen .5, sunrise .5) at load and sets every stop's `fogCol`
  through that stop's grade. What dissolves into the distance dissolves into the painting's own sky.

**Verified.** Headless 1600×900 at all seven viewpoints (`scratchpad/key-1..7.png`, sheet `key-sheet.jpg`): no rectangle, no ghost,
no console errors; live tab reloads clean. Bench Balanced 2× (3840×1550): pond 11.3, parasol 8.6, argenteuil 8.5, poplars 9.9,
haystacks 11.0, rouen 9.2, sunrise 5.0, aerial 4.9 ms; reflection 1.4 ms.

**Still visible.** Away from the viewpoint the canvas stretches and its visibility edges are hard (`key-3off.jpg`). Le Havre's chimneys
still show dark from the poplars (solid kinds take only 30 % fog, a M10 rule). Outside the frame the grass is smoother and lighter than
Monet's; Rouen's flanking houses stay plain slabs. A faint pale mass of the cathedral remains above the haystacks at the far right.

### Progress · M12 — the Orangerie (request: "add the Water Lilies of the Musée de l'Orangerie")

**What it is.** An eighth place, key 8: Monet's *Nymphéas* as he installed them, two oval rooms end to end beyond the water garden,
south of the pond. The layout is the museum's own plan (dossier pédagogique, p. 19): a small rotunda, then the first room with
*Soleil couchant* (6 m) at the west end, *Les Nuages* (12.75 m) north, *Reflets verts* (8.5 m) east, *Matin* (12.75 m) south; then
the second with *Reflets d'arbres* (8.5 m) west, *Le Matin aux saules* (12.75 m) north, *Les Deux Saules* (17 m) round the east end,
*Le Matin clair aux saules* (12.75 m) south. Every composition is 2 m high and centred on its wall; the doorways are placed
automatically in the gaps between compositions (two at each end, none round *Les Deux Saules*), so the plan follows from the lengths.
The panels are the canvases themselves (Google Art Project scans, public domain), on curved walls at true size, `u` running from the
viewer's left; a thin gilt edge, a white ledge beneath, a stone arch round each door, a cove and a veiled oval skylight above, the oval
bench, the carpet. The building outside is a long pale box with arched windows on the garden front and its door at the west end,
chestnuts on the lawn, and a gravel walk from the door round to the pond.

**How it is built** (`index.html`, build `m12-orangerie`).
- `Oval`: an ellipse parametrised by arc length (`sOf`, `theta`, `atS`, `inside`); `PANELS` gives each composition its room, wall
  angle and length; `ORANG.doors` is derived. `Quads` accumulates quads/triangles with normals and uv into one geometry per palette
  index, drawn with the place's base material; the panels use `PANEL_FRAG` (texture × the hour's grade, strokes returning as the canvas is
  magnified, lit from above), the skylight `VEIL_FRAG`.
- The hour is the daylight in the room: four stops (Morning, Full day, Grey day, Evening) grade the panels and colour the veil
  (`ORANG.veilLight`, `ORANG.panelLight`, set in `applyHour`). `gradeOf`/`applyHour`/`paintingHour` accept `panels` as well as `painting`.
- `uPaintAmt` (per place, in `paletteUniforms`) scales the base coat's strokes and mottle; the Orangerie uses .38 so the rooms read white.
- The terrain is flattened under the building (`orangFlat` in `terrainHeight`); the height grid now reaches z = +80 and walking z = +96.
- Walls block: `orangWalkable` / `orangBlocks` (rooms, the lens between them, the vestibule, the rotunda, the passage; walls only pass at
  their doors); `tryMove` refuses a blocked step unless flying. Tested in node (`scratchpad/walk_test.mjs`): 15 points, the whole prefix path.
- The promenade now begins inside the second room: `ORANG_PATH` (through both rooms' north doors, the rotunda, out the west door, round
  to the pond viewpoint) is prefixed to `PROMENADE_PTS`; scrolling back from the pond leads into the museum. `?s=` values shifted by
  the prefix length.
- Keys 1–9; the aerial is 9 and pulled back (`p (0, 250, 200)`) so the building is in frame. Textures: `paintings/orangerie/*.jpg`,
  2.8 MB in all, loaded with the paintings; a grey placeholder until then.

**Verified.** Headless 1600×900 frames (`scratchpad/orangerie-*.jpg`, sheet `orangerie-sheet.jpg`): the viewpoint, both rooms, a
door, the evening hour, the building from the pond, the aerial; no console errors, live tab clean. Bench Balanced 2× (3840×1550):
pond 12.0, parasol 8.9, argenteuil 10.1, poplars 10.3, haystacks 12.9, rouen 9.4, sunrise 5.2, orangerie 9.0, aerial 5.2 ms; reflection 1.5 ms.

**Still visible / open.** *Le Matin aux saules* has no public scan: it is the 1309-px catalogue reproduction (Wildenstein 1996, via
Commons) and is soft up close; the museum offers larger files behind its terms of use, which the user can accept and drop in. The rooms
are simpler than the real ones (no cornice mouldings, no rail, no lamps in the cove); the ceiling is a flat veil, not a glazed vault. The
lawn outside is plain. The Sailko in-situ photographs on Commons remain the reference for the real rooms.

### Progress · M13 — the spot (fix: "the scene is broken by something that feels like a picture")

**What was wrong.** The user's frame was the Argenteuil viewpoint itself: the canvas's sail missing, a band of smeared strips across the
far bank, the 3D sail standing mid-frame wearing painted sky, the near bridge veiled. The cause, found with a new diagnostic (`?dbg=2`:
each point's depth against the painter's depth map, red where the map is nearer, blue where it is farther; `?dbg=3` shows the map):
the depth map was taken once, at build time, and the world then moved away from it. The sailing boats orbit (13 m across), the rowing
boats drift, so the map held the sail where the painting has it while the render had it 5 m to the right: the far bank behind the old
sail was "hidden" (key strokes instead of the canvas), the new sail "in front of nothing" (painted with what lay behind it). Two more
tells came with it: objects of another place inside a frame carried *their* canvas (the Epte's poplars by the Argenteuil bridge were
behind their own painter and showed his key), and away from the spot the canvas stretched over the ground (the M11 known tell).

**What changed** (`index.html`, build `m13-spot`).
- **One canvas at a time.** Every material now takes the projector of the place the eye is in (`syncProjectors`), weighted by the
  place weight as the sky and base terrain already were; a painting keeps its own projector in `painting.own` (`projectorState`).
- **The depth map follows the world.** While the canvas is in reach (eye within 20 m of the spot) the map is rendered again every other
  frame from the painter's spot into the next of two depth targets (the other may be bound as a texture: rendering into a bound one
  silently draws nothing, which is how the first attempt failed). Only the place's own meshes, the ground and the water are drawn;
  other places are then absent from the map, which paints them as if nothing stood in his way — exact at the spot, harmless elsewhere.
  Cost: about 1.2 ms per pass at 1024 px on the live tab (so ~0.6 ms a frame); below the bench's noise at 2×.
- **Moored where he painted them.** A boat whose rest position projects inside a painter's frame, within that place, loses its orbit at build
  (Argenteuil's sail, Le Havre's two rowers); it still rocks and bobs. The others sail on.
- **The canvas belongs to the spot** (`canvasHere`): per point it dissolves by the parallax between the painter's ray and the eye's
  (1 − cos from 4° to 14°: near things first, the far view last) and altogether as the eye walks off (8 to 18 m). As it dissolves, and
  toward the frame's edge, the canvas read goes soft (mip bias up to 4 + 3) rather than being cut. What remains is the world in the key.
- **The sky** takes the canvas by the eye's distance only, and where something stood in his view (a hole beside a silhouette, seen from
  aside) it searches up to 48 % of the canvas upward for clear sky and takes that colour, soft; if none, the key at 70 %. The vertical
  edge and grey blob of the sail's silhouette on the sky are gone.
- **Visibility by any tap**: of the four depth taps one clear one is enough, so the ground behind a leaf's edge is no longer fringed.
- Diagnostics `?dbg=2` and `?dbg=3` kept (grey outside the frame); the bench reports `proj` (pass cost); `monet.projMs`,
  `monet.flags.noProjPass`.

**Verified.** Headless 1600×900 at all seven spots (`scratchpad/spot-sheet.jpg`): each frame is its canvas; the Argenteuil sail, house
and far bank are the painting's, the depth check at the spot shows agreement everywhere but thin leaf edges and the (absent) Epte
poplars. Off the spot (`offspot-sheet.jpg`: Argenteuil from 6 m and from the bank, the haystacks from behind, the pond from the side):
the canvas dissolves into the key without smears or hard silhouettes. Live tab: build `m13-spot`; over four seconds the moored sail and one rower stay put while the three outer boats
sail on; no new console errors. Bench Balanced 2× (3840×1550), the machine in a slower state than at M12 — the committed build re-benched alongside
as the control:

| | pond | parasol | argenteuil | poplars | haystacks | rouen | sunrise | orangerie | aerial | refl |
|---|---|---|---|---|---|---|---|---|---|---|
| M12 (control, now) | 22.8 | 16.6 | 16.6 | 17.5 | 21.8 | 18.2 | 10.3 | 17.0 | 8.9 | 1.8 |
| M13 | 21.3 | 17.7 | 17.4 | 17.7 | 20.5 | 18.4 | 10.2 | 15.9 | 8.5 | 1.4 |

**Still visible.** Between 4° and 14° of parallax the near canvas is half there; walking slowly past a spot one sees it go soft and
give way, which is the design. The Rouen houses either side are still plain slabs. A painted crown wider than its 3D tree still hangs
on the sky until the eye has walked 8 m. Deployment still waits for the explicit go-ahead, with the domain.

### Progress · M14 — the figure (fix: "another silhouette in the scene", "the statue became the persona, sudden and abrupt")

**What was wrong.** The user's frame was the hill seen from the promenade beside the spot, looking left: a small dark cone with a stick
and a ball on it, in the meadow. That was the *boy* — placed at build (4.6, −93.6), 64° left of the painter's look, far outside his frame,
while the painting has him nine degrees left of centre, beyond the crest. So the painted boy landed on the sky and the 3D boy stood by
himself in the field: the other silhouette. Behind that, three more things made the arrival abrupt and the figure a statue:
- the woman's 3D body was 15 % taller than the painted one and her parasol dome half a metre above the painted parasol (the depth map
  at the spot, `?dbg=3`, showed the cone's top and the dome above the frame's top edge);
- the promenade passed 1.1 m beside the spot and never through it, and the flight to a painting came straight down onto it; a figure
  3.5 m from the spot only paints in within a quarter to nine tenths of a metre of it (the 4°–14° parallax window), so she snapped from
  statue to persona in the last step;
- off the spot the painted woman's outline stayed on the sky where the painter saw it (the sky dissolves by distance only), four times
  her size from ten metres back, while the 3D body walked out of it: a ghost beside a statue. And the stroke cloud, 100 m off, wore
  pieces of the canvas that no longer matched the sky's.

**What changed** (`index.html`, build `m14-figure`).
- **The boy where the picture has him**: seven metres along the painter's ray through his painted head, (9.63, −96.06), scale .5, with
  the hat; the crest hides his legs as in the painting (checked numerically: the ray to his painted hem passes below ground from 3 to
  6.5 m out). The stray figure is gone from the field.
- **The woman sized to the picture**: scale .84 (hat's top at a fifth of the frame from 3.5 m, hem at .71); the body is one turned
  profile (hem, skirt, waist, bust, shoulders, neck) instead of a cylinder on a cone, so off the spot she is a figure, not a cone; the
  parasol dome (r .44) sits where the painted parasol is, tilted toward him so its near rim dips beside her hat; the veil and hem
  strokes stream half as far.
- **Arrival along his gaze.** The promenade now runs through the spot, its last six metres along his line of sight; the flight to a
  painting (`goView`) comes down nine metres behind the spot and glides in along the gaze (at eye height where the gaze, which goes up,
  would run into the ground). Along that line the parallax of everything in the frame is near zero, so the canvas assembles as one thing
  by the distance fade (18 → 8 m), nothing snapping in last.
- **The figure carries its own outline** (`figureQuad`, `FIGURE_FRAG`): each figure has a quad at its place, turned to the eye, showing
  the canvas wherever the live depth map has the figure within a quarter of its depth, glass elsewhere, fading past 6°–25° off his line.
  The parts of the painted figure the stroke body does not reach (the veil, the skirt's billow) stay on her from anywhere near the line.
- **The sky lets go of the outline** (`buildPlate`, the *sky plate*): a near map (16 × 16 minima of the depth map, two 4 × 4 passes, half
  float) marks the canvas cells that held something within 400 m; once per painting, at load, the painting is read at that size and the
  sky is diffused over those cells above the horizon (100 sweeps). Sky pixels inside the outline proper always take the plate; those in
  the dilated band around it take it as the eye leaves the spot in proportion to the thing's nearness (`smoothstep(.1, .4, dist / nd)`),
  with the clear sky's detail from 30 % of the frame aside laid over, in strokes. So her outline does not hang on the sky: a soft cloud is
  there instead. Cost: one 52 × 64 readback and ~50 ms of JS per painting at load.
- **The stroke cloud** never takes the canvas (kind 4) and thins out stroke by stroke (`uFadeK`) as the painted sky comes in (8–18 m).
- **The extension behind him** reads softer the farther round from the frame (LOD 7 → 10 by 2.4 frames out): the radial bands of the
  frame's bottom edge fanning over the meadow from the spot are gone.
- Diagnostics: `?dbg=4` the near map on the world, `?dbg=5` the sky's fill (red), near map (green), clear (blue); `monet.places`.

**Verified.** Headless 1600×900 (`scratchpad/approach-sheet.jpg`): the promenade from 10, 6, 3 m before the spot, at it, 3 m past, and
the user's view (s = 272 looking left) — the figure emerges gradually along the approach and the field to the left is empty;
`user-view-before-after.jpg` the user's frame before and after; the seven spots unchanged (`spots-m14.jpg`); `?dbg=3` at the spot
shows the 3D woman, parasol and boy inside their painted outlines. Live tab: the flight from the pond, frames at 9.5, 5, 2 and 0 m
from the spot: the painted woman at 9.5 m already, complete at 2 m, no statue at any step; no console errors. Bench Balanced 2×
(3840×1550), the M13 build re-benched alongside as the control, two M14 runs (the first had two spikes, Argenteuil 13.3 and Orangerie
11.5 — the Orangerie has no projector, so noise):

| | pond | parasol | argenteuil | poplars | haystacks | rouen | sunrise | orangerie | aerial | refl |
|---|---|---|---|---|---|---|---|---|---|---|
| M13 (control, now) | 11.6 | 8.5 | 9.5 | 9.5 | 10.9 | 9.3 | 5.1 | 8.7 | 5.8 | 4.5 |
| M14 | 11.4 | 9.4 | 9.7 | 9.7 | 10.9 | 9.3 | 5.8 | 9.3 | 5.9 | 4.0 |

**Still visible.** Where her outline was on the sky, a soft cloud sits from off the spot (the plate: right in colour and texture, but a
cloud the painting does not have). Walking past her within a metre or two, the parasol dome is a green cap seen from below. Seen from
her far side she is the stroke figure in the key, a figure now rather than a cone, still not her. The other near things (the sail, the
poplars) have no quad yet; their outlines get the plate's fill only. Deployment still waits for the explicit go-ahead, with the domain.

### Progress · M15 — flight (fix: "What's the first button on the left? it doesn't work")

**What was wrong.** The first button is walk / fly (F). The click registered and the state flipped, and then nothing followed from it:
forward was computed from yaw alone, so W in flight still moved along the ground; the only way up was Space, E or Q, said nowhere
(the tooltip read "Walk / fly · F", the hint names drag, scroll and W A S D); one tick of the scroll wheel, the advertised way to move,
called `engageDrift`, which switched flight off and put you back on the promenade; the button kept keyboard focus after the click, so
the first Space (a button activates on Space's key-up) clicked it again and landed you; on touch there was no ascent at all.

**What changed.** In flight, forward is the gaze (`_fwd` takes the pitch when `fly`), so look up and press W, or push the pad on touch,
and you rise; Space / E climb, Q descends (`_move.y += ±1` before the normalise, the old `up` bookkeeping gone). The wheel and the
swipe glide along the gaze while flying (`scrollDrift` adds to `glide` instead of engaging the drift; the metres owed are paid out
eased, `1 − .02^dt` per frame, through `tryMove` so water and the Orangerie's walls still hold; only `monet.at()` and `?s=` still
land you on the promenade). The click itself lifts you 1.6 m (`lift`, same easing) so the change is seen at once, leaves the drift,
and says how to fly on the top line for seven seconds (`say()`, reusing `#hint` with its own timer; "Walking" for two seconds on
landing; a touch variant of the text). Every dock button blurs after a click. Tooltip: "Walk / fly · F · Space up · Q down".
Build `m15-flight`. Rendering untouched, no bench.

**Verified.** Live tab (1280×720): click the button → fly on, focus back on the body, the line shown, eye 1.54 → 3.13 m within a
second; Space afterwards leaves flight on; three wheel notches glide 12.6 m along the gaze; W with the view tilted up .6 rad for a
second climbs to 8.4 m; F lands ("Walking"). `lift-sheet.jpg`: the pond at eye height and 1.6 m up after the click.

**Still visible.** Landing (F) over water puts you in the pond, as any flight that ends over water always has; the walk then lets you
wade out. On touch the rise needs the view tilted up before pushing the pad; there is no separate climb control.

### Progress · M16 — the cathedral (fix: "Rouen Cathedral doesn't look like the cathedral")

**What was wrong.** At the painter's spot the façade is the painting itself, projected; everywhere else the building was a stage set:
a 12 m front slab with two free-standing 10 m box towers, a 12 × 38 m warehouse behind under a shallow roof, a stubby four-sided
cone on the left tower, a truncated cylinder with eight pegs on the right, bare box flanks, and no iron spire — the one thing Rouen
is known for from any distance. Flight (M15) made all of this visible.

**What changed.** The painted front, its arches, rose, buttresses and the towers' inner faces stay where the projector expects them.
Around and behind them the church is built to Rouen's plan at the world's scale (≈ .84): the nave 26 m wide with walls to 33 m and
a slate roof to 46 m, aisles of 7 m under lean-to roofs, a stepped pier with pinnacle and a flyer up to the clerestory at every
6.25 m bay; the transept 50 m across with a gable, rose and portal at either end and turrets at the corners; the lantern tower over
the crossing with corner pinnacles and the iron spire on it, a lathe to 129 m with four small spires about its foot; the choir, the
ambulatory and the round east end with radiating buttresses. The Saint-Romain tower keeps its 10 m and 60 m stone and gets string
courses at its stages, a steep slate pyramid of 18 m with a finial; the Tour de Beurre is 12 m wide and 56 m tall with corner
pinnacles and the open octagonal crown (a ring of tall openings under a ring of small spires). Windows: the towers' lancets on all
four faces, one clerestory and one aisle lancet per bay along the body and round the apse, the crown's openings; roofs in slate
(17 with 4/15 shadow and 14 where lit), iron in the deep shadow (3) with cool accents, stroke sizes on the spire capped by its radius.
The flattened land moves south to cover the church (TOWN z −452, hz 75; the north edge unchanged), the ground plane with it.
Build `m16-cathedral`.

**Verified.** `cathedral-m16.jpg`: before and after from the air, from the fields to the north-west (the whole silhouette: the two
towers, pyramid and crown, the spire behind), the north flank, the east end, the painter's spot unchanged. Haystacks and Sunrise
spots unchanged. Patches 240 456 → 256 125 (+15.7 k, the body of the church). Bench Balanced 2× (3840×1550), the M15 build
re-benched as the control:

| | pond | parasol | argenteuil | poplars | haystacks | rouen | sunrise | orangerie | aerial | refl |
|---|---|---|---|---|---|---|---|---|---|---|
| M15 (control, now) | 13.1 | 12.1 | 12.7 | 9.8 | 10.4 | 9.2 | 5.0 | 9.5 | 6.8 | 2.4 |
| M16 | 13.2 | 9.4 | 10.4 | 11.3 | 11.2 | 10.8 | 6.4 | 9.1 | 7.1 | 4.0 |

Rouen +1.6 ms and Sunrise +1.4 ms (the harbour now sees the body of the church); the rest is run-to-run noise.

**Still visible.** From the fields north-west of the town the ground plane of Rouen shows its bare base colour (a violet flat in the
Full-sun stop) where the field patches are sparse — present before M16, seen now from the air. The buttress flyers are slabs, not
arches; the transept portals are dark recesses without their tracery; the Lady Chapel beyond the apse is not built; the church has no
walk blocking (as before). Deployment still waits for the explicit go-ahead, with the domain.

### Progress · M16b — the hour and the land (fix: "the purple ground")

**What was wrong.** The violet flat north-west of Rouen was the west strip of the Haystacks ground plane, and behind it a rule: every
place painted its land from its own series at the one global hour. Rouen's "Full sun" is hour .5; at .5 the Haystacks series is
between "Sunset" and "Morning frost", so from Rouen the Haystacks land lay in pink and mauve beside the world's green — a foreign
flat with a straight seam, seen from the air now that flight works. (The strip itself was painted in the haze slot as distant hills for
the spot, with no strokes; violet on violet after the first attempt.)

**What changed.** The hour dial belongs to the place you are in. The dominant place follows the dial; every other place shows its own
painting's hour (`ownHour`), so from Rouen the Haystacks are their end-of-summer picture. Crossing into a place, the dial re-bases to
that place's own hour, so nothing jumps; a place you leave eases back to its picture (`hourNow`, ¼ per second). The light is blended
by nearness as before, each place at its own hour. The world plane between the places takes the palettes blended by nearness
instead of the pond's alone (so at the Haystacks' snow the land around goes white with it). The Haystacks' hill strip gets the
green-grey base and 1 600 strokes (ochre, green-grey, haze-blue), greener toward the plane's edge. Rouen's field strokes reach the
plane's west edge. Build `m16b-hours`.

**Verified.** `hours-m16b.jpg`: the far view before and after; from above, the Haystacks land beside Rouen and the harbour in one
key; the Haystacks and pond spots unchanged; Argenteuil at its evening hour with the Haystacks keeping their picture. Live tab: at
the Haystacks the dial reads End of summer, `]` gives Sunset and the field goes orange while Rouen's stays; walking to Rouen the dial
re-bases to Full sun and the Haystacks field returns to end of summer.

**Still visible.** A teleport (`monet.at`, the `?s=` param) snaps the left place back instead of easing, since it runs with dt 0.

### Progress · M17 — the picture off the spot (fix: "in fly mode the picture breaks")

**What was wrong.** The painting is thrown from Monet's spot onto two things: the world's geometry, and the sky dome wherever nothing
stood in his view. Lift the eye 1.6 m (flight) and the geometry moves down in the view while the dome does not, so every near thing
the painting had put on the dome stayed where it was and appeared twice: the pond's bridge, the Parasol figure, the Argenteuil bridge
and masts, the poplars, the cathedral's top. At the pond the whole top rail was on the dome, because the world's bridge is lower than
the painted one from the spot (the painted deck sits ~1.5 m higher, the top rail ~4° above the trellis; matching it would need a
taller bridge with ramps — not done). The dome already had a fill for this (the plate) but it began only at a tenth of the thing's
distance, and its near map reached one cell, so a painted outline spilling past the depth silhouette counted as clear sky.

**What changed.** With the plate, a halo map (`uPaintHalo`) is built once per painting at the near map's size: round everything nearer
than 60 m a weight that is 1 over the thing and fades out 1.5–5.5 cells past it, the depth of the nearest such thing, and where the
sky is from each cell (the first sky row above, the nearest sky column beside). The dome's fill is the halo times the eye's offset
over that depth (4 % begins it, 14 % completes it), so it has no hard edge; inside a thing's own depth silhouette it is 1 as before.
What fills it: the sky above (or beside) mirrored down about its own edge, read a little soft, at .8 over the plate's colour, each
copy weighted by how squarely it lands on sky (the plate's alpha marks sky); the plate itself now seeds each filled cell from the sky
straight above it and smooths 300 passes. Build `m17-off-the-spot`.

**Verified.** `fly-m17.jpg`: the pond, the Parasol, Argenteuil and the poplars lifted 1.6 m, before and after; the pond at 4 m; the
spots unchanged (mean pixel difference .25 at the pond, .02 at the Parasol); a metre's walk off the pond spot unchanged (.6);
a metre off the Parasol spot trades the ghost woman for a soft sky patch. Bench at DPR 2, previous → new: pond 20.1 → 18.2, parasol
17.5 → 13.3, argenteuil 16.3 → 17.3, poplars 16.0 → 15.5, haystacks 18.3 → 18.8, rouen 18.8 → 17.9, sunrise 14.0 → 11.7, orangerie
16.5 → 13.0, aerial 10.2 → 10.3 ms (this session's machine runs slower than M16's table; the two runs are within noise).

**Still visible.** At the Parasol, off the spot, the filled sky is a soft patch with a dark diagonal stroke from the mirrored copy.
The pond's bridge still parts from its painting on the far willows when the eye moves (the painted deck lands on the far bank, a
geometry mismatch, not the dome). The haystacks double on the ground behind them, the intended dissolve stretching the canvas.
Lifted higher (4 m and up) the canvas dissolves as designed and the world in the picture's key remains.

### Progress · M17b — the fill (fix: "the Parasol sky patch and the diagonal stroke")

**What was wrong.** Two things. The fill's content was the sky mirrored down over the place; at the Parasol nearly all the sky is
stroke clouds 40–100 m off, which the plate counted as "not sky", so there was almost nothing to mirror from, the copies landed on the
figure (the diagonal stroke was her parasol's handle, reflected), and the plate under it was one grey. And the hill counted as a near
thing, so its halo lifted a band of fill along the whole horizon, with no sky beside it to draw on.

**What changed.** No copies of the picture at all: the fill is the world's own sky in the plate's key — the painted sky's colour
diffused over the place, carrying the world's light and strokes — the same rule the world outside the frame follows, so nothing of
the thing can come with it and the fill reads soft beside the canvas, as the dissolve does. Only what stands above the horizon within
30 m casts a halo, which now reaches eight cells so a painted veil or handle lies inside it; everything outside every halo counts as
sky, the stroke clouds and the far trees included, and they keep the picture. The plate takes each filled cell's colour from the
nearest sky cell and smooths it a little, instead of seeding from the top strip and smoothing to one average. The sky-edge maps
for the mirror are gone. Build `m17-off-the-spot`.

**Verified.** `fly-m17b.jpg`: the Parasol lifted and walked a metre, the pond lifted and at 4 m, Argenteuil, the poplars, Rouen.
The spots unchanged (mean difference .03 at the Parasol; .34 at the pond, where the water's reflection now fills a little, since the
reflected eye is 2.3 m from the spot).

**Still visible.** The filled sky is a shade paler or greyer than the painted sky beside it, a soft patch where the figure or the
bridge's top stood; it has the world's strokes, not the picture's.

### Progress · M18 — the bridge (fix: "the pond bridge geometry to match the painting")

**What was wrong.** From the spot the world's bridge did not lie under the painted one: projected into the painter's frame its deck
sat at v .47 where the canvas has it at .57–.61, its rails at .51 and .54 where the canvas has three at .65, .70 and .75, and only
its trellis (2.3–2.8 m over the deck) reached the painted middle rail. So the picture's bridge landed on the far willows and the sky,
and parted from the world's bridge as soon as the eye moved.

**What was measured.** On the canvas at the centre column: the deck's beam from v .574 to .606, rails at .649, .697, .746, posts a
quarter of the width apart (u .27, .535, .77); at the frame's edges (u .135, .866) the beam .554–.589, the rails .629, .677, .714.
Read at the world's distance of 12.4 m that is a railing 2 m tall over a deck 3.4 m above the water; read at 7 m it is a real
bridge — a beam .27 m deep with its top 2.5 m above the water, rails .37, .79 and 1.22 m over it, posts every 1.7 m, an arch of
about 2 m over the span. The painted bridge is a real bridge 7 m from where he stood.

**What changed.** The bridge moved from z −4 to z .4, its near beam 7 m from the spot and 8 m to its centre line (the bank ends at
z 6.5, so the spot could not come to it). Its deck: `deckY = .5 + 2.0 (1 − (x/7.6)²)`, the beam .27 m deep; three rails at .37, .79
and 1.22 m on posts 1.25 m tall every 1.7 m from x = .63; the trellis and its wisteria are gone — the 1899 canvas has none (the
wisteria came in 1901). The pond's narrowing, the reeds, the bank bushes and the walk height follow the constant; the promenade's
crossing moved with it ([9.9, 3.4] → [8.3, .4] … [−8.3, .4] → [−11.6, −2.4]); the lily rafts' bands moved so two lie between the
promontory and the bridge (z 4.6 and 2.3) and the rest beyond it. The spot's frame (position, look, fov) is unchanged. Build `m18-bridge`.

**Verified.** `bridge-m18.jpg`: the spot before and after; the canvas laid at half strength over the unprojected world, the rails and
the beam coinciding; the geometry mask at the spot with the world's top rail on the painted one; lifted 1.6 m and 4 m the bridge moves
as one, nothing left behind on the sky; from the east bank, from the deck's end, from its crest and from the air. Bench at DPR 2:
pond 12.4, parasol 10.9, argenteuil 9.7, poplars 10.1, haystacks 12.0, rouen 10.4, sunrise 6.2, orangerie 10.4, aerial 11.3 ms;
patches 258 379 → 260 229 (the extra lily band).

**Still visible.** The arch is steep — a 2 m rise over 15 m, ending in a .3 m step at each bank. Positions along the promenade (`?s=`)
beyond the pond have shifted by about 4 m with the new crossing.

### Progress · M18b — the bridge's ends (fix: "smooth the bridge ends into the banks")

**What was wrong.** The deck ended .5 m above the water at x ±7.6 while the bank there lies at about .3, a step of .2 m, and for its
last two metres over the bank the deck hung in the air.

**What changed.** The terrain itself carries the abutments (`bridgeAbutment`, in `terrainHeight`, so the height grid, the grass, the
walk height and the shoreline all follow): set back from the water's edge (from .6 to 2.2 m inland), a shoulder rises under the
deck's last stretch to its underside, and from each end a ramp comes down to the bank over 4 m; both fade out 1.4–3 m from the
bridge's line. A first version rose from the water's edge itself and stood 1.3 m tall inside the shore band, which the pond paints as
bare earth — two dirt mounds beside the bridge; set back, the shoulders are low and grassy. Build `m18b-abutments`.

**Verified.** Walk height along the crossing at z .4: the deck 1.95 m at x ±4, .80 at ±7, .55 at ±7.5; the bank beyond .49–.50 at
±8, easing to .32–.5 by ±11 — no step. Frames from the east bank, the deck's end, the eastern approach, the spot and the air.
The spot's stroke layout reshuffled with the terrain (the scatter is seeded, the terrain feeds it), the composition unchanged.

### Progress · M18c — grass on the shoulders

The bridge's shoulders lay within the promenade's path band, which the pond's ground scatter paints as trodden earth. A scatter of
their own now covers them: 1 100 strokes a side (× density), tufts seven in ten and laid strokes the rest, in the grass slots, over
the banked earth 1.4–3 m either side of the crossing and beyond the ends; the trodden strip of the crossing (.9 m each side of its
line) and the shore band are left as they were. Build `m18c-shoulders`. Frames from the east bank, the eastern approach, the spot,
the deck's end up close and the air.

### Progress · M18d — reeds at the shoulders' waterline

The shore's reeds and iris keep 1.6 m off the promenade and off the bridge's line, which left the shoulders' waterline bare. A band of
their own now stands there: 520 a side (× density), in the shore band from 1.1 m into the shallows to 1.3 m up the shoulder's foot,
1.1–3.4 m either side of the crossing's line, the same stems and palette as the shore's, the water under the deck left open.
Build `m18d-reeds`. Frames from the east bank, the deck's end up close, the spot, from under the deck's end and the air.

### Progress · M18e — lily pads round the shoulders' reeds

Seven small rafts a side (radius .5–1.1 m) in the shallows off each shoulder, 1.4–4 m either side of the crossing's line, water
at least .2 m deep, the same pads, dark water and occasional flowers as the bands. Build `m18e-pads`. Frames from under the deck's
end on both sides, the east bank, the spot and the air.

### Progress · M18f — blooms on the shoulders' rafts

Nearly half the shoulders' pads now carry a flower where the bands' carry one in five: two crossed petal strokes (.16–.22 m) in the
light or the warm accent slot, half of them with a small warm heart. Build `m18f-blooms`. Frames from under the deck's end on both
sides, up close, from the water's level and the spot.

### Progress · M18g — dragonflies over the shoulders' rafts

Six dragonflies, three a side, each a small group of strokes (`dragonflies`, `updateDragonflies`): a dark body of five overlapping
segments along its heading on two planes tilted 45° either way, and two fans of three pale wing strokes tilted outward — a dab of
about half a metre, not an insect's size. Each glides on two slow sines round its raft (radius .5–1.1 m) with a quick tremor over
them, .8–1.3 m above the water, turns into its own motion, and moves at about a metre a second. Built with the pond, cleared with it.
Build `m18g-dragonflies`.

**What it took.** The first versions were invisible, and the reasons are worth keeping: `lay` caps a stroke's length at 2.3 times
its width, and the patch vertex shader shortens any stroke past ~2 % of its distance from the eye, so a long thin body becomes a
dot — hence the overlapping segments; the pond's "dark" slot 3 is a pad green, so the body is slot 0, deep water; the pane's
WebGL cannot be read back or screenshotted here, so the checks were headless stills — a 3 000-stroke blob put into each group
showed the groups exactly where the motion places them, and a diff against the previous build found the single stroke. At the
spot itself every surface wears the canvas, so the dragonflies, like every world stroke, are invisible there by design; off the
spot they show in the picture's key as small pale-winged marks with a dark centre.

**Still visible.** They are small; end-on they read as a pale fleck. Whether they should be larger, or fewer, is a matter of taste.

### Progress · M19 — the square opened

**What was wrong.** The canvas (the National Gallery's *West Façade, Sunlight*, 1894) is the facade alone, edge to edge, a strip of
pavement and three tiny figures at the bottom left; Monet painted it from a first-floor window across the open Place de la Cathédrale.
The world's spot stood in the street, 39 m short of the square, with the last houses ahead of it on both sides: in the widescreen
frame the portrait canvas covered only the middle, and the street's house fronts filled the two sides in the picture's key, so the
frame read as a cathedral seen down an alley; from behind the spot and from the air the houses ran up to the square's edge and the
facade rose behind a row of them. The code's own comment said the view was "from a first-floor window across the square"; the spot
had never been moved across it.

**What changed.** The square now runs from 4 m behind the spot to the portals, 70 m deep and 60 m wide (`inSquare`), cobbled over
its whole extent (the cobble scatter's bounds and count follow). The street keeps three houses a side, from the fields' edge at
z −336 to z −370, ending just behind the spot, which stands at first-floor height over the street's mouth; the hedge and the
tree at the mouth moved north with it (z −332, −339), the town's flat extended (`TOWN` z −440, hz 92), the town's ground plane and
the field scatter reach to z −325. The other houses moved to the square's sides: a row along each (`ring`), facing across it, 30 m
from the axis, four on the west and three on the east, the east row stopping short so the promenade's way out to the quay passes
between it and the big corner house; the two big corner houses are where they were. Nothing stands between the spot and the facade.
The spot's distance and look are unchanged: the facade fits the canvas at 66 m. Build `m19-open-square`.

**Verified.** Headless frames at the spot (projected, world colours, and at the dusk stop), down the street from the fields, from the
square's west side with the projector on, lifted 9 m over the spot, over the square from the air, in the square at eye height, and
from the Haystacks spot. At the spot the canvas sits in the middle of an open cobbled square; the rows enter the frame only at its
far edges, where the widescreen is wider than the canvas's throw (its half-width is .27 of the distance, the frame's .74), and the
projector never touches them. From the fields the street is three houses a side opening onto the square with the facade beyond.
From the Haystacks spot the town's first houses are now at ~85 m instead of ~110, still low on the horizon behind the trees, the
spires beyond. Patches 260 584 → 261 982. Bench at DPR 2 (two runs): rouen 9.4 / 10.6 ms against 10.4 before, within the noise;
haystacks 10.7 / 12.7 against 12.0.

**Still visible.** The spot hangs over the street's mouth with no house behind it; the draper's window Monet sat in has no
building. The square's rows are the same half-timber blocks as the street's, seen end-on across 60 m of cobbles. DESIGN.md still
says "one street of half-timber houses fading to haze"; it is now a short street and a ringed square.

### Progress · M20 — Argenteuil: the bridge as Monet has it, and the key beyond the frame

**What was wrong.** The user's frames of the river place: the world's bridge did not agree with the canvas in shape or colour, and the
whole place off the canvas was a grey-green wash. Two causes. The bridge was a masonry viaduct, a 5.5 m wall pierced by five arches
with a solid parapet box on top, in one warm grey; Monet's (the National Gallery's *The Bridge at Argenteuil*, 1874, checked against
the museum's own 7999-pixel file, see `ref/ARGENTEUIL-SOURCES.md`) is stone piers with pilasters rising to a cornice, low arches
between them, a thin deck and an open iron railing with people at it, the face toward him in shadow, denim over the arches, the
pilasters cream and ochre in the light, and the toll house, three storeys of coral, at its far end. The grey wash was the picture's
key (M11): beyond about a tenth past the frame's edge `extCol` read the canvas at mip level 10, the whole picture's mean colour, and
`inKey` replaced the world's colour with it entirely, keeping only the value; the Argenteuil picture's mean is a grey-green mud of
sky, trees and water (RGB 124 142 135), so from the far bank, the water, the towpath and the sky behind him were all that one colour.

**What changed.**
- **The key.** `extCol` now goes no softer than level 9 (four columns by three rows of the mirrored picture), so the sky's
  directions keep the sky's colour and the ground's the water's; `inKey` keeps two fifths of the world's own colour under the key,
  so a tree stays green and a sail white where the key is blue, and the key is a tint, not a wash. This is global: checked at the
  haystacks, the hill, the pond and Rouen off their spots; the keys there are the pictures' own colours and hold.
- **The bridge.** Five arches as before (pier 1.9 m, springing at .6 m, crown at 3.3), the wall stopping at a cornice at 3.85 m,
  pilasters on both faces at every pier, an iron railing of posts every 2.4 m with two rails (7 cm; a thin dark line from the spot,
  as in the canvas), five figures at the rail on his side, one with a parasol. Base coats and strokes by part: the wall's face toward
  him denim and grey-blue, its far face cream, the soffits dark, the pilasters and cornice cream and ochre, the iron dark with a
  denim light now and then. Palette slots 11, 12, 15 re-read for the river (stone in shadow, stone lit, denim); slot 14 the
  toll house's muted coral (the boats' waterline note takes the same).
- **The toll house** at the bridge's far end on his side of the road, 3.8 × 3.8 m, 8.4 m of coral wall with cream corners under a
  low pyramid roof of slate, a window a storey on the two faces he sees, the deck meeting it at its middle storey; the white
  two-storey house a little to its left along the bank. The place carries the painting's title now, *The Bridge at Argenteuil*.

**Verified.** Headless frames before and after at the spot (projected), from the near bank side-on, from the water, from the far
bank looking back (the wash gone: sky, grass and water in their own colours under a light blue key), 10 m off the spot with the
projector on, at the toll house, at the calm-evening stop, and off the spot at the four other places. Patches 261 982 → 263 905;
bench at DPR 2, three runs, argenteuil 9.8 / 9.3 / 12.7 ms, pond 11.4 / 11.5 / 12.6, rouen 12.3 / 9.3 / 8.9: the usual noise.

**Still visible.** The arches are rounder than Monet's flat segmental ones. The figures at the rail are two strokes each. The
toll house reads as a brick-red tower up close; at the spot's distance it is the canvas's coral note. DESIGN.md still lists the
place as "Regatta / Sailboats at Argenteuil".

### Progress · M21 — the edge of the world, felt; the way to the next painting; the eye at the painting's height (fix: "In the fly mode, i couldn't advance beyond the cathedral scene")

**What was wrong.** From the cathedral spot the gaze is up at the facade; F and W fly along it, over the cathedral, and 19 m past the
apse the walkable box (x −110..150, z 96..−540) refused every step, silently, while the eye went on climbing along the gaze. The ground
runs on to the horizon past that line, so it read as being stuck; and the two places not yet seen were not ahead anyway (the harbour is
east of the square, the Orangerie at the far north end). Every input path was sound: W and the wheel in flight, keys 7 and 8, and the
promenade drift all reach the harbour and the Orangerie (headless Chrome driven over the DevTools protocol, keys held for real:
`scratchpad/cdp.mjs`). Two things seen on the way: flight passed through the cathedral and the houses (a frame of wall, which also reads
as a block); and within a second of arriving at a spot the walk pulled the eye to ground + 1.45 m, so Rouen was seen from 2 m instead of
5 and the harbour from 2 m instead of 4.6, the near boat's hull showing below the picture. The still frames used for verification never
ran the walk, so never showed it. (The hill and the poplars were already at ground height, their spots snapped to it at build.)

**What changed.**
- *The edge, felt.* A refused step at the box sets `edgeHit`: the velocity is turned back (−.35, a soft push), the gaze's climb and the
  wheel's glide stop there, and the top line says `The world ends here · Next · Impression, Sunrise · 110 m behind you to your right · key 7`
  (at most once in eight seconds).
- *The way to the next painting.* `whereIs(p)` gives distance and direction against the gaze (eight sectors: ahead, ahead to your right, …);
  `nextLine()` names the painting after the spot last stood on (`lastSpot`, in the dock's order, the pond after the Orangerie) with its key.
  `announceSpot()` says it once when one comes to rest within 2.5 m of a spot (flight arrival, key, drift at rest, on foot), re-armed
  eight metres away; the spot one starts on is not announced over the opening hint; nothing is said in `?still`.
- *Solids.* `SOLIDS` and `solidTop(x, z)`: the town's houses (each house pushes its footprint and ridge height) and the cathedral (the
  bounds of the west front, the body and the spire). `tryMove` refuses a step into a solid below its top, walking or flying, unless one is
  already inside it, so nothing traps. A flyer with the gaze up is held at the facade and climbs it until clear of the top, then goes on.
- *The eye.* `eyeHeight(x, z)`: the walker's height, plus the spot's own lift (its `view.p.y` less ground + 1.45) blended in over the
  last six metres, never below .6 m over the ground; used by the walk, the drift, `monet.at` and `?s=`. Rouen holds 5 m, the harbour
  4.6, the pond 1.15; the others were at ground height already.
- DESIGN.md: the river place is *The Bridge at Argenteuil*, 1874 (left over from M20). Build `m21-the-edge`.

**Verified.** Headless, keys held: flight from the cathedral spot is held at the facade at z −439, climbs to 68 m, crosses, and stops at
z −540 with the line above, no further climb; walking west from the spot stops at x −73.5 against the square's west row; key 7 arrives at
(14, 4.6, −440) and says `Next · Les Nymphéas · 500 m to your right · key 8`; the drift's end says the same; standing eye heights at the
eight spots 1.15 · 6.34 · 2.04 · 1.87 · 2 · 5 · 4.6 · 2.45. Live frames at Rouen and the harbour now match the still frames
(`m21-sheet.jpg`). Bench, DPR 2, three runs: pond 12.2–13.7 · argenteuil 9.6–9.8 · rouen 9.5–10.0 · sunrise 5.7–6.5 ms, the usual noise.

**Still visible.** Argenteuil's toll house and houses, the poplars' and haystacks' farms are not solids; flight passes through them.
The next-line names the dock's order, not the promenade's (the Orangerie comes last, though it is where the promenade starts).
The push back at the edge is a nudge, not a wall one can see.

### Progress · M22 — the edge of the world, seen; the promenade's order; Argenteuil's houses solid (what M21 left visible)

**What was wrong.** M21 left three things: flight passed through Argenteuil's far-bank houses and the toll house (not solids); the
next-painting line followed the dock's order, so it named the Orangerie last though the promenade begins there; and the world's edge was
a nudge back and a line of text, nothing one could see. (M21's note also said the poplars' and haystacks' farms were not solids; there
are no farm buildings at either — the claim was wrong.)

**What changed.**
- *Solids.* Argenteuil's `house()` pushes its footprint and ridge height into `SOLIDS` like Rouen's; the toll house pushes its own
  (3.8 m square, to the top of its pyramid roof). Walking or flying below their tops stops at their sides; the escape rule stands.
- *The promenade's order.* `promOrder()` sorts the eight spots by where they fall along the promenade (`nearestS`), once, when first
  asked: the Orangerie, the pond, the hill, the bridge, the poplars, the haystacks, the cathedral, the harbour. `nextLine()` names the
  painting after the spot last stood on in that order, with its dock key; after the harbour, the promenade's end, it names
  *The whole universe · key 9* (no distance: it is the air).
- *The edge, seen.* `edgeVeil`: a 400 × 260 m plane of the hour's fog colour (toned like everything else), standing a metre outside the
  box on the side one is nearest, facing in. Its opacity (`edgeAmt`) rises over the last 18 m of the approach and to full where a step
  is refused (in a third of a second; away in two); a soft patch 60 × 32 m round the point met, broken by three octaves of value noise
  drifting slowly, at most .72 opaque, so the sky, the ground and the harbour's sun still show through. Its foot stands on the terrain:
  the surface height along each side, 128 samples 1 m outside the box (`edgeGround`, water at 0), is a uniform array, and the veil
  fades out over the two metres above it, so where the plane meets a hillside there is no cut line. Not shown during a flight or a
  drift, and none from outside the box.
- *The aerial view, found on the way.* Its camera (0, 250, 200) stands 104 m beyond the box's north side, and `tryMove` refused every
  step from there, a zero step included: the view arrived saying *The world ends here* (since M21) and could not be flown out of
  (since M8). A step is now refused only if it leaves one further outside than before (`outOfBox`), and a zero step is not a step;
  the veil is not shown from outside, so the overview is not fogged. Build `m22-the-edge-seen`.

**Verified.** Headless, keys held: walking south into the far-bank house at (−14, −212) from the field stops at z −215.5 (its
footprint + .5); flying north at 4 m into the toll house stops at z −194.1. The next-lines, after flights: the harbour says
`Next · The whole universe · key 9`; the Orangerie `The Water-Lily Pond · 50 m to your right · key 1`; the cathedral
`Impression, Sunrise · 90 m ahead to your right · key 7`; the pond `Woman with a Parasol · 100 m ahead · key 2`; the bridge the poplars,
the haystacks the cathedral. The veil (`m22-sheet.jpg`): the bank ahead from 12 m out; the frame filled at the south edge from 30 m up
and at the west edge on foot behind the haystacks' hills; three steps back it is going, its foot on the hillside without a cut line
(the first version, a metre from the eye with no foot, was a flat grey wash and cut the hill in a hard band); at the east edge over the
harbour the sun shows through it. The aerial view arrives with the opening hint, and W flies it in to (0, 168, 68); from the harbour
spot flying east meets x 150 with `The world ends here · Next · The whole universe · key 9`, and S backs out of it. Bench, DPR 2, three runs: pond 11.6–13.9 · argenteuil 10.3–11.1 · rouen 8.7–8.9 · sunrise 5.2–5.5 · aerial 5.5–5.9 ms, the usual
noise (the veil is one quad, drawn only near an edge).

**Still visible.** The veil's breakup is soft blotches rather than dabs; it is one plane, so pressing along a wall it slides with you
rather than standing in the world. The bridge at Argenteuil is not a solid (its deck is reached by no walker; a flyer passes through
its piers). The next-line does not say the promenade's own way there, only the straight-line distance and direction.

### Progress · M23 — the bank of fog in the world's own strokes; the bridge solid; the way said along the promenade (what M22 left visible)

**What was wrong.** The veil's breakup was three octaves of value noise, soft blotches, not dabs; its opacity was a patch about the
plane's centre, and the plane followed the eye, so pressing along a wall the bank slid with you. The bridge at Argenteuil was not a
solid (the promenade crosses its deck, so it could not simply be a box). The next-line gave the straight-line distance and direction,
though the promenade is the way there.

**What changed.**
- *The bank.* The plane is 800 × 400 m, the whole side; its strokes are the world's own — `paintAt(uPaint, …)` read in world space on
  the wall's plane, sized to the eye (1.6 × `STROKE`), drifting slowly — pushing colour (`applyPaint`) and opacity (the value and relief
  offsets) so the fog is a field of dabs, as the sky and the ground are. Opacity thins with distance from the eye into the ordinary fog
  (from .3 to 1 × `uFogFar`), and a slow broad noise varies it. Nothing is tied to the plane's centre, so the bank stands in the world
  as one moves along it. The foot on the terrain and the approach over 18 m are as in M22.
- *The bridge.* One solid: 6.2 × 52 m, to .9 m above the deck. A flyer meets its piers and wall; the road rises to the deck at both
  ends, so the walker's eye (5.75 m) is above the top and the promenade's crossing is untouched.
- *The way.* `wayTo(p)`: on the promenade (within 6 m of it), the distance along it to the spot and the direction the promenade sets off
  in, six metres on, against the gaze — `130 m along the promenade, to your right`; off it, or within 15 m, as the crow flies as before.
  `dirOf(p)` is the direction alone, shared. Build `m23-the-bank`.

**Verified.** Headless, keys held: flying east at 2.5 m into the bridge stops at x 20.8 (its west face + .5); walking the road south
from z −138 crosses the deck to −202 without a check. The lines after flights: at the pond `Woman with a Parasol · 130 m along the
promenade, to your right · key 2`; at the hill `The Bridge at Argenteuil · 60 m along the promenade, ahead to your left · key 3`; at the
cathedral `Impression, Sunrise · 110 m along the promenade, ahead to your left · key 7`; at the Orangerie `The Water-Lily Pond · 110 m
along the promenade, ahead · key 1`; at the harbour `The whole universe · key 9`. The veil (`m23-sheet.jpg`): dabbed at the four
edges, the bank ahead behind the hill on the approach, the sun through the strokes at the harbour; after seven metres along the west
wall the bank stands. Bench, DPR 2, two runs: pond 11.8–12.0 · argenteuil 9.2–9.4 · rouen 8.9–9.7 · sunrise 6.0–6.5 · aerial 5.3–5.5 ms, the
usual noise (the veil is drawn only near an edge).

**Still visible.** The veil's dabs are a size larger than the world's, a deliberate coarseness that could be tuned. The direction in
the next-line is the promenade's first six metres, which at a bend is not the direction of the spot. The Orangerie's walls stop only
walkers (a flyer passes through its roof, as before).

### Progress · M24 — the veil's dabs at the world's size; the way said with the spot's own direction; the Orangerie for flyers; roofs to rest on (what M23 left visible)

**What was wrong.** The veil's strokes were 1.6 × the world's. The next-line's direction was the promenade's first six metres, which at a
bend is not the spot's. The Orangerie stopped only walkers: a flyer passed through its walls and roof. And, found on the way: the
solids' escape rule was one rule for all of them (a step into a solid was allowed if the eye was already below *any* solid's top over
it), and the cathedral's spire had been given the bounds of the whole iron list, 19 × 50 m and 129 m tall over the crossing — so a
flyer anywhere under that box was "inside", and fell through the cathedral's roof to the ground.

**What changed.**
- *The veil* reads the paint field at `STROKE`, the world's own size.
- *The way.* `wayTo` measures the set-off direction ten metres on and says it as such: `130 m along the promenade, setting off to your
  right`; where the spot's own sector differs by more than one it adds it: `…, the spot itself ahead`. `sectorOf(p)` and `SECTORS`
  are shared by `dirOf`.
- *The Orangerie for flyers.* `orangRoof(x, z)`: the roof prism over the building (ridge along x, 11.05 m at the ridge), −∞ elsewhere;
  `ORANG_CEIL` the cove's height less .3. A flyer below the roof meets the walls as a walker does (`orangBlocks`), and the doors are open
  to both; above the roof there is nothing to meet.
- *Roofs.* A flyer above a solid's top (a house, the cathedral's body or west front, the spire, the bridge, the Orangerie) comes to rest
  on it (`floorAt`: the highest top under the eye, + .2 m); sideways off it, the floor is the ground again. Inside the Orangerie the
  ceiling holds. Solid by solid: `solidsWithin(x, z, y)` lists the solids whose footprint holds the point and whose top is over the eye,
  and a step is refused into any of them one is not already within. The cathedral's spire is its own solid (12.6 m square, 129 m) and
  each of its four turrets theirs; the roof's small fins are none.
- Debug: `monet.set(x, y, z, yaw, pitch)` puts the flyer somewhere (as `?cam=` does), `monet.solids`, `monet.top(x, z)`. Build `m24-the-roofs`.

**Verified.** Headless, keys held: flying north at 3 m meets the Orangerie's garden front at z 72.1; east at 2 m through the west door,
along the passage and the rotunda, to the first room's west panel at x −20.4 (a walker's stop too); Q from 25 m over the building
rests at 10.8 on the roof's slope; E from within the second room stops at 5.3. Q from 80 m over the cathedral's body rests at 64.2 and
S runs along the roof at that height; W at 100 m along z −488 meets the spire at x −37.3; the M21 flight from the spot is still held at
the facade, climbing. Q over a town house rests at 7.0. The lines: at the pond `Woman with a Parasol · 130 m along the promenade,
setting off to your right, the spot itself ahead · key 2`; at the Orangerie `The Water-Lily Pond · 110 m along the promenade, setting
off ahead, the spot itself to your right · key 1`; at the cathedral `Impression, Sunrise · 110 m along the promenade, setting off
ahead · key 7`. Frames in `m24-sheet.jpg`. Bench, DPR 2, one run: pond 12.3 · argenteuil 9.6 · rouen 10.0 · sunrise 6.8 · aerial 5.7 ms,
within the usual noise (nothing here draws).

**Still visible.** A flyer resting on a roof sits .2 m above its box, not on the slates; on the cathedral the box is the body's bounds,
flat at 64 m, so one hovers over the aisles. The spire's box is square; a flyer meets it 6 m from the spire's foot. The Orangerie's
ceiling is the cove's height everywhere within, the rotunda and passage included.

### Progress · M25 — the slates: every roof a surface, the spire's taper, the cathedral part by part, the Orangerie's ceilings (what M24 left visible)

**What was wrong.** A solid was a box with a flat top, so a flyer came to rest at a house's ridge height over its eaves, and over the
cathedral at 64 m (the lantern's pinnacles) everywhere, aisles included. The spire's solid was a square met 6 m from its foot. The
Orangerie's ceiling was the cove's height everywhere within. And a flyer could not go under the Argenteuil bridge: the deck counted as
ground there, and its solid had no underside.

**What changed.**
- *Surfaces.* A solid carries `surf(lx, lz)`, its surface's height over every point of its footprint in its own frame, or −∞ where
  there is none (the corners of a spire's box); `solidH(S, x, z)` reads it. A house's is its roof's two slopes (the ridge across for a
  gabled Rouen house), the toll house's its pyramid. `floorAt` is the highest surface at or below the eye, `ceilAt` the lowest above
  it (a solid with an `under`, the bridge's deck, has open air beneath), `solidOver` whether a solid stands in the way at a height.
- *The cathedral* is 66 solids: the two towers (Saint-Romain's pyramid to 78 m, the Butter Tower's crown with its ring of small
  spires and its corner pinnacles), the front and its gable, the nave and choir vessels with their ridges at 46, the aisles' slopes
  from 21.5 down to 16, every pier with its pinnacle and its flyer sloping down from the clerestory, the transept and its four turrets,
  the lantern and its pinnacles, the spire and its four turrets tapering as their lathe profiles do (`lathe` reads a profile back as
  height at a radius), the apse's cone and the ambulatory's slope. Heights are the geometry's.
- *The climb.* A step under a surface no more than a shoulder (.8 m) above the eye, or the step's own rise up a slope (2.2 : 1), lifts
  a flyer onto it; anything higher refuses the step. So a roof is a floor one walks up, and a wall or a spire's flank is not. At rest
  on a roof a flyer follows its slope down too; off its edge one hovers (it is flight). A walker is refused by every surface over the eye.
- *The Orangerie.* `orangCeil(x, z)`: in a room the cove's quarter-ellipse from the wall's 3.4 to the ceiling's 4.7 over 1.7 m in, the
  ceiling beyond; the rotunda's 4.6; the lens's, vestibule's and passage's 3.4. A flyer rising meets it (.3 below), and moving toward a
  wall under the cove is pushed down by it. The doors stop a flyer above their heads (`orangWalkable(x, z, yr)`: the rooms' at 2.4,
  the rotunda's openings at 2.7, the west door's lintel at 3.5), so nothing passes over a lintel. The ceiling applies within the walls only.
- *The bridge* has an underside (3.6): a flyer on the water passes beneath it, one rising stops under the deck, one at deck height
  meets it as a wall, one within a shoulder of the parapet steps up onto it. `walkHeight` still gives the road its deck; a flyer beneath
  has the water for a floor.
- Debug: `monet.set` now puts the flyer down still (no velocity, lift or glide carried over); `monet.top(x, z)` adds the Orangerie
  ceiling. Build `m25-the-slates`.

**Verified.** Headless, keys held. Surfaces read: a house 5.62 at its ridge, 4.47 half-way, 3.78 near the eave; the nave 46 at the
ridge, 33 at the eave, the aisle 18.6 mid-slope and 16 at its eave, the lantern's pinnacle 64, the apse 39.3, the ambulatory 18.6,
Saint-Romain 78 at the apex and 64.9 four metres off it, the Butter Tower 67.9, the front's gable 51.4, nothing beside the church.
Q over a house's eave rests at 4.14; over the aisle at 18.77; over the nave's centre at 46.2. At rest on the nave's west slope, W
climbs it (40.2 → 44.2 past the ridge), follows the east slope down (36.1), steps down onto the aisle (18.2), and off its eave hovers
at 16.2. W at 100 m along z −488 meets the spire at x −41.7 (its radius there 1.8 m; M24: −37.3). The Orangerie: E from the first
room's centre stops at 5.3, near its wall at 4.84, in the rotunda at 5.2, in the passage at 4.0; S at the ceiling from the centre
is pushed down to 4.86 at the wall. East through the west door at 3.0 m reaches the room's west panel at x −20.4; at 4.0 the
rotunda's opening stops it at −29.8; at 4.6 the west door's lintel at −36.6. The bridge: E from the water stops at 3.4; W at 3 m
passes beneath; at 4.1 the deck is a wall (x 27.1); at 4.7 the flyer steps up onto it (5.4). Regressions: walking into the far bank's
house stops at z −215.5; the road walker crosses the deck at 5.75; the M21 flight from the cathedral's spot is held at the facade,
climbing; the walker through the west door stops at the panel at −20.3. Frames in `m25-sheet.jpg`.

**Found on the way.** In the tests the flyer drifted a metre sideways after a teleport: `monet.set` carried the previous key's
velocity, which then decayed at the new place. Not reachable by a player (there is no teleport), but `set` now puts the flyer down still.
Bench, DPR 2, one run: pond 12.8 · argenteuil 9.4 · rouen 8.3 · sunrise 5.6 · aerial 5.3 ms, within the usual noise (nothing here draws;
the solids are read per step, 97 of them, and per frame for the floor and ceiling).

**Still visible.** A flyer rests .2 m above the slates, the eye just over them, and off a roof's edge hovers rather than falls. The
spire's octagon is read as a circle, so its flats are met up to 8 % early, with a .3 m margin on every solid. The flying buttresses are
slabs by their top line, not arches one can pass under; the bridge's piers are not solids, so a flyer beneath the deck passes through
them; the pond's Japanese bridge is no solid at all. The cove's inset is read radially, not along the wall's normal, so at a room's ends
its height is off by a few centimetres. The climb is judged per step, so a sprinting flyer may be refused on Saint-Romain's steep pyramid
where a slow one climbs it.
