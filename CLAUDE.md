# reactions — Molecular Collision Model

Standalone single-file sim. Vanilla HTML + CSS + JS, no build step — edit `index.html` directly.

To run locally: `python3 -m http.server` in this directory, then open `localhost:8000/`.

Forked from the `fission` repo and gutted: the nuclear physics, the scripted
fission animation and its scrub/replay machinery are all gone. What was kept is
the chrome — themes, quality selector, zoom, the toolbar, the energy bar.

The fission names are gone — nothing is called a nucleus, a neutron or a scrub
bar any more. `index.html` opens with a map of its sections; read that first.

## Vocabulary

Used consistently in code, comments and UI, so pick these words for new work:

| word | means |
|---|---|
| **atom** | one circle — `{el, lx, ly, r, m}` in body-local coordinates |
| **molecule** / **body** | a rigid group of atoms with `(x, y, angle, vx, vy, spin)` |
| **species** | a recipe for a molecule: `SPECIES.H2 = { els: ['H','H'] }` |
| **energy of motion** | kinetic — translation + rotation. `energyOfMotion()` |
| **energy in fields** | the glowing lobe inside each bond. A picture, not a sum |
| **the band** | the ±`SPEED_SIGMA` speed limit on collision outputs |
| **the dial** | the launch-speed slider, `#speedDial` |

The energy trace stores samples as `{ t, motion }` rather than a bare number,
so a `fields` channel can join it without touching the drawing code.

## The one idea

The world is a **walled box** filling the canvas between the toolbar and the
energy bar. Molecules are **rigid bodies** that translate, spin, and bounce
perfectly elastically off one another and off the four walls.

(It was a wrap-around "asteroids" box until walls replaced it. `wrapDelta` and
`wrapOffsets` are gone — git has them if the wrap is ever wanted back.)

Nothing is scripted. There is no timeline, no phases and no replay: the sim just
runs.

## Atoms and molecules

```javascript
const ATOM_RADIUS     = 30;    // drawn radius of one H atom; most geometry is a multiple of it
const BOND_OVERLAP    = 0.72;  // < 1 so bonded atoms overlap, space-filling style
const ELEMENTS = {
  H: { rgb: '185,192,205', rScale: 1.00, q: 0 },   // grey
  O: { rgb: '232,56,47',   rScale: 1.27, q: 0 },   // red
};
```

`rScale` is the van der Waals radius relative to hydrogen. There is no charge
field on an element: see **Energy in fields** below for why.

Atoms are not drawn as circles any more. What you see is the **valence
electrons** tracing their actual paths — see below. The earlier renderers (a
translucent shell, then an arrow lapping a meaningless circle) are both gone.

Icons (`drawAtomIcon`) stay a plain ring — the arrow is unreadable at icon size.

### The nucleus

Each atom carries one small **proton** at its centre, lifted from fission's
`drawParticles`: the same pink sphere (`#ff1a65` shell, `#ff80b0` core) with a
white "+", at `nucleusDiameter` = 12 px. One proton stands in for the whole
nucleus — an oxygen really has eight — so it is a marker saying "the positive
charge is here", not a count. Shown in both views; in Outline it is what says
how many atoms the silhouette is made of.

**Fission drew it with a per-particle `shadowBlur`. Do not copy that.** The
whole thing — halo, sphere and "+" — is baked once into a sprite by
`protonSprite` and blitted, so it costs a `drawImage` per atom: the hi-res draw
went 2.0 → 1.87 ms with it added. The blur happens at bake time, where paying
once is fine. Size is a dev slider.

## Space-filling view

The **Outline** button. Each atom is a **sphere** at its van der Waals radius,
so a molecule reads as one continuous layer of matter with its nuclei just
showing through.

```javascript
let sphereOpacity = 0.80;   // 1 = solid; 0.80 is "20% transparent"
```

Each atom draws its **body** inside the layer:

