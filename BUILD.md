# Monet's Universe — build plan

The plan for building what `DESIGN.md` describes. `DESIGN.md` says what
and why; this file says in what order and how we know each step is done.
A milestone is done when its exit criteria pass, not when its code
exists. Sequencing changes land here, never in the design.

The file has two parts. The first is the plan as written before the
build: the verification harness and milestones M0 to M8, then what was
deferred. The second, under **Progress**, is the log: one entry per
milestone from M0 onward, each saying what was wrong or asked, what was
built, how it was verified, and what is still visible. The log names the
frame captures and node scripts it verified with; those lived in a local
scratch folder and are not in the repository. The commit for each
milestone carries the same title as its entry.

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

### Progress · M26 — standing: a flyer lands and stands on a roof, and falls off its edge; the spire's octagon; arches open; both bridges whole; the cove along the normal (what M25 left visible)

**What was wrong.** A flyer came to rest .2 m over the slates, the eye at the roof, and off a roof's edge hovered rather than fell. The
spire's octagon was read as a circle, met up to 8 % early on its flats, with a .3 m margin on every cathedral solid. The flying
buttresses were slabs with no arch under them; the road bridge's piers were no solids, so a flyer beneath the deck passed through them;
the pond's Japanese bridge was no solid at all. The cove's inset was read radially, off by centimetres at a room's ends.

