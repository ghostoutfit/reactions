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

**On by default.** `eRound` 1 makes the figure-8's own lobes circular too, so
the two look close — but only this gives each H a ring genuinely of its own, one
closed curve per nucleus, rather than two lobes of one curve meeting at the bond
midpoint. That is the picture an O₂'s lone pairs already show.

```javascript
let bondCircles = true;   // off: the figure-8
let bondCircleR = 0.85;   // ring radius, multiple of the atom's valence radius
```

They overlap because a bond is shorter than two valence radii: H–H sits
`2 × 30 × BOND_OVERLAP` = **43.2 px** apart while each ring is `30 × 0.85` =
**25.5 px**, so the rings cross and share a **7.8 px** lens over the bond. They
stop overlapping at all below `bondCircleR` = 0.72, where the ring radius drops
under half the bond. That lens is the picture — it is where both electrons are, which is what the
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
let loneSpeedFree = 6.0;  // x loneSpeed for an atom with no bonds at all
```

**A free atom's electrons race.** Nothing is bonded and nothing is paired into a
shared lobe, and the speed is the tell: 7.16 laps/s against a bonded atom's
1.19. It is the one thing that says "this came apart" from across the canvas,
after the trails have faded and the halves have drifted apart. Applies to
anything with no bonds, so a He reads the same way.

**Free atoms are spaced by the golden angle, not evenly.** An even ring of n
repeats every `2π/n`, so once the per-frame advance approaches that spacing the
ring appears to stall or crawl backwards — the wagon-wheel effect. An O has six
lone electrons, spacing 1.047 rad, so it bites at an advance of only ~0.52.
Measured at 6×: the ring genuinely turned at 7.16 laps/s and read as going
**backwards at 0.30 rad/frame**. The golden angle has no k-fold symmetry at all,
so there is nothing to alias against and the limit becomes half a lap per frame
— `π` against the 0.75 rad/frame that 6× actually uses, and 1.5 at the top of
the slider, so the whole range is safe. It still spreads six points well (gaps
32.5/52.5/85/52.5/85/52.5°, nothing clumped), so the ring stays a ring, just not
a symmetric one. Bonded atoms keep the even spacing: they are slow, they never
alias, and the even ring is the cleaner picture.

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

### Bond breaking

Collisions pump energy into a molecule's bond, the bond holds it for a while,
and when it holds more than the bond is worth the molecule comes apart.
**One-way, for now**: the halves drift off as free atoms and nothing puts them
back, so a hot world gradually comes apart until there is nothing left to break.

```javascript
let breakBonds  = true;    // dev toggle
let breakEnergy = 26;      // vibration a bond can hold before it lets go
let VIB_TAU     = 2.2;     // seconds — how long a bond holds what it was given
let breakKick   = 0.22;    // separation speed given to the halves, x nominal
let VIB_GLOW    = 0.7;     // how brightly a loaded bond reads, vs a full flare
```

#### Why it is energy, and not the hardness of one hit

The first version broke a bond when a single collision exceeded a threshold,
with hardness measured the way the flare measures it — `jImp / (mass x speed)`.
That had two faults, both found by measuring rather than by reasoning:

1. **It threw away every hit but the biggest.** In 600 s an O₂ took **190 hits
   from free H atoms** and not one of them counted for anything.
2. **A light atom could not satisfy it at all.** The mass in the denominator is
   the *target's*, so an H (mass 1) hitting an O₂ (mass 32) peaked at **1.50**
   across those 190 hits, against a threshold of 4.0. Only another O₂ could
   ever break an O₂ — and since one O₂-on-O₂ collision is hard enough to break
   *both*, a world either lost both at once or kept two immortal O₂ forever.
   A part-broken world was a dead end.

The honest quantity is the kinetic energy in the centre-of-mass frame, which is
what a collision actually has to deposit:

```
E = ½ · μ · vn²        μ = 1 / invMassSum, the contact's effective mass
```

`invMassSum` already exists in `collidePair` and includes the rotational terms,
so a glancing hit that mostly spins the pair deposits correspondingly less.

That alone fixes (2), because **equipartition means a light atom moves fast
enough to carry as much energy as a heavy one**. Measured at High, dose per hit:

| hit on | by | n | mean | max |
|---|---|---|---|---|
| O₂ | H₂ | 436 | 2.46 | 12.84 |
| O₂ | O₂ | 32 | 3.24 | 16.36 |
| H₂ | H₂ | 692 | 3.17 | 15.95 |

An H₂ now lands 76% of an O₂'s mean dose on an O₂, where before it managed a
fortieth of what was needed. **A fast H breaks an O₂ because it genuinely has
the energy to**, not because anything was special-cased for it.

One dose unit is the kinetic energy of a nominal H₂ at temperature 1, so the
speed dial cancels out and the temperature does not — deliberately, the same
way the old measure worked. Energy goes as speed squared, so High is ~9× the
dose per hit and ~3× as many hits: **~27× the pumping rate for 3× the speed**.
That is the entire separation between Low and High, and it is why no
`tempLevel` test appears anywhere in the mechanism.

#### Energy in fields is its own bucket

`vib` is **not** part of `energyOfMotion()`, and that is deliberate. Real
vibrational energy sloshes between kinetic (the atoms actually moving) and
potential (the bond stretched), averaging half and half. These molecules are
**rigid**, so there is no kinetic half to account for and all of it sits in the
bond. The rigidity and the accounting agree — if real vibration is ever added,
this is the split that has to be revisited.

```javascript
function energyInFields()    // sum of b.vib, scaled back into motion's units
```

Both buckets are graphed (`energyTrace` carries `{ t, motion, fields }`, which
is what its original comment was holding room for), both are in the percentage
readout, and `energySplit()` breaks them down with the leak alongside.

#### Collisions pay for the heat

```javascript
let fieldConvert = 0.45;   // fraction of a hit that goes into the bonds
```

Before this, a bond could glow while both molecules bounced off at full speed —
energy appearing from nowhere, and the exact misconception the sim otherwise
teaches against. Measured then: vibration rose by 11.10, 4.09, 0.79, 0.70 on
four separate frames with **ΔKE = 0 every time**.

**It is applied as restitution, which is what makes it exact rather than
approximate.** A collision with restitution `e` loses `(1 - e²)` of the normal
centre-of-mass energy, so `e = sqrt(1 - fieldConvert)` removes *precisely* what
is handed to the bonds. Verified with the speed band off and the decay disabled,
so nothing could mask an error: **total drift 0% at conversions of 0, 0.45 and
0.9.** Not approximately — the same digits.

The energy is split between the bonds that can hold it. **A free atom has no
bond to put it in**, so a hit on one stays perfectly elastic: with nobody to
take it, converting would just delete it.

**`fieldConvert` 0 means nothing ever breaks.** The energy entering the bond and
the energy leaving the motion are the same quantity, so switching off the cost
switches off the heating. That coupling is the point, and the note under the
slider says so.

#### Not sucking the world dry

Converting every hit drains the world. With the speed band off and the decay
disabled, motion fell to **4.9%** of spawn in 120 s at conversion 0.45 — a
world that has stopped moving cannot break anything, and the mechanism would
have strangled itself.

Two things prevent it, and both have to stay: `VIB_TAU` drains the bonds, and
the speed band tops molecules back up off their own spin. At the shipped
defaults, measured over 240 s at High across three seeds, **motion never fell
below 49%** (49/56/59) — there is always plenty left to break with.

`breakEnergy` was **re-tuned from 26 to 6** for this. Each bond now gets
`fieldConvert / storers` of a hit rather than the whole thing — about 0.225× —
so the old capacity put breaking out of reach entirely: 0 breaks in 180 s at
High. Swept again at conversion 0.45, 3 seeds at High and 2 at Low:

| capacity | High: broke | first | last | Low: breaks |
|---|---|---|---|---|
| 3 | 6/6/6 | 0.2–3.7 s | 6.5–8.6 s | 0/0 |
| 4.5 | 6/6/6 | 1.5–9.7 s | 9.8–18.6 s | 0/0 |
| **6** | **6/6/6** | **3.8–31.6 s** | **14.6–38.7 s** | **0/0** |
| 8 | 6/6/6 | 69.7–107.7 s | 82.6–123.5 s | 0/0 |

#### Making the loss visible

The conversion is real, but it was **invisible on screen**, and the reason is
subtle enough to be worth writing down: the speed band is not creating energy
when it restores a molecule. It **sells the pair's spin** to hold their linear
speed inside the band. So the loss lands on rotation, which is far harder to see
than speed.

Measured over 507 collisions: the impulse alone took **−13.5%** of the pair's
linear speed, and after the band it was **+0.1%**. On the hardest third,
**−37.6% became −2.4%**. One hit went 1122 → 531 → 813.

Tying the band's floor to the bond's accumulated load did **not** fix it — a
typical hit leaves `vibLoad` near 0.19, which moves the floor about 3%. What has
to drive it is the energy *that collision* took.

```javascript
let vibDrag = 0.6;        // how far the band lets a molecule sag
const SAG_TAU = 0.8;      // seconds for the slowdown to fade
```

`b.sag` is the share of a body's own kinetic energy that just went into its
bond. `bandTarget()` lowers the band around it by `sqrt(1 - vibDrag · sag)` —
energy goes as v², hence the root — so the molecule visibly slows on impact and
winds back up over about a second. On the hardest third of hits:

| `vibDrag` | after the band |
|---|---|
| 0 | −2.2% |
| **0.6** | **−8.1%** |
| 0.9 | −17.5% |

It does not starve the breaking: still 6/6/6 at High and 0/0 at Low over 900 s,
motion floor 41–57%. Capacity was re-checked at 4.5, 5 and 6 and all three break
6/6/6/6 — the spread in *when* the first break lands (1.2 s to 123 s) is seed
noise, not capacity, so 6 stands. Once the first bond goes the rest follow
within 5–20 s, because free atoms are fast and keep hitting.

#### Breaking is endothermic

```javascript
let breakSlow = 0.5;   // a fragment's nominal speed, x what equipartition gives
```

Breaking a bond **absorbs** energy — 436 kJ/mol for an H–H — so a dissociated
gas is colder than the one it came from. The sim used to say the opposite.
Equipartition gives a fragment `sqrt(2) ×` its parent's nominal speed because it
has half the mass, and `breakKick` added more on top, so **a fully broken world
ran 1.45× faster** than the intact one (motion energy 2.12×). Exactly backwards.

**It has to change the nominal speed, not just scale the velocities**, because
the speed band would pull a slowed fragment straight back up to whatever
`vTarget` says. `breakBody` now scales both: the nominal, so the band holds it
there, and the inherited velocity, so the drop lands at the instant the bond
goes. 0.5 against equipartition's `sqrt(2)` leaves a free atom at **0.71 ×** its
parent's nominal — the same proportion for H and for O, so the world slows
together rather than changing shape.

Measured end to end at the shipped defaults:

| | mean speed before | after | |
|---|---|---|---|
| H₂ → H | 506 | **343** | −32% |
| O₂ → O | 158 | **97** | −39% |

with total motion ending at **0.43×** and the dissociation cost — the kinetic
energy absorbed plus the vibration spent — counted in `breakCostPaid` at 73% of
the spawn energy.

**This trades directly against being able to break anything**, and the trade is
sharp. Slower fragments pump less energy, and the last O₂ is the bond that
depends on them:

| `breakSlow` | High: broke | speed after |
|---|---|---|
| 1.0 | 6/6/**0** | 1.45× (faster!) |
| 0.7 | 6/6/6 | 0.84–0.97× (barely) |
| 0.5 | 5/5/6 | 0.70× |
| 0.35 | 5/3/**0** | — never finishes |

At 0.5 the last bond often never broke. **The fix is that the world starts
faster, not that bonds are made flimsy.**

The speed dial's default went from 4 to **8** (of a ceiling raised 8 → 12, so
the default is not pinned to the end of the slider), and `breakEnergy` sits at
**4.5**.

Why the dial helps at all is worth stating, because the obvious reasoning says
it should not. The dose a hit deposits **is** scale-free in the dial — that is
deliberate, so the threshold means the same at any setting. But the collision
**rate** is not, and `VIB_TAU` is fixed in seconds, so the pumping rate
(dose × hits per second) scales with speed. Measured at capacity 3.5:

| dial | High: broke | last break |
|---|---|---|
| 2 | 3/6/4 | 132–259 s |
| 4 | 6/6/6 | 24–85 s |
| 8 | 6/6/6 | 34–45 s |

Raising the dial does **not** buy sturdier bonds, though — at dial 6–8,
capacities of 8 and 11 broke **nothing at all** (0/0/0). `vibDrag` is a negative
feedback: a molecule that just gave energy to its bond runs slower, so it pumps
less. Capacity has to stay in the 3.5–5.5 band regardless of how fast the world
starts.

Shipped at dial 8 / capacity **2.5**: **6/6/6 at High, 0/0/0 at Low** over
3 × 900 s, everything broken by 10–17 s. H₂ ~1100 → H ~**730** and O₂ ~280 → O
~**176**, both about −35%, total motion ending at **0.41–0.47×**. Capacity was
swept at 1.0, 1.5, 2.0 and 3.0 as well — all break 6/6/6 with Low clean, taking
2.7–5.8 s, 2.3–10.2 s, 4.2–13.1 s and 11.4–23.7 s respectively, so there is
plenty of room either way.

All of these are dev sliders, and they interact: making fragments slower without
either lowering capacity or starting faster leaves bonds that nothing can
break.

#### Where the energy actually goes

The deposit is now exact. **The decay is still a leak** — `exp(-dt / VIB_TAU)`
drops energy out of the world rather than passing it anywhere, and at the
shipped defaults that is **88–214% of the spawn energy over 240 s at High**.
It is no longer silent: `fieldsLeaked` counts it and `energySplit()` prints it,
the way `energyCreated` already counts what the speed band invents.

Physically it should leave as infrared, or to other molecules in later
collisions. Routing it back is the remaining piece, and it needs the care the
speed band got, because feeding it back is exactly how a world runs away.

#### Holding and cooling

`b.vib` accumulates and decays as `exp(-dt / VIB_TAU)`. **The decay is what
keeps Low safe** — without it every molecule breaks eventually however cold the
world is, since vibration would only ever go up. Vibration is pumped whether or
not `breakBonds` is on, so the toggle only decides whether a full bond lets go;
the heat still shows, and the diagnostics can measure what the world *would*
have done without changing it.

A single hit big enough to clear the threshold on its own still breaks
immediately. Accumulation does not replace that case, it contains it.

#### The colour scale is a fraction of the threshold

```javascript
let breakEnergy = 2.5;     // vibration a bond can hold before it lets go
let colourFull  = 0.8;     // full pink at this FRACTION of capacity
```

These used to be one number, so **every change to how easily bonds break
silently rescaled the colours**. Splitting them is right, but an *absolute*
colour scale is not: set it above the threshold and bonds break before the ramp
runs out, below it and they sit pink for ages. Measured with an absolute scale
of 4.5 against a threshold of 2.5, bonds broke at a mean colour load of 0.68 and
the rendered frames only reached 0.44 mean / 0.55 max — the top of the ramp was
a colour you never saw.

As a **fraction of capacity**, the full white → purple → pink range survives any
retuning of breaking, which is the point.

**`colourFull` is under 1 on purpose.** At exactly 1 the ramp reaches pink only
in the same frame the bond breaks — the dose that saturates the colour is the
dose that breaks it — so the pink is never actually drawn. Measured over 3 runs
at High, 120 s each:

| `colourFull` | bonds that reach pink | white | purple | pink |
|---|---|---|---|---|
| 1.0 | 67% | 34.6% | 56.7% | 8.7% |
| **0.8** | **94%** | **26.0%** | **43.5%** | **30.4%** |
| 0.6 | 89% | 41.1% | 31.6% | 27.3% |
| 0.45 | 94% | 39.9% | 13.1% | 47.0% |

0.8 uses the whole ramp most evenly. Below it purple gets squeezed — at 0.45 a
bond mostly jumps from white to pink with barely any purple in between.

`colourFull` is **visual only**; it does not appear in `breakThreshold`. Both it
and the threshold are scaled by the bond's own energy the same way, so an O=O
still sits 498/436 above an H–H on both axes.

#### Choosing the capacity

`breakEnergy` is the vibration an O=O can hold; an H–H holds `436/498` of it,
**22.8 against 26**. Swept against measured behaviour, 3–4 seeds each:

| capacity | High: broke | first break | last break | Low: breaks |
|---|---|---|---|---|
| 18 | 6/6/6 | 3.3 s | 8–15 s | 0/0/0 |
| 22 | 6/6/6/6 | 6–17 s | 18–26 s | — |
| **26** | **6/6/6/6** | **12–47 s** | **40–64 s** | **0/0/0** |
| 30 | 6/6/6/6 | 2.6–32 s | 19–54 s | — |
| 44 | 0/6/0 | — | — | 0/0/0 |

26 gives the longest arc — something to watch rather than a burst — while still
finishing reliably. 44 is past the edge: two runs in three broke nothing at all.

**Low is clean.** Over 3 × 900 s (45 simulated minutes) at Low: **0 breaks**,
peak vibration **11.8** against H₂'s capacity of 22.8, a 1.9× margin on the
worst moment in all of it. Live at High: all six molecules gone by **28.1 s**,
both O₂ included, no console errors, every body finite.

#### What it looks like

A loaded bond glows, so a molecule about to go is visibly holding energy rather
than just flashing when hit — `heat = max(flare, VIB_GLOW · vibLoad)`, with
vibration deliberately reading below a full flare so a collision is still the
brightest thing that happens to a bond. Measured over the load range, in a box
around one O₂'s bond: white-core pixels **219 → 810 → 1146 → 1335 → 1440**, and
total ink +34%. The core *spreads* rather than the lobe simply brightening,
which is the same effect the flare relies on and for the same reason.

#### Fragments

Each keeps the velocity its atom actually had — the body's velocity plus the
spin's contribution at that offset, via `pointVel` — so a molecule that was
spinning fast throws its halves apart harder, with none of it invented. On top
sits `breakKick` along the outward direction, standing in for the bond energy
released. A free atom gets `spin = 0`: there is nothing for it to spin about.

The electron structure falls out for free: `buildElectrons` on a one-atom body
with no bonds makes every valence electron lone, so a free H shows **1** and a
free O shows **6**. Both are radicals, and both look like it.

**Breaks are applied after the collision pass, never inside it** —
`world.breaking` queues them and `stepDissociation` splices. Replacing a body
mid-pass would shift `world.bodies` indices under the loop and let a pair be
missed or tested twice.

#### runBreakTests()

`runBreakTests(seconds)` in the console reproduces all of it headlessly —
`stepWorld` is driven directly, so it does not wait on real time or on the GO
button — and puts the world back as it found it. Three tables: dose per hit
**split by who was hit and by what**, the vibration that adds up to against each
bond's capacity, and what that does to the world.

The by-partner split is the one that matters. The old measure looked perfectly
healthy in aggregate while being impossible for a light atom to satisfy, and
nothing but a partner breakdown would have shown it.

**A trap worth remembering:** `spawnWorld` already multiplies by `tempCurrent`
through `speciesSpeed`, so setting the temperature *after* spawning and then
scaling velocities applies it twice. An early version of this diagnostic did
exactly that and reported vibration ~50× too high at High. Set the temperature
first, then spawn.

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

### The colour ramp is two-dimensional

| axis | what it is |
|---|---|
| `t` | where a pixel sits on the lobe — top is the bond's core, bottom its outer fringe. A resting bond already peaks at `t ≈ 1`. |
| `load` | how much energy **that bond** is holding, 0–1: `max(flare, vibLoad(b))`. |

**`load` is the axis the "Energy in fields" key describes.** The key is not a
gradient across one lobe — it is the sequence a single bond moves through over
time as it takes hits: whitish at rest, purple once it has been worked on, pink
near its capacity. Low and High are a cold bond and a hot one, not the fringe
and the core. Getting that backwards was the first attempt, and it put magenta
on every resting bond.

Three stage ramps in `FIELD_STAGES.energy`, interpolated into `LOAD_BINS` = 17
stacked 256-entry ramps at `setFieldRamp` time, so the per-pixel cost stays one
lookup. Measured core colours:

| load | core |
|---|---|
| 0.00 | (210,238,255) whitish blue — **the original, unchanged** |
| 0.25 | (206,212,255) still whitish |
| 0.50 | (200,161,255) lavender |
| 0.75 | (211,114,242) purple |
| 1.00 | (255,92,208) pink |

`FIELD_STAGE_GAMMA` (1.6, dev slider) holds the cold stage longer, so a bond has
to be properly worked on before it stops looking like an ordinary bond — the
separation between white and purple, rather than a smooth wash across the whole
range. `blue` in the dropdown is the original with no colour response at all.

**Per-bond colour needed a second buffer.** `fieldAcc` sums every bond's lobe
into one accumulator, which cannot say whose energy a pixel is carrying.
`fieldLoad` accumulates the same contributions weighted by each bond's load, and
`fieldLoad[i] / fieldAcc[i]` is the contribution-weighted load at that pixel —
so where two bonds overlap, the colour follows whichever is putting more field
there. Cost after the change: **0.29 ms/frame at hi-res**, 1.8% of budget, no
worse than the 0.43 ms measured before it.

**Two rules hold across every stage.** The alpha column is identical in all of
them — that falloff is the glow, which fades the fringe to nothing so the lobe
has no edge. And the fringe stays **darker than the page**: the lobe is culled
flat at each nucleus (`along < 0 || along > L`) and at a box a couple of sigma
out, and those straight edges are invisible only while the fringe is darker than
what is behind it. A pale fringe measured (232,232,243) at alpha 22 and turned
every bond into a visible grey rectangle. The key's "whitish" belongs at the
**core of a cold bond**, which is where it already was.

**A pre-existing artifact worth knowing:** the lobe reaches further
perpendicular to the bond (2.5σ ≈ 41 px on an O=O) than along it (≈ 27 px), and
is truncated flat at the nuclei, where the `sin` profile climbs from 0 to alpha
130 inside about one low-resolution buffer pixel. Axis-aligned and at rest it
reads as a hard-edged vertical bar. At natural angles it is hard to see, and it
is identical under `blue`.

### Cost

Measured at a full 6+6 world: **0.29 ms/frame at hi-res**, 0.23 detailed, 0.14
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

A scrolling 12-second trace of **two** series, full scale = 2× the energy at
spawn (growing to fit, since switching to High is a 9× jump):

- **Energy of motion**, red — translational + rotational.
- **Energy in fields**, purple — what the bonds are holding. Its sawtooth is the
  mechanism in one picture: sharp rises as collisions deposit, slow falls as the
  vibration decays.

With the speed band off and bonds unable to break, motion alone is flat and a
visible step means the collision solver leaked. With collisions converting into
bonds, the two **trade**, and it is their sum that should hold — verified at 0%
drift. The band and the vibration decay then both break that sum, deliberately
and with both counted; see Bond breaking.

## What is deliberately left standing

Dormant, kept because the next round of work wants it. None of it is live:

- **`#speedDialWrap.bounce-warn`** — the red flash on the dial, commented out in
  the stylesheet. It was fission's "this neutron is too slow" warning; breaking
  now exists and could use exactly that signal for "too slow to break a bond".
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
- Molecules break but never **re-combine**, by design for now — so a hot world
  only ever runs down to free atoms. Recombination is the other half, and the
  natural next piece: free H and free O are now produced in quantity and have
  nothing to do with themselves.
- Internal vibration exists only as a scalar energy store (`b.vib`) that decides
  when a bond breaks and how it glows. The molecule is still perfectly **rigid**
  — nothing actually oscillates, so the atoms never move relative to each other.
- Only two counter slots in the toolbar (`sp1`/`sp2`). Breaking makes this
  worse, not better: a part-broken world holds up to **four** species (H₂, O₂,
  H, O) with two slots to show them in. `updateSpeciesCounters` now sorts by
  count so the two slots go to the two largest populations rather than to
  whichever sat earliest in `world.bodies`, but it is still lossy.
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