1. the element's flat colour
2. **limb darkening** — a gentle fade to dark at the rim, centred a little up
   and left. No specular highlight and no bright side; just enough falloff to
   say "ball" rather than "disc".

```javascript
let sphereDepth = 0.40;   // limb darkening + edge strength; 0 = flat discs
```

Then the **edges**, once per molecule, on top of all its bodies: the outer
silhouette, and the seam.

### The seam is a circle

Two spheres meet in a **circle** — not in the lens their two flat discs make.
That circle lies in the plane perpendicular to the bond, so on screen it is an
ellipse: full width across the bond, squashed along it by however far the bond
is tilted out of the screen.

```
d = centre separation      a = (d² + rA² − rB²) / 2d     (A → the chord)
h = √(rA² − a²)            the circle's true radius
```

A bond lying flat in the screen plane would project that circle **edge-on**,
giving back a bare line — the "overlap" look. So the view is given one fixed
tilt and every seam is drawn at it:

```javascript
let seamTilt = 0.34;   // 0 = edge-on (a line); 1 = face-on (a full circle)
```

Measured on an O=O at r 38.1, d 54.9: seam radius **26.4 px**, **18.0 px** wide
on screen. No atom draws its own full outline any more — inside an overlap, what
belongs there is the seam, not two crossing arcs.

Three earlier versions were wrong:

- A **per-atom radial gradient offset up-and-left** to read as a lit ball. It
  looked like a reflection, and the front atom hard-occluded the one behind it.
- Darkening inward from the **molecule's** silhouette instead, which removed the
  light direction and the occlusion — but with no per-atom edge left, two atoms
  melted into one blob with a pinched waist. (That version also needed 14 thin
  passes to avoid contour banding, and round line caps so the run ends did not
  cut a crease across the interior.)
- **Per-atom full circles** again, subtle enough that the crossing arc was what
  you read. That gave a line, but the wrong one: the outline of the *overlap*,
  two arcs meeting at points, rather than the one circle where the surfaces
  actually intersect.

`drawMoleculeRim` strokes the union silhouette for definition — each atom circle
only where it is not buried inside another, so overlapping atoms read as one
outline rather than a pile of rings. It is a thin line, **not a glow**: the old
version used `shadowBlur`, which is the per-atom cost this project has been
bitten by before.

Cheaper than the electron view: measured at 18 bodies, **0.81 ms** whole frame
(0.56 of it the discs) against ~3.0 ms for the electron view.

Opacity is a dev slider.

## Valence electrons

Ported from the **`bonds` repo's Trails view** (`/Users/jkremer/Projects/bonds`,
`_drawBondElecTrails` and `_stepMolBondElecOne`). Each valence electron is a
moving point paying out a fading trail with an arrowhead at its head: **the
arrow is the electron**, and the trail is where it has just been.

**Only the valence shell is modelled.** Inner shells are ignored entirely — they
take no part in bonding and would be a crowd of rings around every nucleus.

Two kinds of electron, which is the whole point:

| kind | path |
|---|---|
| **bonding** | a lemniscate (figure-8) whose two lobes wrap the two nuclei and whose crossing is the bond midpoint |
| **lone** | runs a plain circle around its own nucleus at a fixed radius |

```javascript
const VALENCE    = { H: 1, O: 6, He: 2 };
const BOND_ORDER = { 'H-H': 1, 'H-O': 1, 'O-O': 2 };   // 2 = double bond
```

Electrons are derived, never hand-listed: each bond takes `order x 2`, each atom
keeps `valence - (what it put into bonds)`. That gives, with no special cases:

| | bonding | lone |
|---|---|---|
| He | 0 | 2 — a full shell, no bond, free all the way round |
| H₂ | 2 | 0 |
| O₂ | 4 (double bond) | 8 |
| H₂O | 4 | 4 (two lone pairs on the O) |

### The lemniscate

