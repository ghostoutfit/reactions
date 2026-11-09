# reactions — Molecular Collision Model

Standalone single-file sim. Vanilla HTML + CSS + JS, no build step — edit `index.html` directly.

To run locally: `python3 -m http.server` in this directory, then open `localhost:8000/`.

Forked from the `fission` repo and gutted: the nuclear physics, the scripted
fission animation and its scrub/replay machinery are all gone. What was kept is
the chrome — themes, quality selector, zoom, the toolbar, the energy bar.

**Naming is still mid-migration.** A lot of ids and a few function names are
inherited from fission and have not been renamed yet (`#scrubBar` is the energy
bar, `#neutronSpeed` is the launch-speed dial, `#newBtnNucleus` is the molecule
icon, `#nucleusWheel` wraps the respawn button, `nucleusIconLoop` draws the
toolbar icons). The behaviour is correct; the labels are not.

## The one idea

The world is a wrap-around ("asteroids") box filling the canvas between the
toolbar and the energy bar. Molecules are **rigid bodies** that translate, spin,
and bounce elastically off one another. There are no walls — anything leaving an
edge returns on the opposite one, and a body straddling an edge is drawn on both
sides so it never appears to pop.

Nothing is scripted. There is no timeline, no phases and no replay: the sim just
runs.

## Atoms and molecules

```javascript
const PARTICLE_RADIUS = 30;    // drawn radius of one H atom; most geometry is a multiple of it
const BOND_OVERLAP    = 0.72;  // < 1 so bonded atoms overlap, space-filling style
const ELEMENTS = {
  H: { rgb: '185,192,205', rScale: 1.00, q: 0 },   // grey
  O: { rgb: '232,56,47',   rScale: 1.27, q: 0 },   // red
};
```

`rScale` is the van der Waals radius relative to hydrogen. `q` is a partial
charge in units of e — **both are 0**, so "Show electric field" currently draws
nothing. Give O a negative `q` and H a positive one to light it up for polar
bonds; `drawElectricHeatmap` already sums over `elemOf(p).q !== 0`.

Atoms render as **a circle drawn by a travelling arrow** (`drawAtom`): an
arrowhead laps the circle forever, paying out a trail that fades behind it. The
trail covers most of the circle so the atom keeps a readable silhouette. Driven
by the wall clock, not sim time, so atoms keep tracing while the sim is paused.

The earlier translucent-shell renderer was dropped — it read as a soap bubble.
Trail length and arrow speed are dev sliders; the defaults are
`ATOM_TRAIL_DEFAULT` / `ATOM_ORBIT_DEFAULT`.

Icons (`drawAtomIcon`) stay a plain ring — the arrow is unreadable at icon size.

Outline mode (`drawMoleculeOutline`) strokes the union silhouette — each atom
circle only where it is not buried inside another.

## What is in the world

```javascript
const SPECIES = { H2: { els: ['H','H'] }, O2: { els: ['O','O'] } };
const composition = { H2: 2, O2: 0 };    // 0–6 of each, both dev sliders
const ATOMIC_MASS = { H: 1, O: 16 };
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

```javascript
const RESTITUTION = 1.0;   // perfectly elastic
```

Verified: a head-on hit reverses both velocities exactly and induces no spin; a
glancing hit deflects both and spins them; energy drift and linear-momentum
drift are both 0 to the precision measured, with 20 molecules over 8 s; atom
interpenetration never exceeded 0 px.

`wrapDelta` gives the minimum-image offset, so a molecule at the left edge can
collide with one at the right edge. The loop sub-steps at 4 ms so fast molecules
cannot tunnel through each other, and clamps `dt` to 50 ms so a backgrounded tab
does not resume with one enormous step.

## Energy graph

A scrolling 12-second trace of total kinetic energy (translational +
rotational), full scale = 2× the energy at spawn, so **a flat line across the
middle is the correct result**. A visible step means energy leaked — it is a
check on the collision solver, not a teaching graph.

## Known gaps

- The world band excludes the full-width strip above the toolbar, even on the
  right where the toolbar panel does not reach. Simple, but it wastes screen.
- Composition lives in the dev panel, not the toolbar.
- The temperature ramp runs even while paused, so velocities change with nothing
  moving. Harmless, but the pause is not a total freeze.
- Molecules cannot be dragged or clicked; the old nucleus-rotation drag is gone.
- No reactions. Molecules bounce; they never break or combine.

## Deployment

Not deployed yet. No git remote is configured — the fission remote was removed
when this repo was copied, so `git push` will not reach fission.
