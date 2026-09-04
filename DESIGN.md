# Monet's Universe — design

A single web page. You open it and you are standing on the Japanese
bridge at Giverny, inside *The Water-Lily Pond*. Drag to look. Walk, or
scroll, and the pond becomes the garden, the garden becomes the meadow
where the woman holds her parasol, the meadow rolls down to the Seine
with its sailboats, the river bends past the poplars to the haystacks,
and the road at the end leads into Rouen, where the cathedral is waiting
in whatever light the hour gives it. Seven Monet paintings, one continuous
geography, nothing between them but distance.

The reference is *A town made of paint* (van-goghs-town.surge.sh): a
three.js world with no textures and no models, every surface made of
brushstroke geometry, keys 1–6 for painting viewpoints, a dock with
walk/fly, light, quality and photo mode. We keep that experience shape
almost exactly. What changes is everything the paint is made of, because
Monet is not Van Gogh.

---

## 1. Thesis: Van Gogh is stroke, Monet is light

Van Gogh's town works because his paintings *are* discrete, directional
strokes: you can rebuild a wall out of ten thousand dabs and it reads as
Van Gogh. Monet's paintings are not made of strokes you can count. They
are made of *broken colour* — small patches of unblended pigment laid
next to and over each other so the eye mixes them — and of *atmosphere*:
form dissolving into haze, water carrying the sky, edges that are never
drawn. The subject of a Monet is not the haystack. It is the light on the
haystack at that hour, and he proved it by painting the same haystack
twenty-five times.

So the three decisions that define this project:

1. **The hour is the primary mechanic, not a slider in a popover.** In
   Van Gogh's town the light control "nudges" the painting's own light.
   Here it *is* the painting: every place in the universe has a series
   of canonical hours taken from Monet's actual series, and moving the
   hour re-paints the whole world. Standing at the haystacks and dragging
   from morning to sunset to snow should feel like flipping through the
   Haystacks room at the Musée d'Orsay.