`a = r + d`, where `r` is the atom's valence radius and `d` is half the bond
separation — so the far edge of each lobe lands exactly on that atom's shell.
The two "visits" are the two lobes of **one** figure-8 (`t` in `[0,π]` then
`[π,2π]`), so the electron crosses the midpoint between them.

**The speed is normalised by `d/a`.** Without it, a bond between a big atom and
a small one runs its two lobes at different speeds and visibly kinks at the
crossing — the bug recorded in the bond-kink memory. Measured on H–O (lobe radii
30 and 38): the lobes would run **14.7% apart**; with the fix they are **0.50%
apart**. This matters for H₂O, before H₂O is even spawnable.

### Width: the lobes should be circles

A bare lemniscate is a narrow bow tie. Its widest half-width is `a/(2√2)` —
**0.354** of the lobe's reach — so the figure-8 read as a pinched ribbon rather
than as two orbits. `eWidth` stretches it perpendicular to the bond:

```javascript
let eWidth = 0.60;   // 1 = bare lemniscate; 1.41 makes a lobe exactly round
```

At 1.45 each H–H lobe measured **51.6 px long by 53.0 px wide**, a ratio of
**1.03** — about as round as stretching can get it. **It ships at 0.60, the
bottom of its slider, and that is inert**: at `eRound` 1 the blend below lands
fully on the circle and never reads this. It only bites on the way down the
roundness slider, where 1.41 is the value to come back to.

### Roundness: the lobes *are* circles

Stretching only takes a lemniscate so far. Even at the width that makes it as
wide as it is long, it still pinches into the crossing, so the pair read as a
bow tie rather than as two orbits. `eRound` morphs each lobe towards a **true
circle** instead — the one tangent to the bond's perpendicular at the midpoint,
radius `a/2`, so the lobe starts at the crossing and its far edge lands on the
same valence shell the leaf reached:

```javascript
let eRound = 1.0;    // 0 = bare lemniscate lobes, 1 = tangent circles
```

Two of them, one per atom, touching at the midpoint. Still one path, still one
crossing, but the bulk of every lap is now circular round a nucleus — the same
picture the H₂ rings below give, in the shape that can also carry an O=O's four
electrons. Measured on O=O at `eRound` 1, a lobe is **59.7 px by 59.7 px**,
aspect **1.000**; at 0 it is 59.7 by 61.2, aspect 1.025 but with the pinch.

Both curves are written in axis coordinates (origin at the crossing, `+x`
towards atom k) and **both begin and end a lobe at the origin moving purely
perpendicular to the bond**, so the blend is smooth at every mix. Checked at
`eRound` 0, 0.5 and 1: across the lobe handoff the velocity direction matches
to **0.04°** and the magnitude to **0.1 px/rad** — no kink anywhere on the
slider. At `eRound` 1 both lobes arrive and leave at **90°**, which is what
tangency means.

`eWidth` only bites below 1, since a circle is already as wide as it is long;
the note under the slider says so. The width *jitter* still applies — it is
kept on the perpendicular, so the lobe breathes but averages a circle.

### Pass-to-pass variation

`eSteer` sets how hard the electron is pulled onto the ideal path — below 1 it
steers toward the ideal *velocity*, and the lag blurs the laps. **It ships at
1**: no lag, the electron sits on the ideal point every frame. So the cloud is
now entirely the three wobbles, re-rolled at **every lobe**:

```javascript
let eRadial = 0.26;   // +-: how far the lobe reaches past the nucleus
let eWander = 0.74;   // +-: how wide the lobe is
let eTilt   = 0;      // +- radians: the whole lobe swings off the bond axis
let eRoll   = 0.12;   // how fast a re-rolled target is taken up, per frame
```

`eWander` carries most of it, which is why it is so much wider than the 0.30
that was enough when steering lag was doing half the work.

