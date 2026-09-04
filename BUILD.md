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
