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

**Heavier molecules start slower.** `speciesSpeed` scales the dial speed by
`sqrt(m_H2 / m)`, so every species begins with the same kinetic energy
(equipartition, as in a real gas mixture at one temperature). An O2 sets off at
a quarter the speed of an H2. Delete that factor if you want equal speeds.

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
- Molecules cannot be dragged or clicked; the old nucleus-rotation drag is gone.
- No reactions. Molecules bounce; they never break or combine.

## Deployment

Not deployed yet. No git remote is configured — the fission remote was removed
when this repo was copied, so `git push` will not reach fission.