**The swing is off.** It earned its keep on a leaf-shaped lobe, where rotating
about the crossing changed what the lobe looked like. A circle rotated about a
point on itself is the same circle somewhere else — so all `eTilt` did was walk
the lobes off the bond axis and drift the crossing, with no gain in texture.

An earlier version had almost none of this and it showed. Both jitters were
multiplied by 0.25 at the point of use, and the width roll was `-random()` —
**one-sided**, so a lobe could only ever come out slightly narrower than the
last, never wider. `eRoll` was also 0.08, which over a lobe's ~20 frames only
reached 81% of the new target before it was re-rolled. The amplitudes are now
applied at full value, both signs, with `eRoll` at 0.12.

All five are dev sliders.

### Two overlapping circles — how an H₂ bond runs

`H₂ bond as two circles` in the dev panel. Each of an H–H bond's two electrons
runs a closed circle around **its own** nucleus instead of the pair sharing one
curve, so an H₂ reads as two overlapping rings — the shape an O₂ already shows,
where each O carries a ring of its own lone electrons.

**Off by default, now that `eRound` exists.** It was briefly the default, and
`eRound` made that unnecessary: at roundness 1 the figure-8's own lobes already
*are* two overlapping circles, without leaving the one-path shape that an O=O
has to use anyway. This stays as the way to see genuinely independent rings —
one closed curve per nucleus — rather than two lobes of one curve meeting at
the bond midpoint.

```javascript
let bondCircles = false;  // on: a ring per atom, instead of the figure-8
let bondCircleR = 1.3;    // ring radius, multiple of the atom's valence radius
```

They overlap because a bond is shorter than two valence radii: H–H sits
`2 × 30 × BOND_OVERLAP` = **43.2 px** apart while each ring is `30 × 1.3` =
**39 px**, so the rings cross and share a **34.8 px** lens over the bond. That
lens is the picture — it is where both electrons are, which is what the
figure-8's crossing says with one line instead of two.

**Scoped to H–H**, because "two circles" only describes a single bond. An O=O
has four bonding electrons and would come out as four stacked rings, so it keeps
the figure-8. Widening it is the element test in `bondIsCircled()`.

**Position is set straight onto the circle, not steered onto it.** Steering was
tried first, for consistency with the lemniscate, and it does not survive a
closed curve: `eSteer` lags, and on a circle that lag cuts the corner inward
every frame instead of being absorbed by a changing shape. Measured over 900
frames it pulled the ring to a mean radius of **25.6 px against a nominal 30**,
dipping to 15.6 — a wobbling blob, not a clean ring. Set directly, the measured
mean is **30.5** with the range 22.8–37.0 that `eRadial` ±26% predicts. So
`eSteer`, `eWidth` and `eTilt` all do nothing in this mode, and the note under
the toggle says so.

Angular speed is `eSpeed × d / R`, the ring twin of the lemniscate's
`eSpeed × d / a` — it holds **linear** pace, so a wider ring turns more slowly
instead of racing. Measured at **10.7 px/frame** against the figure-8's ~10.8.

`eRadial` still applies, re-rolled once a lap rather than once a lobe, so the
laps breathe instead of lying exactly on each other. At 0 the ring repeats
exactly.

### Cost

This is the dominant draw cost, so it is bucketed like bonds does it: the trail
is split into `TIER_BUCKETS` runs and **each is stroked once at its own alpha**,
rather than one stroke per point. That is ~10 strokes an electron instead of 50.

Measured at the heaviest the sliders allow — 6 H₂ + 6 O₂ + 6 He = **96
electrons**, trails full at 48 points, hi-res:

| | ms/frame |
|---|---|
| electrons alone | 0.45 |
| whole frame, field off | 0.40 |
| whole frame, field on | 0.75 |

against a 16.7 ms budget.

**Measure with the trails actually full.** A freshly respawned world reads
~0.26 ms because every trail is empty, which is not a number worth having.