2. **Paint is patches, not strokes.** Surfaces are covered in soft-edged,
   overlapping colour patches (Monet's *tache*), not rigid dabs. Colour
   is decided per patch from a palette *and the hour*, never baked. Edges
   between objects are never clean: patches spill across them.
3. **Atmosphere is a material.** Distance fog in the colour of the hour's
   light, a haze layer over water and fields, and reflections that are
   themselves painted (broken, rippled, delayed) — not mirrors. Half of
   what the user sees at any time should be air and water, because half
   of Monet is.

Everything else follows from these. When a later decision is a close
call, the test is: *does it make it feel more like standing inside the
light of a Monet, or more like a game level with Monet colours?*

---

## 2. What we keep from the reference

- One HTML file, three.js from a CDN via import map, no build step.
- No textures, no models, no image assets. Geometry and pigment only.
  (This is also what keeps us clear of reproducing the paintings: we
  build the *places*, in his palette and touch, we do not display his
  canvases.)
- Deterministic seeded random, so the world is identical on every load
  and a shared photo can be found again.
- First-person camera: drag to look (pointer lock on click for desktop),
  W A S D to walk, F to fly, number keys jump to painting viewpoints
  along a curved camera flight, P for photo mode, R to return home.
- Touch joystick on mobile.
- A quiet bottom-right dock; popovers for viewpoints, light, quality.
- A location caption that fades in on arrival: painting title, place,
  year.
- Photo mode saves a PNG from the canvas (`preserveDrawingBuffer`).
- Quality presets that change stroke density with distance, never the
  shape of the places.
- A tiny debug API on `window` (camera, fps, draw calls, patch count).

---

## 3. What changes

| Van Gogh's town | Monet's universe | Why |
|---|---|---|
| Night sky over everything | Sky and light change per place and hour | Monet is a daylight painter; there is no single sky. |
| Strokes are rigid quads with hard edges | Patches are soft, overlapping, vary in size with distance | Broken colour, not impasto. |
| Light slider nudges ±1 | Hour is a first-class axis with named stops per place, from his series | The series *are* the work. |
| Fog is a mood | Fog is a material with per-hour colour, denser over water and at dawn | Atmosphere is subject matter. |
| Static water (one place) | Water everywhere: pond, Seine, harbour; painted reflection + ripple | Water carries more of Monet than land does. |
| Walk / fly | Walk / fly / **drift** (scroll-to-move along the promenade) | The user should be able to just scroll through the world. |
| 5 paintings + aerial | 7 paintings + aerial | See §4. |
| Town in the centre, paintings around | Water in the centre, town at the far end | Giverny is home; Rouen is a destination. |

---

## 4. The geography

One landmass, roughly 260 × 200 world units (1 unit ≈ 1 m), with the
river as the spine. Read north-to-south as a walk of about four minutes.

```
                      [7] Impression, Sunrise
                          Le Havre harbour — sea, cranes, orange sun
                                  ~
   ~~~~~~~~~~~~~~~~~~~~~~~~ estuary / fog ~~~~~~~~~~~~~~~~~~~~~~~~~~
                                  |
      [6] Rouen Cathedral        |  the town: one street of
          the facade, the square  |  half-timber houses fading
                                  |  to haze
   ----------------- road --------+-------------------------------
                                  |
      [5] Haystacks    fields     |     [4] Poplars on the Epte
          two stacks, stubble     |         a curve of tall trees
                                  |         along the bank
   ~~~~~~~~~~~~~~~~~~~ the Seine ~+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
      [3] Sailboats at Argenteuil |
          the bridge, moored      |
          boats, reflections      |
                                  |
      [2] Woman with a Parasol    |     the flower garden
          the hill, wind, cloud   |     (Grande Allée, nasturtiums)
                                  |
                        [1] The Water-Lily Pond  ← HOME
                            Japanese bridge, willows, lilies
```

### The seven places

| Key | Painting | Year | Place | Canonical hours (from the series) |
|---|---|---|---|---|
| 1 | The Water-Lily Pond (Japanese Bridge) | 1899 | Giverny, the water garden | morning green · afternoon · evening violet |
| 2 | Woman with a Parasol | 1875 | a hilltop meadow, wind | late morning, big cumulus |
| 3 | Regatta / Sailboats at Argenteuil | 1872–74 | the Seine, boats, the road bridge | midday · calm evening |
| 4 | Poplars on the Epte | 1891 | riverbank, a curve of trees | wind, afternoon · pink sunset · autumn |
| 5 | Haystacks | 1890–91 | stubble field, two stacks | end of summer · sunset · snow effect · morning frost |
| 6 | Rouen Cathedral | 1892–94 | the square, the west facade | morning (blue) · full sun · harmony in brown, dusk |
| 7 | Impression, Sunrise | 1872 | Le Havre, the outer harbour | one hour only: dawn, orange sun in grey |

Key 8 (or the last item in the viewpoint list) is the aerial: *the whole
universe*, showing river, garden, fields and the town in one frame.

### Why these seven

- Each is instantly recognisable *as a place*, not just as a canvas.
- Together they cover Monet's four subjects: garden, river, field, and
  the built facade in light — and the one that named the movement.
- Each supports a different hour behaviour, so the hour axis is never
  a no-op: the pond does colour, the haystacks do seasons, the cathedral
  does the same stone at three lights, the sunrise is fixed.
- They lay out plausibly: Giverny → Seine → Epte → fields → Rouen →
  the estuary. It isn't real geography but it is a believable Normandy.

### Transitions

Nothing between places is empty. Between 1 and 2 is the flower garden
(the Grande Allée under its arches). Between 2 and 3 the hill descends to
the towpath. 3 to 4 is the riverbank. 4 to 5 crosses stubble. 5 to 6 is
the road with the first houses of the town. 6 to 7 the street ends at the
quay and the water opens. The fog thickens toward 7 so that Le Havre only
resolves as you arrive, and the sun appears out of it — the reveal of the
whole piece.

---

## 5. Light: the hour system

### Model

A single global scalar `hour ∈ [0, 1]` is *not* enough, because Monet's
hours are not a clock: "snow effect" and "harmony in brown" are stops,
not times. So:

- Each place declares an ordered list of **stops**: `{name, sky, sun,
  fog, palette, ground}`. Stops come from the real series titles.
- The global control is a single slider in the light popover, plus
  `[` `]` keys, moving through the *current place's* stops with blending
  between adjacent ones. The label under the slider shows the stop's
  name: "Haystacks, End of Summer", "Snow Effect", "Sunset".
- Leaving a place carries the *feeling* of the hour, not the index: we
  keep a continuous `t` and each place maps it onto its own stop list.
  Walking from evening haystacks into Rouen gives you dusk on the facade.
- Place 7 (Sunrise) ignores the slider. It is always dawn. The slider's
  label says so.

### What a stop drives

- Sky: a gradient dome painted in patches (see §6), sun position and
  disc, cloud density.
- Sun and hemisphere light colour and intensity.
- Fog colour, near and far distance. Fog colour is always taken from the
  sky near the horizon so distance dissolves into light, not grey.
- The **palette shift**: every patch stores a base colour index; the
  stop provides a 16-entry palette per place, and the shader blends the
  two adjacent stops' palettes. Nothing is re-generated on hour change;
  it re-colours.
- Ground state (snow / stubble / grass) as a palette, not new geometry.
- Water: reflection tint, ripple speed, and how much sky vs. depth.

### Default

Home opens at the pond, mid-afternoon. The first thing a user does with
the slider should be dramatic: we tune the pond's evening stop to go
properly violet.

---

## 6. Paint: the patch system

### Patches

The unit is a **patch**: a small planar quad with a soft (feathered,
slightly irregular) alpha mask generated procedurally in the shader, no
texture. Patches are laid on every surface with these rules:

1. Size scales with distance from the painting viewpoint of that place,
   so near things are small dabs and far things are broad, exactly as a
   canvas treats distance.
2. Orientation follows the *subject*, not the surface normal: water
   patches are horizontal and elongated, foliage patches are round and
   jittered, the cathedral's patches are vertical, the sky's are long
   and horizontal.
3. Two to three layers overlap, with the top layer sparser and lighter.
   Van Gogh's town has one layer. Monet needs the lower layer showing
   through.
4. Colour is a palette index plus a small hue/value jitter; the actual
   RGB is resolved in the shader from the current stop's palette, so the
   hour re-colours everything in one uniform.
5. Patches near an object's silhouette spill across it: 15 % of edge
   patches are placed slightly outside the geometry.

All patches of one place and one material are merged into one
instanced/merged geometry, so draw calls stay in the low hundreds.

### Surfaces

- **Water** is the special case. A flat plane with a painted reflection:
  we render the scene mirrored into a low-resolution target, then draw
  it not as a mirror but as a field of horizontal patches that sample
  the reflection with a slow, rippled offset and a per-patch delay. The
  result should look like Monet's Argenteuil water: the boat is there,
  broken, wobbling, a beat behind. Lilies are patch clusters floating on
  top, casting no shadow.
- **Foliage** (willows, poplars, garden) is clouds of round patches on
  simple trunk geometry; poplars have the patches biased to one side so
  wind reads.
- **Stone** (cathedral) is a rough blocky facade of boxes with the patches
  doing all the detail: no carved tracery, just the *impression* of
  tracery, which is what the paintings do.
- **Sky** is a dome of very broad horizontal patches over the gradient,
  plus cloud clusters at the parasol hill.
- **Figures** (the woman, the boaters) are stylised low-poly, patched
  like everything else, and few. Monet's people are notes, not portraits.

### Motion

Nothing is static. Water ripples, lilies drift, poplars sway, the
parasol's dress and the cloud move, haze drifts, the sun's reflection
in the harbour wobbles. All of it is cheap vertex-shader motion driven
by time, no simulation. The rule: slow. Monet's world does not flicker.

---

## 7. Movement and the "scrolling" feel

Three modes, all live at once:

- **Walk.** W A S D / arrows, drag to look. Default speed 3 u/s. Feet
  stay on the ground mesh; you cannot walk into the river (you stop at
  the bank) but you can fly over it.
- **Fly.** F toggles. Same keys plus Q/E for altitude.
- **Drift.** The scroll wheel (and two-finger scroll, and a swipe on
  touch) moves you forward or back along the **promenade**: a single
  hand-placed spline that threads all seven places in order. You keep
  free look while drifting. This is the "scrolling through Monet's
  world" feeling: no keys, no learning — the page scrolls, and the
  scroll is a walk. If you have walked off the promenade, the first
  scroll eases you back onto its nearest point.

Viewpoint jumps (keys 1–7, or the dock list) fly the camera along a
curved path that rises when the distance is large, as the reference
does, so a jump across the map shows you the universe on the way.

The camera always arrives at a painting's viewpoint in its composition:
position, look direction and field of view chosen so that the framing
matches the canvas. Photo mode from a viewpoint should produce *the
painting's* framing.

---

## 8. Interface

Same discipline as the reference: the interface should feel like a
plaque in a gallery, not a HUD.

- **Dock** (bottom right, appears on first interaction, hides in photo
  mode and pointer lock): walk/fly · viewpoints · hour · | · photo ·
  quality · home.
- **Viewpoint popover**: eyebrow "Stand inside the painting", the seven
  titles with place and year, a swatch of the place's key colour, then
  "The whole universe".
- **Hour popover**: eyebrow "The same place, another hour". A slider
  whose label is the current stop's name in the painting's own words.
  At Sunrise the slider is disabled with the note "This one is always
  dawn."
- **Quality popover**: Light · Balanced · Rich, eyebrow "Broken colour".
- **Location caption**, bottom left: title, then place · year in small
  caps, fades after four seconds.
- Typography: Georgia for titles, system sans for the small labels.
  Colours: the dock is a dark blue-grey glass, accent a warm pale gold.
  We do not use Monet's palette for the UI; the UI must stay out of the
  painting.
- Accessibility: canvas has an aria-label describing the controls; all
  buttons have labels and aria-pressed/expanded; keyboard reaches every
  control; the caption region is aria-live.

---

## 9. Performance budget

Target: 60 fps on an M-series MacBook at 2× DPR, 30 fps on a 2020 phone
at Light quality. Concretely:

| | Balanced | Rich | Light |
|---|---|---|---|
| Patches, total | ≤ 250 k | ≤ 450 k | ≤ 120 k |
| Draw calls | ≤ 200 | ≤ 250 | ≤ 150 |
| Triangles | ≤ 1.5 M | ≤ 3 M | ≤ 0.7 M |
| Reflection target | 512 px | 768 px | 256 px |
| Pixel ratio cap | 2 | 2 | 1.5 |

Places beyond fog distance are culled entirely; the aerial view raises
the fog distance and drops to the Light patch set temporarily.

Load: the world is generated on the client in under 3 s on Balanced; a
loading veil shows the pond's palette washing in, patch by patch, so the
wait is part of the piece.

---

## 10. Technical shape

```
Monet's universe/
  DESIGN.md         this document
  index.html        the whole piece: markup, CSS, one <script type=module>
  (later) surge deploy of the folder, single file, no build
```

- three.js r170 via importmap from a CDN, `mergeGeometries` from addons.
- One custom `ShaderMaterial` for patches (soft mask, palette lookup,
  time-based motion), one for water, one for sky. Everything else is
  three.js built-ins.
- World description is data first: a `places` array of `{name, place,
  year, viewpoint: {p, look, fov}, stops: [...], build(scene)}`; the
  promenade is an array of points; the hour system is a pure function
  `(placeIndex, t) → blended stop`. Someone should be able to add an
  eighth painting by adding an entry.
- Seeded PRNG (LCG, seed `18401926` — his years).
- No analytics, no fonts fetched, no requests after the three.js module.

---

## 11. Build order

Each milestone is *walkable and deployable* on its own; we never sit on
a half-world.

1. **The pond.** Ground, bridge, water with painted reflection, willows,
   lilies, sky dome; patch material; walk + look + dock + caption; three
   hour stops working. This proves the thesis. If the pond doesn't feel
   like Monet at this stage, nothing after it will.
2. **Drift.** The promenade spline and scroll-to-move, first between the
   pond and a placeholder second place.
3. **The garden and the hill.** Woman with a Parasol; wind and cloud.
4. **The river.** Argenteuil with boats and the bridge; the reflection
   system at scale; fly mode.
5. **Poplars and haystacks.** The seasons stops; snow as a palette.
6. **Rouen.** The facade, the street, the three lights.
7. **Sunrise.** Harbour, fog reveal, fixed dawn; the aerial view.
8. **Polish and mobile.** Quality presets, touch joystick, photo mode,
   loading veil, performance pass against §9.

---

## 12. Open questions

- **Sound.** The reference is silent. A single low bed of water and wind
  would suit this world better than it suits Van Gogh's, but it must be
  off by default and behind a dock button. Decide after milestone 4.
- **Seasons vs hours.** The haystacks' snow stop makes the slider do two
  things at once. Alternative: a separate season control. Current
  position: one slider, stops named honestly; revisit if users find it
  confusing.
- **How much of the Grande Allée.** The flower garden could swallow the
  whole patch budget. It is a transition, not a place; keep it to the
  arches and the path until everything else stands.
- **Name.** "Monet's universe" is the folder. The title tag could be
  something more Monet-shaped ("A place made of light"). Not urgent.
