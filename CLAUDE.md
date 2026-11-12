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
let eWidth = 2.40;   // 1 = bare lemniscate; 1.41 makes a lobe exactly round
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
0.10**, which is a lot of lag, and the lag does what it always does on a curved
path: it cuts corners inward. Measured on an O=O, mean distance from the owning
nucleus is **25.7 px at 0.10 against 33.5 at 1.0**, nominal valence radius 38.1
— so the lobes run about 23% tighter than the ideal curve and wander more. That
is the look, not a fault; it is the same mechanism that made circle mode
collapse, used deliberately on a path that can absorb it.

On top of that lag, three wobbles are re-rolled at **every lobe**:

```javascript
let eRadial = 0.26;   // +-: how far the lobe reaches past the nucleus
let eWander = 0.80;   // +-: how wide the lobe is
let eTilt   = 0.54;   // +- radians: the whole lobe swings off the bond axis
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

### A bonded atom's lone pairs ride closer in

```javascript
let lonePairsOnBonded = true;   // dev toggle; false drops them entirely
let loneRadiusBonded  = 0.6;    // x loneRadius, for an atom that has bonds
```

At the full radius an O₂'s four lone pairs sat right where the bonding lobes
are, and the molecule read as one tangle of rings — four rings, only two of
which say anything about the bond. Pulled in to **60%**, they are clearly a
second, inner shell and the bond stays the thing you look at. Measured: 20.6 px
against the 34.3 px they would otherwise sit at.

**Free atoms are unaffected**, which matters: nothing is competing for the space
out there, and the full shell is what makes a broken-off O read as a radical
carrying six. `e.bonded` is settled once in `buildElectrons` rather than
searched for per electron per frame. Verified: O₂ 4 bonding + 8 lone at 20.6 px,
free O 6 electrons at the full radius.

`lonePairsOnBonded` false drops them from bonded atoms altogether — which is a
real chemical omission, since H₂O's bent shape is usually explained by the two
on its oxygen. `buildElectrons` re-runs on every body when either changes.

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
let bondCircles = false;  // on: a ring per atom, instead of the figure-8
let bondCircleR = 0.80;   // ring radius, multiple of the atom's valence radius
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
let loneRadius = 0.9;     // multiple of the atom's valence radius
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

**Three levels, and no speed dial.** `TEMP_LEVELS` is `{ low: 1, med: 2,
high: 3 }`, and since energy goes as speed squared that is **1× / 4× / 9×** the
kinetic energy. The launch-speed slider is gone — temperature is the only speed
control now, which is the honest one, since a gas has no speed knob separate
from how hot it is. `launchSpeed()` returns the constant `LAUNCH_SPEED` = 360,
what the dial shipped at. The buttons sit where the dial was, three of equal
width so switching never resizes the group or shifts the counters beneath it.

**The sim opens at High.** At Low nothing breaks at all — that is the measured,
deliberate design of the threshold, not a shortfall — so a world that opened at
Low opened on the one setting where the main event never happens. Verified on a
cold load: `tempCurrent` 3, only the HIGH button active, H₂ spawning at 1126
against Low's ~375. Switching down and back up still ramps (836 → 209 → climbing
again), since `tempCurrent` chases `tempTarget` rather than jumping.

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

#### Three view modes

`viewMode` is `'trails'`, `'shells'` or `'space'` — the old boolean `showAtoms`
split in two, because "Atoms" was doing two jobs. **It opens on `'shells'`**:
that is the picture that names the parts, and the trails are the one to switch
to once you know what you are looking at.

All three buttons carry the **same** H₂ icon, animated by the shared
`ICON_CANVASES` loop. They are three pictures of the same thing, so the button
should say which *view* you get rather than imply the molecule differs between
them. Each canvas keeps its own jitter so they do not wobble in lockstep. The
bespoke `drawOutlineIcon` that used to sit on the third button is gone.

- **Trails** — the electron trails this sim has always drawn.
- **Shells** — the bonds sim's shells picture: a gradient band at each occupied
  shell radius, electrons as **dots sitting on it** rather than as paths behind
  them. Shares `drawShellBands`, `drawElectronDot`, `ATOM_SHELLS` and
  `shellRadius` with the One Bond tab.
- **Space** — space-filling, unchanged but for the name.

**Both views share one geometry.** `shellRadius` places the band *and* the
electrons on it, in both tabs, so a shell is in the same place relative to its
atom wherever you look. Reactions used to place its shells-view electrons at
`loneRadius x loneRadiusBonded` instead — **0.54 of its own band, 17.5 px adrift
on an oxygen**, the electron floating inside the ring it was drawn with. Those
two constants are *trails-view* tuning: 0.6 exists so a lone-pair ring does not
collide with the bonding lobes in that picture. There are no lobes in the shells
view, so applying it there only broke the agreement. Now both read 1.00.

**Colour is by element in both tabs**, from `ELEMENTS` — H the pale grey, O the
red. The One Bond tab briefly coloured its two atoms apart, blue left and amber
right, so you could tell whose electrons were whose as they separated. That
reads well on an O-O and is a lie everywhere else: the same oxygen was a
different colour depending on which side it sat, and nothing else in the sim
speaks that language. On an O-O you now tell them apart by watching which way
they go.

**`shellRadius` takes the nucleus radius.** The One Bond tab draws nuclei about
9x life and needs its inner shells shoved clear of them; the main tab's nucleus
is the plain 6 px sprite and needs no such clearance, so it passes its own and
the inner shell stays where the physics puts it.

Dots are `SHELL_DOT_R` = 9 px against the One Bond tab's 17 — a world of
molecules, not one bond filling the screen.

### Shells electrons sit still, and outward

```javascript
const shellSpin     = 0;      // rad/s round the ring — none
const shellBondPull = 0.72;   // bonding electrons, towards the bond's middle
let   loneArcFrac   = 0.45;   // share of the ring a bonded atom's lone pairs use
```

**A shell is a place an electron lives, not a racetrack.** The trails view needs
motion — a stationary electron leaves no trail — but in the shells view that
same motion reads as electrons whizzing round the ring, which says something
false about what a shell is.

So the shells view draws lone electrons from **`e.angle0`**, the seat each one
was given at build time, turned only as fast as `shellSpin`. The simulation
keeps running underneath for the trails view; shells just ignores it. Measured
over 10 s: the drawn electron moved **10.3 px** while the simulated orbit angle
advanced **75 radians**, about twelve laps.

The only thing that should move them is the molecule itself rotating, and that
still does, because these are body-local coordinates.

**A bonded atom's non-bonding electrons sit on the far side from the bond**,
spread over `loneArcFrac` of the ring centred on the direction away from its
partner. Evenly round the whole circle they swept straight through the bond
region, which is where the bonding pair already is and the one place a lone pair
should not be. Measured on an O in an O=O, the cosine of each electron's angle
against that away direction — +1 is directly outward, −1 is through the bond:

| | mean | worst | all outward? |
|---|---|---|---|
| whole circle | **−0.25** | −1.00 | no |
| arc 0.55 (198°) | +0.34 | −0.16 | no |
| **arc 0.45 (162°)** | **+0.53** | **+0.16** | **yes** |

Under 0.5 on purpose: at 0.55 the arc spans 198°, so the outermost pair sits at
±99° and tips just past the halfway line onto the bond's side. At 0.45 they land
at **81°, 27°, 27°, 81°** from the away direction — every one of them strictly
outward.

**A free atom keeps the whole circle**, golden-angle spaced: nothing is pushing
its electrons anywhere, so they have no reason to bunch. Verified it still
spans the full range after a bond breaks.

Ring spin and the bond-pull slider are gone — spin is fixed at none and the pull
at its tuned value. `Lone pairs outward` replaces them, and rebuilds every
body's electrons when moved.

**Bonding electrons are SEATED beside the bond's midpoint**, perpendicular to
the axis, alternating sides and tiered outwards so a double bond's four do not
stack. Not the figure-8 position: that is the trails view's business, and drawn
as a dot it made the bonding electrons look like they were perpetually swinging
round the molecule. A shared pair sits in the gap.

Pulling them towards the midpoint from the figure-8 was tried first and is not
enough — at anything short of a full pull they still swing, because the thing
being pulled is itself orbiting. Measured after seating: an O=O's four sit at
**11.4 and 18.3 px** from the midpoint on a 27.4 px half-bond, and move **0 px
over 5 s** relative to the molecule.

The seats use the same geometry the One Bond tab settles into, so the two views
agree about where a shared pair lives.

`Lone pairs outward` and `One Bond centre hold` are dev sliders under **Shells
view**. The latter is One Bond only now — the shells view seats its bonding
electrons outright rather than pulling them anywhere.

Shells is the **cheapest** of the three: 0.14 ms/frame against trails' 0.20 and
outline's 0.16, since a dot is one gradient where a trail is a stack of stroked
buckets. Verified across lone-pairs on and off, and after breaking leaves a
world of single-atom bodies.

## Energy in fields is its own bucket

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

Both directions are exact, and nothing leaks.

| | |
|---|---|
| collision | motion → fields, via restitution `sqrt(1 - fieldConvert)` |
| relaxation | fields → motion, `exp(-dt / VIB_TAU)` returned as speed |
| breaking | fields + motion → **spent**, the one real sink |

**The relaxation used to delete the energy.** A bond's colour softened between
collisions and what it was holding simply stopped existing — 88–214% of the
spawn energy over 240 s at High. It now hands it back to the molecule's motion,
scaled as `sqrt((ke + back) / ke)` since energy goes as v². That is what a bond
does between collisions; nothing is radiated away in this model.

Verified with the speed band off and nothing breaking, so neither could mask an
error. **At both temperatures**, since the two exercise quite different regimes
— Low spends far more of its time near zero vibration:

| check | Low | High |
|---|---|---|
| whole world, 300 s | drift **0** | drift **0** |
| one molecule relaxing fully to zero | lost **0** | lost **0** |

And on a single molecule, instrumented frame by frame: vibration lost
**190881**, `fieldsReturned` **190881**, translation gained 162887, rotation
27994 — **balance exactly 0**. (The rotation share is off-centre wall bounces
moving energy between the two, elastically. A first attempt at this check
counted only translation and looked a third short, which is a good reason to
count both.)

**Two leaks were found by checking Low rather than only High**, both now fixed:

- The decay used to snap `vib` to zero below `1e-4` instead of returning the
  remainder, dropping exactly `1e-4 × doseUnit()` = **13 energy units** every
  time a molecule was left alone long enough to decay that far. It never fires
  in a running world — collisions top vibration up faster than it can fall that
  low, measured 0 times in 300 s at either temperature — but a lone molecule
  reaches it, and it was 0.0035% of a Low world's energy against 0.0013% of a
  High one.
- A molecule at a dead stop has no velocity to scale, so its returned energy was
  dropped. It now goes to **spin**, which is still motion. Verified: vibration
  fully returned, nothing lost, the molecule ends spinning.

`energySplit()` now reports `decay -> motion` and `spent breaking` instead of a
leak.

#### Making the slowdown unmistakable

`breakSlow` ships at **0.25**. It was 0.5, which gave a real but unnoticed
34% drop — measured, stable over 300 s, and still not what anyone saw. The
headroom came from `breakEnergy` dropping to 2.5: the earlier trade, where
slowing fragments starved the last O₂, was measured at capacities of 4.5–6 and
simply does not bite at 2.5.

| `breakSlow` | High: broke | speed after |
|---|---|---|
| 0.5 | 6/6/6 | 0.66–0.76 |
| 0.4 | 6/6/6 | 0.51–0.60 |
| 0.3 | 6/6/6 | 0.42–0.44 |
| **0.25** | **6/6/6** | **0.34–0.37** |
| 0.15 | 6/6/**5** | 0.19–0.21 |

Shipped: H₂ ~1160 → H **~371** and O₂ ~261 → O **~88**, both about **−67%**,
with total motion at **0.10–0.11×**. 6/6/6 at High, 0/0/0 at Low over 3 × 900 s.
0.15 is where it starts failing to finish.

**None of this is faked.** The speed on screen is the speed in the physics.
One perceptual confound to know about, though: `loneSpeedFree` spins a free
atom's electrons at 6×, so a slow free atom still reads as frantic. That is a
separate visual channel from translation, and it was asked for deliberately —
but it is the thing working against "the world is calmer now".

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

## Three view modes

`viewMode` is `'trails'`, `'shells'` or `'space'` — the old boolean `showAtoms`
split in two, because "Atoms" was doing two jobs. **It opens on `'shells'`**:
that is the picture that names the parts, and the trails are the one to switch
to once you know what you are looking at.

All three buttons carry the **same** H₂ icon, animated by the shared
`ICON_CANVASES` loop. They are three pictures of the same thing, so the button
should say which *view* you get rather than imply the molecule differs between
them. Each canvas keeps its own jitter so they do not wobble in lockstep. The
bespoke `drawOutlineIcon` that used to sit on the third button is gone.

- **Trails** — the electron trails this sim has always drawn.
- **Shells** — the bonds sim's shells picture: a gradient band at each occupied
  shell radius, electrons as **dots sitting on it** rather than as paths behind
  them. Shares `drawShellBands`, `drawElectronDot`, `ATOM_SHELLS` and
  `shellRadius` with the One Bond tab.
- **Space** — space-filling, unchanged but for the name.

**Both views share one geometry.** `shellRadius` places the band *and* the
electrons on it, in both tabs, so a shell is in the same place relative to its
atom wherever you look. Reactions used to place its shells-view electrons at
`loneRadius x loneRadiusBonded` instead — **0.54 of its own band, 17.5 px adrift
on an oxygen**, the electron floating inside the ring it was drawn with. Those
two constants are *trails-view* tuning: 0.6 exists so a lone-pair ring does not
collide with the bonding lobes in that picture. There are no lobes in the shells
view, so applying it there only broke the agreement. Now both read 1.00.

**Colour is by element in both tabs**, from `ELEMENTS` — H the pale grey, O the
red. The One Bond tab briefly coloured its two atoms apart, blue left and amber
right, so you could tell whose electrons were whose as they separated. That
reads well on an O-O and is a lie everywhere else: the same oxygen was a
different colour depending on which side it sat, and nothing else in the sim
speaks that language. On an O-O you now tell them apart by watching which way
they go.

**`shellRadius` takes the nucleus radius.** The One Bond tab draws nuclei about
9x life and needs its inner shells shoved clear of them; the main tab's nucleus
is the plain 6 px sprite and needs no such clearance, so it passes its own and
the inner shell stays where the physics puts it.

Dots are `SHELL_DOT_R` = 9 px against the One Bond tab's 17 — a world of
molecules, not one bond filling the screen.

### Shells electrons sit still, and outward

```javascript
const shellSpin     = 0;      // rad/s round the ring — none
const shellBondPull = 0.72;   // bonding electrons, towards the bond's middle
let   loneArcFrac   = 0.45;   // share of the ring a bonded atom's lone pairs use
```

**A shell is a place an electron lives, not a racetrack.** The trails view needs
motion — a stationary electron leaves no trail — but in the shells view that
same motion reads as electrons whizzing round the ring, which says something
false about what a shell is.

So the shells view draws lone electrons from **`e.angle0`**, the seat each one
was given at build time, turned only as fast as `shellSpin`. The simulation
keeps running underneath for the trails view; shells just ignores it. Measured
over 10 s: the drawn electron moved **10.3 px** while the simulated orbit angle
advanced **75 radians**, about twelve laps.

The only thing that should move them is the molecule itself rotating, and that
still does, because these are body-local coordinates.

**A bonded atom's non-bonding electrons sit on the far side from the bond**,
spread over `loneArcFrac` of the ring centred on the direction away from its
partner. Evenly round the whole circle they swept straight through the bond
region, which is where the bonding pair already is and the one place a lone pair
should not be. Measured on an O in an O=O, the cosine of each electron's angle
against that away direction — +1 is directly outward, −1 is through the bond:

| | mean | worst | all outward? |
|---|---|---|---|
| whole circle | **−0.25** | −1.00 | no |
| arc 0.55 (198°) | +0.34 | −0.16 | no |
| **arc 0.45 (162°)** | **+0.53** | **+0.16** | **yes** |

Under 0.5 on purpose: at 0.55 the arc spans 198°, so the outermost pair sits at
±99° and tips just past the halfway line onto the bond's side. At 0.45 they land
at **81°, 27°, 27°, 81°** from the away direction — every one of them strictly
outward.

**A free atom keeps the whole circle**, golden-angle spaced: nothing is pushing
its electrons anywhere, so they have no reason to bunch. Verified it still
spans the full range after a bond breaks.

Ring spin and the bond-pull slider are gone — spin is fixed at none and the pull
at its tuned value. `Lone pairs outward` replaces them, and rebuilds every
body's electrons when moved.

**Bonding electrons are SEATED beside the bond's midpoint**, perpendicular to
the axis, alternating sides and tiered outwards so a double bond's four do not
stack. Not the figure-8 position: that is the trails view's business, and drawn
as a dot it made the bonding electrons look like they were perpetually swinging
round the molecule. A shared pair sits in the gap.

Pulling them towards the midpoint from the figure-8 was tried first and is not
enough — at anything short of a full pull they still swing, because the thing
being pulled is itself orbiting. Measured after seating: an O=O's four sit at
**11.4 and 18.3 px** from the midpoint on a 27.4 px half-bond, and move **0 px
over 5 s** relative to the molecule.

The seats use the same geometry the One Bond tab settles into, so the two views
agree about where a shared pair lives.

`Lone pairs outward` and `One Bond centre hold` are dev sliders under **Shells
view**. The latter is One Bond only now — the shells view seats its bonding
electrons outright rather than pulling them anywhere.

Shells is the **cheapest** of the three: 0.14 ms/frame against trails' 0.20 and
outline's 0.16, since a dot is one gradient where a trail is a stack of stroked
buckets. Verified across lone-pairs on and off, and after breaking leaves a
world of single-atom bodies.

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

## The One Bond tab

**The sim opens here**, not on Reactions. It is the tab that explains what a
bond *is*, and a world full of them is easier to read once you have pulled one
apart by hand. `applyTab('bond')` runs at init rather than a second copy of
that state being written into the markup, so there is one path into it.

A second tab, using fission's tab bar unchanged — same shape, same glow, where
it used to carry a "Temperature" tab beside the main one. `applyTab` swaps which
`toolbar-row` is shown and sets `bondTab.on`; the world loop returns early and
calls `drawBondTab()` instead, so no physics runs and no electrons step.

One bond, big, with nothing else on screen. A dropdown picks **H–H, O–O or
O–H**; drag either nucleus and **both** move, symmetrically about the centre, so
the bond stretches about its midpoint rather than one atom chasing the pointer.
Separation is twice the pointer's distance from the centre.

### What the glow shows: energy STORED, not energy left

```
V(r) = D (1 - exp(-a (r - r0)))^2      a = BOND_MORSE_A / r0,  BOND_MORSE_A = 1.5
```

The glow is `V` — the work done moving the atoms off their natural separation.
**0 at the natural length, rising toward the full bond energy as they are pulled
apart.** So the lobe GROWS as you stretch.

An earlier version had this backwards, drawing `D - V`, the energy still held in
the bond, which *falls* as you pull. That is a real quantity, but it is not what
a stretching bond looks like it is doing: the thing that grows as you pull is
the work, and the glow should grow with it.

The asymmetry comes free: stretching has a long tail, squeezing climbs a wall,
because `exp(-a(r-r0))` runs away below `r0`. So **compressing stores energy
too** — measured, 0.6x the natural length reads 337 kJ/mol on an O=O — which is
right, since you are pushing against repulsion. A spring would have given a
symmetric parabola and said something false about short range.

**At the natural length the stored energy is zero — but the lobe is not.**

```javascript
const BOND_FIELD_FLOOR = 0.35;   // mag = floor + (1 - floor) * work
```

Drawing zero literally left a blank gap between the two atoms, which says
"there is no bond here" — the opposite of what a bond at rest is. The floor is
the bond simply **existing**; `work` is what stretching or squeezing adds on top.
It sits at the bottom of the colour ramp (`load` is still 0 there), so a bond at
rest is the plain blue glow and the purple and pink are what stretching does to
it. Measured at the natural length: **(29,84,204)**, clearly blue, against
0 kJ/mol on the readout.

**Nothing else on screen glows.** The atoms had a halo at their valence radius
and it had to go: it read as a second field, and the eye could not tell which of
the two it was meant to be watching. Verified — a pixel beside a nucleus is
(5,4,15), the background exactly.

### It opens settled

```javascript
const BOND_SETTLE_FRAMES = 300;
```

The dots used to be placed on their own atoms and hauled into the gap by the
anchor, so the tab opened with a visible scramble — the first second of every
load was electrons flying across the bond, which is not a thing bonds do.

`placeBondDots` now runs the step they would have run anyway for a few hundred
silent frames, so the opening frame shows the arrangement you would have seen
after a few seconds of watching. Cheap: four dots, no drawing.

Measured movement over the first second after load: **4.8 px mean, 8.1 px
worst**, against **66.5 px mean and 120.7 worst** without the settle — a
fourteenfold difference, which is the scramble.

### The range, and the zoom

```javascript
const BOND_MAX_MULT = 4.8;   // how far apart it can be dragged, x natural
const BOND_MIN_MULT = 0.6;
const BOND_ZOOM     = 1.2;   // 20% larger than fitting 3 natural lengths across
```

**4.8 rather than 3.2 because that is where the stored energy finally reads as
the whole bond.** The Morse form gives 93% at 3.2 and **99.3% at 4.8**, so the
extra half of the range is what turns "nearly" into "that is the bond".
Measured at full stretch: H–H **433 of 436**, O=O **495 of 498**, O–H **460 of
463** — 99% on all three.

**The fit clamp is computed from the MAXIMUM separation, never the current
one.** Clamping against the current separation would shrink the view as you
pulled, which is the one thing this must not do: the whole reading is that the
gap gets bigger. Sizing for the extreme up front costs a little scale on a
narrow window and keeps the scale honest while dragging. On a 1284-wide canvas
the clamp sits at 3.48 against a base of 2.54, so it does not bite at all and
the 20% lands in full (2.12 → 2.54). At full stretch the pair still spans only
198–1086 px.

### The scale reaches 100%

```javascript
const WORK_FULL_AT = 0.98;   // raw Morse fraction that counts as a whole bond
```

The raw Morse form only ever *approaches* 1 — it is still at 99.3% at the end of
the drag, so the reading never quite says "this is the whole bond". That last
couple of percent is also the flattest part of the curve, where a lot of pulling
buys almost nothing. Dividing by the fraction that counts as whole fixes both,
and because it is a **scale rather than an offset** the natural length still
reads 0.

Measured: raw 97% sits at **3.79 r0** and raw 98% at **4.07 r0**; after the
scale those read **99%** and **100%** exactly.

### ...and then it fades anyway

```javascript
const BOND_CONTACT_HOLD = 3.79;  // x r0 — reads 99% here
const BOND_CONTACT_GONE = 4.07;  // x r0 — reads 100% here, and is gone
```

**The glow holds all the way to 99%, then goes over the last one percent.** So
the lobe is full pink right up to the moment the bond is called whole, and the
"highest energy, no glow" state is everything past it.

Pulled far enough the two atoms simply do not interact, so there is nothing
between them to draw — **even though this is exactly where the stored energy is
greatest.** The glow is the interaction; the number is the energy; they disagree
at long range, and that disagreement is the point.

| separation | stored | contact | readout | lobe |
|---|---|---|---|---|
| separation | reads | contact | lobe |
|---|---|---|---|
| 1.0x (opens here) | 0% | 1.0 | blue (18,28,68) |
| 2.0x | 62% | 1.0 | purple (182,117,241) |
| 3.0x | 92% | 1.0 | pink (243,97,217) |
| 3.5x | 97% | 1.0 | full pink (255,92,208) |
| **3.79x** | **99%** | **1.0** | **full pink, still** |
| 3.95x | 100% | 0.39 | fading (72,14,62) |
| **4.07x** | **100%** | **0.0** | **gone** (5,4,15) |
| 4.8x | 100% | 0.0 | gone |

**Colour follows the stored energy, not the fade.** `load: work`, so a
nearly-broken bond stays pink right up until it stops being drawn, instead of
washing back through purple to white on its way out — measured (174,40,176) at
2.7x and (64,9,76) at 3.0x, the same hue at a fifth the brightness.

### No energy-vs-separation graph

There was one. It plotted the old, wrong quantity, and a corrected version would
have duplicated what the lobe and the kJ/mol readout already say between them.
Removed rather than fixed.

### Electrons: the bonds repo's shells view, in full

Every electron both atoms have, not just the bonding pair — a gradient band at
each occupied shell radius and a glowing dot per electron, straight out of the
bonds shells view. Reactions has never needed inner shells (the main tab is all
valence), so the configurations live in this tab:

```javascript
const ATOM_SHELLS = { H: [1], He: [2], O: [2, 6] };      // electrons per shell
const SHELL_FRAC  = { H: [1], He: [1], O: [0.149, 1] };  // radius, x valence
```

Counts verified: H–H is 2 bonding + 0 on shells; **O=O is 4 bonding + 12 on
shells = 16, two complete oxygens**; O–H is 2 + 7 = 9.

**Inner shells need a clamp.** The lifelike fraction is not enough on its own:
this tab draws nuclei about 9x life so the nucleons are countable, and at 0.149
an oxygen's inner shell lands at **13 px inside an 18.6 px nucleus** — its two
inner electrons buried. bonds hits the same problem and solves it the same way,
so `shellRadius` clamps any non-outer shell out to **1.4x the nucleus radius**.
Measured after: nucleus 18.6, inner shell 26.1, valence 89.1.

### Colour is per ATOM, not per element

```javascript
const ATOM_E_COLOUR = ['#4ea8ff', '#ff9a3c'];   // left atom, right atom
```

An O=O has two oxygens, and the whole point is being able to tell whose
electrons are whose. So the left atom's are blue and the right atom's amber,
whatever the elements are; inner shells stay grey.

**The bonding electrons keep their home atom's colour wherever they are**, which
is what makes pulling the bond apart legible — you watch each one go home.

### Pulling them apart sends the electrons home

```javascript
const BOND_SHARE_HOLD = 1.4;   // x r0 — fully shared in to here
const BOND_SHARE_GONE = 3.0;   // x r0 — fully home by here
```

`bondShare` slides each bonding electron's target from the gap between the atoms
to a seat on its own atom's outer shell. Its wander is scaled by the same
number, so a shared electron roams and a returned one settles.

Measured on O=O: at 2.2x every bonding electron is already on its own side of
the midpoint, and at 3.0x the mean distance from its own nucleus is **89 px
against a valence radius of 89** — they have arrived exactly on the shell. A
bond breaking is its electrons going home, and now it looks like it.

### Nuclei: the real thing, ported from bonds### Nuclei: the real thing, ported from bonds

Protons and neutrons packed into a sphere, not a "+" sprite. Bonds' own
algorithm: rejection-sample points inside a sphere with a minimum separation,
build an adjacency graph, then assign protons to the sites with the **fewest
proton neighbours first**, so like charges end up spread rather than clumped.

```javascript
const NUC_PARTICLE_R = 10;   // bonds' PARTICLE_RADIUS — the packing's own unit
const NUC_NUCLEON_R  = 6;    // drawn radius in canvas px — the legibility knob
const NUCLEONS = { H: [1,0], O: [8,8], He: [2,2] };
```

**Size had to stop being about proportion.** The honest figure — measured in the
bonds sim — is that a nucleus is **2.6–3.3%** of its valence shell (H 12.31/480,
O 31.03/945). Even at 3x that an oxygen is 6.1 px across and its sixteen
nucleons about **1.5 px each**: invisible, which defeats the point of drawing
them. So the *nucleon* is the fixed size and the nucleus is however big the
packing makes it. An O lands at **18.6 px of radius** against a ~65 px valence
shell — roughly 9x life, and you can count the particles.

| | nucleons | drawn radius |
|---|---|---|
| H | 1 p | 7.4 px |
| He | 2 p + 2 n | 11.7 px |
| O | 8 p + 8 n | 18.6 px |

Verified: proton/neutron counts correct for all three, minimum separation
**20.8** against an allowed 20.8 (the packing constraint is tight, not
violated), and the arrangement is **deterministic** — seeded from the element,
so switching pairs and back does not reshuffle a diagram.

Each nucleon gets a small per-particle wobble so a nucleus is not a dead
diagram, and they are drawn back to front by `z` so the ball reads as a ball.
Cost at the heaviest pair (O=O, 32 nucleons, each a base gradient plus a
shadow-blurred core): **0.19 ms/frame** against a 16.7 ms budget.

### It opens stretched, and the lobe is boosted to suit

```javascript
const BOND_START_MULT    = 1.34;   // opening separation, x natural
const BOND_FIELD_BOOST   = 1.30;   // lobe magnitude
const BOND_FIELD_SQUEEZE = 0.85;   // lobe width
```

**It opens at 1.0**, the natural length: a bond at rest, 0 kJ/mol stored, the
lobe at its floor. Earlier versions opened stretched so there was a visible gap
for the shared pair to sit in; opening at rest instead makes the first move you
make — pulling — the thing that puts energy in.

The lobe is then drawn bigger and brighter than a faithful reading would make it
there — which has a real cost, since **brightness is the energy reading**.
Measured at a boost of 1.30, the lobe saturates over roughly **0.75x to 1.40x**
the natural length: a band straddling the bottom of the well, where the bond is
83% whole or better either side. It comes back *off* saturation below 0.75,
because the repulsive wall drops the held energy fast. At the 1.55x opening the
ramp sits at **0.89** — bright, with headroom.

So across that band the colour says "as whole as it gets" rather than resolving
83% from 100%. The kJ/mol readout and the Morse curve still separate them
exactly; it is only the colour that runs out, and only where the answer is
"this bond is fine".

### The One Bond tab tunes the field separately

```javascript
let BOND_FIELD_WIDTH = 0.30;   // own copy of FIELD_WIDTH
let BOND_FIELD_GAIN  = 0.90;   // own copy of FIELD_GAIN
let BOND_ALPHA_POW   = 1.8;    // opacity taper — the reason for the split
const BOND_COLORMAP  = new Uint8Array(LOAD_BINS * 256 * 4);
```

The main tab's numbers are set for a dozen small lobes glimpsed at once while
molecules fly past. This tab has one lobe filling the screen, held still and
read. Different jobs, different numbers, so **nothing is shared** —
`drawFieldHeatmap` takes an optional `{ width, gain, colormap }` and behaves
exactly as before when given none.

**The opacity taper is what forced the split.** The shared ramp climbs to alpha
140 inside the first 12% of its range, because on the main tab a lobe has to
register in a glance. `BOND_COLORMAP` replaces that column with
`alpha = t^BOND_ALPHA_POW`, which keeps the outer reaches genuinely faint.
Verified the two stacks differ in **alpha only** — 4318 entries differ in alpha,
**0** in RGB, so the hues stay identical and only the fade changes.

Measured along and across the lobe at the opening separation, the distance over
which alpha falls from 90% to 10% of its peak:

| `BOND_ALPHA_POW` | peak alpha | fade along the bond | fade across it |
|---|---|---|---|
| 1.0 | 183 | 64 px | 84 px |
| **1.8** | **140** | **60 px** | **63 px** |
| 2.6 | 107 | 54 px | 52 px |

1.8 is where the two roughly match, so the lobe fades at the same rate in both
directions instead of reading as a slab that is soft on top and cut off at the
ends.

`buildRampStack(stages, gamma, out, alphaFn)` builds either stack; `setFieldRamp`
rebuilds both, so changing the hues on the dev panel keeps them in step.

All five — width, brightness, taper, rest floor and stretch boost — are dev
sliders under **One Bond tab**, in the tab bar's purple rather than the main
field controls' blue.

### Two things the main tab's lobe does that this tab must not### It opens stretched, and the lobe is boosted to suit

```javascript
const BOND_START_MULT    = 1.34;   // opening separation, x natural
const BOND_FIELD_BOOST   = 1.30;   // lobe magnitude
const BOND_FIELD_SQUEEZE = 0.85;   // lobe width
```

**It opens at 1.0**, the natural length: a bond at rest, 0 kJ/mol stored, the
lobe at its floor. Earlier versions opened stretched so there was a visible gap
for the shared pair to sit in; opening at rest instead makes the first move you
make — pulling — the thing that puts energy in.

The lobe is then drawn bigger and brighter than a faithful reading would make it
there — which has a real cost, since **brightness is the energy reading**.
Measured at a boost of 1.30, the lobe saturates over roughly **0.75x to 1.40x**
the natural length: a band straddling the bottom of the well, where the bond is
83% whole or better either side. It comes back *off* saturation below 0.75,
because the repulsive wall drops the held energy fast. At the 1.55x opening the
ramp sits at **0.89** — bright, with headroom.

So across that band the colour says "as whole as it gets" rather than resolving
83% from 100%. The kJ/mol readout and the Morse curve still separate them
exactly; it is only the colour that runs out, and only where the answer is
"this bond is fine".

### The One Bond tab tunes the field separately

```javascript
let BOND_FIELD_WIDTH = 0.30;   // own copy of FIELD_WIDTH
let BOND_FIELD_GAIN  = 0.90;   // own copy of FIELD_GAIN
let BOND_ALPHA_POW   = 1.8;    // opacity taper — the reason for the split
const BOND_COLORMAP  = new Uint8Array(LOAD_BINS * 256 * 4);
```

The main tab's numbers are set for a dozen small lobes glimpsed at once while
molecules fly past. This tab has one lobe filling the screen, held still and
read. Different jobs, different numbers, so **nothing is shared** —
`drawFieldHeatmap` takes an optional `{ width, gain, colormap }` and behaves
exactly as before when given none.

**The opacity taper is what forced the split.** The shared ramp climbs to alpha
140 inside the first 12% of its range, because on the main tab a lobe has to
register in a glance. `BOND_COLORMAP` replaces that column with
`alpha = t^BOND_ALPHA_POW`, which keeps the outer reaches genuinely faint.
Verified the two stacks differ in **alpha only** — 4318 entries differ in alpha,
**0** in RGB, so the hues stay identical and only the fade changes.

Measured along and across the lobe at the opening separation, the distance over
which alpha falls from 90% to 10% of its peak:

| `BOND_ALPHA_POW` | peak alpha | fade along the bond | fade across it |
|---|---|---|---|
| 1.0 | 183 | 64 px | 84 px |
| **1.8** | **140** | **60 px** | **63 px** |
| 2.6 | 107 | 54 px | 52 px |

1.8 is where the two roughly match, so the lobe fades at the same rate in both
directions instead of reading as a slab that is soft on top and cut off at the
ends.

`buildRampStack(stages, gamma, out, alphaFn)` builds either stack; `setFieldRamp`
rebuilds both, so changing the hues on the dev panel keeps them in step.

All five — width, brightness, taper, rest floor and stretch boost — are dev
sliders under **One Bond tab**, in the tab bar's purple rather than the main
field controls' blue.

### Two things the main tab's lobe does that this tab must not

- **It clips.** `FIELD_GAIN` is set so resting bonds sit at the top of the ramp
  and collisions drive past it, because a flare reading as a white core
  *spreading* is the whole effect. Here brightness **is** the reading, and a
  clipped core throws away the top of the range exactly where the bond is most
  interesting. `mag` is scaled by `FIELD_GAIN` so the strongest bond at its
  natural length peaks at the top and never clips. Measured centre pixel across
  the drag: (142,198,251) at `r0`, (76,27,159) at 2×, (55,7,67) at 2.8× —
  smooth, never pinned.
- **It is wide.** `FIELD_WIDTH` is a fraction of the bond *length* and the
  gaussian reaches 2.5σ, so at this scale a squeeze of 1 gave a lobe reaching
  ~193 px either side of a 257 px bond — taller than the bond is long, which
  reads as a blob with two nuclei in it. `squeeze: 0.5` puts the reach at about
  a third of the length and the bond is the shape again.

`drawFieldHeatmap` grew optional `segs`, `boxW` and `boxH` arguments for this;
with none it behaves exactly as before.

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