Trails live in **body-local coordinates** and are transformed at draw time, so a
molecule can translate and spin without smearing or rebuilding them. Each is a
fixed `Float32Array` ring buffer, so nothing is allocated per frame.

`stepElectrons` runs **once per frame, outside the physics sub-steps** — the
phase logic is tuned in per-frame units and breaks up with large or uneven dt.

It also runs **outside the `running` gate**. Pause stops the molecules moving
and spinning; it does not stop an atom being an atom, and a frozen electron
reads as a broken picture rather than a paused one. It also means the trails
are already drawn on page load, before GO is ever pressed.

### Lone electrons

Deliberately the simplest thing that works: a constant angular speed round a
circle of fixed radius. No jitter, no damping, no sector confinement. An atom's
lone electrons are spaced evenly and all travel the same way, so they read as
one ring at one radius instead of a scribble.

```javascript
let loneRadius = 0.7;     // multiple of the atom's valence radius
let loneSpeed  = 0.125;   // radians per frame at 60 fps — one lap in 0.84 s
```

At `loneRadius` 1.0 the ring sits on the valence shell, which is physically
where a lone pair belongs — **but that is also how far the bonding lobe reaches
past each nucleus, so the two overlap and the figure-8 gets hard to pick out.**
Around **1.7** pushes the ring clear outside it. It ships at **0.7**, which
solves the same collision the other way: the lone ring is tucked inside the
bonding lobe rather than fighting it for the same radius. All three are one
slider apart.

They do pass through the bond region on the way round, which a real lone pair
would not. The plain circle is the point, not the lobe geometry.

Speed, trail length, steering, lone orbit radius and speed, and a lone-pairs
on/off switch are all dev controls.

## What is in the world

```javascript
const SPECIES = { H2: { els: ['H','H'] }, O2: { els: ['O','O'] }, He: { els: ['He'] } };
const composition = { H2: 4, O2: 2, He: 0 };    // 0–6 of each, dev sliders
const ATOMIC_MASS = { H: 1, O: 16, He: 4 };
```

`spawnWorld()` builds a queue from `composition`, shuffles it so a crowded world
does not cram one species into the gaps the other left, then places each at a
random position, heading and spin — retrying up to 400 times for a clear spot.

### Spawn speed spread

Speeds come from a standard normal **truncated to ±1σ** (`spreadSpeed`), so the
fastest and slowest molecules sit one standard deviation either side of the
peak. Rejection, not clamping — clamping would pile molecules onto the two end
values, putting bumps exactly where the distribution should be thinnest.
`SPEED_SIGMA` = 0.25, so the bounds are the mean ±25%. Spin uses the same bell,
signed.

Note the bounds are ±1σ of the *generating* normal. Truncating at ±1 leaves the
sample with a realised sd of about 0.54σ, so the bounds sit ~1.84 realised sd
out. Bounds at exactly 1 realised sd is not achievable by any bell: for support
±h the sd cannot exceed h, and reaches it only when all the mass is on the two
endpoints.

**The spawn shape does not last.** Elastic collisions thermalise the world to a
right-skewed Maxwell–Boltzmann (Rayleigh, in 2D) distribution within about five
seconds even at 6+6, and it then holds steady — measured cv 0.13 at spawn, 0.48
at 5 s, 0.52 at 20 s and 60 s. That is the physically correct equilibrium and
the sim reaches it on its own; the Gaussian is only the initial condition.
Making the bell persist would need periodic velocity rescaling, which is not
physical.

`speedHistogram()` in the console prints mean, sd, cv, bounds and an ASCII
histogram per species — run it right after a spawn and again after a minute.

**Heavier molecules start slower** — `equipartition`, a dev checkbox, on by
default. `speciesSpeed` scales the dial by `sqrt(m_H2 / m)`, so every species
begins with the same kinetic energy, as a real gas mixture at one temperature
does; an O2 sets off at a quarter the speed of an H2. Off, every species starts
at the dial speed and an O2 carries 16x the energy of an H2.