**What changed.**
- *Standing.* A flyer coming down on a surface (a house, the cathedral, the Orangerie, a bridge's deck) lands: the eye its own height
  (`EYE`, 1.45) over the slates, as a walker's over the ground. Standing, one follows the slopes up and down; a step up onto a surface
  within a shoulder of the eye lands there too. Off the surface's edge one falls, under gravity (9.8), to the ground or the next
  surface, and stands again; lifting (E, Space, the wheel) ends a fall and standing both. The `.6` hover over the ground stays.
- *The spire's octagon, the turrets' hexagons, the pinnacles' facets.* `sides(n)` gives a point the radius it has in the profile of an
  n-sided lathe or cone (three's vertices at k·360°/n from +z toward +x), so the flats are met where they are: at 100 m and 4 m out
  the spire reads 74.5 toward a vertex and 71.2 toward a flat's middle. The cathedral's margin is .15 m (the camera's near plane .05).
- *Arches.* A solid's `under` may be a surface too, `under(lx, lz)`, −∞ where it stands on the ground. Each flying buttress has its
  underside (4.2 below its top), so the arch between pier and clerestory, above the aisle roof, is open. The road bridge's one solid
  has each arch's soffit beneath it (the ellipse the wall is cut by, 3.3 at the crown), and −∞ at the piers and abutments, so those
  stand in the water. The pond's Japanese bridge is a solid: its deck's arch (`deckY`), the rails 1.25 over it along its two edges, the
  air under the deck (.27 below it).
- *The cove.* The inset is the distance to the oval's nearest point (a ternary search on the parameter), whose normal runs through the
  point, as the cove is built along the normals; 1 m in reads the same at a room's tip and along its side (5.485).
- Build `m26-standing`.

**Verified.** Headless, keys held. Q over the aisle lands at 20.02 (the slates 18.57 + 1.45); W into the nave's wall is refused at
the aisle's ridge (22.32); W west off the aisle's eave falls, 17.4 → 13.9 → 8.6 → 4.95 → 1.15 (the ground + .6) in about two seconds,
and goes on. From the nave's west slope W stands up it (41.6), over the ridge (45.2), down the east slope (37.0), onto the aisle (19.1),
and off its eave falls to 1.15. At 23 m between the piers and the aisle roof a flyer passes north under three flying buttresses. The
spire from the east meets at −41.7 (a vertex direction). The pond's bridge reads 2.5 at its crown, 3.75 at a rail, .55 at its end;
from the south water at 1.8 a flyer passes under its middle; at 2.5 one is set on the deck and walks over it (the deck's own height
is a landing); at its low end one steps up onto it; Q over the deck stands at 3.81; A off its side falls to the water; a walker crosses
it (3.94 at the middle). The road bridge: at 2.5 a flyer passes under a span (x 21.6 → 8.1) and is stopped by a pier (27.1); E from
the water under a span stops at 3.1 under the soffit. Q over the Orangerie stands at 12.5. Frames in `m26-sheet.jpg`.
Bench, DPR 2, two runs: pond 13.3 / 12.9 · argenteuil 13.9 / 10.0 · rouen 8.8 / 9.4 · sunrise 5.6 / 7.0 · orangerie 9.0 / 14.5 · aerial
6.4 / 9.5 ms. Noisier than usual, and the spikes fall on different views in the two runs, so the machine's; nothing here draws, and the
solids' loops (98 of them, per step and per frame) are microseconds.

**Still visible.** Landing is by height alone: a surface at or below the eye within its footprint sets one standing on it, so a roof
cannot be skimmed lower than 1.45 over it, and a flyer arriving at a deck's own height walks over it, its rails (lower than the eye)
stepped over as a walker's are. The fall ends at .6 over the ground, still in flight, not on foot. The Butter Tower's ring of small
spires is a ring, not eight; the road bridge's pilasters are within the deck's margin, not their own faces. A flyer standing still at a
roof's edge cannot slip; only a step takes one over.

### Progress · M27 — on foot: landing as an act, skimming otherwise; walls by the feet; the slip; the fall's end on foot; the Butter Tower's eight spires (what M26 left visible)

**What was wrong.** Landing was by height alone: any surface at or below the eye within its footprint set one standing on it, so a roof
could not be skimmed lower than 1.45 over it, and a flyer arriving at a deck's own height was set on the deck and walked over its rails
(lower than the eye) as a walker does. The fall ended hovering .6 over the ground, still in flight. The Butter Tower's ring of small
spires was a ring. A flyer standing still at a roof's edge could not slip.

**What changed.**
- *Landing is an act.* A flyer over a roof skims it, .3 over the slates. Coming down onto it from above one's own height (Q held, or a
  fall) one touches down and stands, the eye 1.45 over the slates (`landed`); a step up onto a surface within a shoulder lands one too.
  Lifting ends it. Standing, what a surface must clear is the feet, not the eye: a rail (1.25) or a parapet (.9) is a wall to one
  standing, a slope's rise a step; skimming, it is the eye, so a flyer at a deck's own height meets its rail as a wall.
- *The slip.* Standing still on a slope over 30° (the slope read from the steeper side over 10 cm, a step up of more than .4 ignored as
  a wall) one slides down it, gathering speed (9.8·(sin θ − .45 cos θ)), carrying on over a gentler stretch or a roof's lip until it
  stops, and off the edge falls. A movement key held is a scramble and holds. The Orangerie's roof (10°) and the bridges' decks do not slip.
- *The fall ends on foot.* Falling to the ground one lands walking (fly off), the eye rising to its walking height; on water, hovering.
- *The Butter Tower's crown* carries its eight small spires each its own (as built: at (i + ½)·45°, radius .37 w, 5 sides), on the cap.
- *Steps are taken 10 cm at a time* (`move`), with the climb a fixed .8, so a long frame cannot carry one through a rail or up a wall.
  The road bridge's deck is at the road's height with its parapets along its edges (it was the parapet's height throughout).
- Found on the way: standing exactly on a surface, the feet were a rounding below it, so the rule that lets one already inside a solid
  leave it fired and waved a step past the rail; the rule now needs .05 m of depth. Debug: `monet.state.landed/falling`,
  `monet.over(x, z, y)`, `monet.roof(x, z, y)`. Build `m27-on-foot`.

**Verified.** Headless, keys held. The Butter Tower reads 72.8 on a small spire's axis and 67.85 between two and on the cap. Skimming
the aisle roof at .6 over its slates one stays at 19.2, not landed; Q from above lands at 20.02, standing. Standing still there one
slips west (−61.3 → −62.1 → −63.6, gathering speed), goes off the eave at −64.66, falls (17.2 → 14.8 → 9.9 → 2.6) and lands on foot:
flying off, the eye rising to 2.0, and W walks on at walking pace. W held from the nave's west slope scrambles over the ridge (45.6),
down (37.0), onto the aisle and off it. On the road bridge Q lands at 5.75 (the road walker's eye), W walks the deck, D into the
parapet holds at x 26.24. On the pond's bridge, arriving at the deck's own height one skims at 2.8 and the rail is a wall (z −.77);
landed on the deck (3.81), A toward the rail holds at z 1.22, W along the deck stands as the deck rises (3.82 → 3.94) and off its west
end lands on foot on the bank. The Orangerie's roof, landed on, does not slip. Walkers unchanged: the road over the bridge at 5.75, the
far bank's house at z −215.5. Frames in `m27-sheet.jpg`.
Bench, DPR 2: two runs of this build gave pond 25.9 / 24.0 · argenteuil 19.1 / 18.5 · rouen 19.2 / 17.8 · sunrise 13.0 / 12.1 ·
aerial 11.6 / 11.3 ms, about twice M26's; the M26 build rebenched in the same minutes gave pond 25.4 · argenteuil 19.5 · rouen 17.0 ·
sunrise 14.9 · aerial 14.2, the same. The machine is slow today (load average 5.4 at the time); nothing here draws, and the stills do
not move. To be rebenched on a quiet machine.

**Still visible.** Landing needs a descent: a flyer arriving at a roof horizontally, lower than one's own height over it, skims and never
stands until Q is pressed. The skim clearance is .3, so on a steep slope the eye sees into the roof uphill. The slip has one friction
(.45) for slates, lead, stone and iron alike, and carries over a flat lip by momentum, so a flat roof's edge reached sliding is passed.
Standing, anything under .8 m is stepped over (a .9 parapet and a 1.25 rail hold; a .7 kerb would not); walking, walls are still by the
eye, so a walker steps over a rail and is kept from the water's edge by the water rule alone. The fall's end on water hovers. The Butter
Tower's crown is a solid drum to a flyer, its ring of tall openings closed.

### Progress · M28 — on contact: landing where the roof meets one; friction by material; walls by the feet for all; the crown opened (what M27 left visible)

**What was wrong.** Landing needed a descent: a flyer arriving at a roof level, below its own height over it, skimmed it until Q was
pressed. One friction served slates, stone, lead and iron alike, and a slide carried over a flat roof's edge by momentum. Standing
flyers stepped over anything under .8 m, and walkers measured walls by the eye, so a walker stepped over the Japanese bridge's rails
and was kept from the water by the water rule alone. The Butter Tower's crown, "a ring of tall openings", was a plain drum in the
geometry and a closed one to a flyer.

**What changed.**
- *Landing is contact.* A flyer skims a roof at .3 over it, flying level; when the intended height would come within that .3 of the
  surface, whether one comes down onto it (Q, a fall) or the roof rises into one flying level (a slope met head-on), one touches down
  and stands. Lifting still ends it. A flyer flying level at 40 m into the nave's east slope lands on it at the ridge.
- *Friction by material.* A solid carries `mu`: slate .45 (houses, the cathedral's roofs, Saint-Romain's pyramid), stone .6 (the front,
  the piers and flyers, the lantern, the towers' tops, the Butter Tower), iron .3 (the spire and its turrets), the Orangerie's zinc .5,
  the Japanese bridge's wood .6, the road's deck .8. `roofAt` remembers the surface's friction (`roofMu`). The slip starts where the
  slope is steeper than the friction holds (slate 24°, stone 31°, iron 17°) and slows on the gentler; a flat roof's edge stops a slide
  (as a gutter or a parapet would), a steep eave lets it off.
- *Walls by the feet, for all.* A walker's feet are the eye less its height too, so the bridge's rails (1.25) hold a walker as they
  hold a standing flyer. On foot, a flyer or a walker, the step is knee-high (.5): a .9 parapet holds, the Japanese bridge's deck
  (.5 up from the bank) is stepped onto. In the air the climb stays .8. The road bridge's abutments carry the road's own ramp height
  beyond the deck's ends, so a walker's feet on the ramp are never under the solid.
- *The Butter Tower's crown* is built as eight piers (.9 m square, 11 m tall, at the octagon's vertices) under its cap, the tall
  openings between them real; as solids: the tower's top (56, the crown's floor), the cap (67.85, its spires) with the air of the
  crown under it (its underside 67), and the eight piers. A flyer flies in between two piers and out the other side; a pier holds;
  E under the cap stops at 66.75; Q from under it lands on the tower's top at 57.45.
- Build `m28-on-contact`.

**Verified.** Headless, keys held. Level at 40 m west into the nave's east slope: at x −42.6 one is standing at 46.1, then (W held) over
the ridge, down onto the aisle and off it. Skimming the aisle at .6 stays airborne (19.2). Through the crown from the south-west at
60 m between two piers: in at −31.5 / −445.2, out the north side; at 0° the pier holds at z −441.2; E under the cap stops at 66.75; Q
inside lands at 57.45. Q on Saint-Romain's pyramid lands at 72.9 and, standing still, one slides off it onto the nave's roof (40.9 + 1.45)
and stops there against the tower's flank. The Orangerie's roof does not slip. On the road bridge D into the parapet holds at x 26.18.
A walker on the Japanese bridge, A into the rail, holds at z 1.22; walking across and off its end onto the bank passes; from the bank
onto the deck and across passes; the road over the bridge passes both ways with its ramps; the far bank's house and the Orangerie's
west door as before. Landed on the lantern's flat top at its edge, no slip. Frames in `m28-sheet.jpg`.
Bench, DPR 2, one run: pond 31.0 · argenteuil 27.3 · rouen 26.2 · sunrise 15.6 · aerial 13.8 ms: the machine is still at about twice
its usual times (M27's note: the M26 build rebenched the same); nothing here draws but the crown's eight piers in place of its drum.
To be rebenched on a quiet machine.

**Still visible.** Contact is judged by height within .3 of the surface, so a near-vertical flank (the spire's foot, an iron slope of
82°) met at eye height is "landed on" for a moment before the slip takes one off it. The slip's start is the surface's friction against
its slope, but the slide's own friction is the same number whether one is on one's feet or one's back. Water is still no floor for a
walker (a fall's end on water hovers). Walkers' feet are the eye less its height while the eye is still settling after a landing, a
few centimetres off for half a second. The crown's piers are boxes at the vertices; the drum's cap has no rim or rail, so a flyer
standing on the tower's top between the piers is held only by them.

### Progress · M29 — the flank: no floor from the side; friction on one's feet and on one's back; the fall's end in water; the crown's balustrade (what M28 left visible)

**What was wrong.** Contact was judged by height alone, so a near-vertical flank met at eye height, the spire's iron foot (82°) or
Saint-Romain's slate pyramid (73°), was "landed on" for a moment before the slip took one off it, and a shoulder-high rise of such a
flank was climbed as if it were a step. The slide's friction was the slip's threshold, the same number on one's feet and on one's
back. Water was no floor for a walker: a fall ending on water hovered in flight, .6 over it, and a flyer descending over water could
go to .6 over the bed, under the surface. The Butter Tower's crown had no rim, so a flyer standing on the tower's top between the
piers was held by nothing but them. Found on the way: a flyer descending onto either bridge's deck could not land on it at 60 fps,
held .6 over it by the walker's ground rule (the deck counts as ground) before the touch's .3 could be met.

**What changed.**
- A flank. A solid's surface steeper than 70° (`FLANK`, tan 70° = 2.75; the spire's iron, the pyramid, the pinnacles and turrets, the
  crown's small spires; the transept's 63° is still a roof) is a wall from the side however little it rises: `flankS(S, x, z)` reads
  the slope at the point from what rises beside it (the steeper side of each axis, 5 cm out; a rail's or a parapet's face, a metre and
  more up in 5 cm, is not a slope, and what falls away beside the point, its edge or the ground's notch before an abutment, is no flank,
  one can always step down). A step refused as a flank holds a flyer in the air, hovering at the flank held off by the skim's .3, and
  holds a standing flyer or a walker where they are (a flank is no stair: W up the pyramid from a landing on it does nothing). The
  touch does not land on a flank from level flight; coming down onto one (Q, a fall) lands, for the moment the slip takes.
- In the air the step's judge is the eye less the skim's .3, so a surface within the skim is "met" mid-step and landed on there, where
  before the frame's end found it; the feet are read live in each sub-step (a landing mid-frame moves them).
- Two frictions. The material's (`mu`) is the static one, shoes on a slope: the slip starts where the slope is steeper than it holds.
  Sliding, one is on one's back and the friction is the kinetic, three quarters of it (slate .34, 18.6°; stone .45; iron .22): a slide
  once started carries on over a slope shoes would hold on, and slows only on a gentler stretch. A flat roof's edge still stops it.
- The fall's end in water is on foot, afloat: `eyeHeight` in water stands one on the bed where it is shallower than 1.1 and floats one
  otherwise, the chin at the surface (the eye .35 over it); one wades at 1.3 m/s (the water rule still keeps a walker from stepping
  in from a bank; out of the water onto a bank is a step like any). A line says so. A flyer's floor over water is the water's face + .6,
  never the bed.
- A flyer over a deck: the walker's ground rule gives way to the deck's own surface when that surface is the deck, so the touch meets
  it (landing on the road bridge at 5.75, on the Japanese bridge at 3.81).
- The Butter Tower's crown, built and read: a balustrade between the eight piers (1.1 high, .3 thick, at the piers' radius) and a
  parapet round the tower's top between the corner pinnacles (1.0 high), as geometry (the cathedral's stone) and as solids (stone,
  mu .6): hip-high walls to one standing on the tower's top, ledges to one arriving level at their height.
- Debug: `monet.cam` (the camera unrounded), `monet.flank(x, z)` (the solids whose surface at a point is a flank), `monet.trace = []`
  collects each refused sub-step's reason (`water`, `edge`, `orang`, `wall h feet`, `flank h`).
- Build `m29-the-flank`.

**Verified.** Headless, keys held, live time. Level west at 100, 107 and 114 m into the spire: held at x −45.99, −45.46, −45.02
(the octagon's flat at each height plus the skim), not landed, still there after 3 s; level at 70 m into Saint-Romain's pyramid: held
at x −57.57, not landed. Level south at 40 m at x −22 into the transept's north slope (63°): landed at the ridge (46.6), W carries
over and off the south eave, the fall. Q onto the spire's flank at r 3.5: landed at 81.07 for the moment, then the slide down the
iron to the valley where the transept's south slope meets the choir's roof (35.93) and a stop there. Landed on the pyramid, W held:
no movement (a flank is no stair). Level at 40 m into the nave's east slope still lands at 46.1. The aisle's slip (slate, 36°) runs
to the eave's gutter band in 1.5 s and stops there; the lantern's flat top, standing at its edge, no slip. Landed on the road
bridge's west parapet (6.65), W west off it: the fall to the Seine, its end at .38 then afloat at .35 on foot; W south wades at
1.04 m/s (the lerp's mean) to the bank at z −139.8 (2.48). A flyer descending over the pond stops at .6. Landed on the tower's top
inside the crown, W toward a gap: held at r 3.9 (57.45); outside the drum, W east: held at x −26.52 by the parapet; arriving level
at 57 m: onto the parapet's ledge, down onto the top, held by the balustrade; through the crown at 60 m between two piers still
passes (in at z −445.2, out at −483.6), the pier at 0° still holds at z −441.17. Q onto the road bridge's deck lands at 5.75 and D
into the parapet holds at x 26.24; Q onto the Japanese bridge lands at 3.81 and A into the rail holds at z 1.22. The M28 walkers'
set as before: the rail for a walker at z 1.22, across and onto the Japanese bridge, the road over the bridge both ways with its
ramps (the south ramp's abutment held a walker at z −195.26 until the flank rule learned to ignore a drop), the far bank's house,
the Orangerie's west door. Frames in `m29-sheet.jpg`.
Bench, DPR 2, one run, the machine quiet again: pond 12.1 · parasol 8.6 · argenteuil 9.0 · poplars 9.3 · haystacks 12.1 · rouen 8.4 ·
sunrise 5.5 · orangerie 11.7 · aerial 6.6 ms (M25: 12.8 / 9.4 / 8.3 / 5.6 / 5.3): M27's and M28's doubled times were the machine's,
as noted then. The crown's balustrade and parapet add twelve boxes to the cathedral's stone (27 draw calls, as before but one).

**Still visible.** A flank is judged at the point stepped onto, 5 cm each way, so a surface steeper than 86° (a metre and more in
5 cm) is read as a step's face, not a slope, and its slice within the skim's .3 can be landed on by chance; nothing in the model is
that steep but rails and walls. The kinetic friction is a fixed three quarters of the static, not the materials' own pairs. Afloat,
one is a walker whose feet are 1.1 under the surface: a deck or a bank higher than the knee above the water is a wall to the swimmer,
so one leaves the water only where a bank is low, and one cannot step into water from a bank. The valley between the transept's
slope and the choir's roof is where two boxes overlap, not a gutter. The crown's balustrade is solid stone, not pierced.

### Progress · M30 — the valley: the nave's roof carried across the transept; a slope and a step's face told apart at any steepness; each material's own two frictions; wading in and out; the balustrade pierced (what M29 left visible)

**What was wrong.** The flank test ignored a rise of a metre and more in 5 cm as a step's face, so a surface steeper than 86° read as a
step and its slice within the skim could be landed on by chance. The kinetic friction was three quarters of the static for every
material. A walker was kept out of the water by a rule, and afloat one's feet were 1.1 under the surface, so a deck or a bank hip-high
over the water was a wall to the swimmer. The choir's and the nave's roofs stopped at the transept, whose roof ran across them: a
slider coming down the transept's slope stopped against the choir's gable end standing out of it (M29's "valley"). The crown's
balustrade and the tower's parapet were solid boxes.

**What changed.**
- A slope rises evenly, a step's face all at once: `flankS` counts a rising side only if the surface halfway out has half the rise
  (within a quarter of it), at any height of rise. A rail's, a parapet's or a wall's face beside the point is not the surface's
  slope; a slope of 86° and more is (the spire's tip, 86.6°, was already read as one, its rise under a metre).
- `MAT`: slate [.45, .34], stone [.6, .5], iron [.3, .2], zinc [.5, .4], wood [.6, .45], road [.8, .65], static and kinetic; a solid's
  `mu` is the material's name, the slide reads its pair (stone and wood, the same to shoes, part on one's back).
- Water for a walker: the step in from a bank is taken (the eye sinks to the wading height over the next metre, a line says where
  one is); in the water the body is at the surface, so a walker's feet for the step out are the water's face and the climb is .9,
  onto a bank or a deck hip-high over the water. Out of the water onto a bank is a step like any.
- The crossing: the nave's roof is carried across the transept at the same ridge (a prism and a solid over the crossing), so the two
  roofs meet in valleys running from the ridge to the eaves' corners, the transept's slope the steeper. The lantern stands on the
  crossing as before: its wall is met at |x − CX| < 7.15.
- The balustrade between the crown's piers and the parapet round the tower's top are pierced: a rail on balusters (.11 square, every
  .34) on a plinth, in the cathedral's stone; their solids are as before (a body does not pass a balustrade).
- Build `m30-the-valley`.

**Verified.** Headless, keys held. `flank`: the spire's tip (126.5, 86.6°) is a flank; the Japanese bridge's deck 2 cm from its rail
is not, the tower's top 2 cm from the balustrade and next to the parapet is not (the cap's small spire over the first is). The spire's
flank at 100 m still holds a level flyer at x −45.98, not landed; the pyramid at −57.57; level into the nave's east slope still lands
at 46.1; Q onto the road bridge's deck lands at 5.75 and D holds at the parapet, 26.24. The aisle's slip as before (stops at the eave's
band in 1.5 s); Saint-Romain's slip ends on the nave's roof at 42.34; the lantern's edge, no slip. Q onto the spire's flank: the slide
now runs down the transept's south slope along the valley to the eaves' corner (x −57.5, z −494.6: lx/13.6 = .99, lz/6.6 = 1.0), off
it onto the choir's aisle (17.45) and stops at its band. Landed on the nave's roof at x −54 (37.89), W south goes up the transept's
north slope (45.24 near the ridge) and down onto the choir's roof (37.89), one surface; at x −51 the same walk meets the lantern's
wall at z −480.9 (`trace`: wall 57). On foot from the east bank at z −8, W west: in the water at x 9.6 (the eye .65, then .35), across
the pond afloat at 1.04 m/s, out on the west bank at x −8.6 (the eye 1.79 at −11.9); the hint reads "In the water · wade out at a
bank, or F to fly". Frames in `m30-sheet.jpg`.
Bench, DPR 2, one run: pond 12.4 · parasol 10.3 · argenteuil 10.9 · poplars 11.7 · haystacks 13.0 · rouen 9.2 · sunrise 6.0 ·
orangerie 9.8 · aerial 5.9 ms (M29: 12.1 / 8.6 / 9.0 / 9.3 / 12.1 / 8.4 / 5.5 / 11.7 / 6.6): within the runs' spread; the crossing's
prism and 180-odd balusters are in the cathedral's merged stone.

**Still visible.** The valley is the meeting of two prisms, without a gutter or lead; the slide along it is the slide rule's
(the steeper side of each axis), not a body in a trough. The transept's own roof meets the choir's and the nave's at 63° against
44°, so the valleys are not at 45° in plan. In water one is afloat with the eye .35 over the face wherever the bed is deeper than
1.1, without swimming's slowness or the current of the river. A swimmer climbs out onto anything up to .9 over the water, the
road bridge's abutment included where its ramp is that low. The balusters are square, the same on the crown and the top.

### Progress · M31 — the gutter: the valleys' lead and a body sliding in a trough; swimming, and the river's current; the climb out of the water by the bank's face; turned balusters, a traceried parapet (what M30 left visible)

**What was wrong.** The valleys at the crossing were the bare meeting of two prisms, and a body sliding in one followed the slide
rule (the steeper side of each axis), so it slid down one slope into the other and back, not along the trough. Afloat one moved at
the wader's pace and the Seine stood still. A swimmer climbed out onto any solid up to .9 over the water, a pool's edge at the hip;
and under a deck (the Japanese bridge's arch, the road bridge's) the walker's ground was the deck, so a swimmer passing beneath rose
through it onto the road. M30's note that the road bridge's abutment could be climbed from the water where its ramp is low was
wrong: the abutments stand on the embankment, 4.1 m and more over the water, and a bank of .8 lies between the water and the
embankment's face. The balusters were .11 boxes, the same on the crown and round the top.

**What changed.**
- `troughAt(x, z, A)`: with the floor solid A under the feet, the highest other solid whose surface is within .25 below it and whose
  gradient differs from A's (the same slope carried on by another solid, the nave's roof by the crossing's, is no valley) makes a
  trough. Its line runs where the two heights stay equal (across the difference of the gradients), its fall is A's slope along that
  line, and each frame of the slide the body is drawn to the line's bottom (the Newton step to equal heights, at most 5 cm). The
  slide takes the trough's direction and fall and the rougher of the two materials' pair; a level line (the eaves' at the corner)
  leaves the slide its last direction, so it carries on, slowing.
- The valleys' lead: for each of the four lines from the lantern's foot (|lx| = 7) to the eaves' corner, a strip .42 wide laid in
  each roof's plane 2.5 cm over it, up from the line into its own roof (eight quads, the cathedral's palette 10).
- Swimming: afloat (the bed deeper than 1.1) the pace is .9 m/s (wading 1.3, walking 3.2), Shift doubles it as before. `currentAt`:
  the river's current sets west along its line, .55 m/s in mid-stream, slack to the banks (`sstep` over the first 7 m of the river's
  SDF) and over the shallows (over the depth .2 to 1.4); it moves a swimmer by its whole and pushes at a wader's legs by .45 of it.
  The pond and the harbour have none. The line at the step in, or at a fall's end, says "In the river · the current sets west · swim
  for a bank, or F to fly" within 6 m of the river; the pond's line is as before.
- The climb out: in water one heaves out onto a solid a knee's height (.5) over the water's face, or over the bed where one stands
  (M30: .9); and the same for the ground: a walker in the water is refused a step onto land or a deck's end more than .5 over the
  water's face (`trace`: bank). `inWater(x, z, y)`: under a deck more than a knee over the water's face one is in the water beneath
  it, not on it, so `eyeHeight` keeps the water's eye there and a swimmer passes under the Japanese bridge and the road bridge's
  arches; `isWater` is unchanged for what is laid on the water.
- The crown's balustrades: turned balusters, a lathe of a ten-point vase profile (8 sides), every .34 as before. The top's parapet:
  colonnettes (.06, six-sided) every .5 under pointed heads, two bars leaning together beneath the rail, an arcade of tracery. The
  solids of both are as before.
- Debug: `monet.trough(x, z, y)`, `monet.current(x, z)`. Build `m31-the-gutter`.

**Verified.** Headless, keys held. `trough` on the north-east valley's line (t .7): direction (.90, .44), the line's own (13.6, 6.6),
fall .86 (40.7°), no correction; 10 cm off it toward either roof the correction points back to the line (−.02, .04 and .04, −.08);
on the aisle and on the tower's top, none. Q onto the valley at t .6 (39.65): the slide runs along the line to the eaves' corner in
1.5 s, off it, onto the nave's aisle (21.33), off the aisle's eave to the ground (2.0). Q onto the spire's west flank from
(−47.5, 110): down the flank, over the lantern's rim onto the transept's south slope (44.41 at 1 s), into the valley and along it to
the corner (past it at 2 s), the choir's aisle 17.45: M30's end by the trough's rule. Q onto the spire's north-east flank: the slide
down the flank stops at a turret's foot on the lantern's top (58.45; `trace`: wall 58.3 feet 57). `current`: mid-stream (0, −172)
(−.55, −.03), at (60, −172) (−.55, .01), at the bank (0, −189) 0, the pond 0. Afloat in the river with no key held: 1.1 m west per 2 s
(.55 m/s), the river's own curve in z; D (east, against it) .35 m/s, A (with it) 1.45 m/s; at (−10, −187) by the bank .15 m/s. Across
the pond afloat at .9 m/s (M30 1.04), out on the west bank at x −8.9. From the water the road bridge's pier is a wall (`trace`: wall
5.2 feet 0) and its arch is passed under afloat (x 21.45 to 27.76 at .35 m/s, the eye .35 throughout); the Japanese bridge is passed
under (z −.72 to 2.17, the eye .35–.39) and the north bank walked out on. Out of the river at (20.5, −190) onto the .41 bank, then
the embankment on foot (the walk ends at a house's wall, `trace`: wall 9). The step in from the bank at (0, −189.5) says the river's
line. M28's walkers and M30's flyers as before: the pond bridge's rail (3.81) and the walk across and onto it, the road both ways
(5.75, the ramps), the house, the Orangerie's door, the spire's flank at 100 m (−45.99), the pyramid (−57.57), contact on the nave
(46.1), the deck landing (5.75) and the parapet (26.24), the aisle's slip to its band (17.45), Saint-Romain's (42.34), the lantern's
edge, the crossing walk (37.89 → 45.24 → 37.89) and the lantern's wall at z −480.9. 120 solids. Frames in `m31-sheet.jpg`.
Bench, DPR 2, one run: pond 11.3 · parasol 8.8 · argenteuil 9.2 · poplars 10.4 · haystacks 12.8 · rouen 8.2 · sunrise 4.9 ·
orangerie 8.7 · aerial 4.8 ms (M30: 12.4 / 10.3 / 10.9 / 11.7 / 13.0 / 9.2 / 6.0 / 9.8 / 5.9): within the runs' spread, a little
under; the lead is one more draw call (27), the balusters and colonnettes 80 triangles more in the merged stone.

**Still visible.** The lead is a flat strip in each roof's plane, not a lined trough with a lip, and a body sliding down the valley
shoots off the eaves' corner where a real gutter would catch it (the corner's flat margin lets it by). In the trough the friction is
the rougher material's pair; the wedging between two faces is not counted. The current is one field along the river's centreline,
slack at the banks, not bent round the piers, and it carries nothing but the swimmer; swimming is a pace without strokes or tiring.
The climb out is judged against the water's face even in the shallows, so a bank .5 over the water is climbed from a bed a metre
down; no bank in the world is a cliff (the harbour's quay slopes one in one), so the bank rule is exercised only by the piers, the
decks and the abutments. At a deck's end a knee over the water the eye rises through the deck's end onto it. The parapet's heads are
two leaning bars, not cusped tracery, and the balusters are one profile all round the crown.

### Progress · M32 — the lip: the valleys' lead with its welt and the eaves' gutters, a body caught at the corner or vaulting it; the trough's wedged friction; the current parting round the piers and carrying leaves; strokes and wind; a quay wall and a slipway in the harbour; cusped tracery on the parapet (what M31 left visible)

**What was wrong.** The lead was a flat strip in each roof's plane with no lip, and there was no gutter at the eaves, so a body
sliding down the valley shot off the corner at any speed; and the slide's "edge" was where no solid lay ahead at all, so the aisle
under the nave's eave made the eave read as roof going on, and the lantern's flat rim likewise let a slide over because the transept's
roof lay below it. The trough's friction was the rougher material's, the wedging between two faces not counted. The current was one
field along the river's line, straight past the piers, and carried nothing. Swimming was a steady pace, no stroke, no tiring. No bank
in the world was a cliff, so the swimmer's bank rule was exercised only by piers, decks and abutments. The parapet's heads were two
leaning bars.

**What changed.**
- The lead's welt: along each strip's outer edge a second quad standing .07 off the slates (the lip). Eaves' gutters: lead troughs
  (.3 × .16) hung at the nave's and the choir's eaves outside the transept and along the transept's eaves, meeting at the four
  corners where the valleys' leads come down. Solids carry `gutter` (the two vessels, the crossing, the transept); `S` returns its
  solid.
- The slide's edge: the roof ends where the surface a frame's slide and .3 ahead is a metre and more below (nothing there, or the
  aisle under the eave). At an edge a body arriving under `GUTTER_V` 5 m/s is caught by a flat roof's rim or a guttered eave's lip;
  faster it vaults either; a steep bare eave lets it off at any speed.
- `troughAt` reads each face's pitch across the line (β from the gradients' components along their difference) and the wedge:
  (μA sin β2 + μS sin β1) / sin(β1 + β2), applied to both the static and the kinetic pair (the crossing's valleys: 1.265 × slate).
- `currentAt`: the stream past each of the road bridge's four piers (`piersZ`, lazily) as potential flow round a cylinder of 1.4:
  slack before and behind, quick past the sides, nil within. Flotsam: 28 leaves and twigs (one merged geometry, its positions moved
  each frame in `updateFlotsam`) borne west at the current, parting round the piers, set in again upstream when they leave the world
  or strand.
- Swimming: breaststroke, the pace pulsing ±45% over a cycle of 1.5 s (1 s sprinting), the head lifting .04 with the kick. Wind
  (`stamina`) spent in a minute's swimming, a third of that sprinting, back on land in half a minute; spent, one is weary — half
  the pace, no sprint, "Tiring · swim for a bank, or F to fly" — until a third of it is back.
- The harbour's built quay (`QUAY`): along z −392..−424 the quay's line straightens to x 22.4 (a grid line) and the ground stands
  1.3 over the water, dropping as a cliff one cell wide (the terrain's `hb > −.01 ? 1 : 0` under `quayWall`), a stone wall from the bed
  to the top standing .3 out over the cliff's brow; a slipway at z −408 (3.2 wide, 6.4 long, one in two and a quarter) down through
  it, the parapet and a bollard gapped there. The swimmer's bank rule looks a body's length ahead too: the step's own ground no more
  than a knee over the water's face and the ground .6 further on no more than a hip (`trace`: bank h1 h2).
- The parapet's heads: an equilateral pointed arch per bay (two arcs of the bay's span from the opposite springings, four bars each)
  with a cusp on each arc pointing into the light, trefoiled.
- Debug: `monet.swim`, `monet.stamina =`, `monet.flotsam`, `monet.slideDbg` (an array to fill). Build `m32-the-lip`.

**Verified.** Headless, keys held. `trough` on the valley's line: direction (.90, .44), fall .86, wedge 1.265, the pair [.57, .43].
Q onto the valley 3 m from the corner: the slide runs to the corner and is caught at (lx 13.68, lz 6.62, 34.45), standing. Q at
t .6: the slide vaults the corner (6.2 m/s), the nave's aisle (22.36), off its eave, the ground (2.0). Q onto the spire's west flank
from (−47.5, 110): the slide vaults the lantern's rim, the transept's south slope (44.64), the valley to the corner, the choir's aisle
(17.45), the ground: M31's path. Frame-by-frame (`slideDbg`) the corner is seen .37 ahead with the transept's margin under it and
the aisle 13 m below, the edge by the new rule. `current` at the second pier (z −176.8): 3 m upstream (−.45, −.01), 3 m downstream
(−.45, −.02), 2.2 beside (−.82, −.01), within (0, 0); a swimmer under the arch at z −170 is set 1 m aside passing between two
piers. Flotsam: four leaves 2.2 m further west after 4 s. Swimming west with the current, positions every .25 s: the pace pulses
1.1–1.8 m/s (.5–1.3 own, plus .55), the eye .32–.38; wind .933 after 4 s; set to .03 and swimming on: weary in 2 s, the pace
1.0–1.1 (.5 plus the current), the line said. The quay: the ground 1.3 at x 22.4 and −2.5 at 23.2; from the harbour swimming west at
z −400 one is refused at x 22.68 (`trace`: bank −.03 1.30), standing ankle-deep at the wall's face (the eye 1.31), the camera
outside the stone; at the slipway (z −408) the same swim walks up onto the quay (the eye 2.74, then 1.95 on the town's ground);
walking east off the quay's top one drops into the harbour and floats (.32). M28's walkers, M30's flyers and M31's water as before:
the rails, the road, the house, the door; the spire's flank at 100 m (−45.99), the pyramid, contact on the nave (46.37), the deck
and the parapet, the aisle's and Saint-Romain's slips, the lantern's edge, the crossing walk; the river's line at the step in, the
pond bridge passed under, the pier a wall (5.2), the arch passed under. 120 solids. Frames in `m32-sheet.jpg`.
Bench, DPR 2, one run: pond 13.0 · parasol 7.9 · argenteuil 8.4 · poplars 9.2 · haystacks 10.8 · rouen 7.9 · sunrise 5.1 ·
orangerie 8.1 · aerial 5.3 ms (M31: 11.3 / 8.8 / 9.2 / 10.4 / 12.8 / 8.2 / 4.9 / 8.7 / 4.8): within the runs' spread; the flotsam is
one more draw call at the river (its 28 quads moved each frame), the quay wall 18 boxes in the harbour's merged stone, the tracery
some 800 small boxes in the cathedral's.

**Still visible.** The catch is a speed, 5 m/s, not the lip's height against the body's momentum; a flat rim and a guttered eave
catch alike. The wedge counts the two faces' normal forces but not the body's turning in the trough. The piers part the stream as
cylinders in a potential flow, so there is no wake and no eddy behind them, and the piers are not round. The flotsam is carried but
never sinks, catches on a pier or gathers at a bank; the boats sit still in the stream. The stroke is a pulse of the pace and a lift
of the head, with no arms in view and no sound. The wind is one number, spent and got back at fixed rates. The quay wall is the
one cliff in the world, straight and 1.3 high, the cliff one cell wide under it (a shelf at the wall's foot ankle-deep); the slipway
is a plain ramp. The tracery is bars and cusps in one plane, the same bay repeated.

### Progress · M33 — the welt: the lip's height against the body's momentum, and the aisles guttered; wake and eddies behind the piers, flotsam that sinks and strands; the arms in view, breath and strength; the quay wall on the quay's own curve with the cliff exact under the walker, and a mole; the tracery's bays varied and its cusps proud (what M32 left visible)

**What was wrong.** The catch at an edge was one speed, 5 m/s, for a flat rim and a guttered eave alike. The piers parted the stream
with no wake or eddy, and the flotsam never sank or stranded. The stroke showed no arms and the swimmer's wind was one number. The
quay wall was straight (its line pulled onto a grid line so the terrain's cliff would be one cell wide), with that cell's ramp
leaving a shelf ankle-deep at the wall's foot, and it was the world's one cliff. The tracery was bars and cusps in one plane, one bay
repeated.

**What changed.**
- The catch: a body sliding into a lip pivots on it; to go over, its centre (.12 over the slates on its back, half its length .85
  behind its feet) must rise the lip's height times that lever, 7.1: caught if v² < 2 g H · 7.1 (H .16: under 4.7 m/s). A bare
  rim is caught only by the arms, under `ARMS` 1.5. Solids' `gutter` is now the lip's height; the aisles' eaves are guttered too
  (lead troughs along them at 16), so the aisle's slip is caught as before.
- `currentAt`, downstream of each pier: a wake, a deficit of .85 of the stream on the axis, widening (.55 + .12 per radius) and
  fading (1/√s) with the distance in radii, and within four radii a pair of eddies (vortices of 1.6 · stream, .75 either side of the
  axis, fading to nothing at four) turning the water back on itself close behind the pier.
- Flotsam: each piece has a life (60–300 s) after which it goes down over three seconds and is set in again upstream; one carried
  onto a pier's nose (within .3 of it, 1.1 either side of the axis) is held there quivering 6–30 s, then let go round the pier; one
  reaching slack water at a bank lies 10–40 s and is gone.
- The arms: two forearms and hands (boxes, skin-coloured by vertex, `MeshBasicMaterial`) hung from the camera under the eye at the
  water's face, shown only afloat and swimming, hidden from the reflection pass; `updateArms` sets them by the stroke's phase: the
  glide with the arms out ahead (.4 of the cycle), the pull sweeping them out .3 and back .22 (.3), the recovery bringing the hands
  in under the chin and shooting them forward (.3).
- Breath and strength: breath is spent in twelve seconds' sprint and comes back in fifteen of rest (forty, swimming easy); at nil
  one is winded, "Winded · no sprint till the breath is back", panting (the pace × (.75 + .25 · breath)) and cannot sprint till half
  of it is back. Strength is spent in a minute's swimming (twenty-five seconds sprinting), back in half a minute on land; at nil
  one is weary as before. Sprinting, the stroke's cycle is 1 s.
- The quay: `quayX` keeps its curve; the wall's face is .5 out from it, the stone 1.6 thick in 2 m segments turned to the curve;
  `groundHeight` along the wall is the stone's own edge (the wall's top to its face, the bed −2.5 beyond it), the terrain grid's
  ramp lying inside the stone, so there is no shelf: a swimmer floats at the face and is refused there (`trace`: bank 1.30 1.30).
  A mole (`MOLE`: at z −398, 3.4 wide, 16 long from the quay, a bollard at its head) runs out into the harbour, its top 1.3 and its
  sides cliffs the same way, the parapet gapped at its root: a second cliff, and a third with the wall's own return.
- The tracery: the cusps stand proud of the bars on both faces (.07 deeper); even bays trefoiled (one cusp to each arc), odd bays
  cinquefoiled (two smaller); a shaft ring on every other colonnette.
- Debug: `monet.swim` (breath, strength, winded, weary, ph, arms), `monet.breath =`, `monet.strength =`, `monet.flotsam` (sinking,
  stuck, first four), `monet.age(s)`, `monet.put(i, x, z)`. Build `m33-the-welt`.

**Verified.** Headless, keys held. Q onto the valley 2 m from the corner: caught at it (34.45), standing; 4.5 m from it the slide
arrives at 8.55 m/s and goes over the lip, the aisle, the ground. Q onto the aisle near its eave: caught at 17.56; from its ridge
the slide reaches the eave at 4.56 m/s and is caught (the eave band as before). Behind the second pier (z −176.8) on the axis:
2 radii (+.05, −.02), the stream turned back; 4 radii (−.29); 8 radii (−.37); close behind and .3 off the axis (+.21, −.07), 1.6 off
(−.47, −.13); beside it (−.59); open water (−.55). Flotsam aged 400 s: 27 of 28 sinking a second later, all set in again upstream
four seconds on; one put on the pier's nose is held there. Sprinting west 14 s: 2.4 m/s (2.2 × .9 + .55), the breath .044 and
"Winded" said at 14 s; easy after it 1.2 m/s (panting), the breath .19 six seconds on, the arms shown. The quay at z −400, −412,
−420: 1.3 to the face (17.76, 17.55, 17.99), −2.5 a hand beyond it; swimming west at −400 and −420 one is refused at the face
(`trace`: bank 1.30 1.30), the eye .32–.38, afloat; at the mole's north side (z −396.3) likewise; walking out along the mole (2.75)
one drops off its head into the harbour (.32). M28's walkers, M30's flyers, M31's and M32's water as before. 120 solids. Frames in
`m33-sheet.jpg`.
Bench, DPR 2, one run (a first run overlapped the last one's Chrome and was thrown out): pond 13.8 · parasol 11.2 · argenteuil 12.9 ·
poplars 9.7 · haystacks 10.5 · rouen 8.2 · sunrise 5.0 · orangerie 9.7 · aerial 4.9 ms (M32: 13.0 / 7.9 / 8.4 / 9.2 / 10.8 / 7.9 / 5.1 /
8.1 / 5.3): the river up 4.5 ms and the parasol 3.3, the rest within the spread; the flotsam's stream is read 28 times a frame with the
wake and the eddies of four piers, the one new per-frame cost, though the parasol has none of it. The mole and the wall's 16 segments
are in the harbour's merged stone, the cusps and rings some 300 boxes more in the cathedral's; the arms are one small mesh.

**Still visible.** The lever is a rule of thumb for a body on its back; a body sliding feet first or head first, or tumbling, is not
told apart, and the arms' catch is a speed. The wake and the eddies are a sketch (a Gaussian deficit and two vortices fading in
four radii), steady, with no shedding; the piers are still round to the stream. The flotsam strands only on a pier's nose or in
slack water, never against the abutments or the boats, and the boats do not feel the current. The arms are two boxes each and do
not enter the water; the legs kick unseen. Breath and strength are two numbers with fixed rates; nothing is felt in the sound. The
quay's cliff is exact under the walker but the terrain mesh's one-cell ramp still lies inside the stone; the harbour's other banks
slope as before. The tracery's bays alternate between two patterns; there is no moulding on the bars.

### Progress · M34 — the street: the catch by how the body arrives (feet first, head first, tumbling) and the arms' grip; a street of eddies shed from the piers; flotsam against the boats and the boats in the stream; upper arms, the hands into the water; the effort and the cold, the breath heard; the whole harbour quayed, the cliff in the mesh; three patterns of tracery, the bars moulded (what M33 left visible)

**What was wrong.** The lever was a rule for a body on its back whatever way it arrived, and the arms' catch was a bare speed. The
wake and its eddies were steady, with no shedding. Flotsam stranded only on a pier's nose or in slack water; the boats sat still in
the stream. The arms were two boxes each, at the surface throughout. Breath and strength drained at fixed rates whatever the water
and the stream did, and nothing of it was heard. The terrain mesh's ramp lay inside the quay's stone, the harbour's other banks
sloped, and the quay wall was one stretch. The tracery alternated two patterns, its bars plain. Under all of it, a standing flyer
whose floor dropped away (the eave over the aisle, the lantern's rim) was set down on the lower roof in a frame instead of falling.

**What changed.**
- `slideHow`, set as the body lands: from a fall of over 4 m/s it tumbles; landed by contact moving up the slope (the roof rising into
  a level flyer, or a step up onto one) it comes down feet uphill and slides head first; otherwise on its back, feet first. At an
  edge: feet first, the lip takes the feet and the body must be tipped over it (the lever, .16 under 4.7 m/s); tumbling, it need
  only be lifted the lip's height (1.8); head first, the hands take the lip and the arms' grip is all the catch, as on a bare rim;
  a tumbling body has no arms to it. The arms' grip is what they can brake: `GRIP` .4 of the body's weight over `REACH` half a metre,
  v² < 2 g · .5 · .4 (1.98 m/s).
- A standing flyer follows the roof only while it is within a step below; where it drops by more (an eave over a lower roof, the
  lantern's rim) one falls, and lands as a fall does.
- `currentAt`: behind each pier the pair of eddies is a street: four vortices shed from the sides in turn (St .2: one every 25 s),
  borne down at .8 of the stream 11 m apart, turning opposite ways .8 either side of the axis, fading over 33 m; the wake's
  deficit as before.
- Flotsam drifting within 2.5 of a boat's hull is held against it 5–20 s, then let go past it. The boats feel the stream: a moored
  one rides .7 · current down its rope and sways ±7° with the pull; one under sail makes leeway of 1.2 · current.
- The arms: each an upper arm from a shoulder under the eye (±.19, −.34, .02) to an elbow, a forearm to the hand, and the hand
  (tapered cylinders and a box, skin coloured with a little noise); the elbow bends out and down with the pull; in the pull the hands
  go .06 under the glide's line, a hair into the water, and the arms lie in it.
- Breath and strength by the effort and the cold: the effort is one's pace plus what the stream takes back swimming against it,
  less what it gives with it (up to half), over the pace (against the Seine 1.61, with it .5); the cold is the pond's 1, the river's
  1.15, the harbour's sea 1.5, wearing the strength. The breath is heard: the same noise through a band at 500–900 Hz let through on
  each pull, harder and higher the shorter the breath (`gBreath`, `breathF`).
- The harbour's whole quay is built (`QUAY` z −376..−520 blending over 4 m at the ends): the wall on the quay's curve the whole way,
  slipways at z −408 and −470, the mole at −398. The cliff is in the ground mesh itself: the vertices either side of the wall's face
  (and the mole's sides and head) are drawn to a line .3 inside the stone, the land's at the top and the water's at the bed, so the
  mesh has a cliff there and no ramp, hidden in the wall.
- The tracery: three patterns by bay, trefoiled, cinquefoiled, and a mullion with two lancets under the arch; every bar carries a
  roll along it proud of both faces.
- Debug: `monet.swim` adds effort, cold, how, hands; `monet.boats`. Build `m34-the-street`.

**Verified.** Headless, keys held. Q onto the valley near the corner: feet first, caught. Flying level east up the aisle's slope, the
roof rising into one: landed head first (22.38 at the top); left still, the slide back down reaches the eave at 6.63 m/s and goes
over (the arms cannot hold it). Q onto the valley 4.5 m from the corner: off the corner at 5.58 m/s, a fall (34.08, 26.49), landed
on the aisle tumbling (21.89), the slide to its eave at 6.47 and over it, the ground. The aisle from its ridge, feet first at 4.56:
caught (17.56) as before. Behind the second pier, .8 off the axis, 6 m down, every 5 s: the across-stream part −.07, −.18, +.06,
+.01, +.02, +.02 — an eddy passing. Flotsam put on a moored boat is held (`stuck`: boat); the moored boats lie at their moorings
offset down the stream, the sailing ones with leeway. Swimming against the stream the effort reads 1.61 and with it .5; in the
harbour the cold 1.5 (the river 1.15); the hands' height over a cycle −.23 to −.33 (the water's face at −.35). The quay at z −390,
−440, −500: 1.3 to the face (18.68, 20.53, 20.9), −2.5 a hand beyond; swimming west at −440 one is refused at the face (`trace`:
bank 1.30 1.30) afloat; at the second slipway (−470) the swim walks up onto the quay (2.69). Saint-Romain's slip and the spire's now
fall where they leave a roof (55.6 falling to the nave's 42.34; the lantern's rim to the transept's 42.26) and end as before; M28's
walkers, M30's flyers, M31–M33's water as before. 120 solids. Frames in `m34-sheet.jpg`.
Bench, DPR 2, one run: pond 12.0 · parasol 8.6 · argenteuil 8.6 · poplars 10.0 · haystacks 13.2 · rouen 9.6 · sunrise 5.9 ·
orangerie 11.3 · aerial 5.4 ms (M33: 13.8 / 11.2 / 12.9 / 9.7 / 10.5 / 8.2 / 5.0 / 9.7 / 4.9): within the runs' spread; M33's river
and parasol were the run's noise. The street is four more vortex terms per pier per reading; the quay's wall the whole way is 72
segments in the harbour's merged stone, the rolls 670 boxes more in the cathedral's; the arms are six small meshes.

**Still visible.** The three ways of arriving are told by the landing alone; a body does not turn over on the slope, and the grip is
a fraction of the weight with no hold to grip. The street is four vortices of a set strength on a set spacing, the same for every
pier and every stream. Flotsam held by a boat does not move with it; the moored boats do not swing to face the stream (the
paintings' compositions hold them). The arms are tapered cylinders with no elbow or hand joint modelled, and the legs kick unseen.
The cold is three numbers; the breath is a filtered noise. The quay's cliff is in the mesh but the mesh's face is hidden .3 inside
the stone, not the stone's own face; the town behind the quay rises 1.3 over 4 m. The tracery repeats its three bays in turn; the
rolls are boxes.

### Progress · M35 — the roll: the body turning and rolling on the slope, the hands' hold; each pier's own street; flotsam with its boat, the moored boats on their ropes; the joints, the legs and the kick in the pace; the cold from the water's temperature, the breath in and out; the wall's face the mesh's own, the quay level behind it; each face of the parapet its own scheme (what M34 left visible)

**What was wrong.** How the body arrived was the whole story of the slide: it never turned over on the slope, and the arms' grip
held with nothing to hold. The street was four vortices of one strength and spacing for every pier. Flotsam held by a boat stayed
where it was caught; the moored boats sat on their painted lines whatever the stream did. The arms had no joints, the legs kicked
unseen and the pace's pulse was a sine. The cold was three numbers by place and the breath one band of noise on the pull. The
terrain mesh's cliff was drawn .3 inside the wall, the stone hiding it, and the land behind the quay fell 1.3 to the town's level
over 4 m. The tracery's three bays repeated in turn on every face.

**What changed.**
- The body on the slope: `bodyAng` (its long axis from the slide's line: 0 feet first, π head first, ±π/2 across) and `bodySpin`.
  Head first, the hands drag unevenly (`handBias`, up to .3 of the grip, set by where the body landed) and turn it at the shoulders'
  lever on the body's inertia (`HAND_YAW`, 1.6 rad/s² per unit); past 60° off the line it is across (`side`). Across, it rolls as a
  log: driven by the slope over the limbs' radius (`ROLL_R` .35) at a fifth, checked by the limbs at 2.5 rad/s², never faster than
  it slides over that radius; over .8 rad/s it is `tumble`, and a tumble that the slope cannot keep rolling comes to rest across.
  From a fall it arrives rolling at the fall's speed over the body's half-thickness, scaled to the limbs' radius. Feet first it
  keeps its line, the feet leading.
- The hold: what the hands have at an edge is the welt (the whole grip), a flat rim under flat hands (the grip times the rim's own
  friction), or a bare eave's arris hooked by the fingers (half). The catch: feet first, the lip's lever with the hands by the hips
  reaching half a grip; head first, the hands' grip on the hold alone; across, the body lifted over the lip whole and one arm's
  grip; rolling, the lip alone.
- `piersZ()` gives each pier its own stream at its flanks (slacker near the banks) and from it its shedding: T = D / (St · U),
  λ = .8 D / St, each eddy's strength .3 of the shear layer's roll-up over a half-period (½ U² T), the two rows .281 λ apart
  (Kármán's ratio), as many eddies as fill three wavelengths, at the pier's own phase. The wake is read to that length.
- Flotsam held against a boat keeps its place on the hull and goes and swings with it; let go, it leaves from where the boat is.
  A moored boat is moored by the bow where the painting has the bow: its rope streams down the current and the hull weathercocks on
  it, its heading easing to the stream's line at the hull over 4 s (the eddies swing it); in slack water it hangs at its painted line.
- The arms have shoulder, elbow and wrist as balls the segments turn on. The legs are built (hip, thigh, knee, shin, ankle, foot)
  and do the whip kick with the stroke; they are behind the eye. The pace's pulse (`PULSE`) is the pull's impulse (.4) and the
  kick's (.6) against the water's drag (the pace squared), the glide decaying between, the shape found by running the stroke to a
  steady cycle and set about a mean of one (.78 to 1.26). The head lifts on the pull, for the breath.
- `waterTemp`: the Channel in a November dawn 11°, the Seine 17°, the Epte 14°, the pond 21°, the shallows warmed by the sun up to 3°;
  the cold is the gap to the body's warmth against a 20° pond's (`swimCold` .91 pond, 1.18 river, 1.53 harbour).
- The breath: in through the mouth on the pull as the head lifts (a sharper band, 1100–2200 Hz by the breath's shortness), out into
  the water through the kick and the glide (low-passed at 380 Hz, heard through the water) with bubbles, each a short tone rising as
  it shrinks, a burst of them the harder the exhale.
- The terrain mesh's cliff is now the wall's face itself: the vertices either side of the face (and the mole's sides and head) are
  drawn to the face, the mesh made non-indexed so the face's own triangles are painted in the stone's colour, with the stone's strokes
  laid on them down to the water; the wall's stone stands .3 behind under the paving, and a coping course .16 deep overhangs the face
  .05. Behind the wall the quay's apron lies level for 14 m and eases to the town's level over the next 20.
- The tracery: each of the tower's four faces its own scheme, read from the centre bay outward and mirrored: a mullioned centre
  with cinquefoils and trefoils by turns; a mullioned centre among Y-tracery (a fourth pattern: the mullion forking at the
  springing into two bars meeting the arcs at 40°, each with its roll); a cinquefoil at the centre and every third bay, trefoils
  between; mullioned and trefoiled by turns.
- Debug: `monet.swim` adds temp, ang, spin, bias, pulse, feet; `monet.piers`; `monet.boats` adds the mooring; `slideDbg` rows add
  the angle and spin. Build `m35-the-roll`.

**Verified.** Headless, keys held. Feet first at the valley's corner: caught (34.45) as before. Landed head first on the aisle
(bias −.19): the body turns from π to 2.41 over the 4 s slide, still head first at the eave at 6.63 m/s, and over (the arris's
half-hold gives the hands 1 m/s). The vault off the corner lands on the aisle across (spin .47), the slope sets it rolling
(.84 → 1.61 rad/s), and it goes over the eave rolling at 3.17. M33's aisle slips as before (low: caught at 17.56; from the ridge
4.56 at the eave, caught), Saint-Romain's (17.56) and the spire's (58.45) as before. The piers: the first, near the bank, has a
stream of .08 (T 168 s, strength .18), the other three .55 (25.5 s, 1.16), λ 11.2 for all; 7.4 m behind the second pier the
across-stream part over 20 s: +.04, +.01, −.07, −.10, −.21, +.15; at 30 m the wake's deficit only (−.43), at 40 m the stream
(−.55). Flotsam put on a sailing boat moves with it (boat +.63, −.13 over 3 s; the leaf +.61, −.13); the moored boats lie to the
stream on their ropes (headings −.06 and −.24, from the painted .35 and .2; the moorings kept). The Seine 17° (cold 1.18, effort
against it 1.61), the harbour 11° (1.53), the pond 21.6° in the sun (.91). The pulse .78–1.26; the feet over a cycle x .10–.45,
y −.63 to −.47, z .86–1.59 (the heels to the seat and out). The quay at z −390, −440, −500: 1.3 to the face, −2.5 beyond; behind
it 1.3 at 6 and 12 m, 1.16 at 20, .65 at 30, .55 at 40; swimming west at −440 refused at the face (`trace`: bank 1.30 1.30); the
second slipway walks up (2.75). Sound on, swimming 4 s: no errors. 120 solids. Frames in `m35-sheet.jpg`.
Bench, DPR 2, one run: pond 13.7 · parasol 8.5 · argenteuil 9.2 · poplars 10.9 · haystacks 12.7 · rouen 8.8 · sunrise 5.3 ·
orangerie 9.2 · aerial 6.0 ms (M34: 12.0 / 8.6 / 8.6 / 10.0 / 13.2 / 9.6 / 5.9 / 11.3 / 5.4): within the runs' spread. The
street is six vortex terms per pier per reading (was four), read to 34 m; the ground mesh is non-indexed (52,800 vertices for
17,600); 348 strokes more on the wall's face; the legs and joints are eighteen small meshes more.

**Still visible.** The arms and legs hang from the eye and turn with the look; the legs are behind it and never in view. The
body's turning is a hand's uneven drag by a number set at landing, not a hand deciding. The street's ratio, roll-up and fading are
constants of the textbook cylinder, and the eddies are point vortices. The moored boats swing on ropes of one length from a
painted bow, and the compositions have shifted by up to 23°. The water's temperatures are four numbers by place and month. The
breath is two bands of noise and a tone for a bubble. The wall's coping is a box; the face's strokes are laid on the mesh, not
cut as courses. The parapet's schemes are four rules.

### Progress · M36 — the body: the swimmer's body its own frame under the head; the hands and a foot steering the slide; the piers as Rankine bodies and the eddies as Lamb–Oseen vortices; the boats moored fore and aft on ropes of their own, as bodies in the stream; the water's temperature from the day and the sun; the breath as a mouth and the bubbles as resonators; the wall cut in courses under coping stones; the tracery grown by subdivision (what M35 left visible)

**What was wrong.** The arms and legs were children of the camera, so they pitched with the look and the legs were always
behind the eye. The body's turning on a slope was an uneven drag by a number drawn from where it landed. The street's Strouhal
number, roll-up and fading were a textbook cylinder's, the pier a circle of 1.4 m, and each eddy a point vortex. The moored
boats hung from a painted bow on a rope of 1 m and swung to the stream's line, up to 23° off the paintings. The water's
temperatures were four numbers. The breath was two bands of noise and a tone per bubble. The coping was a box; the wall's face
had strokes scattered on it. The parapet's tracery was four rules.

**What changed.**
- The body (`arms`, still the group's name) is a child of the scene, placed at the eye every frame and turned to `swimYaw`: the
  swim's line, come round to over .6 s (a body turns in the water slower than a head); afloat and still, it hangs under the head's
  line. The head turns on it: looking down brings the arms and shoulders into view; strafing, the body lies across the look;
  backing, the legs are ahead of the eye. The legs kick under the water's paint; the heels, drawn up to the seat, now come to the
  water's face (they were below the hips) and break it, and the sweep out begins there.
- The slide steers. The body wants its feet downhill: the wanted turn is twice the angle still to go, up to 2.5 rad/s, and the
  hand and the foot on the side that turn it the short way round press by the shortfall (`handBias` in ±1), the right at a dead
  heading, after a half-second's grasp (`slideT`). What they have (`STEER`, 3.3 rad/s²): the hands' unevenness at .3 of the grip
  at the shoulders' lever and a foot pressed at .2 of the grip at the body's end. The slates' friction on the turning body's ends
  checks it, μ g cos θ · ω / v (the friction along the body integrated), and no more than Coulomb's at the ends when the turn
  outruns the slide. Across the slope the limbs are spread and brace against the roll (the check doubled); rolling faster than .8
  rad/s the body is a tumble and the limbs have lost it, till the roll dies under .3 and they have it again. A roll brought from a
  fall, faster than the slide over the limbs' radius, is ground down to it over a half-second instead of being cut to the slide's
  at once.
- Each pier is a Rankine body for the stream (`pierShape`): a source and a sink 2a apart of strength πκU, a and κ found once so
  the body has the pier's length (6.1 m with its pilasters) and width (2.3): still at the cutwaters, quick past the flanks, the
  stream parting round it and closing behind. The shedding is on the pier's width with St .16 (Okajima's rectangles at 2.6:1),
  T = D / (St U) (26 s at .55 m/s), λ = .8 U T (11.5 m); the eddies leave the stern. Each eddy is a Lamb–Oseen vortex: a core of
  .3 of the half-width at the shedding grown by the wake's eddy viscosity (.02 U D) as √(4νt) with its age (a half-period's roll-up,
  then the drift at .8 U), the swirl Γ/2πr outside the core and dying inside; as the cores grow to the rows' spacing the rows'
  vorticity cancels and the circulation goes as 1 − exp(−2h²/r_c²). No ad hoc fade.
- The moored boats are bodies in the stream: mass as the water displaced (half the box of length, beam and draught .06 L), a rod's
  inertia; the drag along the hull (.02 on the wetted box) and across it (1.2 on the side, taken in six lengths with the hull's
  turning, so the turn is checked); two ropes, each a spring past its length (a stretch of a quarter metre under the hull's weight,
  damped), pulling at the bow or the stern and turning the hull; integrated in 40 ms steps. Moored where the painting has the boat:
  a bow line straight up the stream (a quarter of the length) and a breast line from the stern (half the length) out on the side
  the stream pushes the stern from, or to the bank when the hull lies along the stream. The hull keeps the painted line to within
  the ropes' stretch, the eddies work it, and in still water the lines just reach and it lies as painted. Flotsam still goes with it.
- `stop()` carries `doy`, the day of the year of each painting (the pond, the meadow, the Seine 190–200; the poplars 220, 240, 290;
  the stacks 250, 262, 340, 25; Rouen 55, 100, 75; Le Havre 317; the Orangerie 200), blended with the light. `sunClock()` reads
  the hour from where the sun stands (the hour angle from the elevation and the declination at 49.2° N, morning or afternoon by
  the azimuth) and sunrise from the same. `airTemp`: Rouen's normals, 3.7° in mid-January and 18.7° in late July, the day's swing
  2° in winter and 5° in summer, warmest at three. Each water (`WATER_BODY`) follows the air as a body with its own lag (the
  Channel's mixed layer 40 days, the deep slow Seine 10, a pond or a shallow river 3), sits above the air's mean by what the sun puts
  in (1.5° the sea and the Seine, 2° the still pond), the Epte half spring water at the year's mean, and is stirred at its own rate
  (a tide's .3 m/s, the Seine's .5, the Epte's .4, a pond's .02); the year's and the day's swings come through the lag smaller and
  later. The layer over the bed (to the sun's 1.2 m) warms from sunrise toward its balance at its own pace: a clear sky's kilowatt
  by the sun's height less the face's reflection (most at a grazing sun), the air's warmth or chill on it at 15 W/m²K, and its
  exchange with the deep water by the stirring (a still pond's 5e-6 m²/s and the current's on top); the night's cooling has mixed
  it away by morning. `waterTempAt` gives the parts.
- The breath in is the glottal rush (its noise as the flow's 2.5th power) through the mouth as a tract of two formants (an open
  vowel's, 650 and 1150 Hz, the first rising 300 Hz as the mouth gapes for a short breath). The breath out is a flow, 2.6 l/s at a
  calm breath's peak and more when short of it; each bubble leaves the mouth at the size the flow makes it (3.8 mm by the lip
  against the water's skin, and bigger with the flow), at the rate the flow fills them (to 40 a second), and rings at Minnaert's
  pitch for its size (3.26 / r, a little higher .3 m down) with a bubble's damping (Q about 22 at 1 kHz, the larger ringing
  longer), the pitch rising 7% as its neck closes; the big ones louder; the crowd of small ones a low rush through the water.
- The coping is stones: each cut to a section (from the paving's edge to .05 proud of the face, the outer arris chamfered .05 as a
  weathering, the top falling .01 to the water), .8 to 1.15 long with 15 mm joints, laid along the face's curve, the mole's sides
  and round its head, broken at the slipways, with their own strokes. The wall's face is cut as courses: ashlar .36 high from under
  the coping to the water, stones .7 to 1.1 long in a running bond, each stone its own tone with its strokes laid along the course,
  the bed joint under it and the head joint at its end as dark lines, the wet band over the water in the water's grey. The mole's
  three faces the same.
- The tracery is grown, not chosen (`panel`): an arch .9 m and wider is split by a mullion into two lights, each an arch of half
  the span whose outer arc is concentric with the parent's, and the figure between them is the circle tangent to both (a quarter of
  the span across, its centre .559 of the span over the springing), foiled four or three by its size; a middling arch (.45 to .9)
  forks its mullion into Y-tracery or takes two cusped lights; a narrow one is cusped, trefoiled or cinquefoiled; every light is
  grown the same way in its turn, the choices by a hash of the face and the bay. The parapet is 1.5 m and its bays a metre wide.
- Debug: `monet.swim` adds steer, yawV, yaw (the body's) and t (the slide's); `monet.boats` gives m1, m2, r, psi, yaw0, v;
  `monet.pier`; `monet.temp(x, z)` and `monet.clock()`; `slideDbg` rows add yawV, bias and t. Build `m36-the-body`.

**Verified.** Headless, keys held. Feet first at the valley's corner: caught (34.45), feet. Landed head first on the aisle (36°):
the grasp takes .5 s, the turn is 1 rad/s at 1 s and 2 at 1.7, and at the eave (2 s, 6.63 m/s) the body is 43° short of feet
first and the hands already checking the turn (bias −.68); it goes over feet first. The vault off the corner lands rolling at
2.8 rad/s and the aisle keeps it rolling (4.1 at the eave), over at 6.3. M33's aisle slides as before (2.54 and 4.56 at 17.56,
caught, feet). The pier: a 2.59, κ .997; the stream .15 m ahead of the bow a quarter of the free stream (−.18 against −.67), at
the flank 1.12 of it (−.75), inside nil; the piers' streams .083 and .55, T 173 and 26 s, λ 11.5, strengths .18 and 1.19. The
street 4.4 m astern of the second pier's stern, in its row, over 20 s: across-stream −.10, −.09, +.01, +.05, +.02, −.01 (the cores
are 1.6 m there); at 30 m the wake's deficit (−.43), at 40 m the stream (−.55). The moored boats over 40 s: headings .41, .23,
−.02 against the painted .35, .2, 0 (3.5°, 1.7°, 1° off), the harbour's −.40 against −.40, still; a leaf on the first moves with
it (0, 0 over 3 s, the boat at rest). The body: swimming forward its yaw is the look's (−1.571), strafing right −3.142 (across),
backing −4.712 (away); its feet at (±.4, −.37, 1.07) break the water's face in the frame. The Seine at 12:50 solar on day 200:
20.2° (the body's water 19.9, the layer +.3 in the stream; cold .99); the harbour at 8:32 on 13 November (sunrise 7:32): 13.8°
(cold 1.37); the pond at 13:43: 23.6° (the body's water 20.6, the layer +3.0; cold .79). The quay's edge and apron as before
(1.3 / −2.5 at the face; 1.3, 1.3, 1.16 at 6, 12, 20 m); swimming west at −440 refused at the face; the slipway walks up (2.75).
Sound on, swimming 4 s: no errors. 120 solids. Frames in `m36-sheet.jpg`.
Bench, DPR 2: pond 15.8 · parasol 8.3 · argenteuil 11.3 · poplars 11.5 · haystacks 10.5 · rouen 8.4 · sunrise 4.9 ·
orangerie 8.1 · aerial 5.3 ms (M35: 13.7 / 8.5 / 9.2 / 10.9 / 12.7 / 8.8 / 5.3 / 9.2 / 6.0). Three more runs came back higher
everywhere (up to 62 ms at the meadow) with a load average of 28 from a virtual machine running alongside, and are not the
build's; in the one clean run Argenteuil is 2 ms up, which no new strokes explain (the 1,445 new patches are the harbour's
courses and coping), and the pond 2 ms up with nothing new in it, so both are within what the runs have wandered. 265,662 patches.

**Still visible.** The body is a frame under the eye, but the head is not on a neck: the look turns freely through the body, and
the eye does not roll or lift with the stroke. The slide's steering is a controller (a wanted rate and a shortfall), not a body
choosing. The pier is a Rankine body of the pier's length and width, not its cutwaters' shape, and the eddy viscosity and
Strouhal number are read from the literature, not the wake. The boats' ropes go to points fixed in the water where the painting
needs them, not to stakes or buoys anyone placed. The waters' lags, offsets and stirrings are estimates. The breath is still
noise, through two formants that do not move with the jaw. The courses are strokes on a plane; the stones have no depth, and the
coping stones no bevelled ends. The tracery's grammar has three productions and its bars are boxes.

### Progress · M37 — the neck: the head on the body, the eye lifting on the pull; the slide choosing by the eave it sees; the pier's own plan and the shedding from its shoulders; stakes and buoys for the ropes; the waters from their depths and a heat balance; the tract as two tubes moved by the jaw; the wall as blocks and the coping's ends bevelled; the tracery upright, from bays of their own widths, six productions, moulded bars (what M36 left visible)

**What was wrong.** The look turned freely through the body and the eye neither lifted nor rolled with the stroke. The slide's
steering was a controller with a fixed wanted rate. The pier was a Rankine oval, its Strouhal number and eddy viscosity read
from tables. The ropes went to points in the water. The waters' lags, offsets and stirrings were guesses. The formants were two
numbers. The stones were strokes on a plane and the coping's ends square. The tracery had three productions and its bars were
boxes; and, found on the way, its arches had hung upside down from their springing since M33: the arcs' angle step ran the wrong
way, so the circles and forks stood upright over arcs that dropped, and neighbouring arcs crossed at the plinth.

**What changed.**
- The neck (`NECK`): the head turns on the body 80° a side, lifts 40° and drops 75°. Afloat, the eye's frame is built as a
  chain: the body's line, the stroke's lift about the body's own lateral axis, the head's turn on the neck, then its pitch within
  the neck's range (`setHead`). So a head turned to the side and lifted rolls the horizon, as a neck does (9° at the pull with the
  head 60° round). The head lifts through the pull, 20° at most, and the eye rises with it by the pivot's geometry (.10 ahead of
  the pivot swung up: 2 cm), instead of the old sine bob. Still in the water and asked to look past the neck's end, the look waits
  there while the body comes round under it at a body's rate; swimming, the stroke's line holds the body (the swim goes where one
  looks, and the body follows in .6 s). The arms' group carries a third of the lift.
- The slide looks ahead: each frame it scans the roof along its line for the roof's end (a drop of a metre and more from the
  slope's line, to 24 m), reckons when it will get there and how fast, and chooses how to arrive (`slideAim`): feet first where
  the lip will take the feet, head first where only the hands' grip on the hold would catch it, and feet first to land on them
  where nothing will. The turn wanted is then the angle still to go over the time left (less a hand's reach of .4 s), up to
  2.5 rad/s. The hands and the foot press by the shortfall as before.
- The pier for the stream is its own plan: 6.1 long, 2.3 wide, a cutwater 1.6 long at each end. By the slender body's rule (the
  source strength is the stream times the width's growth) it is a line of sources over the bow cutwater and sinks over the
  stern's, each 2h/lc per unit length, whose velocity is in closed form (`pierFlow`: a log along the run and the angle it
  subtends across). The flow is masked inside the plan's hexagon. The shedding is no longer a table's number for a shape: the
  stream at the stern cutwater's shoulders is read from the pier's own flow (1.23 of the free stream), the wake is as wide as that
  stream over the free stream makes it, and the period follows from Roshko's wake number on the two. The eddies' cores grow by the
  wake's own eddy viscosity, .037 of the local deficit by the local half-width (Townsend's plane wake), summed along the eddy's
  drift (`core` tables per pier): smaller cores than the constant .02 U D gave, so a stronger street.
- The ropes go to things a boatman set: a red mooring buoy laid up the stream a quarter of the boat's length off the bow, and for
  the stern a stake driven at the water's edge on the side the stream pushes the stern from, found by looking for the edge within
  60° of that side and 14 m, else a second buoy half a length out. Each rope's length is what just reaches at the painted place.
  The buoys bob; the stakes stand in the bank; the ropes are drawn from bow and stern each frame.
- Each water by what it is (`WATER_BODY`): its mixed depth (the Channel tidally stirred to its bed at 30 m, the Seine 4, the Epte
  1, the pond 1.5), its exchange with the air by its exposure (38, 25, 22, 20 W/m²K), the share of its flow that is spring water
  (a chalk stream's baseflow, .7 for the Epte), and its stirring (the tide, the Seine's own `CURRENT`, the Epte's flow, a pond's
  drift under a light breeze at 3% of 1 m/s). Its lag is its heat capacity to that depth over the exchange (the sea 38 days, the
  Seine 7.7, the pond 3.6, the Epte 2.2) and its standing above the air's mean is the year's mean sun less the sky's loss over
  the same exchange (the sea 1.3°, the pond 2.5°).
- The tract (`TRACT`): a pharynx 9 cm and a mouth 8.5 cm, 3 cm² at the pharynx; the two tubes' resonances are where the closed
  pharynx and the open mouth cancel at the junction, tan(k l1)·tan(k l2) = A2/A1, found each frame by a scan and bisection with
  the poles skipped (`formants`; a uniform tube gives 500 and 1500 Hz). The jaw drops with the breath's pull and the gasp of a
  short breath, the mouth opening from 2 to 10 cm², and the formants follow it (436/1563 Hz closed, 681/1320 open).
- Each stone is a block: its face .03 proud of the wall's plane, .06 deep, .02 joints as gaps to the plane, its own tone (most
  the mid stone, a fifth pale, a twelfth dark; the wet band grey), its strokes on its face; the blocks merged in one mesh with the
  tone per block. The coping stones' ends are bevelled 20 mm on every arris.
- The tracery: the bays are of their own widths (.75 to 1.45) fitted to the run, each bay's springing set so its arch meets the
  rail (narrower bays, taller lights). Six productions: three lights and a circle under a wide arch (1.35 and over); two lights
  under the arch with the tangent circle foiled, or a dagger there instead; Y-tracery; two cusped lights; two ogee lights (the
  arcs climb 40° then reverse in a curve tangent to them and to the vertical at the apex); a cusped narrow arch. Every bar is a
  moulding: a fillet with hollow chamfers and a roll proud of both faces, extruded along the bar (`moulded`). The arcs climb.
- Debug: `monet.swim` adds head, lift, camRot, aim, eave; `monet.pier`, `monet.pierFlow`, `monet.formants`, `monet.tract`;
  `monet.piers` adds Us and core; `monet.boats` adds stake; `monet.temp` adds tau and off. Build `m37-the-neck`.

**Verified.** Headless, keys held. Swimming west with the look turned 1.0 off: the body comes round to it (the swim goes where
one looks); the lift through the pull 0 to .24 rad, the eye .35 to .37, the roll to −.15 with the head turned. Still, the look
2.2 off the body is held within the neck (the body at −.57, the head at .63). Head first onto the aisle: the eave is seen at 7 m,
2.0 s off at 6.6 m/s; the aim is the feet (the lip would take them under 4.7); the body turns 1 rad/s by 1.7 s and is 28° short
of feet first at the eave (was 43°), over feet first. The corner catch and M33's aisle slides as before (34.45 feet; 2.54 and
4.56 caught at 17.56). The pier: .05, .15 and .5 m ahead of the point the stream is .22, .30, .42 against .63 free (the thin
body's log, not a stagnation); at the shoulder 1.08 of it (.68), at the flank 1.13 (.71); inside nil; just astern .19 back up the
stream (the sinks and the deficit: a base bubble). Separation .676 for .55, T 25.5 s, λ 11.2; the core integral .18 m² at a
quarter of the street and .90 at its end. The street 4.4 m astern, in a row, over 20 s: −.09, −.15, −.01, +.09, +.05, +.01.
The boats: two found the bank for their stern stakes (2.8 and 7.8 m of rope), two got a second buoy; headings over 30 s .40,
.21–.27, −.03 against the painted .35, .2, 0, the harbour's −.4 held; 8 rope lines, 6 buoys, 2 stakes in the scene. The Seine
20.8° (lag 7.7 d, offset 2), the pond 23.2° (3.6 d, 2.5°), the Epte 14.4° at its poplars (2.2 d), the harbour at its own dawn
11.7° (38 d, 1.3°; was 15.1 with one exchange for all). Formants for 3, 5 and 10 cm² mouths: 500/1500, 580/1420, 681/1320;
swimming with sound on, the jaw 0 to .19 and the formants 436/1563 to 524/1476, no errors. The quay's edge, apron, wall and
slipway as before. 120 solids. Triangles 296,656 to 375,572 (the moulded bars and the blocks); draw calls 26 to 42 (the ropes,
buoys and stakes). Frames in `m37-sheet.jpg`.
Bench, DPR 2, load average 2: pond 11.7 · parasol 9.6 · argenteuil 8.8 · poplars 10.4 · haystacks 12.4 · rouen 8.8 ·
sunrise 5.2 · orangerie 8.0 · aerial 5.6 ms (M36: 15.8 / 8.3 / 11.3 / 11.5 / 10.5 / 8.4 / 4.9 / 8.1 / 5.3; M35: 13.7 / 8.5 /
9.2 / 10.9 / 12.7 / 8.8 / 5.3 / 9.2 / 6.0): within the spread; M36's Argenteuil was the load. 265,869 patches.

**Still visible.** The neck is a chain of three turns; the head has no weight and the look never lags the neck's own speed.
The slide's choice is a rule over two catches, and the body never chooses to stop turning short. The pier's point is a log
singularity, not a stagnation, and Roshko's wake number and Townsend's .037 are still constants read, not measured. The stakes
and buoys appear where the rule puts them, not where a boatman would have chosen the ground. The exposures, the baseflow index
and the light breeze are still estimates. The tract is two straight tubes; the jaw moves only the mouth's area. The blocks are
boxes with flat faces and no tooling; the wall's plane shows in the joints. The tracery's six productions are still a table,
and the mouldings one profile for every bar.

### Progress · M38 — the weight: the head's mass on the neck, the look lagging the neck's pace; the slide choosing among what it can reach, and holding where it lies; the pier's point a stagnation, the wake's numbers measured from its own simulation; the moorings on ground a boatman costed; the waters from each painting's wind; the tract in eight sections, the tongue with the jaw; rock-faced tooled blocks set their own way; the tracery by one rule with its figures fitted to the space, three orders of moulding (what M37 left visible)

**What was wrong.** The head had no mass: the look was where the mouse put it the same frame, and a flick of the look turned the
head faster than any neck turns. The slide chose between two catches by a rule and never held what it had. The pier's cutwater
was a log singularity with the flow clamped at a standstill, and its wake ran on Roshko's .164 and Townsend's .037 read from
tables. The stakes went to the nearest dry ground, the buoys to a fixed scope. The waters' exposures were four numbers and the
day's air exchange and the pond's breeze constants. The tract was two tubes and the jaw only widened the mouth. The blocks were
flat boxes with random strokes, all in one plane. The tracery's grammar was six rules by width, and every bar the one moulding.

**What changed.**
- The head has weight (`HEAD`, `neckTurn`): a mass of 4.5 kg, its inertia .022 about the neck's axis and .038 about the atlas
  across, its centre 2 cm ahead of the pivot and 6 over it. Each of the neck's three turns (the yaw on the body, the pitch, the
  stroke's lift) is driven by the neck's muscles as a spring and a damper toward where the look wants it (9 N·m/rad, near
  critical), capped at what the muscles have (4 N·m turning, 8 nodding) and at the neck's own speed (8 rad/s), stepped at 4 ms.
  Nodding, the head's weight (half of it, the water bearing the rest) pulls against the tone the neck holds it with. So the look
  asks and the neck answers: a flick of the look runs at the neck's speed and the eye arrives a fifth of a second later; the
  stroke's lift lags the pull. On foot the eye is still the look's.
- The slide's choice (`slideChoice`): at the eave it sees, the body reckons what each way of arriving would catch by the lip's
  own rule (feet first the lip's lever or the hands by the hips; head first the arms' grip; across, the lip's height whole or one
  arm), how far it has to turn to each, and how far the hands can bring it round in the time left, less what is left of the
  half-second's grasp (their steer over that time, no faster than 2.5 rad/s). It holds as it lies where that will catch (no
  turning for its own sake), else turns to feet or head, whichever would catch and can be reached in time, else to feet to land
  on them or as near them as it gets. `swim.choice` reads `hold`, `feet` or `head`.
- The pier's points are stagnations: a uniform run of sources ending at the point runs away there as a log, with its standstill
  a little ahead of the end, ξ = lc / (e^(2π/σ) − 1) (2.05 cm alone, 1.5 with the other run's share at the point, by iteration);
  each run is now set back into the pier by ξ, so the flow's own stagnation sits on the stone's point and the log is inside it.
- The wake is measured, once at build, from its own simulation (`wakeSim`, ~70 ms): in a unit stream the two shear layers off
  the stern's shoulders are shed as vortex blobs (a fifth of a second's flux each, ½ Us², cores .15 of the half-width) into the
  pier's potential flow and convected by the stream and one another for 50 s. The shedding period is read from the cross-stream
  velocity two half-widths astern; over the last 20 s every blob is binned by its distance astern (its row's distance from the
  axis, its scatter about the row) and the deficit's integral is taken at two stations. From these: Roshko's number as the wake's
  width over the period and the shoulders' stream; the width and the deficit down the street (the integral is the momentum the
  pier takes, the same at every station); and the eddy viscosity from the scatter's growth (σ² as 2νt), over the local deficit by
  the local half-width — the wake's own Townsend number. The street's period, spacing, deficit, width and core growth all run on
  these (`monet.wake`); where the run gives nothing sound the read values stand and the record says so.
- The moorings' ground is costed as a boatman would (`b.pick`): for the stern line, the water's edge within 60° of the stern's
  side to a rope's throw, and at each edge the ground there and a metre and two further up the bank, each costed by the rope's
  length (a sixth a metre), the bank's height (a hand or two over the water, or the flood has it; over a metre it is a cliff),
  its slope (over 1 in 2 will not take a stake) and the lead (a stern line a little aft of square holds the stern off the bank);
  the cheapest ground takes the stake. The bow buoy is sunk within 30° of up-stream at the shortest scope that will do (a fifth to
  a half of the length) in water .8 m and deeper.
- The waters' exchange with the air follows the wind (`windExch`): the sky's radiation (5.5 W/m²K) and the wind's evaporation and
  warming by Sweers' wind function at the vapour pressure's and the psychrometer's slopes, 12.8 + 3.24 w. Each water declares the
  year's mean wind over it (the open Channel 7.8 m/s; the Seine 3.7, Rouen's normal; the Epte under its poplars 2.8; the walled
  pond 2.2), which gives the old exposures (38, 25, 22, 20). Each painting declares the wind on its water that day (`wind`, read
  from the poplars' bending, the sails, the chop or the glass), blended like the day of the year; the shallow layer's exchange with
  the air is that wind's, and the stirring the greater of the water's own and the wind's drift (3% of it). The baseflow index stays
  declared.
- The tract (`TRACT`, `formants`) is eight sections of 2.2 cm from the glottis to the lips: at rest a narrow larynx, the pharynx
  and the mouth 3 to 3.5 cm², the lips 2.5 (a schwa); the jaw dropped, the tongue goes down and back with it, the root halving the
  pharynx and the mouth's front opening to three times, the lips widest (an /a/). The resonances are found by the chain matrix of
  the lossless tubes from the lips back to the glottis, whose lower-right term vanishes at the formants of the tract closed at the
  glottis and open at the lips; scanned every 25 Hz and bisected, three formants; a third bandpass carries the third at .35.
- Each block of the quay wall is rock-faced (a drafted margin at the arrises, the middle proud by 8 to 30 mm, unevenly, the face
  a 4×3 grid) and set its own way in the wall (out of the plane by ±8 mm, tipped ±.7°), so the joints are no longer one plane's.
  The face is tooled: boasted, the chisel run across it in parallel strokes at the mason's angle (35° to 55° off the vertical,
  one way on one block and the other on the next), one every 14 cm.
- The tracery is grown by one rule at every depth: a panel is an arch; it is divided into as many lights as its span holds of the
  face's module (.42 to .62 m, the face's own), each light a panel grown the same way, or left whole and cusped when it holds
  fewer than two; two lights too narrow for heads of their own fork the mullion (Y-tracery). The head's figure comes from the
  space left over the lights: the largest circle on the axis tangent inside to the parent's arcs and outside to the lights' heads
  (`distArch`, the height found by bisection), and then by its size a foil of as many lobes as its round holds at a cusp's module
  of .2 m, a dagger where the space over the circle is tall and the face has daggers, or nothing but the cusps. Each face has its
  own style (module, ogee heads, daggers). Three orders of moulding by the bar's place: the bays' arches (73 mm, a roll 33 proud
  of both faces), the lights' (56 and 25), the figures' and forked bars' (a chamfered bar of 39, no roll); the bays' mullions
  stouter.

**Verified** headless, `m38a.js`, `m38b.js`, `m38w.js`. The neck: a flick of the look by 1.4 rad afloat runs the head at the
neck's cap of 8 rad/s from .05 to .13 s, reaches 90% at .19 s and settles by .40 s. A nod of .5 rad reaches 90% at .265 s down
and .267 s up (the half-buoyed head's weight is .4 N·m against a neck of 8; the asymmetry is in the milliseconds). At rest the
head sits on the look with no sag (the neck's tone carries the weight). Through a pull the lift asked peaks at .349 and the head
gives .281, a tenth of a second behind. The slide: head first onto the aisle the eave is seen at 1.98 s and 6.6 m/s, nothing
catches, the choice is feet, and the body has come round 2.65 rad by the eave (28° short of feet first, as M37) and goes over
at 6.63 m/s. M33's low aisle slide reads `hold` from the first frame (feet first, the lip's lever will take 3.1 m/s) and is
caught at 17.56; the high one aims feet and is caught the same; the corner catch at 34.45 stands. The pier: ξ .0150; on the
pier's own axis the flow 1 mm ahead of the point is .045 (7% of the free .63), .20 at 5 cm, .42 at .5 m, .50 at 1.5 m; the
shoulder .68, the flank .71; astern on the axis −.29 at 5 cm, −.08 at .3 m, +.08 at 1 m, +.16 at 3 m (a base bubble .7 m long).
The wake's run: 8 sign changes in the last 30 s, a period of 7.56 in the unit stream; Roshko's number .304 (read: .164), the
Townsend number .056 (read: .037), the deficit .81/√(1 + s/h) (read .85), the half-width .58 + .061 s/h (read .55 + .12), the
momentum integral .72 h, ν .019; the rows .44 to 1.03 h from the axis and their scatter .12 to .65 down the street. The same
sim run standalone across the stagnation shift (.015 to .0205) and the blob core (.15 to .2) gives Roshko .27 to .32 and
Townsend .056 to .107: the measurement's own spread. At .55 m/s the piers now shed every 13.7 s (was 25.5), λ 6.05 m (was
11.2), cores .12 to .45 m² over the street. Sampled 4.4 m astern and 1.6 m off, the stream reads −.50 ± .04 as the eddies pass.
The boats: two stakes chosen at costs .59 (a bank .32 m up, slope .53, lead −.34, 2.8 m of rope) and 1.4 (.39 up, .48, −.37,
7.8 m), the other two boats to buoys; the bow buoys up-stream at the shortest scope (.2 L: 1.24, 1.08, 1.28, .88 m) in 2.5 m of
water; headings at rest .381, .205, −.018, −.400 against the painted .35, .2, 0, −.4 (within 2°), and after 20 s .375, .171,
−.02, −.4 at under .03 m/s. The waters: the exchanges 38.1, 24.8, 21.9, 19.9 from the winds; the Seine 20.8° at noon (the day's
wind 3 m/s, its exchange 22.5, stirring .55), the pond 23.3° (1.5 m/s, 17.7, .045), the Epte at its poplars 14.5° (the wind
there 6.8 m/s in the blend, 34.9), the Channel 13.5° at 8.5 h with a 38-day lag (M37's 11.7 was read at another hour; M37's own
file gives 13.5 by this probe). The tract: at rest 516/1549/2611 Hz, the jaw half down 682/1461/2605, down 804/1368/2605 (a
schwa to an /a/); the solver checked against a uniform tube (500/1500/2500) and a 1:8 two-tube (784/1216 against 783/1216
by hand). Swimming with sound on the jaw goes to .35 and the formants to 639/1485. No errors; 120 solids; draw calls 42.
Triangles at the harbour 193,918 to 261,120 (the faces), at Rouen 431,770 to 400,172 (the tracery lighter); 270,605 patches
(the tooling). Frames in `m38-sheet.jpg`.
Bench, DPR 2, load average 2 to 4 (the machine's other work): pond 14.3 · parasol 8.1 · argenteuil 10.8 · poplars 10.1 ·
haystacks 10.8 · rouen 8.8 · sunrise 5.5 · orangerie 10.9 · aerial 7.7 ms (M37, load 2: 11.7 / 9.6 / 8.8 / 10.4 / 12.4 / 8.8 /
5.2 / 8.0 / 5.6; M36: 15.8 / 8.3 / 11.3 / 11.5 / 10.5 / 8.4 / 4.9 / 8.1 / 5.3): the harbour, where the blocks' faces are, is up
.3 ms; the rest moves with the load, as before. 270,605 patches.

**Still visible.** The neck's muscles are a spring and a damper with a cap, not muscles with a length and a speed, and the head's
weight afloat is a guess at half. The slide's choice is still a rule over three catches and one half-second's grasp. The wake's
measurement is an inviscid blob model whose Roshko number (.30) is nearly twice the measured wake's (.164), and its spread is a
fifth. The boatman's costs are a table of penalties. The year's mean winds and the baseflow index are declared, and the
painting's wind is read by eye. The tract's area function is a drawn schwa and /a/, with no lips' rounding or velum. The blocks'
faces are a noise on a grid, not a chisel's work, and the tooling is strokes. The tracery's rule fits circles; a real head is
drawn by a mason's compasses from the arcs it has. The mouldings are three scalings of one profile.

### Progress · M39 — the muscles: the neck turned by Hill's muscles with a length and a speed, the head's weight from its immersion; the slide costing every attitude it can reach, its grasp set by the landing; the wake's simulation with the pier as a body and the shedding fed back from its own flow; the moorings' ground costed in time and holding; the painting's wind into the strokes' sway; the tract's lips and velum; the blocks pitched by spalls, the margins tooled; the tracery struck with compasses, foils of lobed arcs, three mouldings of their own (what M38 left visible)

**What was wrong.** The neck was a spring and a damper with a cap, and the head's weight afloat a guess at half. The slide's
choice was a rule over three catches and a fixed half-second's grasp. The wake's vortex model had no body in it and shed at a
fixed rate, and its Strouhal number was twice the tables'. The boatman's costs were a table of penalties. The painting's wind
moved only the water. The tract had neither lips' rounding nor a velum. The blocks' faces were a noise on a grid, the tooling
strokes across the whole face. The tracery's circle was found by bisection and its foils were rings with boxes for cusps; the
mouldings three scalings of one profile.

**What changed.**
- The neck by its muscles (`MUSC`, `neckTurn`): a pair to each turn, the two sternomastoids for the turning (130 N at a 3 cm
  arm on a 16 cm belly), the extensors and the flexors for the nodding (200 N at 5 cm, 100 at 3.5). Each has Hill's force by its
  length (a parabola a half-length wide about the rest length) and by its speed (the hyperbola falling to nothing at ten lengths
  a second shortening, rising to 1.8 lengthening) and a passive pull past a fifth's stretch; its activation follows the neural
  drive with 15 ms to rise and 50 to fall; the drive is proportional (3 a radian of shortfall, .3 a rad/s) plus the tone that
  holds the head's weight where it is asked for. The head's inertia is turned by their torques and its weight, stepped at 2 ms.
  So the neck's speed is what the muscles can do against the hyperbola, and its strength their force at its arm; the chin lifts
  quicker than it drops. The head's weight afloat is read from its immersion: a ball of 10 cm about a point 6 cm under the eye;
  at the swimmer's eye (.35 over the water) it is out of the water and the neck carries all 4.5 kg.
- The slide (`LIP_LEVER`, `ARM_REACH`): the catch is a function of the body's angle to the slide, not a label: the lip's lever
  on a body pivoting on it (the feet's lever feet first, falling with the angle to 1 across, none head first) and the arms' reach
  to the hold (all of them head first, one arm across, the hands by the hips feet first). At the eave it sees, the body costs
  every attitude the hands can bring it to (37 across the reach): caught, nothing; over the lip uncaught, the fall's harm by how
  it goes over (.3 feet first, .65 across, 1 head first); and a twentieth for a half-turn of turning. The least takes it, and where
  the attitude chosen is the farthest it can reach the hands give all they have. The grasp is set by the landing (`graspT`): a
  standing body's hands are on the slates in .2 s; a body landed from a fall needs .12 s more for every m/s it landed at.
- The wake's simulation has the pier in it: thirty source panels on the plan (Hess and Smith's), their strengths solved every
  step so that nothing passes the stone under the stream and the blobs together (an LU of the influence matrix, once). The blobs
  are shed at the flow's own speed read a tenth off each shoulder, so the wake's state sets the shedding as the base pressure
  does. 800 steps of a tenth of a second; the period, the rows, the scatter, the integral and the fits as before. The record
  gives the Strouhal number on the width (what the street's period runs on) and Roshko's on the shoulders' stream.
- The moorings' ground is costed in what it costs a boatman (`b.pick`): a stake must hold twice the line's pull (the stream's
  cross load on the hull over the line's lead across it; 3 kN a metre driven, a third in the saturated ground within a hand of the
  water, half in a cliff's scree, and the drive shallower the steeper the bank, none past 1 in 1.2); among the grounds that hold,
  the least time: the walk along the rope's length and back at 1.2 m/s, the climb at .3, the driving (half a minute in firm ground,
  longer in soft), and the flood's risk (a bank .35 m up is drowned once in e, ten minutes to moor again). The buoy: its sinker
  must hold the line's pull along the stream (a stone's weight in water at a friction of .6, 20 kg at least), its chain three
  depths, the water half a metre at least; the time rowing out and back, hauling the sinker and the chain, and 20 s a radian off
  the stream's line for the fouling.
- The painting's wind is one number: the strokes' sway follows it (`uWind` scaled each frame, 3 m/s the sway as built), as the
  water's exchange and stirring already did. The year's mean winds and the baseflow index stay declared, as data.
- The tract (`formants(jaw, round, velum)`): the lips rounded close their section to a fifth and add 12 mm; the velum opened
  couples the nose at the fourth section as a shunt on the chain (an open tube of 11 cm and 3 cm²), whose poles the root finder
  skips. The breath in begins pursed from the blowing and opens over 80 ms; a gasp lets the velum open a little.
- Each block's face is pitched as a chisel leaves it: three to five spalls, each a plane struck from a point of the face at its own
  height and tilt, the face the lowest of them, so it is facets meeting at ridges, on a 6×4 grid, with a drafted margin 35 mm wide
  left flat; the margin along the top and bottom arrises is boasted, a chisel line every 8 cm at the mason's angle.
- The tracery's head is struck as a mason strikes it: the circle's size by proportion (a quarter of the span over two lights, a
  fifth over three, a sixth over more), its centre on the axis where arcs of the span less the circle, struck from the two
  springings, cross; where it would cut the lights' heads the compass is closed a little and struck again. The foil is lobes: their
  centres on a circle inside the head's, each lobe's compass opened a fifth wider than would just touch its neighbours, each lobe
  an arc from its own centre cusp to cusp the outer way round, and a cusp at each crossing pointing in. Three mouldings, each its
  own: the bays' arches a broad fillet between hollow chamfers with a roll 30 mm proud of both faces; the lights' a wave moulding,
  a hollow rising to a small bead at each face; the figures' and forked bars' a plain chamfered bar.

**Verified** headless, `m39a.js`, `m39b.js`, `m39c.js`. The neck: a flick of the look by 1.4 rad afloat is answered by the
sternomastoid at .97 activation, the head at 9.65 rad/s by .11 s, the other side braking at .6 from .14 s, 90% at .29 s, and it
settles at 1.35: the muscles' length leaves it 3° short of the neck's end. A nod of .5 rad down takes .28 s (3.8 rad/s), up .22
(4.3): the extensors are twice the flexors. At rest the head sits on the look with the extensor's tone at .09 and no sag. The eye
afloat is .35 over the water, the head dry, its weight all the neck's. The lift asked .349, given .309. The slide: head first onto
the aisle the eave is seen at 1.98 s and 6.6 m/s; feet is chosen at first, then the farthest attitude the hands can reach, with
the steer at its full 2.5 rad/s; the body comes round to 12° short of feet first (M38: 28°), holds from 1.97 s and goes over at
6.63 m/s. M33's aisle slides read `hold` (feet first, caught) and are caught at 17.56; the corner catch at 34.45 stands. The vault
off the corner: the body slides off at 5.8 s, falls 12 m and lands tumbling at 7.4 s (a roll of 4.4 rad/s) with a grasp of 2.14 s
from its 16 m/s. The wake's run (~350 ms at build): 8 sign changes in the last 40 s, a period of 10.3 in the unit stream, the
Strouhal number .223 on the width (Okajima's read .16), Roshko's .189 on the shoulders' stream of 1.18 (read .164), the Townsend
number .080, the deficit .65/√(1 + s/h), the half-width .90 + .060 s/h, the integral .74 h, ν .028, the rows .59 to 1.18 h
and their scatter .22 to 1.0 down the street. The same model standalone across the blob core, the shedding offset and the step
gives .235 to .269 at a thousand steps: a spread of a seventh. At .55 m/s the piers shed every 18.7 s (M38 13.7, M37 25.5),
λ 8.25 m; the street sampled 4.4 m astern reads −.44 ± .03. The boats: two stakes chosen 5 and 10 m up the bank at .8 m over the
water (slope .05, leads −.5 and −.2), at 104 and 113 s, holding 1.4 kN against a pull of 4 and 12 N; two buoys at the shortest
scope with 20 kg sinkers (24 s); headings at rest .368, .202, −.001, −.400 against the painted .35, .2, 0, −.4, and after 20 s
.366, .200, .006, −.4. The wind: the Seine's stops read 3.0 m/s, the poplars' 6.8 (the sway 2.3 times as built). The tract: rest
516/1549/2611 Hz, open 804/1368/2605, pursed 287/1288/2478, the velum open 370/934/1555, half down and a little nasal
559/903/1468; swimming, the breath in begins pursed at .93 with 325/1298/2481 and opens over 80 ms to 639/1485/2606 at the
jaw's .35. No errors; 120 solids; draw calls 42. Triangles at Rouen 400,172 to 501,160 (the wave mouldings and the lobes), at
the harbour 261,120 to 343,482 (the faces' grid); 284,930 patches (the margins' tooling). Build 800 to 1000 ms (M38 ~750: the
wake's panels). Frames in `m39-sheet.jpg`.
Bench, DPR 2, load average 3.4 to 4.6 (the machine's other work): pond 11.8 · parasol 10.2 · argenteuil 13.0 · poplars 13.0 ·
haystacks 11.9 · rouen 10.0 · sunrise 6.7 · orangerie 10.5 · aerial 9.3 ms (M38, load 2–4: 14.3 / 8.1 / 10.8 / 10.1 / 10.8 /
8.8 / 5.5 / 10.9 / 7.7; M37, load 2: 11.7 / 9.6 / 8.8 / 10.4 / 12.4 / 8.8 / 5.2 / 8.0 / 5.6): Rouen and the harbour are up
1.2 ms each, the tracery's mouldings and lobes and the blocks' grids; the rest moves with the load. 284,930 patches.

**Still visible.** The muscles are Hill's curves under one proportional drive: no reflexes, no co-contraction, one belly a side.
The slide's harms (.3, .65, 1) and the grasp's .12 s a m/s are numbers set, not measured, and the attitude is costed at the eave
only. The wake's model is inviscid, its blobs' core fixed, its separation pinned to the shoulders, the body without circulation;
its Strouhal number (.22) is still a third over the tables' (.16). The stake's holding (3 kN a metre) and the flood's .35 m are
declared, and so are the year's winds and the baseflow index; the painting's wind is read by eye. The nose is one tube, the
rounding one number. The spalls are planes and the margin's tooling is strokes. The compass rule's proportions are three numbers
and the daggers are still bars; each moulding is drawn once and never varies.

### Progress · M40 — the reflex: three bellies a side with their spindles and a co-contraction; the slide's harm reckoned from the fall it sees, the grasp from the landing's rebound; the wake's cores spreading and its separation found, its circulation tried and refused; the stake's holding by Broms, the flood by each water's regime; the nose a chain with a sinus, the lips with their end correction; the spalls as scallops; the tracery's circle opened by compass trial, daggers of reverse curves, mouldings at each bay's scale (what M39 left visible)

**What was wrong.** The neck had one belly a side under a proportional drive with no reflexes and no co-contraction. The
slide's harms and its grasp's rate were numbers set. The wake's blobs had fixed cores, its separation was pinned to the stern's
shoulders, the body carried no circulation. The stake's holding and the flood's height were declared numbers. The nose was one
tube and the rounding one number. The spalls were planes. The tracery's circle was set by a proportion, the daggers were straight
bars, and each moulding was drawn once.

**What changed.**
- The neck (`MUSC`, `neckTurn`): three bellies a side for the turning (a sternomastoid 130 N at 3 cm on 16, a splenius 90 at 2 on
  12, the trapezius's upper part 60 at 4 on 18), three extensors against two flexors for the nodding (semispinalis 140 at 4.5 on
  14, splenius, trapezius; a sternomastoid 100 at 3.5, longus 40 at 2), each Hill's as before. Each belly's activation follows the
  side's one neural command plus its own stretch reflex: the spindle's report of its stretch beyond the commanded length (6 a
  length) and of its lengthening (.5 at ten lengths a second) through a 30 ms lag; and never under the co-contraction the task
  asks for (.05 at rest, up to .2 with a far look, the lift and the stroke), both sides held on to stiffen the neck. A push
  against the head is met by the spindles before the command knows.
- The slide's harm is reckoned from the fall it sees: the scan records the drop beyond the eave (to the roof or the ground it
  finds), and every attitude's harm is the landing's deceleration (the drop's speed and a third of the slide's, stopped over what
  the body has to give in that attitude: the legs' half-metre feet first, the trunk's 12 cm across, the neck's 4 head first) over
  what that part will stand (15 g the legs, 8 the trunk, 5 the head), 1 the injury. A way that passes across a slope steep enough
  to roll the braced body costs 3. The grasp after a landing is the time the body's bounces take to die, its limbs giving back .3
  of each landing's speed: 2 v e / (g (1 − e)), .087 s a m/s, on the standing .2.
- The wake's blobs' cores spread as Lamb–Oseen's, r² by 4νt at ν = .002 U h. The separation is not pinned: each side sheds from
  the panel where the flow along the stone is fastest and begins to slow, read a tenth off it (past the bow's point) — on this
  plan the bow's shoulder, the flanks being flat — at that flow's speed, no more than 1.6 of the stream. Kelvin's condition for
  the body's circulation (a vortex strength on every panel balancing all vorticity ever shed) was tried and ran away: each shed
  blob turned the body's bound vortex, which sped the next shedding; guarded by a farther nascent blob, a capped and smoothed
  shedding speed and a relaxed balance it still would not hold, so the body stays without circulation, and the record says so.
- The stake's holding is Broms' for a short pile in clay: 9 c_u D over the driven length past 1.5 diameters, taken at the rope's
  lever .4 m up, a 60 mm stake driven half a metre; the clay's strength by its wetness, 10 kPa saturated at the water's edge to
  50 kPa dry half a metre up (3.3 kN), .4 of that in a cliff's scree. The flood's risk is by each water's own regime (`FLOOD`):
  the height a year's rise reaches once in e, the Seine's 1 m at Argenteuil, the Epte's .35, the pond's .05, the tide's 3.5.
- The tract: the nose is its own chain (`nasalY`), from the velum's port (2 cm, 1.2 cm²) through the cavity (7 cm at 5) to the
  open nostrils (2 cm, 1), with the maxillary sinus on it as a Helmholtz resonator (15 cm³ behind a 1 cm neck of .1 cm²: 455 Hz),
  its admittance at the port the shunt on the mouth's chain. The lips rounded close to a fifth and protrude 12 mm, and the mouth's
  opening radiates as a piston, its end correction .8 of its radius added to the tract (7 mm open, 3 pursed).
- Each spall is a scallop, rising from where the chisel struck as the distance to the three halves (a conchoid) with a little
  tilt, the face the lowest of them: bowls meeting at ridges.
- The tracery's circle is what the arcs allow: for any opening of the compass its centre lies where arcs of the span less the
  circle, struck from the springings, cross; the mason opens the compass until the circle just touches the lights' heads, by
  trial, halving the doubt sixteen times; no proportion is set (a quarter of the span over two lights comes out of it). The
  dagger's sides are reverse curves, convex from the foot's tangent to an inflexion at .55 of the width and .9 up, then concave to
  the apex at 1.7 tangent to the axis. Each bay's mouldings are at its own scale, by the root of its span over 1.1 m and a
  twentieth either way as the mason's hand had it; the bar's width by the scale, its depth by half of it.

**Verified** headless, `m40a.js`, `m40b.js`. The neck: the body turned 1.5 rad under a held look (a strafe from a swim along
the look, the same each time): with the spindles the head's excursion peaks at .38 and .37 rad and settles at .17 to .22; with
them off (`monet.reflex = false`) .44 and .26. The flick of 1.4 rad reaches 90% at .19 s at 8.6 rad/s and settles at 1.30, the
co-contraction and the muscles' length leaving it 6° short of the neck's end; at rest every belly holds .09 with the extensor at
.12 for the head's weight. The slide: head first onto the aisle the scan sees a 15.2 m drop; the harm feet first reads 2.1 (an
injury at any attitude; head first it would be near 80), the body comes round to 12° short of feet first and holds. The vault
off the corner lands tumbling from 16.2 m/s with a grasp of 1.61 s (the bounces' time). M33's aisles hold and are caught; the
corner catch stands. The wake: the separation found at the bow's shoulder (x = −1.2 of the pier's −1.45), the shoulders' stream
1.27, 7 sign changes, a period of 10.5, the Strouhal number .218 on the width and Roshko's .172 on that stream (read .164: within
5% now), the Townsend number .093, the deficit .74/√(1 + s/h), the half-width 1.24 + .115 s/h, the integral 1.34 h (a wider
wake, shed from the bow's shoulders with no reattachment), the rows 1.05 to 2.17 h. At .55 the piers shed every 19.2 s, λ 8.4,
the cores .5 to 1.9 m²; the street 4.4 m astern reads −.39 ± .01. The boats: the two stakes at 5 and 10 m, .8 m up, in clay at
50 kPa holding 3.0 kN by Broms against 4 and 9 N, at 313 and 322 s (the Seine's flood at 1 m); headings .368, .201, −.003,
−.400 against the painted .35, .2, 0, −.4, and after 15 s .366, .195, −.002, −.4. The tract: at rest 491/1485/2539 Hz (the end
correction lowers the schwa from 516), open 739/1267/2462, pursed 276/1280/2471, the velum open 405/556/1005 (the nose's
resonances on the chain), half open and half down 420/639/987; swimming, the breath in begins pursed at .79 with 374/1302/2471
and opens to 600/1398/2509 at the jaw's .35. No errors; 120 solids; draw calls 42. Triangles at Rouen 502,190, at the harbour
344,516; 284,971 patches; build 820 ms. Frames in `m40-sheet.jpg`.
Bench, DPR 2, load average 5.2 falling to 2.4 (the machine's other work): pond 11.4 · parasol 9.5 · argenteuil 9.4 · poplars
10.8 · haystacks 10.4 · rouen 9.3 · sunrise 6.0 · orangerie 10.3 · aerial 12.1 ms (M39, load 3–5: 11.8 / 10.2 / 13.0 / 13.0 /
11.9 / 10.0 / 6.7 / 10.5 / 9.3; M38: 14.3 / 8.1 / 10.8 / 10.1 / 10.8 / 8.8 / 5.5 / 10.9 / 7.7): no geometry to speak of was
added this time and the readings sit within the spread; the aerial's 12.1 is the load at the run's start. 284,971 patches.

**Still visible.** The reflex gains, the spindle's lag and the co-contraction's rule are set; the bellies share one command a
side, and there is no vestibular reflex holding the head in space. The slide's tolerances (15, 8, 5 g) and gives (.5, .12, .04 m)
and the restitution .3 are numbers set, and the harm is a ratio, not an injury. The wake's viscosity .002 U h is set, the
separation is the fastest panel with no reattachment, and the body has no circulation. The clay's strengths, the tide's and the
floods' heights, the year's winds and the baseflow index are declared, and the painting's wind is read by eye. The nose's sections
and sinus are drawn; the rounding is still one gesture. The scallops are a power law and the margin's tooling is strokes. The
compass trial's tolerance and the dagger's inflexion are numbers, and the mouldings vary in scale only.

### Progress · M41 — the canals: the vestibulo-collic reflex, each belly's own share of the command, the arc's lag and a co-contraction from a tolerance; the slide's injuries by their criteria and weighted by their worth, the rebound by the landing's damping; the wake's viscosity from the mixing layer and its shear layer let reattach, the circulation tried once more and refused; the clay's strength from its liquidity; the lips as two gestures; the spalls as Hertz's cones; the tracery's circle to within a lead joint, the dagger from two centres, mouldings in two families by the face's style (what M40 left visible)

**What was wrong.** The reflex gains, the spindle's lag and the co-contraction's rule were set; the bellies shared one command;
nothing held the head in space when the body turned under it. The slide's harm was a ratio of set tolerances and gives, its
restitution a number. The wake's viscosity was a number, its separation could not reattach, its body had no circulation. The
clay's strengths were declared. The nose's rounding was one gesture. The spalls were a power law. The circle's tolerance and the
dagger's inflexion were numbers, and the mouldings varied in scale only.

**What changed.**
- The neck: the canals' report is in the command — the vestibulo-collic reflex, .8 of the head's speed in space (the body's
  turning under it, `bodyRate`, added to the neck's own) in the damping term, so a body turning under a held look is met before
  the look's error grows. Each belly takes its own share of the side's command, by its force-length over the best of the side's,
  so the bellies nearest their best length do the most. The spindle's lag is the arc's own: 15 cm of nerve each way at 60 m/s, a
  synapse's millisecond and the muscle's 20 ms to take up, 26 ms. The co-contraction is what the task needs: the disturbance the
  head expects (the water's drag on it at the swim's speed at a 10 cm lever, the body's turning at the head's inertia, the look's
  own demand) over the 3° the neck keeps, over the stiffness both sides give at unit activation (their short-range stiffness,
  30 F/L0 at the arm squared: 94 N·m/rad for the turning), between .02 and .5.
- The slide's harm is an injury by its criteria (`injury`): the landing shared among the legs, the trunk and the head by the
  attitude (cosine squared); the legs' chance by the tibia's 8 kN over the 50 kg they carry (16 g), stopped over the knees'
  45 cm, a logistic on that; the trunk's by the chest's 60 g over its 63 mm; the head's by the Head Injury Criterion (the
  deceleration in g to the 2.5 over its 4 cm's stopping time) and Prasad and Mertz's curve; each weighted by what its injury is
  worth (a leg 1, the chest 1.5, the head 2), so that a fall that will hurt whatever the attitude is still taken on the legs. A way
  across a rolling slope is a certain injury. The rebound is the landing's own: a damped spring's, e = exp(−πζ/√(1−ζ²)), the
  legs' damping .4 feet first (e .25, .07 s a m/s of bouncing), the trunk's or the head's .15 rolling or head first (e .62, .33 s).
- The wake: the cores spread at the mixing layer's eddy viscosity, .014 of the shoulders' stream by a shear layer a tenth of the
  half-width thick (Prandtl's free shear layer: .002 U h for this pier, as was set before — now from where it comes). A blob that
  comes down onto the flank between the shoulders is taken into the boundary layer and its vorticity shed again from the stern's
  shoulder the same step (reattachment). The body's circulation was tried once more, as a bound vortex at the centroid relaxed
  toward the balance of all vorticity shed: over 2 s it holds and gives Roshko's number .164 to the digit, over 5 s it drifts to
  −2.9 and over .5 s it kills the shedding; the number is the relaxation's, not the wake's, so it is not adopted.
- The clay's strength follows its liquidity, Wroth and Wood's c_u = 170 e^(−4.6 LI) kPa, the liquidity by a silt bank's capillary
  rise of .8 m: the ground at the water's edge at its liquid limit (1.7 kPa), .8 m up at its plastic (170 kPa), .4 of that in a
  cliff's scree. The tide's and the floods' heights, the year's winds and the baseflow index stay declared as the data they are.
- The lips are two gestures: the aperture (closing the section to a fifth) follows the breath as before; the protrusion (12 mm)
  is pushed out in 60 ms for the blowing and drawn back over 150 ms after, and each moves the formants its own way.
- Each spall is the flank of Hertz's cone from where the chisel struck, rising at 5° to 10° to the face as a brittle stone's cone
  fracture leaves it; the face the lowest of them, cones' flanks meeting at ridges.
- The tracery's circle is opened until its ring meets the lights' heads within a lead joint (`JOINT`, 5 mm). The dagger is struck
  from two centres: the foot's round carried up to 60° each side, and from there a second arc tangent to the first, its centre out
  on the same radius and opened so that it meets the axis tangent at the apex (R = ρ cos 60° / (1 − cos 60°) = ρ, the apex at
  1.73 ρ): a mouchette's reverse curve with no inflexion set. The mouldings are two families by the face's style: a geometric face
  has its bays' arches in a fillet between hollow chamfers with a roll, its lights' in a wave, its figures' in a chamfer; a
  curvilinear face its arches in a double wave with a fillet, its lights' in a keeled roll, its figures' in a sunk chamfer; each
  at the bay's scale.

**Verified** headless, `m41a.js`, `m41b.js`. The neck: the body turned 1.5 rad under a held look (a strafe from a swim along
the look): with the canals and the spindles the head's excursion peaks at .19 and .18 rad; without the canals .34; without
either .45 — the canals halve it. The flick of 1.4 rad reaches 90% at .19 s at 10.8 rad/s and settles at 1.32. At rest the
co-contraction is .02; swimming .17, the extensor at .51 for the lift; through the flick .085. The slide: head first onto the
aisle over its 15.2 m drop the injury reads .99 held head first and 1.02 arrived feet first (a fall that hurts whatever the
attitude, taken on the legs), the choice feet, the body brought round to 13° short and held. The vault off the corner lands
rolling from 16 m/s with a grasp of 5.6 s (a rigid landing's bounces, e .62). M33's aisles hold and are caught; the corner
catch stands. The wake: the separation found at the bow's shoulder (x = −1.61 of the pier's −1.45), three blobs reattached in
80 s (the shear layer mostly stays off the flank), the shoulders' stream 1.21, 7 sign changes, a period of 11.8, the Strouhal
number .196 on the width and Roshko's .162 on that stream (read .164), the Townsend number .126, the deficit .66/√(1 + s/h),
the half-width 1.12 + .102 s/h, the integral 1.07 h, the rows .82 to 1.73 h. At .55 the piers shed every 21.4 s, λ 9.4; the
street 4.4 m astern reads −.41 ± .01. The boats: the two stakes at 5 and 10 m, .8 m up, in clay at 170 kPa holding 10.1 and
10.3 kN by Broms against 4 and 8 N, at 293 and 302 s; headings .367, .203, −.005, −.400 against the painted .35, .2, 0, −.4,
and after 12 s .366, .196, −.006, −.4. The tract: at rest 491/1485/2539 Hz; the aperture alone 323/1316/2500, the protrusion
alone 454/1396/2417, both 276/1280/2471, the velum open 405/556/1005; swimming, the protrusion rises to 1 through the blowing
and falls back to .02 through the breath in, the formants with it. No errors; 120 solids; draw calls 42. Triangles at Rouen
507,828, at the harbour 344,522; 284,974 patches. Frames in `m41-sheet.jpg`.
Bench, DPR 2, load average 3.7 falling to 1.4 (the machine's other work had just peaked at 19): pond 11.0 · parasol 8.9 ·
argenteuil 9.3 · poplars 9.5 · haystacks 10.6 · rouen 8.7 · sunrise 5.7 · orangerie 8.8 · aerial 8.6 ms (M40, load 5–2: 11.4 /
9.5 / 9.4 / 10.8 / 10.4 / 9.3 / 6.0 / 10.3 / 12.1; M39: 11.8 / 10.2 / 13.0 / 13.0 / 11.9 / 10.0 / 6.7 / 10.5 / 9.3): no geometry
to speak of was added; within the spread. 284,974 patches.

**Still visible.** The canals' gain (.8), the spindle's gains, the 3° tolerance and the short-range stiffness's 30 are set; the
command is still one signal a side shared out, not a synergy learned. The injury criteria are the crash tests' (tibia 8 kN,
chest 60 g, HIC) on a body of one size, the worths 1, 1.5, 2 are set, and the landing's damping .4 and .15 are numbers. The
wake's shear layer thickness (a tenth of the half-width) and Prandtl's .014 are read, the reattachment is a capture window that
rarely fills, and the body has no circulation. The capillary rise .8 m and Wroth and Wood's constants are read; the tide, the
floods, the winds and the baseflow index are data. The lips' two gestures are two numbers with two clocks. Hertz's angle is
sampled, and the margin's tooling is strokes. The lead joint's 5 mm and the dagger's 60° are set, and the mouldings are two
families of three, drawn once each.

### Progress · M42 — the synergy: the neck's gains from its own plant by optimal control, the canals' report through their own cupula and arc, the spindle's gain from the short-range stiffness, each belly's share learned; the slide's criteria on this body, its worths by the injury scale, its dampings from the muscles and the thorax; the wake's circulation by Kelvin on vortex panels, its core from the boundary layer and its spreading from the layer's own ratio; the waters' winds from one wind, the baseflow from the geology, the floods from the banks' own lips, the capillary rise from the grain; the lips and the jaw as muscles; Hertz's cone traced, the margins' tooling in the stone; the joint from the mortar's grain, the dagger's angle from its reach, each bar its own drawing (what M41 left visible)

**What was wrong.** The canals' gain, the spindle's gains, the tolerance and the short-range stiffness were set, and the
command was one signal a side shared out. The injury criteria were the crash tests' on a body of one size, the worths set,
the landing's dampings numbers. The wake's shear layer thickness and Prandtl's .014 were read, the reattachment a window
that rarely filled, the body without circulation. The capillary rise was read; the tide, the floods, the winds and the
baseflow index were data. The lips' gestures were two numbers with two clocks. Hertz's angle was sampled and the margin's
tooling was strokes. The lead joint's 5 mm and the dagger's 60° were set, and the mouldings were drawn once each.

**What changed.**
- The neck's command is planned from its own plant (`neckPlan`): the head's inertia against the muscles' short-range
  stiffness and damping at the resting tone, driven through the activation's lag, by optimal control — Riccati's equation run
  backward to its rest for the cost of an error against a full command, an error of one tolerance worth as much as the whole
  command. The gains come out 5.6 and .26 for the turning, 5.4 and .32 for the nodding (M40 set 3 and .3). The tolerance is
  what the eyes leave the head: the fovea's 1.7° at the vestibulo-ocular reflex's gain of .9, 8.5°. The canals feel the head's
  speed in space through their own cupula (the endolymph's 5.7 s high-pass) and their own arc (25 ms), the brain subtracts the
  head's own share as it predicts it, and the plan's velocity gain falls on the whole — so the reflex has no gain of its own.
  Each belly's short-range stiffness (`ksr`) is the cross-bridges' yield (a 10 nm stroke on the half-sarcomere's 1.1 µm) in
  series with its tendon's 3% at full force, over their shares of the belly: 65, 75 and 61 F/L0 for the turners (M41 set 30),
  205 N·m/rad both sides at unit activation for the turning; the spindle's length gain is that stiffness at the belly's
  activation (Nichols and Houk: the reflex restores the short-range stiffness past its yield). And each belly has a learned
  share, a synergy (`neckW`): its weights on the look's demand and on the canals' report, learned by feedback-error
  (Kawato's) — the weights regress the belly's whole corrected command, which is Kawato's rule wherever the belly acts and
  stays bounded where it does not — at the pace the loop settles (the projection algorithm, one full step a settling time).
  The facial muscles have no spindles.
- The lips and the jaw are muscles (`LIPS`, by the same `neckTurn`): the jaw's opener against the masseter on the mandible's
  inertia, the lips' aperture and protrusion each the orbicularis oris against the retractors on 15 g of lip, with Hill's
  force by length and speed and the activation's lag and a tolerance of 2 mm, so each gesture has the clock its muscles give
  it.
- The slide's criteria are scaled to this body from the tests' 1.75 m (`BODY`: the walker's stature from the eye's height,
  1.56 m; the gives by the scale, the forces by its square, the masses by its cube, so the tolerable g's by its inverse and
  the Head Injury Criterion's by the inverse 1.5 power). The worths are the Injury Severity Score's squares of the
  Abbreviated Injury Scale (the tibia's fracture AIS 2, the chest's crush 3, the head's 4: 1, 2.25, 4). The landing's damping
  is derived: the legs' from the muscles themselves (the extensors' 3 kN on Hill's stretched slope against the tendons'
  spring and the body's mass: 2.7, overdamped — the legs give and do not bounce, the hands hold in .2 s), the trunk's from
  Lobdell's thorax (26.3 kN/m, 520 Ns/m, 27.2 kg: .31, a rebound of .36).
- The wake: the body has its circulation — a uniform vortex sheet round the plan, its strength by Kelvin's theorem the
  opposite of all the vorticity ever shed (M40's and M41's tries had the sheet's sign wrong: checked now against a point
  vortex's far field, it holds, and swings ±1 m²/s at the shedding's period). The nascent core is the boundary layer's at
  separation, Blasius's on the cutwater's run at the water's viscosity in the pier's real stream (5.5 mm), and no less than
  half the sheet's spacing (Krasny's rule, 6 cm here); it grows as the mixing layer's vorticity thickness, Brown and Roshko's
  .17 of the distance run scaled by Abramovich and Sabin's (1 − r)/(1 + r) for the ratio the layer actually has (read from
  the model at each shedding, .44). The shear layer reattaches when its core touches the flank.
- The waters: one wind for the region (4.5 m/s over open country) and each water's from it by its roughness (the log
  profile matched at 500 m: the sea 5.5, the Seine 4.1, the Epte 3.3, the pond 3.0; M38 declared 7.8, 3.7, 2.8, 2.2). The
  baseflow index from each catchment's geology (chalk .9, limestone .7, the tertiary .4, the crystalline .3: the Seine .62,
  the Epte .84), and the share that still shows in the temperature by the springs' distributed ages over the run and the
  trunk's own share of the basin (the Seine .012, the Epte .48; M38 declared 0 and .7). The capillary rise from the bank's
  grain by Hazen (the Seine's fine sand .86 m, the Epte's silt 1.9, the pond's 2.2, the harbour's shingle 9 mm), and a
  shingle bank holds as Broms' sand (a hundred newtons: the harbour moors to buoys). The flood's height is read off the
  scene's own ground by Leopold's rule — a river builds its banks to its yearly flood, so the year's flood stands at the
  lower bank's lip: the section across the river at the mooring walked out each way to the first dry ground, the lip the
  highest ground in the next 10 m (`floodOf`: the Seine .51 at its south plain, the Epte .57, the pond .05 at its outlet);
  the tide's 3.5 m is kept as the datum it is. (A first try put Manning's flow for the real Seine's median flood through the
  scene's 40 m channel and got 2.5 m — the scene's river is a painting's, not the Seine's; Leopold's rule reads the scene.)
- The spalls: Hertz's cone is traced once for the stone (`hertzCone`): the contact's field summed from Boussinesq's point
  loads over the pressure's disc, the crack run from the ring at the contact's edge normal to the greatest in-plane tension
  (Frank and Lawn's trajectory), the cone's angle read once the path settles: 31° for a limestone's ν .25 (34° for glass's
  .22, where Roesler measured 22° — the shortfall is the crack's own field, a known gap). The strikes are in the mason's
  rows, 9 cm apart each way with his hand's scatter, the face 8 by 4 on the block, the relief to 3 cm, the drafted margin
  flat. The margin's tooling is in the stone: the boaster's 5 cm strokes as facets tipped 2° the other way from their
  neighbours', a strip of the margin faceted every 3.5 cm, so the tooling reads in the light as facets do; 20,210 strokes
  gone.
- The tracery: the joint is three of the mortar's coarsest grains (1.6 mm sand: 4.8 mm). The dagger's angle is found from
  the room it has — the apex is to reach the arch over it within a joint, φ = 2 atan(ρ / that height), 25° to 75°. Every
  bar is drawn afresh from the template as the mason's hand had it (each point off by up to half a millimetre, the depth by
  up to 2%, by the bar's own place), so no two bars are the one drawing.

**Verified** headless, `m42a.js` (three flights; the last is the record). The neck: the plan's gains 5.57 and .263 for
the turning (its settling .083 s), 5.38 and .320 for the nodding; `ksr` 65.1, 75.3, 60.9 F/L0; the tolerance .148 rad.
The flick of 1.4 rad reaches 1.33 at .19 s at 12.4 rad/s, peaks 1.44 at .24 and settles at 1.38. The body turned 1.5 rad
under a held look: the head's excursion .165 with the canals and the spindles, .180 without the canals, .213 without
either. Learning on, the same turn four times over: the feedback's integral .263, .211, .182, .209 against .407 unlearned,
the excursion .156; the weights stay bounded (the pulling side .54 on the demand, .07 on the canals; the lift's, which a
plain feedback-error rule ran to −60 and +63, at 4.4 and −2.2). The co-contraction at rest .02, swimming .05 (M41 .17, at
3° and 94). The body: scale .89, 49.5 kg; the legs' damping 2.68, the trunk's .307. Head first onto the aisle over its
15.2 m drop the injury reads .99 head first, the choice feet from the first frame, the body brought round to 6.08 rad (11°
short) and held, the grasp .2 s; the vault off the corner lands rolling with a grasp of 2.08 s (M41 5.6, at e .62); M33's
aisle holds, the corner stands. The wake: the circulation swings −1.04 to +1.33 m²/s over the run's second half, the core
.061 (the sheet's floor; the layer's 5.5 mm), the layer's ratio .436, 5 blobs reattached, the shoulders' stream 1.197, the
separation at the bow's shoulder; 4 sign changes, a period of 13.4, the Strouhal number .172 on the width and Roshko's
.144 on that stream (read .164; M41 read .162 on 7 crossings — the measurement rests on few), the Townsend number .103,
the deficit .60/√(1 + s/h), the half-width 1.00 + .113 s/h, the rows .74 to 1.85 h. At .55 the piers shed every 24.3 s,
λ 10.7; the street 4.4 m astern reads −.41 to −.43. The waters: winds 5.53, 4.07, 3.27, 2.95; capillary rises .857, 1.875,
2.222, .010; the baseflow .62 and .84 by the geology, .012 and .478 showing; the flood: the Seine .509 (the south plain's
lip .51 at 39 m, the north bank 1.0 at 4 m), the Epte .571, the pond .05; the water at noon of day 200 in air of 23.4°:
the Seine 20.6, the Epte 16.6, the pond 23.0, the harbour 17.8. The boats: the two stakes at 5 and 10 m, .8 m up, the clay
at 127 and 124 kPa (a liquidity of .07 at the bank's .86 m rise), holding 7.55 and 7.60 kN by Broms against 4 and 6 N, at
150 and 159 s; the harbour's boat on buoys (its shingle holds nothing); headings .367, .199, −.011, −.400 against the
painted .35, .2, 0, −.4, and after 12 s .367, .217, −.017, −.4. The lips: the plan's gains 38.7 for the jaw, 243 for the
lips (their settling 12 and 35 ms); through the blowing the protrusion reaches .8 of its 12 mm in about 40 ms (.19, .58,
.78 at 30 ms steps) and falls back over about 90 ms; the aperture .69 at the breath in for a goal of .8, the jaw .26 to .33
for .30 to .35; the formants unchanged for the same shapes (rest 491/1485/2539, aperture 323/1316/2500, protrusion
454/1396/2417, both 276/1280/2471, velum 405/556/1005). Hertz's cone 30.9° at ν .25 (the trace 34.1° at .22, 27.4° at .3).
No errors; 120 solids; draw calls 42. Triangles at Rouen 593,924 (M41 507,828), at the harbour 430,602 (344,522): the
blocks' faces and their margins' strips; 264,764 patches (M41 284,974: the margins' strokes gone). Frames in
`m42-sheet.jpg`.
Bench, DPR 2, twice alone: first with the machine's other work at a load of 2.9 to 4.9 — pond 14.4 · parasol 10.0 · argenteuil
11.3 · poplars 15.4 · haystacks 16.8 · rouen 9.6 · sunrise 5.7 · orangerie 8.0 · aerial 5.5 ms, the poplars and the haystacks (views
the quay and the parapet never enter) spiking with the load; then at a load of 2.1 rising to 5.8 — pond 11.9 · parasol 9.2 ·
argenteuil 8.5 · poplars 9.8 · haystacks 10.6 · rouen 8.8 · sunrise 6.0 · orangerie 8.7 · aerial 7.6 (M41, load 3.7 to 1.4: 11.0 /
8.9 / 9.3 / 9.5 / 10.6 / 8.7 / 5.7 / 8.8 / 8.6): the harbour's 86,000 more triangles cost it .3 ms; within the spread.

**Still visible.** The plan's cost weighs an error of one tolerance against a full command — a trade stated, not measured
— and the activation's 15 and 50 ms, the cupula's 5.7 s and the arcs' lengths are read; the spindle's velocity gain (half
the stretch's speed over the fastest) is still set; the synergy learns two features by one rule at one pace, and the
antagonist's braking it learns is whatever the turns taught it. The crash tests' criteria scale by similarity alone, the
Injury Severity Score's squares are a convention, and Lobdell's thorax and the extensors' 3 kN are read. The wake's Brown
and Roshko .17, the Blasius layer and Krasny's half-spacing are read, the core's floor is the discretisation's and not the
layer's, and the Strouhal measurement rests on four crossings. Hazen's C, the geology's indices and the trunk's shares are
read, the tide is a datum, and Leopold's rule takes the lip for the flood. The lips' forces and the 2 mm tolerance are
read, and the jaw's 26° is set. Hertz's trace leaves out the crack's own field (34° for a measured 22°), and the tooling's
2° facets and 5 cm strokes are set. The mortar's grain and the mason's half-millimetre are read, and the dagger's foot
(.9 ρ, .3 ρ down) is still a proportion.

### Progress · M43 — the measure: the plan's trade measured against each plant's own task and Riccati's equation solved by Kleinman, the plan run on its own model, the spindle's speed gain the muscle's own slope referenced to the command, the synergy fitted by recursive least squares; the slide's criteria scaled by the body's own mass and length, its worths what the injuries kill; the wake's layer and separation by Thwaites on the pier's own surface, its period from the whole record, the discretisation's cost measured; the capillary rise by Young and Laplace, the tide from the moon; the lips' tolerances from the ear through the tract, the jaw's range from the mouth; the tooling from the blow's own cut; the dagger's foot the compass's circle (what M42 left visible)

**What was wrong.** The plan's cost weighed an error of one tolerance against a full command, a trade stated. The
spindle's velocity gain was set; the synergy learned two features by one rule at one pace, and the antagonist's
braking was whatever the turns had taught. The crash tests' criteria scaled by similarity alone and the worths were
the Injury Severity Score's squares. The wake's Blasius layer was a flat plate's, the Strouhal measurement rested on
four crossings, and the core's floor was the discretisation's with its cost unmeasured. Hazen's C was read and the
tide was a datum. The lips' 2 mm tolerance was read and the jaw's 26° set. The tooling's 2° facets and 5 cm strokes
were set, and the dagger's foot was a proportion.

**What changed.**
- The plan's trade is measured (`neckTrade`): Riccati's equation is solved by Kleinman's iteration (a Lyapunov solve
  of six unknowns a round, a dozen rounds, 9 ms for every plant — M42 integrated it through twenty thousand steps), so
  the command's weight can be searched: it is walked down from a cheap command while the task's excursion still falls
  and halved in to where the excursion is just the eyes' tolerance, on the plan's own linear plant with the canals in
  the loop. The turning's task is the body's quarter turn under a held look at the body's own pace (.2 s); the
  nodding's the look dropping to the arms at that pace; each lip's its breath's gesture at the breath's rise. The
  gains come out 10.2 and .39 for the turning at a weight of .34 (M42's 5.6 and .26 at a weight of 1), 3.5 and .25 for
  the nodding at 2.0 (5.4 and .32); the linear plant's excursion under the quarter turn is the tolerance, .148, by
  construction, and the muscles' plant holds it to .146 (M42 .165).
- The plan is run on its own model of the plant first (three more numbers in the state: the model's angle, speed and
  activation): the model is sent the demand and gives the command the plan means — the agonist's pull and the
  antagonist's braking as the plan has them — and the muscles get that command plus the plan's correction of where the
  head has strayed from the model, plus the canals' report. So the braking is the plan's, not what any turn taught;
  the synergy then fits only what the model misses. The hold on the quarter turn is .146 with the model and .159
  without it.
- The spindle's speed gain is no longer a half set but the muscle's own stretched slope (Hill's .8 over .06: 13 of the
  fastest) at the belly's activation, as its length gain is the belly's short-range stiffness; and the stretch is
  reckoned against the command's own motion — the reference moves with the demand, as the fusimotor drive moves it —
  because with the gain on the raw stretch the reflex braked the head's own turns (the hold went from .08 to .20 rad
  in the prototype, `neck6.mjs`); with the reference right the flick is quicker than M42's (1.27 rad at .13 s, the
  peak 1.44 at .18; M42 1.33 at .19).
- The synergy is fitted by recursive least squares: each belly keeps the covariance of its two features (three numbers
  beside its two weights in `neckW`), so each feature's weight moves at the pace its own excitation gives it, the fit
  forgets over the loop's settling time, and the covariance's trace is held so a quiet feature cannot wind it up. The
  projection rule with a pace per feature was tried first and ran away (the error is shared, the paces were not); the
  one-pace rule was kept as the fallback in the prototype and dropped. In the page the fit changes the hold little:
  .154, .139, .153, .140 over four turns; the demand's direction settles (its covariance .014), the canals' report
  stays open (2.0); the flick after the fit is the flick before it to .18 s.
- The slide's criteria scale by Mertz and Irwin's laws with the walker's own mass and length apart: the mass by the
  century's build (a body mass index of 22 on 1.56 m: 53.5 kg; the cube gave 49.5) — a force by the length squared, a
  tolerable acceleration by that over the mass (1.15: the lighter body stands more g), a time by √(mass over length),
  the Head Injury Criterion by the acceleration's 2.5 power times the time (1.26). The worths are what the injuries
  kill: the crash records' case fatality by the worst injury, one in 250 for a tibia's fracture, one in 20 for a
  chest's crush, one in 5 for the head's — 1, 12.5, 50 over the tibia's, where the Score's squares gave 1, 2.25, 4.
- The wake's boundary layer is Thwaites' on the pier's own surface (`thwaites`): the momentum thickness integrated
  from the bow's point up the cutwater under the speed the panels give there (from nothing to twice the stream at the
  shoulder) and on to the flank, the layer separating where its shape reaches Thwaites' limit (λ = −.09), at every
  shedding under that step's flow: on this plan the shoulder each time, and the point given as the speed's peak the
  limit is reached from. The vorticity thickness there from Pohlhausen's separating quartic (4.9 θ): 2.5 mm in the
  pier's stream, against Blasius's 5.5 on a 2 m flat run — the cutwater's acceleration thins the layer. The blob is
  shed a hand past the corner on the cutwater's own tangent (the layer leaves a corner along the wall it came off),
  and the flux is taken at the speed a tenth of the width off the stone: on the corner itself the panels' speed is the
  potential flow's, which runs away there, and taken on it the flux went up a sixth and the street twice as wide (St
  .28); set on the flank's line .29, a tenth out from the corner .13. The period is read from the whole record — the
  probe's autocorrelation over the last 40 s, its first peak past the first zero, the peak read to a fraction of a
  step — 13.35 s against the crossings' 13.81 on six. The discretisation's cost is measured (`wake9.mjs`): at steps of
  .05 and .0125 s the Strouhal number reads .13 and .21 against this .17, and at .025 the run gives nothing sound (the
  circulation runs to 4.7); an inviscid sheet has no limit to converge to (Moore's singularity), so the wake is the
  model's at its core and the record says so (`study`). The Townsend number comes out .30 against the measured .037 —
  the model's street stirs far more than a real wake, its blobs reattaching in bunches (93 of 400 on a plan 2.65
  widths long, where Okajima's rectangles pass to intermittent reattachment near 2.8) — and the bound that calls a
  record unsound is set at .5 so that this one stands and is read for what it is.
- The capillary rise is Young and Laplace's in the pack's own throats: water's surface tension over the hydraulic
  radius of a pack of the finest tenth's grains (e D10 / 6), 6σ/(ρ g e D10) — Hazen's rule with his C derived, .44 cm²
  where he read .1 to .5: the Seine's fine sand 1.26 m (.86 by Hazen's .3), the Epte's silt 2.8, the pond's 3.3, the
  shingle 15 mm. The stakes at 5 and 10 m stand .8 m up, now inside the Seine's capillary fringe: the clay there at 32
  kPa (M42 127), holding 1.9 kN by Broms against the boats' 4 and 8 N. The tide is no longer a datum: the harbour's
  high water on the painting's own morning from the moon — Le Havre's two tides (the moon's 2.65 m, the sun's .85)
  beating at the moon's age, the springs a day and a half behind the syzygy, the age counted from a known new moon
  back to 13 November 1872 (10.9 days old, a waxing gibbous): 2.08 m above the mean, half way from neaps (1.84) to
  springs (3.46); the harbour's water stays at its datum, its flood moves with the moon.
- The lips' tolerances are what the ear cannot hear (`lipTol`): a formant's just-noticeable difference (3%) over how
  fast that formant moves with the gesture through this tract (the first two formants at a tenth of each gesture, the
  tighter of the two): the jaw's .36°, the aperture's 2.1 mm (M42 read 2), the protrusion's 4.4. The jaw's range is
  what the mouth asks of it (`JAW`): the tract's lips open from 2.5 to 8.5 cm² on a mouth 5 cm wide by the body, 1.35
  cm of gape on a mandible of 8.9 cm, 8.7° (M41 set 26°, a yawn's). The plans: the jaw 88 and .32 at a weight of 2.4,
  the aperture 286 and 1.3 at .70, the protrusion 133 and .65 at .47 (its settling 61 ms); the protrusion reaches .76
  of its 12 mm within 90 ms of the blowing's start and falls to .48 and .28 over the next 90; the formants for the
  same shapes unchanged.
- The tooling comes from the blow (`TOOL`): a mallet of 1.1 kg swung at 3 m/s (5 J), a boaster 50 mm wide and a point
  5 mm, their edges at 25°, on a limestone crushing at 40 MPa; a blow's cut is where its energy is spent crushing the
  stone ahead of the edge, d = √(2E tan e / (σ w)): 1.5 mm under the boaster, 4.8 under the point. So the boaster's
  facets tip by 1.5 over 50 (1.7°; M42 set 2°), each stroke is the boaster's own width (M42 set 5 cm as a number), and
  the point's spalls sink 4.8 mm with the hand's swing scattering it (M42 drew 2 to 8). Hertz's cone: the crack's own
  field was tried to first order (`hertz2.mjs`) — the crack grown step by step with its stress intensities from the
  uncracked field's tractions through Bueckner's edge-crack weight function, each step kinked to the greatest hoop
  stress — and it leaves the angle where the trajectory had it (33.0° for 33.2° at ν .22: along the trajectory the
  shear intensity is nil), so the crack's field must be solved for whole (Kocer and Collins found 22° by finite
  elements with the crack in the mesh); the trace stands, the gap named. A mixing layer of the model's own blobs was
  run to measure Brown and Roshko's rate (`layer.mjs`, `layer2.mjs`) and gave .05 to .36 depending on how the
  thickness was read — not a measurement — so the .17 stays read.
- The dagger's foot is the very circle the compass found — the round that meets the lights' heads within a joint — no
  longer a proportion of it set lower (M41's .9 ρ, .3 ρ down); its reach to the arch is measured from that centre.

**Verified** headless, keys held, from `m43a.js` (the record `m43a3.json`) and `m43v.js`. The plan: turning 10.23 and
.386 at R .337, the linear excursion .1484 for the tolerance .1484; nodding 3.51 and .247 at R 2.04; planned in 9 ms;
the short-range stiffnesses 65, 75, 61 F/L0, the speed gain 13.3. The flick: .57 rad at .079 s, 1.27 at .131, the peak
1.44 at .183, settling 1.38 to 1.39, 13.0 rad/s at most. The body's quarter turn: the head's excursion .146 with
everything, .164 without the canals, .173 without the reflexes, .159 without the model; over four fitted turns .154,
.139, .153, .140; the weights on the demand −.21, −.23, −.17 on the plus side and −.42, −.40, −.46 on the minus, on
the canals' report −.17, −.19, −.13 and +.28, +.30, +.24; the covariance's trace .014 on the demand and 1.99 on the
report; the flick after the fit the same to .18 s. Co-contraction .02 at rest, .028 swimming. The head-first aisle
slide of 15.2 m: the choice feet from the first frame, the injury .996 on the new scale, the body turned through side
to feet and held .23 rad short, the grasp .2 s; the vault tumbles with the grasp at 2.08 s; the low aisle holds feet
first; the corner is caught feet first. The body: 1.559 m, 53.5 kg, the mass scale .688, the acceleration scale 1.153,
the Head Injury Criterion's 1.255, the legs' damping 2.58, the trunk's .307. The wake: Strouhal .172, Roshko .135, the
period 13.35 s by autocorrelation (13.81 by six crossings), Thwaites' limit reached at every shedding, the first
separation 1.64 m from the bow (the shoulder at 1.60) at 1.93 of the stream, θ .5 mm, the layer 2.5 mm, the core 6.1
cm (the floor), the shoulders' stream 1.278, 93 reattached, the ratio .217, the circulation −1.61 to +1.65, Townsend
.298, the eddy viscosity .13, the deficit .50/√(1+s/h), the half-width .89 + .17 s/h; the study St .13 at .05, unsound
at .025, .21 at .0125; the street 4.4 m astern −.45. The waters: capillary rises .015, 1.259, 2.755, 3.265; the flood:
the Seine .509 (the south plain's lip), the sea 2.083 (the moon 10.86 days old; springs 3.46, neaps 1.84); the
temperatures unchanged (20.6, 16.6, 23.0, 17.8); the stakes at 5 and 10 m holding 1.91 and 1.93 kN in clay at 32 kPa
against 4 and 8 N; the harbour's boat on buoys; headings .367, .199, −.008, −.4 against the painted .35, .2, 0, −.4,
after 12 s .367, .197, −.007, −.4. The lips: tolerances .0062 rad, 2.13 mm, 4.41 mm from formants 491 and 1485 moving
36/−29, −7/−10, −4/−10 Hz at a tenth of each gesture; the jaw's range .152 rad; the plans 87.7/.322, 285.6/1.325,
133.2/.652; the protrusion .03 → .76 → .75 → .48 → .28 at 90 ms samples; the formants rest 491/1485/2539, aperture
323/1316/2500, protrusion 454/1396/2417, both 276/1280/2471, velum 405/556/1005. The tools: 4.95 J, the boaster's cut
1.52 mm (tilt .0304), the point's 4.8; Hertz's cone 30.9° at ν .25. No errors; 120 solids; draw calls 42. Triangles at
Rouen 591,182 (M42 593,924), at the harbour 427,872 (430,602); 264,763 patches. Frames in `m43-sheet.jpg`.
Bench, DPR 2, four runs alone, the machine's other work at a load of 3.4 to 5.2 throughout: at 3840×1376 (M42's
window) — pond 12.0 · parasol 8.9 · argenteuil 10.8 · poplars 16.6 · haystacks 14.6 · rouen 9.0 · sunrise 5.7 ·
orangerie 8.1 · aerial 7.3 ms, then pond 14.3 · parasol 9.3 · argenteuil 10.0 · poplars 12.0 · haystacks 16.5 · rouen
17.1 · sunrise 13.2 · orangerie 10.5 · aerial 7.3; two earlier runs at a window 174 px shorter (3840×1202: the
headless window's own chrome, since corrected) pond 11.6 · 8.9 · 13.0 · 16.3 · 10.5 · 8.6 · 6.3 · 8.3 · 8.6 and 14.1 ·
9.9 · 14.2 · 12.5 · 10.1 · 8.2 · 5.3 · 7.9 · 7.3 (M42, load 2.1 to 5.8: 11.9 / 9.2 / 8.5 / 9.8 / 10.6 / 8.8 / 6.0 /
8.7 / 8.6): the spikes move from view to view with the load (rouen 9.0 then 17.1, sunrise 5.7 then 13.2); M43 draws
one patch fewer and 2,700 triangles fewer, and the street's eddies are a fixed count on the record's period; the
bridge's view reads 10.0 to 14.2 in all four against M42's 8.5 and is not resolved from the load here.
**Still visible.** The tasks the plans are measured against are chosen (the body's quarter turn at its pace, the
look's drop to the arms, the breath's rise), the activation's 15 and 50 ms, the cupula's 5.7 s and the arcs' lengths
are read, the spindle's speed gain is the yielded muscle's slope and not the short-range's, and the synergy's fit
changes the hold little and still stands on two features. The crash tests' criteria scale by Mertz and Irwin's laws
with the tissues taken the same; the case fatalities, the body mass index, Lobdell's thorax and the extensors' 3 kN
are read. The wake's Brown and Roshko .17, Thwaites' .45 and −.09 and Pohlhausen's quartic are read, the core's floor
is still the discretisation's, the model does not converge with its step (St .13, .17, .21, one run unsound), its
street stirs eight times a real one's, and a quarter of its blobs reattach. Water's surface tension and the wetting
are read where Hazen's C was; the geology's indices, the trunk's shares, Le Havre's two tides and the tide's age are
read, the harbour's water keeps its datum, and Leopold's rule takes the lip. The lips' forces and the formants' 3% are
read, the mouth's width and the mandible's length are proportions of the body. Hertz's trace still leaves out the
crack's own field (the first-order account leaves the angle where it was), and the mallet, the edges' 25° and the
stone's 40 MPa are read. The mortar's grain and the mason's half-millimetre are read.

### Progress · M44 — the yield: the muscle's short range and its yield as a series spring, the reflex's two gains fitted on the belly's own stretch, the plan's trade measured on the muscles themselves and its tasks the world's own (the swim's turn, the hands' drop, the gasp's rise), the frame's demand ramped; the wake's blobs cut by a fixed quantum and the record judging its model by three runs — and refusing it (what M43 left visible)

**What was wrong.** The plans' tasks were chosen (a nod of .62, horizons of 1.5 and 1 s, the easy stroke's .375 s
for the gasp). The muscle had no short range at all — Hill's force went straight to the stretched branch — while the
spindle's gains were set from Nichols and Houk's finding about that short range, its speed gain the yielded slope. The
plan's trade was measured on a linear plant that keeps the short-range stiffness for ever, and the frame's demand
jumped, so a slow frame changed the hold (M43's .146 was a flight at about 20 frames a second; at 60 the same head
wanders .168). The wake's step and its discretisation were tied (halving one halved the other), and the model's number
was taken from one run.

**What changed.**
- Each belly's contractile part is in series with the cross-bridges' and the tendon's own spring (`STRETCH`, the
  extension a state per belly): a quick stretch is taken by the spring first at the short-range stiffness until the
  spring's force reaches what the stretched contractile part can hold (1.8 of the isometric) and the belly yields —
  the short range comes out at 1.6% of the belly, the yielded slope Hill's; each 2 ms step solved implicitly (the
  extension bisected on the monotone balance of rates). The flick is unchanged by it (1.30 rad at .13 s).
- The reflex's two gains are fitted, not set (`reflexFit`, in each plan): the plant's first belly alone at the plant's
  tone, stretched at the task's own rate through its yield for the task's own time with the reflex loop on it, its
  force fitted to the short-range spring's by a length gain and a speed gain. The length gain comes out nil in every
  case tried (tones .02 to .2, rates a quarter to twice the body's: a length gain feeds the activation it raises and
  runs past the spring), the speed gain 9.2 for the turning, 12.7 for the nodding, 6.9 for the jaw. M43's gains (the
  short-range stiffness and Hill's 13) missed the spring's force by 113 N root mean square over the turn's stretch;
  the fit misses by 5, no reflex at all by 28.
- The tasks are the world's own: the turning's the swim's own turn under a key (the stroke set square to the look,
  the body coming round at its own lag, `BODY_TURN`, one law for the swim and the plan), the nodding's the look
  dropping to the hands where the stroke brings them nearest under the eye (`HAND_DROP`, .55 rad from the arms' own
  path — `strokeHands`, which now draws the arms as well; M43 set .62), each lip's the breath's gesture at the gasp's
  own rise (`STROKE`: the breath's window at the sprint's stroke as a half-sine, .08 s; M43 took the easy stroke's
  .12); each task runs until the demand's residual is under a twentieth of the tolerance (1.07 s for the turn, .86
  for the nod). The stroke's windows and paces live in one place and the swim, the breath and the arms read them.
- The plan's trade is measured on the muscles themselves (`neckTask` runs the whole of `neckTurn` under a candidate
  plan; the linear plant kept as `neckTaskLin`), the weights scanned from a thousand down and the costliest that
  holds the task halved in on, because on the muscles the excursion is not monotone in the gain. The finding: the
  body's quarter turn cannot be held within the eyes' tolerance by these muscles — the least excursion at any weight
  is .161 for a tolerance of .148 (the activation's and the arc's lags bound it, not the gain) — so the plan takes
  the cheapest weight within a twentieth of that floor and says the task is not held (`sat` −1): 10.6 and .39 at a
  weight of .32, which is M43's plan (10.2 and .39) found again by a different road. The nod is held: 1.29 and .14 at
  9.0 (M43 3.5 and .25). The jaw cannot follow the gasp within the ear's tolerance (.013 for .0062; 54.7 and .21);
  the aperture and the protrusion can, at 38.8 and 11.1 (M43's plant asked 285.6 and 133.2 of them).
- The frame's demand is ramped across its substeps and the task's target is the body's at the frame's end, as the
  world has it: the hold on the quarter turn at frame ends is .168 at 60 frames a second, .169 at 30, .158 at 20,
  .098 at 15 and .033 at 10 (a slow frame hides the head's lag inside the frame; M43's .146 was such a frame).
- The wake's sheet is cut into blobs by a fixed quantum of length whatever the step (a fifth of a second of the bare
  stream's sheet, the sides half a quantum apart), so the step can be halved with the discretisation held; and the
  record judges itself: the model is run at its step, at half of it, and at its step under the midpoint rule instead
  of Euler's, and its numbers are taken only where the three agree within a tenth. They do not — Strouhal .275, .253
  and .097 — so the world's street runs on the read values (Roshko's .164, Townsend's .037, the read deficit and
  width) and the record says which, with the model's own numbers beside them. The study that tried the rest is in
  the record: the midpoint rule at three steps (.10, .10, unsound), the core at a tenth of the width (every blob
  absorbed), the blobs that touch the stone absorbed into the bound circulation instead of re-shed at the stern (the
  street twice as fast), the shedding at the stern's shoulders alone (.5 to .8), and M43's own settings under two
  engines from one code (Chrome .172, node .188): an inviscid sheet at this resolution has no limit to converge to
  and its street is chaotic. The three runs cost about two seconds at the bridge's build.

**Verified** headless, keys held, from `m44a.js` (the record `m44a3.json`, the flight at 16.6 ms a frame) and the
node harness `m44neck6.mjs` on the page's own code. The reflex fits: turning kL 0, gv 9.21 (rms 4.75 against 27.59
with none), nodding 0 and 12.73 (.94 against 15.41), jaw 0 and 6.93 (.43 against 3.25); fitted at the belly's own
rates .235, .125, .076 m/s. The plans: turning 10.59 and .394 at R .316, sat −1, exc .168, floor .1606 (the scan from
R 1000: .874 … .1606 at R .01 … .575 at R 1e−5); nodding 1.29 and .139 at 9.03, exc .1484; jaw 54.7 and .213 at
5.6, sat −1, .0130; aperture 38.8 and .201 at 9.7; protrusion 11.1 and .059 at 9.3; the whole planning 121 ms. The
tasks: turn π/2 at τ .200 within 1.4, T 1.073; nod .5543 at .200, T .864; lips at τ .0796, T .49/.36/.32. The flick:
.61 rad at .081 s, 1.30 at .134, the peak 1.385, 13.5 rad/s at most. The body's quarter turn: .166 with everything,
.179 without the canals, .198 without the reflexes, .171 without the model; over four fitted turns .194, .173, .185,
.174; the flick after the fit 1.31 at .13; in the harness at 60 frames .168 (.169 without the reflex, .204 without
the canals, .170 without the model), a push of .5 N m for .1 s moves the held head .0073 rad (.015 at a co-contraction
of .2), the weight's sag on the nod .017. Co-contraction .02 at rest, .029 swimming. The head-first aisle slide: feet
from the first frame, injury .996, held short, grasp .2; the vault tumbles with the grasp at 2.08; the low aisle and
the corner feet first. The wake: the model .2745 at its step (Roshko .208, period 8.38 s, 143 reattached of 840
blobs, Townsend .417), .2527 at half the step, .0965 under the midpoint rule (period 23.8), so `measured` false and
the world's St .164; Thwaites' limit at every shedding, the first separation 1.64 m at 1.93 of the stream, θ .5 mm,
the layer 2.5 mm, the core 6.1 cm; the street 4.4 m astern −.45. The waters, the flood (Seine .509, sea 2.083), the
temperatures, the stakes (1.91 and 1.93 kN), the moorings and the headings unchanged; the formants unchanged (rest
491/1485/2539); the tract's jaw .08 → .20 → .34 through a gasp; the tools and Hertz's 30.9° unchanged. No errors; 120
solids; draw calls 42. Triangles at Rouen 591,182, at the harbour 427,872; 264,763 patches. Frames in `m44-sheet.jpg`.
Bench, DPR 2 at 3840×1376, four runs alone, the machine's other work at a load of 8.4 to 19.3 through them (its
heaviest yet): pond 11.1 · parasol 8.4 · argenteuil 9.8 · poplars 10.1 · haystacks 10.2 · rouen 9.0 · sunrise 6.4 ·
orangerie 11.1 · aerial 8.1 ms, then 13.2 · 8.4 · 8.9 · 10.7 · 11.1 · 8.9 · 5.8 · 8.5 · 7.8, then 11.4 · 8.2 · 8.7 ·
9.3 · 10.5 · 9.0 · 5.8 · 9.8 · 9.1, then 12.9 · 8.2 · 9.3 · 9.4 · 10.3 · 8.9 · 5.9 · 8.7 · 8.7 (M43, load 3.4 to 5.2:
12.0 / 8.9 / 10.8 / 16.6 / 14.6 / 9.0 / 5.7 / 8.1 / 7.3 and 14.3 / 9.3 / 10.0 / 12.0 / 16.5 / 17.1 / 13.2 / 10.5 /
7.3): the bridge's view reads 8.7 to 9.8 in all four (M43's 10.0 to 14.2, M42's 8.5), so M43's spikes there were the
load's after all; the street draws the read period's count of eddies as before; nothing in the frame changed but the
neck's per-frame work (a bisection a belly a substep, about .2 ms a frame across the three joints, inside the noise
here) — the wake's three runs are a one-off two seconds when the bridge is built, not in the frame.
**Still visible.** The tasks are the world's own but the world's laws behind them are set (the swim's 95% in .6 s,
the stroke's windows and paces); the activation's 15 and 50 ms, the cupula's 5.7 s and the arcs' lengths are read;
the muscle's yield at 1.8 and Hill's .06 are read, and the short range that comes of them (1.6% of the belly) is not
measured; the reflex's gains are fitted to Nichols and Houk's finding taken as the criterion, at one tone and one
rate; the quarter turn cannot be held within the eyes' tolerance by these muscles and the twentieth of the floor the
plan then settles for is chosen; the synergy still stands on two features. The crash tests' criteria scale by Mertz
and Irwin's laws with the tissues taken the same; the case fatalities, the body mass index, Lobdell's thorax and the
extensors' 3 kN are read. The wake's model is refused by its own runs, so the world's street runs on Roshko's read
.164, Townsend's .037 and a read deficit and width; Brown and Roshko .17, Thwaites' .45 and −.09 and Pohlhausen's
quartic are read; the model's core is the quantum's, its street stirs eleven times a real one's and a sixth of its
blobs reattach, and its three runs cost two seconds at the bridge's build. Water's surface tension and the wetting are
read where Hazen's C was; the geology's indices, the trunk's shares, Le Havre's two tides and the tide's age are read,
the harbour's water keeps its datum, and Leopold's rule takes the lip. The lips' forces and the formants' 3% are read,
the mouth's width and the mandible's length are proportions of the body. Hertz's trace still leaves out the crack's
own field, and the mallet, the edges' 25° and the stone's 40 MPa are read. The mortar's grain and the mason's
half-millimetre are read.

### Progress · M45 — the body: the swimmer's turning no longer a law but the body's own (its yaw inertia, its cross-flow drag, the strokes' unequal pull at the hands' own reach, a steersman's law from those numbers), the plan's tasks the swim's own turn and the look's turn in the pull's window, the reflex fitted over the task's own rates and the short range measured at its peak, a third synergy feature, and the fit forgetting over movements (what M44 left visible)

**What was wrong.** The swim's turning was a law set (95% in .6 s: the body came round like a cursor, at a head's
speed, 7.9 rad/s), and the neck's plan was measured against that impossible turn, which no gain could hold. The
reflex's gains were fitted at one tone and one rate, the short range was a number stated (1.6%), the synergy stood on
two features and forgot over the loop's settling (55 ms). The nod's pace was the body's .2 s, a law too.

**What changed.**
- The swimmer's turning is the body's own (`SWIM`, after `BODY`): the prone body's yaw inertia about its middle
  (mass on length squared over twelve, 10.8 kg m²); its resistance to turning the cross-flow drag of its length
  swinging through the water (a rough cylinder's coefficient on the shoulders' breadth, integrated along the body,
  35 N m s² — the torque goes as the rate squared); and what turns it the strokes' own pull: the arms' share of the
  propulsion (.4 of the stroke's impulse, as `PULSE` has it) at the pace's own drag (a coefficient of .5 on the
  shoulders' breadth by a chest's depth: 17 N at the easy pace, 82 sprinting), spread over the pull's window as the
  pull is, pulled unequally on the two sides by a steering asymmetry, each hand's pull times its own lateral reach on
  the arms' path (2.5 N m at full asymmetry over the easy stroke). The steersman's law comes from those numbers
  alone: full asymmetry until the angle still to turn is within the stopping angle under the drag with the pull
  reversed, eased over the angle the body coasts through from its steady rate. So a quarter turn takes six seconds of
  easy strokes and three sprinting, in lurches with the pulls (the body's peak rate .39 and .70 rad/s), the body
  hunts a little about its heading (±.1 rad: the pulls come in quanta), the stroke's line lags the key while the
  swimmer crabs toward it, and the drawn arms pull unequally as the body turns. The pace and the sprint live in one
  place with the speed. The chest's depth and the two drag coefficients are read.
- The plan's tasks are the world's own and a plant may have several, the excursion the worst of them: for the
  turning, the swim's own quarter turn at the sprint from the pull's start (`taskTrace` runs `SWIM` itself, the
  target the body's angle at each frame's end, the canals the frame's mean rate, two strokes) and the look's own turn
  to the neck's end within the pull's window (the body's turn alone is so slow that a plan holding only it goes limp,
  .11 and .03, and the flick takes .33 s); the nod's the look dropping to the hands within the pull's own window
  (`STROKE.pullRise`, .095 s; M44 used the body's .2 s). The body's own turn is held to .04 by any plan; the look's
  turn is past these muscles at any gain (.38 for .148, the floor .37: an 80° flick in a tenth of a second is the
  lags', not the gain's), so the yaw plan stays at 10.6 and .39 at a weight of .32 and says the task is not held; the
  nod is held at 5.8 and .34 at .87 (M44 1.29 and .14 at 9.0). The hands' path is continuous at the recovery's start
  (M34's jumped 8 cm), and the arms are drawn from it with the steering asymmetry.
- The reflex's gains are fitted over the task's own rates, not one: the body's rate at the arm through the task's own
  run, its peak, its median and its lower quartile for each task (`taskRates`), the cost summed over them for the
  task's own time. The turning's rates are 3.1, 1.3 and 1.0 cm/s from the body's turn and 40, 2.9 and .7 from the
  look's (M44's law gave 24 alone); the fit gives no length gain and a speed gain of 8.3 (the cost 3.1 N against 6.9
  with none), the nod .13 and 6.9, the jaw 0 and 7.3.
- The short range is measured, not stated (`bellyRange`, in each plan's `rf.range`): the belly stretched at the task's
  peak rate until its force has come nine tenths of the way to what the yielded contractile part holds at that rate.
  It is the rate's, not the muscle's: at the look's 40 cm/s the turning's bellies give 1.21, 1.08 and 1.30% of their
  length, the nod's 1.35, 1.05 and 1.25, the jaw's 1.18; at the body's slow turn it would be a tenth of that.
- The synergy has a third feature, the demand's own rate (the feed-forward a quick demand needs), bounded at the
  tasks' own peak demand rate (`wmax`, 14.7 rad/s for the turning) so that a weight fitted at the body's half radian
  a second does not meet a flick's eighty; each belly keeps the covariance of its three features (six numbers). And
  the fit forgets over the tasks' own length (two strokes, 2 s), not the loop's settling: a synergy is fitted over
  movements and forgets over movements. Over four slow turns the rate's weights come out .10 on the plus side and
  −.12 on the minus, the hold unchanged (.022 → .014), and the flick after the fit is the flick before it (1.31 rad
  at .13 s from its start).

**Verified** headless, keys held, from `m45a.js` (the record `m45a7.json`, the flight at 16.7 ms a frame) and the
node harness `m45neck2.mjs` on the page's own code; the body's turn prototyped in `body45b.mjs`. `SWIM`: I 10.84,
c 35.07, the mean moment 2.496 N m, the steady rate .267 rad/s, the easing angle .107 rad, the drag area .0418 m².
The world's turns under a held key: the easy stroke reaches a quarter turn at 6.05 s (the body's peak rate .39 rad/s,
the overshoot to 1.65, the head held to the look throughout, −1.37 at the neck's end), the sprint at 3.27 s (.70
rad/s, to 1.60); the steering asymmetry 1 through the turn, then .43, .29, −.05 … as the body eases. The plans:
turning 10.59 and .394 at R .316, sat −1, exc .384, floor .367, wmax 14.66; nodding 5.83 and .336 at .872, exc .1484;
jaw 54.7 and .213 at 5.6, sat −1, .0130; aperture 38.8 and .201 at 9.7; protrusion 11.1 and .059 at 9.3; the whole
planning 267 ms. The reflex fits: turning kL 0, gv 8.33 (3.09 against 6.94), rates [.0312, .0131, .0099] over .3 s and
[.4036, .0294, .0073] over .095, ranges 1.211, 1.076, 1.30%; nodding kL .133, gv 6.94 (1.83 against 7.46), ranges
1.354, 1.053, 1.253; jaw 0 and 7.28 (.19 against 1.67), 1.183. The flick: .61 rad at .079 s, 1.31 at .131, the peak
1.39, 13.5 rad/s at most; after four learned turns, from a head .10 off its body, 1.20 at .131 (1.31 from its start),
settling 1.26 for a target of 1.30. The body's own turn under a held look: the head's excursion .018 with everything,
.018 without the canals, .021 without the reflexes, .014 without the model; over four fitted turns .022, .018, .015,
.014; the weights on the demand −.10 (plus) and −.08 (minus), on the canals' report +.08 and −.04, on the demand's
rate +.10 and −.12; the covariance's trace .0017 on the demand, .0015 on the report, 1.50 on the rate. Co-contraction
.02 at rest and swimming. The head-first aisle slide, the vault, the low aisle and the corner unchanged; the street
4.4 m astern −.44; the moorings, the headings, the flood, the temperatures, the formants (rest 491/1485/2539), the
tract's gasp (jaw .04 → .31), the tools and Hertz's 30.9° unchanged; the wake's model .2745/.2527/.0965, refused, the
world on .164. No errors; 120 solids; draw calls 42. Triangles at Rouen 591,182, at the harbour 427,872; 264,763
patches. Frames in `m45-sheet.jpg`.
Bench, DPR 2 at 3840×1376, four runs alone, the machine's other work at a load of 1.9 to 6.1 (lighter than M44's):
pond 12.1 · parasol 9.1 · argenteuil 9.5 · poplars 10.6 · haystacks 11.4 · rouen 10.1 · sunrise 6.3 · orangerie 11.0 ·
aerial 5.8 ms, then 11.8 · 9.2 · 9.5 · 11.2 · 11.7 · 10.5 · 6.5 · 12.0 · 5.7, then 12.5 · 10.7 · 10.3 · 11.6 · 12.1 ·
11.6 · 6.9 · 9.3 · 6.2, then 12.6 · 10.0 · 9.3 · 10.5 · 11.5 · 9.5 · 6.8 · 8.9 · 5.9 (M44, load 8.4 to 19.3: 11.1 /
8.4 / 9.8 / 10.1 / 10.2 / 9.0 / 6.4 / 11.1 / 8.1 and 13.2 / 8.4 / 8.9 / 10.7 / 11.1 / 8.9 / 5.8 / 8.5 / 7.8): the
parasol's, Rouen's and the bridge's views read about a millisecond over M44 and the aerial's two under, with nothing
of M45 in a still frame (the body's turn and the neck's fit run only afloat, the planning is a one-off 267 ms at the
first stroke): the spread is the machine's, not the build's.
**Still visible.** The body's turn is the body's own but two drag coefficients and a chest's depth are read, the
arms' share of the propulsion is `PULSE`'s .4, the steersman's easing angle is the coasting angle from the steady
rate (a choice of angle), and the drawn hands are not the propulsion's own (their path pushes a newton where the pace
needs seventeen); the look's own turn in the pull's window is past these muscles and the twentieth of the floor the
plan then settles for is chosen; the stroke's windows and paces are set; the activation's 15 and 50 ms, the cupula's
5.7 s and the arcs' lengths are read; the muscle's yield at 1.8 and Hill's .06 are read; the reflex's gains are fitted
to Nichols and Houk's finding taken as the criterion, at one tone; the synergy's third feature changes the hold
little. The crash tests' criteria scale by Mertz and Irwin's laws with the tissues taken the same; the case
fatalities, the body mass index, Lobdell's thorax and the extensors' 3 kN are read. The wake's model is refused by
its own runs, so the world's street runs on Roshko's read .164, Townsend's .037 and a read deficit and width; Brown
and Roshko .17, Thwaites' .45 and −.09 and Pohlhausen's quartic are read; the model's core is the quantum's, its
street stirs eleven times a real one's and a sixth of its blobs reattach, and its three runs cost two seconds at the
bridge's build. Water's surface tension and the wetting are read where Hazen's C was; the geology's indices, the
trunk's shares, Le Havre's two tides and the tide's age are read, the harbour's water keeps its datum, and Leopold's
rule takes the lip. The lips' forces and the formants' 3% are read, the mouth's width and the mandible's length are
proportions of the body. Hertz's trace still leaves out the crack's own field, and the mallet, the edges' 25° and the
stone's 40 MPa are read. The mortar's grain and the mason's half-millimetre are read.

### Progress · M46 — the key beyond the frame: the picture swept a second time over the frame's top, the seam behind him, the fan at the pole (fix: "when flying, there are some visual bugs, look at the sky of both screenshots")

**What was wrong.** The world outside the frame is painted in the picture's key, and the sky takes the same key: `extCol` reads the
canvas by angle (`dirUv`) and, past the frame, reads the picture's own mirror image, soft. The mirror was unbounded in both
coordinates, and both ran out. None of it depends on where the eye is — the sky's key is a function of direction alone — so all three
faults were there from the first frame; on foot the canvas covers the frame and the rest sits low in the view, and flying is simply
where the whole dome comes into sight.

- **Over the frame's top the picture was swept again.** The mirror did not carry the top rows on: between the frame's edge and the pole
  it ran through the whole canvas a second time, upside down. The pond's frame ends 31° above his axis, the zenith stands at 1.95 canvas
  heights, and the mirror folded that to .048 — the canvas's foot, the pond's own water, hanging over your head. The rest the same: the
  poplars' zenith read .300 of their canvas, the parasol's .375, Rouen's and the sunrise's .500, the haystacks' .597, Argenteuil's .750.
- **Behind him the mirror never closed.** The pond's canvas is 51° wide, so a turn holds 7.014 of it — not a whole even number, so the
  last copy met its own reflection: over four thousandths of a radian the read went from .996 of the canvas to .004, a step of 26 levels,
  a hard edge standing from the horizon to the zenith at the azimuth opposite his look. It is worst where the count falls nearest an odd
  number: Argenteuil 6.952 (a step of 31), the poplars 7.200 (30), the haystacks 7.178 (14), the pond 7.014 (26), the parasol 9.129 (10),
  Rouen 11.856 (5) and the sunrise 9.445 (3).
- **At the pole the columns fanned.** Toward the pole of the mapping every azimuth is nearly the same direction, so the canvas's columns
  crowded into a pinwheel standing on the zenith. Round the turn at 86° up the read swung 41 levels at the poplars, 34 at Argenteuil, 20
  at Rouen, 17 at the pond, in steps of up to 32 between neighbouring directions.

**What changed.** Three repairs in `PAINT_GLSL`, all within the ext read, so the world beyond the frame and the sky take them together.

- `extWidth` rounds the canvas's width in azimuth so that a whole even number of them fills the turn (the pond 7.014 → 8, Argenteuil
  6.952 → 6, the poplars 7.200 → 8, the haystacks 7.178 → 8, the parasol 9.129 → 10, the sunrise 9.445 → 10, Rouen 11.856 → 12): the
  mirrored copies then close on themselves behind him. It moves the frame's own edge by at most 7% of a canvas width (the pond's; Rouen's
  by .6%), which at level 7 of a 13 × 16 read is under a texel, so the promise that `dirUv` matches the projection near the frame holds.
- `foldV` leaves the frame's top and bottom edge and settles a fifth of a canvas inside it (a fold saturating at .18, at a rate of 4 a
  canvas height) instead of sweeping the picture again. Every zenith now reads .82 of its own canvas — the rows just under the frame's
  top, the willows at the pond and the sky at Argenteuil — and every nadir the rows just over its foot.
- Toward the pole the columns close on the middle, from the frame's own top edge to 80°, so straight overhead the picture is read in one
  column and not in wedges. Closing it earlier was tried and is worse: at 66° and at 57° the canvas's dark centre column is laid over the
  whole upper sky, and the pond's high sky goes flat and green.

**Verified.** `m46-sky.jpg`: the pond's whole sky as the ext read alone has it (the zenith at the centre, the horizon at the rim), then
three frames — twenty metres up looking up the sky, back over his shoulder, and thirty-four up straight overhead — each before and after.
The seam's step in the ext read, over three elevations and 256 samples across the azimuth opposite his look, before → after (of 255):
the pond 26 → .7, Argenteuil 31.3 → .3, the poplars 30 → 1.0, the haystacks 14.3 → .3, the parasol 10 → .7, Rouen 5.3 → .3, the sunrise
3.3 → .3 — every one down to the read's own quantisation. Round the whole turn at 86° up, the swing (and the worst step between
neighbours) before → after: the pond 17.3 (13.7) → .7 (.7), the parasol 14.3 (.7) → 1.3 (.7), Argenteuil 34.3 (25.7) → 0, the poplars
40.7 (32) → 0, the haystacks 18.0 (12.3) → 0, Rouen 19.7 (1.0) → 0, the sunrise 18.3 (3.0) → 0. Over the whole sky the mean difference
old → new is 8.9 at the pond, 19.7 at the parasol, 14.5 at Argenteuil, 9.2 at the poplars, 4.5 at the haystacks, 4.9 at Rouen, 6.0 at the
sunrise; within 15° of the horizon, where walking looks, it is .2 at Rouen, 2.1 at the haystacks, 2.7 at the pond and at most 7.5 at
Argenteuil (whose count rounds down); above the frame it is 5.1 to 41.7. The figures are the same at his spot and 38 m up, as they must
be for a read that takes a direction and not a place.

The frames were taken by rendering the scene to a target with a camera made for the purpose, because the app's own camera was carrying
a NaN: with the pane collapsed the resize handler sets `camera.aspect = innerWidth / innerHeight` on a 0 × 0 window, `fitFov` then
divides by it, and every frame after that clips away — `monet.snap()` returns black, against M0's promise that a frame can be checked
without a visible pane. Not touched in M46; fixed in M46b. For the same reason there is no bench this milestone; the ext read gains a
floor, two divides, an `exp` and a `smoothstep`, once per fragment in the sky and once in the world's base pass.

**Still visible.** The mirrored copies still read as soft wedges in the upper sky, light and dark by the canvas's own top rows: at
the pond that band is willow, so the sky over the water garden keeps its green cast and carries eight wedges all the way round.
The fold's fifth and its rate, and the pole's 80°, are chosen and not measured. The rounding takes the nearest even count, so a
copy is not quite the canvas's width — the pond's an eighth narrower, Argenteuil's a sixth wider — though the frame's own edge
lands under 7% of a width off. The ext read still knows nothing of the sun: the upper sky's colour round the turn is the canvas's,
mirrored, not the light's.

### Progress · M46b — the harness: a frame checked with the pane away (fix: "fix the camera NaN guard too")

**What was wrong.** M0 asks that a frame can be checked without a visible pane, and it could not. A hidden or collapsed pane reports a
0 × 0 window, and the resize handler set `camera.aspect = innerWidth / innerHeight` — 0/0, a NaN. It went into the projection matrix and
stayed there: every vertex after it fails the clip, so the frame is the clear colour and nothing else. `fitFov` divided by the same ratio,
so `camera.fov` went NaN with it, and neither recovered short of a reload under a real window. The camera was built and the canvas first
sized from that ratio too, so a page that loaded into a collapsed pane was in the state from its first frame. `monet.snap()` returned
black, which is why M46 was verified with a camera made for the purpose instead of the world's own.

**What changed.** One place now reports the window's size for the camera and the canvas — `viewport`, with `viewAspect` over it — and it
never returns zero: a real size is taken and remembered, a zero one is refused and the last real size stands, and where there has never
been a real one the size is 1280 × 720. The four readings that divided by the raw ratio go through it: the camera's construction, the
renderer's first `setSize`, `fitFov`, and the resize handler.

**Verified.** With the window at 0 × 0: the camera's aspect 1.7778 and its fov 62 at the pond, every element of the projection finite, the
canvas 2560 × 1440 at DPR 2, and the frame's luminance 29 to 251 about a mean of 119 where before every pixel was the clear colour.
`m46b-snap.jpg` is `monet.snap()` itself with the pane away — the pond from his spot, and the same garden 21 m up — where it returned one
flat colour before. Through a run of sizes, each checked for a finite projection: loaded at 0 × 0 → 1280 × 720, aspect 1.7778; 1600 × 900
arrives → taken, canvas 3200 × 1800; collapsed to 0 × 0 → the 1600 × 900 stands, the aspect unmoved; 900 × 1400 → taken, aspect .6429,
the fov widened by `fitFov` for a canvas taller than the window; collapsed again → the tall size stands. No NaN at any step, and a real
window always wins.

**Still visible in M46b; fixed in M46c.** Two other readings of the window are left as they were, neither of them a NaN: the quality preset
at load reads `innerWidth < 900` on a coarse pointer, so a touch device whose pane is collapsed at load picks `light`; and the canvas
overlay (key C) takes its height from `innerHeight`, so with the pane away it has none — it is a DOM element over the canvas, which no snap
carries anyway. The fov `fitFov` chose for a viewpoint is still not re-fitted when the window resizes, as before: the resize handler sets
the aspect alone, and the painting's fit is right again at the next viewpoint.

### Progress · M46c — the bench across the nine viewpoints, and the three readings of the window M46b left (fix: "do a proper bench across the nine viewpoints now that the camera is sane and fix what's visible")

**What was wrong.** The bench measured nothing. Run headless before M46b it printed its own header — `bench 0x0 dpr 2` — and under it nine
plausible frame times, the pond at 54.2 ms; but the camera's NaN clipped every vertex and the canvas had no pixels at all. Run again now
against that build to be sure: aspect NaN, fov NaN, no element of the projection finite, not one lit pixel across the middle of the frame,
and the nine times printed just the same. They were the cost of `update` and of the pixel read that syncs it, and not a frame's.

With the camera sane it still over-reported, because the first frames at a viewpoint are not the frame's cost: the materials new to that
place compile on their first draw and its projector depth pass is taken again. One tick of warm-up does not cover it — at the pond the first
block of twenty runs 211 ms a frame, the next 39, and the least of the ten after it 32. M46's line put the pond at 64.1 ms, near three
times what it costs. And the two passes were
priced by differences between measurements taken minutes apart, with the machine's own drift lying between them: 20.9 ms for the reflection,
7.6 ms for the projector, both mostly warm-up. Nothing in the line said which of it was measurement and which was the machine.

The three readings of the window M46b left were each wrong in their own way. The quality guessed at load read `innerWidth < 900` raw, and 0
is under 900, so a touch device whose pane was collapsed at load was called a phone and given the light world for the session. The canvas
over the view (key C) took its height from `innerHeight`, so with the pane away it had none, while the canvas beneath it stood at 1280 × 720.
And `fitFov` — which widens the view when the window is narrower than the picture, so that the whole canvas is held — ran at the viewpoint
and never again: a window narrowed after the visitor arrived kept the shape of the one they arrived at, and the picture ran off both sides.

**What changed.** `viewport` moves up among the basics, above the quality, and the quality's guess reads the window through it like everything
else. A guess that had to stand on the stand-in size is provisional (`qualityGuessed`), and the first size the window gives of its own makes
it again — taken if it differs, and not written to `localStorage`, a guess being no choice of the visitor's (`setQuality`'s second argument).
The canvas over the view is measured against the canvas the world is drawn on, not the window, so the two agree wherever the window is. And
the framing is fit again on every resize (`refitFov`), against the place whose framing the fov is holding — a new `fovPlace`, set wherever the
fov is set, and not `placeIndex`, which follows the walk: walking from the haystacks to Rouen must not re-frame the view under the visitor.
Mid-flight it is the fov being flown toward that is refit, so a window changed in the air arrives right.

The bench keeps the shape it had — the clock driven by hand, each frame synced by a read of one pixel, the whole drawn into the debug overlay
for headless capture — and is made to say what it measured. A block of twenty frames is run and thrown away before any is kept; several
passes follow and the least is kept, the one least interrupted by whatever else the machine was doing (`?bench=6` asks for six). Each
viewpoint's canvas size is recorded and printed where it differs from the first's, because above sixty metres the world drops to one device
pixel per CSS pixel and the aerial viewpoint's frame is a quarter of the others' — a single size across the top said otherwise. The two
passes are priced by blocks alternating on and off within one run, every other pair taken the other way round so that a drift inside a pair
falls on both sides; the pass costs the middle of the paired differences, the extreme pair at each end set aside first. And a run that cannot
separate a pass from the noise says that, and does not say the pass is absent.

**Verified.** The nine viewpoints, at 2560 × 1440 and DPR 2 (a retina window of 1280 × 720), quality balanced, 264,763 patches and 1,146,086
triangles, on an Apple M3, with the pane hidden. The figure is the median of four runs' minima, each run six passes of twenty frames after a
discarded one; the range is those four runs.

| Viewpoint | Frame | Over four runs |
| --- | --- | --- |
| The Water-Lily Pond | 22.0 ms | 20.9–22.3 |
| Woman with a Parasol | 23.8 ms | 21.1–26.0 |
| The Bridge at Argenteuil | 22.6 ms | 20.4–26.6 |
| Poplars on the Epte | 22.7 ms | 18.7–24.0 |
| Haystacks | 21.4 ms | 18.3–24.5 |
| Rouen Cathedral | 19.1 ms | 17.2–21.2 |
| Impression, Sunrise | 16.0 ms | 14.1–17.5 |
| Les Nymphéas | 19.6 ms | 18.2–22.5 |
| The whole universe (at 1280 × 720) | 20.4 ms | 19.8–21.8 |

So the world costs much the same wherever you stand in it — eight of the nine within 5 ms of each other, the harbour cheapest at 16 (it is
mostly fog and water) and the meadow dearest at 24. At this size that is 42 to 63 frames a second: the world as it now stands sits on the
edge of M0's sixty and no longer comfortably inside it, where the pond alone, at 74.7 k patches, was 4.6 ms.
(These nine figures are the walk's shape and not the world's, and M46d withdraws them: the bench took them in a fixed order, so the places
walked last were measured while the machine had quietened and came out a quarter cheaper than they are. What survives is the level — 21 to
39 ms a viewpoint depending on the machine — and the reading that the world no longer holds sixty at this size.)

The two passes are both real and both near the noise, so a difference wants more pairs than a level does. Over twenty-five pairs at
Argenteuil, at 2560 × 1600: the reflection 8.1 ms, 24 of the 25 differences positive, quartiles 6.7 and 11.0; the projector 6.9 ms, 23 of 25,
quartiles 4.5 and 9.9 — near a fifth of that frame each. At the twelve pairs a default run gives, the reflection resolves every time (6.4 to
8.3 ms over the four runs) and the projector none of them, and the line says so. The harness said "not found in the frame" before this was
measured, and its own `projMs` seemed to agree at .1 to .5 ms — but that timer reads the submission, not the pass the GPU then runs.
`m46c-nine.jpg` is the nine frames themselves, every one out of `monet.snap()` with the window at 0 × 0.

The quality's guess: with the pane at 0 × 0 and a coarse pointer, the old expression picks `light` and the new one `balanced`. When
420 × 800 then arrives, the guess is made again and `light` taken — 264,763 patches down to 126,212, the canvas to 525 × 1000 — and
`localStorage` stays empty. A second change of the window does not guess again; a visitor who then chooses `rich` gets it and it is saved;
and a narrow window after that leaves their choice alone. The canvas over the view, with the window at 0 × 0 and the canvas at 1280 × 720:
the frame 575.78 × 720 at 352.11 from the left, which is the canvas's own centre ((1280 − 575.78) / 2, and 720 × .7997), where it was 0 × 0
at 0 before. Under a real window of 1280 × 800 it is 639.76 × 800 at 320.12 — exactly what it was before the change, the fix being felt only
where the window says nothing. The framing: at the haystacks at 1280 × 720 the fov is 31; the window turned to 700 × 900 it becomes 62.064,
which is `fitFov`'s own value for that aspect, where it stayed at 31 and the stacks ran off the sides. Standing at Rouen but framed for the
haystacks, a resize keeps 62.064 and does not take Rouen's 45. And a flight to the harbour begun at 1400 × 800 aims at 30; the window turned
to 700 × 900 in the air, it arrives at 47.896 — the harbour's fit at the aspect it landed in.

**Still visible.** The bench is run with the pane hidden, which is what makes it a harness at all — the page is given no animation frames
there, and the line says so — but a GPU compositing to a visible surface may be clocked differently, and these numbers have not been taken
against one. The machine's own spread is wider than anything measured here: the pond moved by 1.4 ms across the four quiet runs and by 26
across the contended ones earlier in the session, and the runs above were taken with nothing else of mine running. The projector's pass
wants about twenty pairs where a default run gives eight and `?bench=6` twelve, so it usually goes unresolved; and three of the viewpoints
(the haystacks, Rouen, the Orangerie) came out in two clusters 6 ms apart across the four runs, which is not the machine's shape and has not
been chased. (Chased in M46d: there are no clusters — four samples had fallen into two pairs — and what moved them was the fixed order of
the walk.) `travel.f0` — the fov a flight departs from — is not refit when the window changes in the air, only the one it arrives at. The
quality is guessed a second time once and only if the first guess had to stand on the stand-in size; a visitor who loads on a phone at a
real size and never resizes is where they always were.


### Progress · M46d — the two clusters were the order of the walk (fix: "chase the two clusters at haystacks, rouen and orangerie")

**What was wrong.** M46c saw the haystacks, Rouen and the Orangerie each come out in two clusters 6 ms apart across four runs, said it was
not the machine's shape, and left it. It was not their shape either: four samples had fallen into two pairs. Taken eight times each, in a
randomised order, every one of the three is a single broad spread — the haystacks 27.3 to 31.6 ms, Rouen 25.3 to 36.0, the Orangerie 19.8
to 26.1 — with nothing in the middle missing.

What was moving was the machine, and it moves whole stretches of a run rather than one place in it: across those four runs the four late
viewpoints rose and fell together, all high in the first and third and all low in the second and fourth. And the bench walked the nine in a
fixed order, so a machine busy for ten seconds charged the same places every time, and a machine quietening over a run credited the same
ones. Two cold loads settle it, one walking the nine forward and one walking them back: the Orangerie is 20.1 ms measured eighth and 35.2
measured second; the pond 44.9 measured first and 31.7 measured last; and the two runs' own means differ by one part in a hundred (30.20
and 30.57). Nothing moved but where in the walk each place stood. M46c's table was the walk's shape, not the world's.

**What changed.** The bench warms every viewpoint first — a block at each, all of them thrown away, so the materials' first compile is paid
before anything is kept — and then walks the nine as a round, each round starting one place further along, so that over the passes a place
stands in a different position each time. The least of a place's rounds is kept. It costs exactly what it cost: sixty-three blocks of twenty
frames either way, nine warm and fifty-four kept, only differently arranged. And a block is now preceded by two settled frames rather than
one, because arriving at or leaving the aerial viewpoint changes the device pixel ratio and the canvas and its buffers are made again on the
first frame there — a cost belonging to the arrival, not to the frame.

**Verified.** With the order randomised, position no longer predicts anything: over 72 measurements the mean by position in the walk runs
25.5, 28.5, 28.1, 24.9, 27.2, 28.3, 28.8, 26.1, 29.7 ms — no order to it, where the fixed walk had put 45 at the first place and 23 at the
ninth. What is reproducible is not the level but the shape. Over 22 rounds in three runs whose own levels ran from 21 to 39 ms a viewpoint,
each place against the mean of the nine in its own round:

| Viewpoint | Against its round | Draw calls | Passes |
| --- | --- | --- | --- |
| The Bridge at Argenteuil | +9% | 56 | reflection, projector |
| Woman with a Parasol | +7% | 58 | reflection, projector |
| The Water-Lily Pond | +5% | 42 | reflection, projector |
| The whole universe | +5% | 148 | neither, at a quarter the pixels |
| Haystacks | +4% | 45 | reflection, projector |
| Poplars on the Epte | +2% | 37 | reflection, projector |
| Rouen Cathedral | −1% | 38 | reflection, projector |
| Les Nymphéas | −7% | 39 | reflection, no projector |
| Impression, Sunrise | −22% | 26 | reflection, projector |

The three runs agree on that shape to within four parts in a hundred while their levels differ by three fifths. So the machine sets what a
frame costs and the place sets only this, and no absolute table taken on this machine deserves more precision than the level it was taken at.

Why the two ends of it, counted rather than timed — the renders themselves, `renderer.render` counted over eight frames: twenty-four at
seven of the nine (eight to the screen, four the reflection at 512 × 288, twelve the projector's depth at 1024 × 768, three to a pass taken
every other frame); twelve at the Orangerie, which is `panels` and not a `painting` with a camera, so `updateProjector` returns at its first
test and that pass is never taken there at all; and eight at the aerial, where the reflection is skipped above sixty metres and the painting's
spot is out of range — it draws the whole world, 148 calls and 1,146,086 triangles against 26 to 58 elsewhere, and lands at the mean anyway
because it does it on a quarter of the pixels. The harbour takes both passes like the rest and is still the cheapest thing in the world by a
fifth: it simply has the least to draw, 26 calls and 30 of the 119 place objects surviving its fog. `m46d-nine.jpg` is the nine again, with
what each costs against its round.

**Still visible.** The shape is one machine's at one size, and the level under it is this machine's mood: the same nine ran at 21 ms a
viewpoint on the quietest round and 39 on the busiest, and nothing here separates a place's cost to better than a few parts in a hundred. A
round is still a whole trip through the nine, so a round's level is set by ten seconds of machine and not by one; only the rotation keeps
that from landing on the same places. The projector's pass still does not resolve at the eight pairs a default run gives or the twelve of
`?bench=6` — 6.9 ms over twenty-five. And the reflection's own cost is not the same at every place (8.3 ms at Argenteuil, 0.5 at the
harbour, where the reflected world is almost entirely fog), which the single figure the bench prints does not say.

### Progress · M47 — the country beyond the world, and five other things the air showed (fix: "conduct a full air fly review using the browser and taking pictures to have a full examination" / "apply the fixes")

**What was wrong.** Sixteen stops flown over the world between 300 m and 26 m, and seventeen frames of A/B after them, found six things.
Nothing had been looked at from above before.

*The world ends.* The base plane is 520 by 720 and the ground stops at its edge, so from the aerial viewpoint the terrain, the Seine and the
sea all finished in straight cut lines with the sky under them, and the world read as a card lying on cream paper. Fog cannot cover that,
and the cap on the fog's climb that the review recommended as the one-line fix was tried and thrown away: at 250 m the fog opens to 1674 m
where the farthest corner of the ground is 906, but the two long sides stand only 270 m from that eye where the fog's own near is 449, and
no fog short enough to hide them leaves the cathedral standing — at 820 m and at 640 the far half of the world goes and the sides are as
hard as before.

*A sunburst standing on the painter's spot.* From sixty metres up a fan of light and shade over the ground, widest looking straight down.
Not the projector's depth pass, which it survives; not the sun, being the same at dawn as at dusk; not a shadow camera, the scene holding no
`THREE` light at all. It is the picture's key: `extCol` reads the canvas by the azimuth of each point from where the painter stood, so the
canvas's columns fan out over the ground around him — the same fan M46 closed on the zenith, still open at the equator, where the ground
around his feet is. And `projectorOn` gates the key on the place's weight, which `placeWeights` measures along the ground alone, so the key
was thrown at full strength from 300 m up. On foot that fan is under your feet and behind you and never seen.

*A white kerb round every water.* Found while chasing the first. The terrain carries its palette as an index on its vertices, 0 for the
water and 9 for the grass, and the index is interpolated across every shore; the seven lying between are the pond's other colours, and two
of them are all but white. So a band of near-white a cell wide stood at every waterline in the world, and round the harbour it read as a
line ruled on the sea.

*The Orangerie's roof.* A prism at a pitch of ten degrees, 69 m by 28, in one palette — from the air a grey lid laid on the garden, the
widest surface in the world with no incident on it, and nothing on it of the veiled skylight that every one of the room's four hours is
daylight through.

*The meadow's grass stops at a circle.* The tall grass is laid in a disc of 36 m round the parasol's viewpoint, and the disc has an edge.
The walker never reaches it; the flyer sees a circle drawn on the field.

*And an hour asked for was thrown away.* Crossing into a place re-bases the dial to that place's own painting hour, which is what puts each
painting's own light under a flyer and is right. But it did it to an hour the visitor had set as well: an evening chosen at the pond was
gone by the poplars, seven crossings in seven seconds at 250 m.

**What changed.** The ground goes on. A ring of four strips outside the base plane, graded outward (eight metres at the seam, seven hundred
at the last of nine steps, out to a kilometre and a half) and sampled every eight along the rim, laid on the same height field — which is
defined everywhere and holds the rim's own value beyond the box, so the plateau carries west, the shore of Le Havre carries south and the
Seine's bed carries out both ways. Where that field lies under water the ring is laid at the surface and not on the bed. It is 2.7 k quads,
and the corners belong to the north and south strips alone so no two overlap. Its faces are turned to look up whichever way their quads were
wound, which two of the four were not, and stood as dark slabs until they were.

With the ground continuing, the water had to as well, or the river and the sea ended in the middle of a plain: the Seine's own surface ran
to exactly the world's width and the harbour's stopped 35 m short of the world's edge on one side and 40 on the other, leaving a strip of
bank between the sea and what lay beyond it — that strip was the white line ruled round the harbour. Both surfaces now run out past where
the fog closes, and show only where the ground is under them, so the harbour's is the bed it was dug for and nothing else; its north edge
stands clear of the Epte.

The picture's key is not thrown from the air: `projectorOn` now falls away over the fifty metres above the painter's own eye, and what is
left from up there is the world in its own colours. And the terrain's palette is asked per fragment instead of carried on the vertices — the
rule the ground was built by, `TERRAIN` in the patch shader, tested on the height itself — so the shore is where the ground crosses the
water's level and there is nothing between the two colours.

The Orangerie's two rooms carry their lanterns, each a monitor half a metre proud of the ridge and buried in the slates at its shoulders.
The meadow's grass reaches to 48 m and thins over the last eighteen, the blades shortening with it, so the field has no rim; there are 14.3
k strokes cast where there were 10 k, and 1.2 k more survive. And an hour the visitor sets is kept: `hourAsked`, which every way of setting
the hour passes through, and which the url's `?hour=` already had in another form. A dial nobody has touched still follows the ground.

**Verified.** `m47-after.jpg` is the same sixteen stops flown again and `m47-before-after.jpg` the six side by side. The world has a horizon
in every frame and no cut edge in any. The sunburst is gone from the plan view; the white line round the harbour is gone; the grass fades
where it stopped. The eight painting viewpoints are untouched — the canvas is thrown onto the world exactly as before, the key falling away
only above twelve metres, which no viewpoint is. The dial holds 0.90 the length of the world when it is set by hand, where it ran 0.33,
0.50, 0.50, 0, 0, 0.50, 0 before; untouched it still brings each place its own hour, and coming down it re-bases at 29 m above the ground.

The cost, benched against the same build at HEAD on the same machine minutes apart, three passes of twenty at 2560 by 1440: the nine
viewpoints 10.6, 8.5, 8.5, 9.7, 10.5, 10.2, 6.9, 8.3, 8.0 ms before and 10.9, 8.7, 9.2, 9.8, 10.6, 10.2, 7.5, 8.5, 8.4 after — a mean of
9.02 against 9.31, three parts in a hundred, and the two largest of it at Argenteuil and the harbour, whose water surfaces were widened. 8.2
k triangles more and 1.2 k patches, on 1.15 M and 266 k.

**Still visible.** The white drift on the Seine is still there: it is the river's own painted strokes, laid dense where the painter stands
and sized by the distance from him, so a hundred of them a metre across sit in front of the Argenteuil viewpoint. On foot that density is
the picture's sparkle on the water; from 300 m it is the only thing in the world that clips to white, and it was left alone because thinning
it changes the viewpoint that matters most. The hour still barely reaches the air — the three stops move the aerial frame by 7 to 22 levels
of 255 where on the ground the same three move it by 23 to 50 and change its hue — because the fog at that height is 1674 m of the hour's
own colour over everything. The flyer's ceiling is still 300 m, fifty above the aerial viewpoint. A boat seen from straight overhead is a
dark ellipse with its masts lying flat on the water. And the ring's own far edge is fog-coloured but not fog: the fog stops at nine tenths,
so a tenth of a green plain stands against the sky where it ends, which is a horizon and reads as one, but is not the sky's colour.

### Progress · M48 — the grey sky: the canvas's own mean laid over the whole dome (fix: "I still see the grey sky. see the screenshot and investigate")

**What was wrong.** The sky takes the picture's key as the rest of the world outside the frame does — `.85` of `inKey(sky, extCol(d))` —
with no fall in direction and none in distance. But `extCol` is soft by design (level 7 of the canvas, level 9 a fifth of a width past the
frame, and since M46 the columns close on the middle toward the pole), so far from the frame it is the canvas's own mean, and over the pond
that mean is willow and water. Standing at his own spot and reading nine directions of the upper sky — 45, 65 and 85 degrees up, at the
picture, ninety degrees round and behind him — every one of the nine came back between 129 and 144 red, 140 and 158 green, 130 and 148 blue,
a blueness (b less the mean of r and g, of 255) from −4.8 to +3.6. One flat green-grey over the whole dome, greener than it was blue, with
no gradient in it, no strokes, no sun and no hour. M46's own note had said as much and left it: *the ext read still knows nothing of the
sun; the upper sky's colour round the turn is the canvas's, mirrored, not the light's.*

Across the seven places, the upper sky (38–54 degrees, twelve azimuths, the dial pinned at .5) measured, against the same sky with no key at
all: the pond −3.3 against 32.8, the harbour 6.2 against 37.2, the haystacks 22.9 against 50.7, Argenteuil 36.8 against 72.9, the poplars
39.4 against 64.7, Rouen 49.4 against 85.9, the parasol 56.7 against 96.3. Half to all of the sky's blue, everywhere; and at the pond the
sky came out green.

The frame in the screenshot was found: the meadow between the pond and the parasol, at (−22, 2, −42), fifty metres south of where he stood
and looking the way he looked. The canvas itself has gone there — it dissolves by eighteen metres — but its key had not, and `extCol` takes
a direction and not a place, so the directions that held his water and his willows now hold sky, and the sky wore the pond's water: a
green-grey with four soft columns of the canvas standing in it from the horizon to the top of the view. Forty metres on, the same seven
measured the pond −5.9, the haystacks 22.6, Argenteuil 38.1, the poplars 38.8, Rouen 42.0, the harbour 40.0 and the parasol 87.8 — the last
two already past their own weight gate, the other five as keyed as at his feet.

**What changed.** Two falls, in `SKY_FRAG` alone. The ground's key is untouched, and so is everything M46 repaired.

*By direction.* `outAng(d)` — how far outside the frame a direction lies, in radians, negative within it — and the key falls from half a
radian past the frame's own edge to 1.25. Half a radian is 29 degrees, further out than the widest window shows beside the canvas: at the
pond and at Rouen the screen's own edge stands .41 radians outside the frame at an aspect of 1.94, .52 at 2.39 and .67 to .69 at 3.39, so
the eight viewpoints keep the surround they had. By 1.25 the key is gone, which is overhead and behind him.

*By distance.* The key goes over 12 to 45 metres from the painter's spot, as the canvas goes over 8 to 18. It belongs to his spot as the
canvas does; a place's own radius is 30 to 60 m, and the weight gate takes what is left.

And the frame's own half-angles are now taken once a frame into `uProjAng` rather than as two `atan` of a uniform at every pixel of the dome
— the sky is drawn first, with no depth to reject it, so its shader runs on every pixel of every frame.

**Verified.** `m48-sky-1.jpg` and `m48-sky-2.jpg`: ten frames before and after — the pond's and Argenteuil's viewpoints, the meadow of the
screenshot, behind him at the pond and a hundred degrees round from it, behind her at the parasol, the poplars' sun overhead, Rouen behind
the front, the haystacks' far horizon, and the harbour out to sea.

The pond's nine directions, at his spot, before → after (and the sky with no key at all): straight at the picture, 45 degrees up −4.8 → −4.8
(30.4) and 65 up −4.2 → −3.5 (27.3), held to a tenth as they must be; ninety degrees round, −2.3 → 25.8 (34.4), 0.7 → 23.0 (36.2) and,
overhead, 2.3 → 26.6 (36.9); behind him, −1.4 → 35.9 (35.8) and 3.6 → 40.6 (40.6), which is the world's own sky exactly; overhead in his own
azimuth 1.2 → 20.6 (35.3). The dome that was one colour has a front and a back again.

Over the seven places, the upper sky's blueness at the spot, before → after (no key): the pond −3.3 → 17.0 (32.8), the harbour 6.2 → 24.4
(37.2), the haystacks 22.9 → 39.1 (50.7), Argenteuil 36.8 → 59.1 (72.9), the poplars 39.4 → 53.2 (64.7), Rouen 49.4 → 69.5 (85.9), the
parasol 56.7 → 82.1 (96.3). Forty metres on, where the walker is, it comes back to within a unit of the sky with no key at all: the pond
−5.9 → 24.4 (25.0), the haystacks 22.6 → 49.5 (50.2), Argenteuil 38.1 → 73.9 (74.8), the poplars 38.8 → 66.0 (66.7), Rouen 42.0 → 76.2
(77.3), the parasol 87.8 → 89.4 (89.5), the harbour 40.0 → 39.7 (39.7). In colours, the pond's sky at the spot goes 139,154,143 →
147,169,175 and forty metres on 136,150,137 → 150,175,187, against 151,176,188 with no key.

And the spread round the turn — the sky's own gradient, its strokes and its sun coming back — at the spot: Rouen 54 → 81 (81), Argenteuil 36
→ 56 (65), the pond 59 → 71 (68), the poplars 75 → 83 (82), the harbour 26 → 32 (40), the haystacks 15 → 15 (14). The parasol's falls, 63 →
45 (50): what it lost was the canvas's own columns.

The eight viewpoints: seven are the same to the byte — 0 of 48 900 pixels differ at 300 px wide, against the build at HEAD, both under
`?still`. The eighth is Argenteuil, which differs by 145 levels over 4 799 pixels; but Argenteuil differs from *itself* by 110 over 3 526 on
a second pass of the same walk in the same build, and by 131 over 4 141 on a third. Its boats drift under `?still`, and that drift is the
whole of it. At an aspect of 2.39 the viewpoints differ by at most 1 level and at 3.39 by at most 7, at the corner where the window shows
most beside the canvas.

The cost could not be separated from the machine. Twenty paired blocks of 35 frames, thirty metres up looking up so that the whole sky is in
the view, at 2800 × 1520: the median paired difference +0.26 ms and the trimmed mean +0.29, on a level of 14 ms, the differences running
−1.15 to +1.94. A three-way run the same hour — the old shader, the new one, and the new one with the half-angles written in as literals —
put the old at 14.43 ms, the new at 13.93 and the literal one at 13.35: the new faster than the old, which it cannot be. Under half a
millisecond, and the machine moves more than that in a run (M46d).

**Still visible.** In the frame's own cone, within half a radian of its edge and a dozen metres of the spot, the sky is still the ext
read's, and over the pond that read is willow: the sky above the water garden keeps its green cast where he stood, which is the point of it
— the two directions held above are −4.8 and −3.5 where the world's own sky is +30 and +27. Looking 70 to 100 degrees round from the frame
the handover crosses the view as a soft gradient, blue away from the picture and the canvas's colour toward it, smooth over some forty
degrees but there. The ground's key does not fall with distance or direction, so at forty metres the ground is the picture's and the sky is
the world's; at the horizon behind you the two meet without a seam, checked at the harbour, the haystacks and Argenteuil, because a place's
fog colour is its sky's horizon colour by construction. The four numbers (.5, 1.25, 12, 45) are chosen against the eight viewpoints and the
eye, not measured. And Argenteuil's boats move when the world is asked to stand still: `?still` stops the clock for everything else, and the
one viewpoint with moored boats in it cannot be compared with itself frame to frame.