## Temperature

A Low/High toggle on the toolbar. High runs at `TEMP_HIGH` = 3x the speed, which
is **9x the kinetic energy** — the energy readout says so, and the graph rescales
to fit the excursion.

Switching does not teleport the world to a new speed: `tempCurrent` chases
`tempTarget` with an exponential approach (`TEMP_TAU` = 0.26 s) and every
molecule is rescaled by the per-frame ratio, so you watch everything accelerate
over about a second. `stepTemperature` is fed **wall-clock** dt, so the ramp
takes the same time on screen wherever the playback-speed slider sits.

## Quality tiers and frame rate

```javascript
const TIER_FPS      = { 'hi-res': 60, 'detailed': 60, 'simple': 30 };
const TIER_SEGMENTS = { 'hi-res': 28, 'detailed': 16, 'simple':  8 };
```

Each tier sets both trail detail and a frame cap; `worldLoop` holds to
`frameBudgetMs` and the skipped time still counts towards the next step's dt, so
physics does not slow down with the frame rate.

`_runPerfBenchmark` drives the **real** `drawAtom` into an offscreen canvas —
24 atoms at hi-res detail, GPU-synced — and `_startupBenchmark` picks a tier from
it. `runPerfTest()` in the console re-measures and prints the timings next to the
thresholds they are compared against, without changing the current tier.

Measured at a full 6+6 world on a 2024 laptop: hi-res ~2.0 ms/frame, detailed
~0.7, simple ~0.45, against a 16.7 ms budget.

**The arrowhead glow is a cached sprite, not `shadowBlur`.** Canvas shadowBlur
cost 8.3 ms a frame here — 88% of the entire hi-res draw. `glowSprite` bakes a
radial gradient once per element and size and blits it; same look, 4.7x faster
overall. Do not reintroduce shadowBlur in a per-atom path.

A body stores its atoms once in **body-local** coordinates and derives world
positions each frame from `(x, y, angle)`, so a molecule can never stretch or
come apart. `bodyAtoms(b, dx, dy)` is the only way to get world positions.

## Collisions

Atom-against-atom, resolved as a 2D rigid-body impulse with rotation
(`collidePair`). The rotational terms are what make a glancing hit set a
molecule spinning instead of just shoving it sideways.

**Rotation is a full energy channel**, and always has been — measured, a
glancing hit puts 62% of the pair's kinetic energy into spin, an oblique one
57%, and a perfectly head-on one 0% (no torque arm). Nothing special was needed
to "add" it.

### Speed band — deliberately not pure

`rebalancePair` holds a molecule leaving a collision inside the same
±`SPEED_SIGMA` band the spawn uses, so nothing crawls or streaks. Dev toggle
(`clampSpeeds`), on by default.

Settled across **the pair**, not one body at a time: a slow molecule rarely has
spin of its own to sell, but the one it just hit is usually the fast one. The
pair's total energy is measured, translation is pulled into band, and the
remainder is what the two spins are set to carry — so the pair's energy is
conserved by construction.

`softBand` applies a **soft knee** rather than a clamp. The outer `KNEE` = 35%
of each half-band is compressed so speeds approach the limits asymptotically
and never reach them. A hard clamp parked 40% of the population on exactly the
floor speed; the knee cuts that to 9% while keeping the bounds exact.

Two places energy genuinely changes, both counted and reported by
`energySplit()`:
- **spin cap** (`MAX_SPIN` = 14 rad/s) — rare, a handful of events per minute
- **floor top-up** — both molecules slow with no spin left to sell, so the
  shortfall is created from nothing

Net effect, measured at 6+6 over 30 simulated minutes: speeds stay strictly
within 0.75–1.25 of nominal with 0 violations, rotation settles around 40% of
the energy, and total energy climbs to roughly 150–180% of spawn and then
**plateaus** — it oscillates there rather than running away, because the
ceiling and the spin cap shed what the floor adds.

A tighter `MAX_SPIN` does **not** help the floor (min stayed ~0.65 at every
value tried from 2 to 25) and bleeds energy badly — −46% at `MAX_SPIN` = 2.
Do not reach for it as a fix.

With the band on, the energy graph is no longer a conservation check; the note
under the percentage says so, and switching the toggle rewords it.

```javascript
const RESTITUTION = 1.0;   // perfectly elastic
```

Verified: a head-on hit reverses both velocities exactly and induces no spin; a
glancing hit deflects both and spins them; energy drift and linear-momentum
drift are both 0 to the precision measured, with 20 molecules over 8 s; atom
interpenetration never exceeded 0 px.

The loop sub-steps at 4 ms so fast molecules cannot tunnel through each other,
and clamps `dt` to 50 ms so a backgrounded tab does not resume with one
enormous step.

### Walls

`collideWalls` tests **every atom against all four walls**, and the contact is at
the atom, not at the molecule's centre — so the same rigid-body impulse runs as
for a molecule-molecule hit and a corner-on bounce sets the molecule spinning.
`WALL_RESTITUTION` = 1.0, so a molecule leaves with exactly the energy it
arrived with.

b.x moves as each contact resolves, so the atom's world position is recomputed
for every wall; a molecule in a corner touches two at once.

Verified: a square-on hit reverses velocity exactly (400 → −400 px/s) with zero
induced spin; an off-centre hit leaves it spinning at 3.15 rad/s with 2.9% of
its energy in rotation. **Energy drift 0 and wall overlap 0 px over 90
simulated seconds at 6+6, with nothing ever escaping.** Linear momentum is
*not* conserved, and should not be — the walls are external.

Wall bounces are settled by `settleBodySpeed`, a single-body version of the
speed band: a wall cannot take the surplus, so it is paid to and from that
body's own spin. Measured with the band on, 90 s at 6+6: speeds stayed strictly
within 0.75–1.25 of nominal, **0 violations in 5400 samples**, 143 floor
top-ups, energy plateauing at 148% of spawn.

`drawWalls` strokes a faint rectangle at the boundary. At zoom 1 the box lines
up with the toolbar, the energy bar and the screen edges so it is barely
visible — but zoomed out there is canvas beyond it, and without the line the
molecules look like they bounce off nothing.

## Energy in fields

The "Show energy in fields" checkbox draws one **lobe between each pair of
bonded atoms** — zero at each nucleus, brightest at the midpoint. The single
idea it has to carry: a bond is two nuclei both pulled toward the space
between them, because that is where the shared electrons spend their time.

**It is a picture, not a calculation.** There are no partial charges, no test
probe and no electron cloud, and it deliberately does not feed anything. When
the energy bar charts arrive they do their own arithmetic — the two are the
same idea, not the same numbers, and that is on purpose.

```javascript
const BOND_ENERGY = { 'H-H': 436, 'H-O': 463, 'O-O': 498 };  // kJ/mol
let FIELD_WIDTH = 0.30;   // lobe half-width as a fraction of bond length
let FIELD_GAIN  = 1.8;    // value that maps to the top of the colormap
```

Brightness scales with real bond dissociation energy, so O=O sits a little
above H–H. **Break mode should read `BOND_ENERGY` too** — it is the number that
decides what an impact is strong enough to pull apart, and having the glow and
the threshold disagree would be worse than either being wrong.

Each body carries `bonds` as index pairs. Atoms are laid out in a row and each
bonds to the previous one, so a diatomic is `[[0,1]]` and water would be
`[[0,1],[1,2]]` — two lobes off the oxygen, no special casing.

### The flare

Molecules are perfectly rigid, so a bond at rest never changes and the lobe
would be wallpaper. Instead a collision lights it: `b.flare` is set from the
impulse (`jImp / (mass × vTarget)`, which is ≈1 for a square hit at the
species' nominal speed) and decays with `FLARE_TAU` = 0.30 s. `FIELD_GAIN` is
set so resting bonds sit mid-colormap and leave the top of the range for
flares. This is the only motion in the view until molecules can vibrate.

`FIELD_WIDTH`, `FIELD_GAIN` and `FLARE_GAIN` are dev sliders — they were dialled
in against the picture, so they stay adjustable.

### Cost

Measured at a full 6+6 world: **0.43 ms/frame at hi-res**, 0.23 detailed, 0.14
simple, against a 16.7 ms budget. `fieldTest()` in the console re-measures.

Nothing is cached. The old fission heatmap kept a composite behind
`heatmapDirty`, which worked because a nucleus sat still; here every molecule
moves every frame, so the cache could never hit and it was removed.

The cost only works because it draws **bonded pairs only** — 12 lobes at a full
world, each covering a box about a bond long. The fission version drew a tube
between *every* charged pair, which here would be 276 pairs over a 57k-pixel
grid, about 15M iterations a frame. If you ever add intermolecular lobes, cull
them hard by distance or that number comes back.

## Energy graph

**Hidden by default** — SHOW LINE GRAPH on the energy bar brings it back,
along with the key and the percentage readout, which share its `.visible`
classes.

Showing or hiding it changes the energy bar's height, which moves the bottom
wall, so `setGraphShown` calls `restretchWorld()` — the same re-fit the window
resize uses. Without it the world keeps its old box and leaves a gap between
the bottom wall and the bar.

A scrolling 12-second trace of total kinetic energy (translational +
rotational), full scale = 2× the energy at spawn, so **a flat line across the
middle is the correct result**. A visible step means energy leaked — it is a
check on the collision solver, not a teaching graph.

## What is deliberately left standing

Dormant, kept because the next round of work wants it. None of it is live:

- **`#speedDialWrap.bounce-warn`** — the red flash on the dial, commented out in
  the stylesheet. It was fission's "this neutron is too slow" warning; break
  mode wants the same signal for "too slow to break a bond".
- **`z` on atoms** is always 0 but `drawAtoms` already sorts by it, so a molecule
  that is not a flat row has a draw order waiting.
- **`formulaOf`** does full Hill ordering, so it already prints H₂O correctly.
- **`world.bodies[].id`** is stable per body — the handle for selecting one
  molecule and charting it alone.

## Known gaps

- The world band excludes the full-width strip above the toolbar, even on the
  right where the toolbar panel does not reach. Simple, but it wastes screen.
- Composition lives in the dev panel, not the toolbar.
- The temperature ramp runs even while paused, so velocities change with nothing
  moving. Harmless, but the pause is not a total freeze.
- Molecules cannot be dragged, clicked or selected.
- No reactions. Molecules bounce; they never break or combine.
- No internal vibration — a molecule is perfectly rigid, so a collision has only
  translation and rotation to put energy into.
- Only two counter slots in the toolbar (`sp1`/`sp2`), so with H₂, O₂ and He all
  present one of them is not shown. Three species now exist, so this one is live.
- `makeBody` lays atoms in a straight row, so an H₂O would come out linear
  rather than bent at 104.5°. The electron structure is already right (4
  bonding, 2 lone pairs) — only the geometry is missing.

## Deployment

`origin` is **`github.com/ghostoutfit/reactions`**. (The fission remote was
removed when this repo was copied, so nothing here can reach fission.)

GitHub Pages is **not enabled** on that repo — the API 404s — so there is no
`ghostoutfit.github.io/reactions/` URL yet. Open `index.html` locally, or serve
the directory, until it is switched on.

This sim is deliberately **not** on the SSP site. It was briefly built to
`sim.ssp.org/reactions/` and that was removed: reactions lives only in
ghostoutfit.
