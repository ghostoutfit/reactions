# reactions — Molecular Collision Model

Standalone single-file sim. Vanilla HTML + CSS + JS, no build step — edit `index.html` directly.

To run locally: `python3 -m http.server` in this directory, then open `localhost:8000/`.

Forked from the `fission` repo and gutted: the nuclear physics, the scripted
fission animation and its scrub/replay machinery are all gone. What was kept is
the chrome — themes, quality selector, the toolbar, the energy bar.

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
`_drawBondElecTrails` and `_stepMolBondElecOne`), where each valence electron
was a moving point paying out a fading trail with an arrowhead at its head. The
trail is gone from this sim; **the motion is not.** Every electron still walks
its figure-8 or its ring every frame, because where it is on that path is what
the One Bond tab animates and what the shells view seats it by.

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

Drawing the trails used to be the dominant cost, and the numbers below are what
it was when it was measured — 6 H₂ + 6 O₂ + 6 He = **96 electrons**, trails full
at 48 points, hi-res, against a 16.7 ms budget:

| | ms/frame |
|---|---|
| electrons alone | 0.45 |
| whole frame, field off | 0.40 |
| whole frame, field on | 0.75 |

The trails are gone and the shells stack draws instead. On one machine, same
synthetic world, the stack costs **1.31×** what the trails pass did — 1.70 ms
against 1.30 — so the renderer benchmark was re-pointed at the shells stack and
`TIER_THRESHOLD_MS` was multiplied by that ratio (6.0 → 7.9, 14.0 → 18.3) to
keep a given machine in the tier it was already getting.

Electron positions live in **body-local coordinates** and are transformed at
draw time, so a molecule can translate and spin without its electrons being
recomputed.

`stepElectrons` runs **once per frame, outside the physics sub-steps** — the
phase logic is tuned in per-frame units and breaks up with large or uneven dt.

It also runs **outside the `running` gate**. Pause stops the molecules moving
and spinning; it does not stop an atom being an atom, and a frozen electron
reads as a broken picture rather than a paused one. It also means the molecules
are alive on page load, before GO is ever pressed.

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
long after the halves have drifted apart. Applies to
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

Speed, steering, lone orbit radius and speed, and a lone-pairs
on/off switch are all dev controls.

## What is in the world

```javascript
const SPECIES = { H2: { els: ['H','H'] }, O2: { els: ['O','O'] }, He: { els: ['He'] },
                  H2CO3: { els: ['C','O','O','H','O','H'] } };
const composition = { H2: 4, O2: 2, He: 0, H2CO3: 0 };   // 0–6 of each, dev sliders
const ATOMIC_MASS = { H: 1, O: 16, He: 4, C: 12 };
```

`spawnWorld()` builds a queue from `composition`, shuffles it so a crowded world
does not cram one species into the gaps the other left, then places each at a
random position, heading and spin — retrying up to 400 times for a clear spot.

### The reaction picker

The composition is a set of dev sliders, which is the right tool for tuning and
the wrong one for teaching: a reaction is a set of molecules that belong
together, and choosing one should not mean setting three numbers in the right
order. `REACTIONS` is that choice, and the **picker in the red MOLECULES box**
is where it is made.

```javascript
const REACTIONS = {
  water:    { composition: { H2: 4, O2: 2, He: 0, H2CO3: 0 }, icon: 'H2',    recombine: true  },
  carbonic: { composition: { H2: 0, O2: 0, He: 0, H2CO3: 2 }, icon: 'H2CO3', recombine: false },
};
```

`setReaction` writes the whole composition, respawns, and puts the dev sliders
back in step (`syncCompositionSliders`) — a stale slider is a lie about what the
next NEW will spawn. The NEW button under the picker draws the reaction's own
molecule and carries its formula, so the two read as one control: choose the
reaction, press NEW. Both stretch to the button's width, which the 96 px
molecule canvas fixes, so changing the selection cannot resize the group or
shift the temperature buttons beside it. Measured: the pink box is 352×138 and
`tempGroup` starts at x=319 on **both** reactions.

**`recombine` is part of the reaction, not a global.** Water's two halves —
breaking, then coming back together — are both the point there. Carbonic acid is
the breaking half only: its atoms go free and stay free. Without that flag the
stray H's and O's it leaves behind would start making water, a reaction nobody
asked that world for. `stepRecombination` needs both the dev toggle and the
reaction's own flag. Verified: 150 s of carbonic acid at High ends with
`joins=0`, `recombLatched=false` and twelve free atoms — 2 C, 4 H, 6 O.

### Carbonic acid in the Reactions tab

Two assumptions held while every species here had at most two atoms: **atoms lie
in a row**, and **atom k is bonded to atom k-1**. Carbonic acid breaks both. It
is a tree, not a chain, and it holds a C–O and a C=O at once, which no table
keyed by the element pair can tell apart (see "A bond TYPE, not an element
pair"). So:

```javascript
const MOLECULE_SHAPES = {
  COOHOH: { geometry: () => sbCarbonicAcid(),
            bonds: [[0,1,2],[0,2,1],[2,3,1],[0,4,1],[4,5,1]] },
};
```

- A species may bring **its own geometry**. The geometry is `sbCarbonicAcid`,
  the same function the Sandbox builds its molecule from: one Lewis structure,
  two tabs reading it. Measured in the world: C=O 50.0, C–O 58.1/58.1, O–H
  49.0/49.0; O=C–O 125.5 twice, (HO)–C–(OH) 109.0, H–O–C 106.8 twice.
- A bond may carry **its own order** as a third entry. Everything that reads a
  bond goes through `bondOrderOf` / `bondEnergyOf`, which fall back to the
  element-pair tables when no order is given, so H₂, O₂ and water are
  pixel-for-pixel what they were. Electrons measured: C=O four, every single
  two; lone electrons C 0, each O 4, each H 0. Fields reads **559 kJ/mol per
  molecule**, the same figure the Sandbox gives.
- `moleculeIconLayout` now reads `moleculeGeometry` instead of computing a row
  of bond lengths, so the NEW button draws the shape the molecule actually is.
  For a diatomic it comes back to the same two numbers it always produced.
- `drawWorld` draws every atom in the world in **one layer-sorted list**, not
  per body, and gives each one `drawAtomBody` first. Carbon has to come out
  above the oxygens around it, and in a six-molecule world those oxygens can
  belong to a different body than the carbon does. `byLayer` is stable, so every
  atom that is not carbon stays in the order it was drawn in before. Measured on
  a carbonic acid in the scene: the carbon disc reads luminance 14–19 all the way
  round against a background of 21 — black, the same as in the Sandbox.

### The speed the data means, and the speed you see

**This is the first piece of a deliberate split between the two.** Equipartition
says `v` goes as `1/sqrt(m)`, and that is true and it tells the wrong story. A
water is 18 to an H₂'s 2, so a finished water crosses the screen at **a third**
of the speed the hydrogen it was made from did — and it arrives there carrying
*more* energy, because forming the bond released some. The bars say the motion
went up, the screen says everything slowed down, and a student believes the
screen.

So the mass dependence is split. `massVisual` is the exponent the **drawing**
uses; the accounting keeps the true 1/2 whatever the drawing does.

```
drawn speed = base × (H2_MASS / m) ^ massVisual
true  speed = base × (H2_MASS / m) ^ 0.5
```

At 0.5 the two are the same. At 0 every species is drawn at the same speed, so
the only thing left that can make the screen faster or slower is the
temperature. Between them the mass spread is compressed rather than removed.
Dev slider, *Show of that*, under the equipartition toggle.

**0.25 is measured.** Mean body speed on screen two seconds in, against the same
once the reaction has finished — four runs each at High:

| `massVisual` | start | end | end/start |
|---|---|---|---|
| 0.5 (true) | 389 | 302 | **0.77** — the complaint: it slows down |
| 0.35 | 368 | 427 | 1.16 |
| **0.25** | 306 | 500 | **1.63** |
| 0.15 | 405 | 620 | 1.53 |
| 0 | 390 | 848 | 2.17 |

The crossover is just under 0.35. At 0.25 the world plainly ends faster than it
started, and a real mass difference survives on screen: a water at 0.58 of an H₂
rather than 0.33, an O₂ at half rather than a quarter.

#### What keeps it honest

A distortion of the same kind as water's exaggerated pivot and the bar chart's
raised zero — the number is wrong and the behaviour it shows is right. What
makes it safe is that **it is confined to the drawing**, and there were exactly
three places where the data was reading drawn velocities. All three now read
`trueMotionEnergy()` or undo the boost:

- **The anchor.** `anchorEnergyTotal` sets the run's constant total. Measured
  across four spawns each: mean `totalKJ` 5,239 / 5,244 / 5,210 at `massVisual`
  0.5 / 0.25 / 0 — the same number, within the spawn spread's own ±5%.
- **The thermostat.** `stepTemperature` moves `totalKJ` by what it put in, and
  must put in the same amount whatever the drawing is doing.
- **The collision dose.** This one bit hardest. The dose goes as the square of
  the closing speed, so drawing the heavy molecules faster made every hit they
  were in land harder: at `massVisual` 0 the world broke **4.3× as many bonds**
  in 90 s as it did at 0.5 (means 51.6 against 12.0). A slider that rewrites the
  reaction's dynamics is not a visual slider. Dividing the deposited energy by
  `displayBoost(A) × displayBoost(B)` takes it back out — exact when the pair
  are boosted alike, the geometric mean in between. After: means **11.5 / 11.2 /
  14.7** across the same three settings, and four waters in every run.

`trueMotionEnergy()` corrects **translation only**. Spin is not set from
`speciesSpeed` and carries no display boost, so correcting it would be inventing
a distortion to undo — the first version divided the rotational term too and
moved the anchored total by a third across the slider's range.

#### What this lever cannot do

It makes the world *end* faster than it started, but only because the heavy,
slow O₂ is gone by then. It cannot make the released bond energy itself visible,
because the speed band pins every body to `vTarget`, which is a function of
temperature and mass and **not** of how much energy the world actually holds —
surplus goes to spin and, past `MAX_SPIN`, out. Making an exothermic reaction
*look* exothermic needs the band's target to follow the energy, which is a
separate change.

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

**The sim opens at Low, with strong water on.** At Low nothing breaks at all —
that is the measured, deliberate design of the threshold, not a shortfall — so
the world opens **calm**, and the first thing to do is turn the heat up and watch
bonds start to go. The breaking is something the student *causes* rather than
arrives into.

That reverses an earlier default. The sim used to open at High, on the argument
that Low is the one setting where the main event never happens, so opening cold
opened on nothing. The opposite argument won: opening mid-event means the
student never sees the before. Low is the before, and HIGH is the move. Keep
both halves together — Low alone, with water still breakable, is a world where
nothing at all happens; **strong water on** is what makes the heat-up end in a
product that stays, so the run reads one way: cold → hot → water.

Verified on a cold load: `tempLevel` `'low'`, `tempTarget` and `tempCurrent` 1,
only the LOW button active, `strongWater` true with the STRONG WATER button
carrying `active`, H₂ spawning at 90–435. Ramping to High from that default
reaches `tempCurrent` exactly 3.000 with every speed scaled 3.000×, since
`tempCurrent` chases `tempTarget` rather than jumping. The defaults live in
three places that must agree — the `tempLevel`/`tempTarget`/`tempCurrent`
declarations, `let strongWater`, and the `active` class on the LOW and STRONG
WATER buttons in the markup, since neither control reads its state back out of
the DOM on load.

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

Each tier sets both drawing detail and a frame cap; `worldLoop` holds to
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

`drawWalls` strokes a faint rectangle at the boundary. The box lines up with the
toolbar, the energy bar, the bar chart's panel and the screen edges, so the only
line that really shows is the one against the panel — which is the one worth
having, since nothing beyond it is part of the world.

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

#### Bond storage: three ways, on one dev control

The stack above is a modelling **choice**, and the other two choices are worth
being able to see beside it. Dev panel, under Bond breaking, as `vibMode`:

| setting | stacks? | colour ramp | bar chart | breaks on |
|---|---|---|---|---|
| `shown` *(ships)* | yes, to `bondLevels` | reads it | counts it | the running total |
| `none` | no | nothing to read | nothing to count | one hit alone |
| `hidden` | yes, identically | reads **zero** | counts **nothing** | the running total |

**`none` — one hit or nothing.** A hit either carries a whole dissociation
energy on its own or the bond shrugs it off: the dose goes straight back into
the motion the restitution took it out of, in the same step, and the bond is
left exactly as it was. No bite out of the motion, so no `sag` either — the
band has nothing to make allowances for when the energy never actually left.
This is the honest hard-sphere reading, and it is the setting for asking whether
accumulation is doing real work or just making the sim feel busier.

Note the restitution still runs and the energy is handed back by scaling each
body's velocity, so the pair separates in the direction an inelastic bounce set
but at elastic speed. Same energy, very slightly different bounce. Deciding
before the impulse instead would be a bigger restructure for no visible gain.

**`hidden` — the same stack, with the lights off.** Stacks and breaks *exactly*
like `shown`; the colour ramp reads zero and the bar chart never counts it, so a
bond gives no warning and the bars do not move until it actually goes. This is
provably display-only: `vibLoad` is called from two places, both of them
drawing, and `fieldsKJ` reaches the physics nowhere — `anchorEnergyTotal` runs
only at spawn, when nothing is stored, and `motionKJ` is read only by the bar.
It is the setting for asking whether the ramp and the rising fields bar are what
make the breaking legible, or whether the world reads the same without them.

Hiding it loses nothing: motion is read off the constant total as the remainder,
so `hidden` simply reports the stored energy as still being in motion. Breaking
then moves the bars by the whole dissociation energy at the moment it happens,
instead of part of it leaking across beforehand.

Measured — two H₂CO₃ taken to atoms, which needs 8 breaks and has no
recombination to confound it, twelve 90 s runs at Med:

| | mean breaks | first break |
|---|---|---|
| `shown` | 4.3 | 16.7 s |
| `hidden` | **4.3** | 19.4 s |
| `none` | 3.8 | **28.4 s** |

`hidden` lands on `shown`'s number exactly, which is the claim. At **High** all
three reach 8 of 8 in every run — single hits there routinely carry a whole
dissociation energy, so the stack has little left to contribute, and `none` only
delays the first break (≈5.9 s against ≈3.0 s). At **Low** none of the three
breaks anything in 90 s. So the setting bites hardest at Med, which is where the
question it asks is interesting.

Changing the setting **hands back whatever the world is holding** — the same
contract the strong-water toggle keeps — rather than leaving energy in a field
the new mode cannot fill, release or show. Measured across a `shown → hidden`
switch holding 220 kJ/mol: fields −220, motion +220, real total unchanged. With
the stack off, a join's leftover goes to speed rather than a glow, since a bond
that cannot hold a collision's energy cannot hold a join's either.

The dev diagnostics — `energySplit`, `vibPeak`, `runBreakTests` — stay honest in
all three. The secret is kept from the picture, not from the instruments.

#### The bond that would actually let go

With two atoms there was one candidate and this question did not exist. With six
there are five bonds, and the weakest is **not necessarily one that breaking
would take an atom off**: carbonic acid's weakest is a C–O at 358, but that
oxygen is carrying a hydrogen, so pulling it would throw off an OH, not an atom.

`breakableBond(b)` is the weakest bond with a **leaf** on one end — an atom in
exactly one bond. For carbonic acid that is an O–H at 463. Every number the
break uses comes from that one bond: `bondEnergyFactor` (and so the threshold),
`bondLevels`, and what the dissociation costs. A molecule here is a tree, and a
tree with any bonds at all has a leaf, so it is never null.

`bondLevels` taking the breaking bond's order rather than the molecule's biggest
matters for the same reason: carbonic acid holds a C=O, but the bond under load
is the O–H that is going to go, and a single bond has one level wherever it sits.
Measured on a fresh H₂CO₃: breakable bond `[2,3]`, threshold 1.395
(= 1.5 × 463/498), one level.

#### Shatter or peel

| bonds | what happens |
|---|---|
| 1–2 | **shatter** — every atom goes free at once |
| 3+ | **peel** — one bond lets go, one atom leaves, the rest stays whole |

The shatter is what breaking has always done. It is exactly right for a
diatomic, which has nothing else one bond could do, and **water depends on it**:
the recombination gate waits for every body in the world to be a single atom
before it opens, so peeling a water into H + OH would shut that gate forever.
That is why the line is drawn at the bond count rather than at the reaction —
nothing in the water world has three bonds, so nothing there peels.

Six atoms vaporising on one knock says the wrong thing. A collision does not
undo a molecule, it takes a piece off it, and the piece coming off one at a time
is the thing worth watching. Measured, carbonic acid goes:

```
H₂CO₃ → H + CHO₃ → O + CHO₂ → H + CO₂ → C + O + O
```

— the O–H at 463 first, then the C–O at 358 once that oxygen is a leaf, then the
other O–H, and the last three together when only two bonds are left. Still every
atom free in the end.

The residue is **not rebuilt from any table**; there is no table it could come
from. `assembleBody` takes an explicit geometry and bond list, and the residue
is assembled from the coordinates its atoms already had, with the surviving
bonds renumbered and their orders carried over. Measured at every one of eight
breaks: **zero atoms move** at the instant the bond goes, except the one that
left.

#### The slowdown goes on the atom that leaves

This is the one place the peel departs from the shatter, and it was forced by
measurement. `breakSlow` cuts a fragment to a quarter of its speed — a
sixteenth of its energy — and the speed band then holds it there for good, since
`vTarget` goes down with it. One break, one cooling: fine for a diatomic, which
has nothing left to break.

Carbonic acid needs four. With the residue slowed too, the second break **never
came**: both residues sat at 54 px/s with a threshold of 1.08 dose units, a hit
between them deposited 0.7 into each, and in 117 s of High nothing else broke.
A molecule that can only ever lose its first atom is not a molecule coming apart.

So the dissociation's cost is paid by the piece you watch fly off, and the
molecule left behind carries on at the speed it was going. The world still cools
as it dissociates — every freed atom is slow, and in the end that is all there is
— but it cools as the atoms leave rather than all at once. The kick is
momentum-neutral here, unlike the shatter's: with two pieces there is a right
answer, and it is the light one that visibly flies off while the heavy residue
barely recoils.

With that, the journey measured over three runs, from the Low default:

| | run 0 | run 1 | run 2 |
|---|---|---|---|
| 20 s at **Low** | 0 breaks | 0 breaks | 0 breaks |
| first atom off at **High** | 2.9 s | 10.5 s | 1.9 s |
| all 12 atoms free | 12.5 s | 24.4 s | 9.8 s |

Low holds for 180 s with nothing breaking at all.

#### What the bars cannot pay for

A carbonic acid world is the first one where **full dissociation costs more than
the world was anchored with**, and it is worth knowing before the bar chart is
read in anger. Twelve free atoms are 6,000 kJ/mol of fields; two H₂CO₃ start at
1,118, so pulling them apart costs 4,882. At High the world spawns with about
5,203 kJ/mol of motion — a margin of only 22%, because equipartition gives a
six-atom molecule the same kinetic energy as an H₂ while it has five bonds to
pay for. The spawn bell can push a particular world under that line.

The total itself stays exact — measured drift **0** across a full breakup — but
`motionKJ` is `max(0, total − fields)`, so a world that runs out reads zero
motion while its atoms are still visibly moving. Measured: at High it happened
in 1 run of 3, and only in the last moment of the run; at Med the breakup stalls
part-way (4–6 of 8 breaks in 120 s), which is arguably the right answer — a gas
that cold should not come apart — and motion can bottom out there too. Fixing it
properly means a reverse reaction, not a bigger number.

#### Two view modes

`viewMode` is `'shells'` or `'space'`, and **it opens on `'shells'`** — the
picture that names the parts.

There was a third, **Trails**, which drew each valence electron's recent path as
a fading streak with an arrowhead for the electron. It was this sim's original
picture and it has been removed, along with the per-electron ring buffers that
fed it, `drawElectrons`, `TIER_BUCKETS`, `TIER_TRAIL_LEN`, `TRAIL_MAX`, the
`Trail length` dev slider and the view zoom that framed it. The electron
*motion* stayed: every electron still walks its figure-8 or its ring each frame,
because where it is on that path is what the One Bond tab animates and what the
shells view seats it by. Git has the rest.

Both buttons carry the **same** H₂ icon, animated by the shared
`ICON_CANVASES` loop. They are two pictures of the same thing, so the button
should say which *view* you get rather than imply the molecule differs between
them. Each canvas keeps its own jitter so they do not wobble in lockstep.

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
two constants size the *figure-8* picture: 0.6 exists so a lone-pair ring does
not collide with the bonding lobes in it. There are no lobes in the shells
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

**Real nuclei here too**, not the "+" sprite: an oxygen should look like eight
protons and eight neutrons wherever you meet it. `WORLD_NUCLEON_R` is 3.2
against the One Bond tab's 6, because this view has a dozen molecules in it, and
that puts an O's nucleus at **9.9 px** of radius against its 38.1 px valence
shell.

**The inner shell is drawn with its electrons.** Reactions models only valence
electrons — bonding and lone pairs are all it ever needs — so an inner shell had
a band round it and *nothing on it*. `drawInnerElectrons` fills them in straight
from `ATOM_SHELLS` rather than simulating them, because nothing is ever going to
happen to them: they are there to say the atom has an inside. Grey, dimmed, and
smaller than the valence dots.

**The inner-shell clamp went 1.4 → 1.8 nucleus radii.** With a real packed
nucleus rather than a sprite there is more to clear, and at 1.4 the inner shell
sat 4 px off the nucleons and read as part of them. At 1.8 an oxygen's lands at
**17.9 px** — 8 px clear of a 9.9 px nucleus, and well inside the 38.1 px
valence shell. Shared with the One Bond tab, where it moves the inner shell
26.1 → 33.5 against a 111.3 px valence shell.

Cost in shells view with all this: **0.56 ms/frame**.

### Shells electrons sit still, and outward

```javascript
const shellSpin     = 0;      // rad/s round the ring — none
const shellBondPull = 0.72;   // bonding electrons, towards the bond's middle
let   loneArcFrac   = 0.45;   // share of the ring a bonded atom's lone pairs use
```

**A shell is a place an electron lives, not a racetrack.** The trails view needed
motion — a stationary electron left no trail — but drawn as dots that same
motion read as electrons whizzing round the ring, which says something false
about what a shell is.

So the shells view draws lone electrons from **`e.angle0`**, the seat each one
was given at build time, turned only as fast as `shellSpin`. The simulation
keeps running underneath — the One Bond tab animates it — and shells just
ignores it. Measured
over 10 s: the drawn electron moved **10.3 px** while the simulated orbit angle
advanced **75 radians**, about twelve laps.

### ...but they still move

```javascript
let shellWobble = 0;                 // own accumulator, advanced every frame
const SHELL_WOBBLE_AMP = 0.075;      // x the shell radius
const SHELL_WOBBLE_HZ  = 0.62;
```

With the ring spin off and the bonding electrons seated, the shells view had
nothing moving in it at all. **Paused, it looked switched off rather than
held** — which is what the default view changing from trails to shells cost.

So each electron wobbles **about its seat**, rather than running a lap of the
ring. That is the distinction the spin got wrong: an electron is somewhere,
imprecisely, not racing round a track. On a 38.1 px shell it drifts about
**4.6 x 3.3 px**.

The two frequencies are deliberately not a round ratio so a pair never settles
into lockstep, and a per-electron seed keeps each on its own schedule.

**Its accumulator advances every frame whether or not the sim is running**,
because an atom does not stop being an atom when you press pause. Verified:
paused, with the molecule itself held perfectly still, **1807 pixels change in
half a second**. Both tabs use it.

The other thing that moves them is the molecule itself rotating, and that still
does, because these are body-local coordinates.

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
stack. Not the figure-8 position, which the electron still walks every frame:
drawn
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

Shells was the **cheapest** of the three when there were three: 0.14 ms/frame
against trails' 0.20 and outline's 0.16, since a dot is one gradient where a
trail was a stack of stroked buckets. (Per atom. The whole shells STACK — bands,
inner electrons, dots and nucleons — costs more than the trails pass did; see
the renderer benchmark.) Verified across lone-pairs on and off, and after breaking leaves a
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

## Two view modes

(This section repeats the one above, from an earlier edit; both are kept in
step until one of them is deleted.)

`viewMode` is `'shells'` or `'space'`, and **it opens on `'shells'`** — the
picture that names the parts. The third, **Trails**, has been removed; the
section above says what went with it.

Both buttons carry the **same** H₂ icon, animated by the shared
`ICON_CANVASES` loop. They are two pictures of the same thing, so the button
should say which *view* you get rather than imply the molecule differs between
them. Each canvas keeps its own jitter so they do not wobble in lockstep.

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
two constants size the *figure-8* picture: 0.6 exists so a lone-pair ring does
not collide with the bonding lobes in it. There are no lobes in the shells
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

**Real nuclei here too**, not the "+" sprite: an oxygen should look like eight
protons and eight neutrons wherever you meet it. `WORLD_NUCLEON_R` is 3.2
against the One Bond tab's 6, because this view has a dozen molecules in it, and
that puts an O's nucleus at **9.9 px** of radius against its 38.1 px valence
shell.

**The inner shell is drawn with its electrons.** Reactions models only valence
electrons — bonding and lone pairs are all it ever needs — so an inner shell had
a band round it and *nothing on it*. `drawInnerElectrons` fills them in straight
from `ATOM_SHELLS` rather than simulating them, because nothing is ever going to
happen to them: they are there to say the atom has an inside. Grey, dimmed, and
smaller than the valence dots.

**The inner-shell clamp went 1.4 → 1.8 nucleus radii.** With a real packed
nucleus rather than a sprite there is more to clear, and at 1.4 the inner shell
sat 4 px off the nucleons and read as part of them. At 1.8 an oxygen's lands at
**17.9 px** — 8 px clear of a 9.9 px nucleus, and well inside the 38.1 px
valence shell. Shared with the One Bond tab, where it moves the inner shell
26.1 → 33.5 against a 111.3 px valence shell.

Cost in shells view with all this: **0.56 ms/frame**.

### Shells electrons sit still, and outward

```javascript
const shellSpin     = 0;      // rad/s round the ring — none
const shellBondPull = 0.72;   // bonding electrons, towards the bond's middle
let   loneArcFrac   = 0.45;   // share of the ring a bonded atom's lone pairs use
```

**A shell is a place an electron lives, not a racetrack.** The trails view needed
motion — a stationary electron left no trail — but drawn as dots that same
motion read as electrons whizzing round the ring, which says something false
about what a shell is.

So the shells view draws lone electrons from **`e.angle0`**, the seat each one
was given at build time, turned only as fast as `shellSpin`. The simulation
keeps running underneath — the One Bond tab animates it — and shells just
ignores it. Measured
over 10 s: the drawn electron moved **10.3 px** while the simulated orbit angle
advanced **75 radians**, about twelve laps.

### ...but they still move

```javascript
let shellWobble = 0;                 // own accumulator, advanced every frame
const SHELL_WOBBLE_AMP = 0.075;      // x the shell radius
const SHELL_WOBBLE_HZ  = 0.62;
```

With the ring spin off and the bonding electrons seated, the shells view had
nothing moving in it at all. **Paused, it looked switched off rather than
held** — which is what the default view changing from trails to shells cost.

So each electron wobbles **about its seat**, rather than running a lap of the
ring. That is the distinction the spin got wrong: an electron is somewhere,
imprecisely, not racing round a track. On a 38.1 px shell it drifts about
**4.6 x 3.3 px**.

The two frequencies are deliberately not a round ratio so a pair never settles
into lockstep, and a per-electron seed keeps each on its own schedule.

**Its accumulator advances every frame whether or not the sim is running**,
because an atom does not stop being an atom when you press pause. Verified:
paused, with the molecule itself held perfectly still, **1807 pixels change in
half a second**. Both tabs use it.

The other thing that moves them is the molecule itself rotating, and that still
does, because these are body-local coordinates.

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
stack. Not the figure-8 position, which the electron still walks every frame:
drawn
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

Shells was the **cheapest** of the three when there were three: 0.14 ms/frame
against trails' 0.20 and outline's 0.16, since a dot is one gradient where a
trail was a stack of stroked buckets. (Per atom. The whole shells STACK — bands,
inner electrons, dots and nucleons — costs more than the trails pass did; see
the renderer benchmark.) Verified across lone-pairs on and off, and after breaking leaves a
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

This table is keyed by a **sorted element pair**, which is why it cannot hold
the carbon bonds: C–O and C=O collide on the key `'C-O'`. The One Bond tab
reads `BOND_TYPES` instead; this one stays as it is and keeps serving Reactions
and Sandbox, which have no carbon in them.

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

Using fission's tab bar unchanged — same shape, same glow, where it used to
carry a "Temperature" tab beside the main one. `applyTab` swaps which
`toolbar-row` is shown and sets `bondTab.on`; the world loop returns early and
calls `drawBondTab()` instead, so no physics runs and no electrons step.

The bar reads in the order the sim is meant to be taken: **One Bond, Reactions,
Sandbox** — one bond on its own, then bonds breaking in a crowd, then build your
own. Every tab's control row carries `margin-top: 20px`, because the boxes hang
their labels above their own top edge and without it the label rides up into the
tab bar's band; the same offset on all three keeps the boxes at one height
whichever tab you are on.

One bond, big, with nothing else on screen. A dropdown picks the bond; drag
either atom and **both** move, symmetrically about the centre, so the bond
stretches about its midpoint rather than one atom chasing the pointer.

### A bond TYPE, not an element pair

Everything in the sim used to be keyed by a sorted element pair —
`BOND_ENERGY`, `BOND_ORDER`, `bondLen` all take `[a,b].sort().join('-')`. One
entry per pair, so a pair has one energy, one length and one order. That held
while every pair in the sim had a single bond worth drawing, and **it stops
holding the moment carbon arrives: C–O and C=O are the same two elements.**
Nothing keyed by the sorted pair can tell them apart.

So the One Bond tab selects from `BOND_TYPES`, where each row carries its own
order, energy and length:

```javascript
const BOND_TYPES = {
  'H-H': { els: ['H','H'], order: 1, D: 436, lenMult: 1    },
  'O=O': { els: ['O','O'], order: 2, D: 498, lenMult: 1    },
  'O-H': { els: ['O','H'], order: 1, D: 463, lenMult: 1    },
  'C-O': { els: ['C','O'], order: 1, D: 358, lenMult: 1    },
  'C=O': { els: ['C','O'], order: 2, D: 799, lenMult: 0.86 },
};
```

`bondTab.type` is a key into this, where it used to be `bondTab.pair`. The tab
reads `bondTabEls / bondTabOrder / bondTabD / bondTabR0 / bondTabSymbol` and
never consults the element-pair tables, so the C=O gets **four** bonding
electrons where the C–O gets two — measured, and that is the visible difference
between the two entries besides the length.

**The element-pair tables are deliberately untouched.** Reactions and Sandbox
model molecules that form and break by collision, and putting carbon into that
is a far bigger piece of work than adding a row here. Verified after the change:
`BOND_ENERGY_MAX` 498, `kjPerSim` 0.0026, `bondEnergy(O,O)` 498, the Sandbox
ADD buttons still H₂ and O₂ only.

**`lenMult` is the length, relative to what the pair's `bondLen` gives.** The
original three sit at 1 and are pixel-for-pixel what they were. A double bond is
genuinely shorter than its single — C–O is 1.43 Å, C=O is 1.23 — and when both
are in one list that ratio is the reading: 0.86, so 58.1 px against 50.0.

The dropdown is **built from `BOND_TYPES` at startup**, not written out in the
HTML. The list IS the table: a bond cannot be in the menu without an energy, or
in the table without a way to pick it.

### Carbon

Added to all five element-keyed tables: `ELEMENTS` (`rScale` 1.42, from C's
1.70 Å van der Waals against H's 1.20), `NUCLEONS` `[6,6]`, `ATOM_SHELLS`
`[2,4]`, `SHELL_FRAC` `[0.18,1]`, `VALENCE` 4.

Size does part of the work on its own: at 1.42 it is the biggest atom here, and
the nucleus shows six protons against oxygen's eight.

### Carbon is yellow, and yellow draws itself

`#ffee00` = `255,238,0`. It was CPK black for a long stretch, and black needed a
pile of machinery no other element here wants. **All of it is gone**, because a
colour lighter than the background needs none of it:

| black needed | yellow needs |
|---|---|
| `body: true` and `drawAtomBody`, a filled disc, because an atom drawn as light has nowhere to be black | nothing — it is drawn as light, like H, O and He |
| a separate `ink`, because black dots on a black atom are not dots | nothing — one colour does the band, the dots and the icon ring |
| that ink split three ways, the band dimmed by `carbonShellDark` and the dots forced to `0,0,0` | nothing |

`ELEMENTS.C` is now an ordinary row in the table: `rgb`, `rScale`, `layer`.

**`layer` stays, and it was never about the colour.** `ELEMENTS.C.layer` is 1
and `byLayer` paints higher layers last. Carbon is the centre of everything it
is in, so it is always the one with neighbours overlapping it, and the sort is
what keeps a C–O reading as a carbon in front of an oxygen rather than a blend
of the two.

The electrons are deliberately *not* in that sort — they are painted after every
body, on both the Sandbox and its thumbnails. The shared pair in a C–O gap falls
**inside** the carbon's own radius (the bond is 58 px, the carbon's radius 51),
so a carbon painted over them would hide the very dots the Lewis reading is made
of. That was true when carbon was an opaque black disc and it stays true now
that it is a ring, because the ring's glow is still drawn over that gap.

#### What was kept and what went

`drawAtomBody` and `BODY_CORE` **stay, dormant.** No element sets `body: true`
any more, so the function returns on its first line every time. It is the
general answer for an element darker than the background, the measurements in
its comment are the record of what that costs, and the next dark element will
want it.

`elemShellInk` and `elemEInk` **stay, as names.** Both fall through to `elemInk`
for every element in the table today. They are kept separate because they are
separate questions and carbon spent a while answering them differently: an
element dark enough to need that again sets `ink` for its band and `eInk` for
its dots, and no call site has to change.

The two dev sliders **went**: *Carbon opacity* drove `bodyAlpha`, which nothing
reads now, and *Carbon shell dark* drove `carbonShellDark`, which no longer
exists. A dev control that moves nothing is worse than no control — the sliders
come back with the machinery if it is ever woken up.

Verified after the change: all four of `elemInk`, `elemShellInk`, `elemEInk` and
`ELEMENTS.C.rgb` return `255,238,0`; `body` false; `layer` still 1; the dev
panel opens with its remaining 52 sliders and no console errors. Checked by eye
in all four places carbon is drawn — the Sandbox scene, the Sandbox ADD button
thumbnails, the Reactions world and its picker icon, and One Bond's C–O.


### The scene had to come off the floor

Black on a background two values off black is not a colour, it is a hole. The
themes each carry a near-black `bg` and the canvases cleared to a hardcoded copy
of theme A's `#05040f` = `rgb(5,4,15)`. The scene now sits a measured `bgLift`
above whatever the theme's base is — **the theme still owns the hue, the lift
owns how far off the floor it sits** — weighted toward blue the way the base
colours are, so lifting it does not wash the tint out to grey.

Default 34, giving `rgb(20,18,49)` on theme A. Dev slider **Scene lift**; at 0
it reproduces `rgb(5,4,15)` exactly, which is the check that nothing else moved.

**The bar chart's panel had to come with it.** It was a fixed
`rgba(4,3,20,0.88)`, which over the old `(5,4,15)` scene composited to `(4,3,19)`
— all but the same colour, which is why it read as a strip of the background
with a glowing edge rather than a slab. Lifting the scene alone turned it into a
dark rectangle beside a lighter scene. `panelBg()` now derives it from the scene
by the ratios it used to sit at, so it stays a hair darker and a hair bluer than
whatever the scene is, on every theme.

### The glow needed a wider ceiling, and then a compressed one

Two measured problems arrived with the carbon bonds, both from the same cause:
**the old list spanned 436–498 kJ/mol, a factor of 1.14, and the new one spans
358–799, a factor of 2.23.**

**`BOND_SHOWN_MAX`, not `BOND_ENERGY_MAX`.** One Bond's lobe is normalised
against the deepest well *in the dropdown* (799), because on that tab
brightness is the reading and a clipped core throws away the top of the range.
It is NOT normalised against `BOND_ENERGY_MAX`: that constant is the Reactions
currency — `kjPerSim()` is defined in terms of it and every main-tab field
brightness is scaled by it — so moving it to 799 would silently re-denominate a
tab nobody asked to change. The consequence is honest and worth knowing: **the
three original bonds are dimmer than they were**, because they now share a
scale with a bond 1.6× deeper than the old top.

**`bondShownGamma` stops the weakest bond vanishing.** At strict proportionality
a C–O at rest measured `rgb(33,30,40)` against a background of `rgb(5,4,15)` —
all but invisible. Selecting a bond and seeing nothing is not a reading. The
depth term is therefore `(D / BOND_SHOWN_MAX) ** bondShownGamma`, default 0.6,
on the **Depth contrast** dev slider; 1 restores strict proportion.

| bond | D | mag at γ=1 | mag at γ=0.6 | rest colour at γ=0.6 |
|---|---|---|---|---|
| C–O | 358 | 0.448 | 0.618 | rgb(59,56,68) |
| H–H | 436 | 0.546 | 0.695 | rgb(75,76,88) |
| O–H | 463 | 0.579 | 0.721 | rgb(81,82,94) |
| O=O | 498 | 0.623 | 0.753 | rgb(93,85,97) |
| C=O | 799 | 1.000 | 1.000 | rgb(166,169,180) |

The ordering is untouched — a deeper well is always brighter — and the
strongest bond still lands exactly at the top of the ramp, so nothing clips.
What the gamma changes is only how far the bottom of the range falls.

### Grabbing: the whole atom, and no snap

**The handle is the atom, not the nucleus.** `bondGrabR(el, z)` is the drawn
valence radius, floored at `BOND_GRAB_MIN` = 26 px so a small atom at a low zoom
is still comfortably hittable. The faint ring each atom wears is drawn at that
same radius, so the affordance is the truth: the ring is exactly the area that
answers to a pointer. It was a fixed 40 px disc before, well inside atoms that
draw 85 px across.

**The drag works from an anchor, not from an absolute.** On pointerdown the grab
records the pointer's distance from the centre (`grabD0`) and the separation at
that instant (`grabSep`); every move then sets `sep = grabSep + 2·(d − grabD0)/z`.
Setting `sep` to twice the raw distance instead — which is what it did — meant
grabbing anywhere but the exact nucleus jumped the separation by twice the
offset. **Grab the outer edge and the bond snapped open before you had moved at
all.** Measured after the fix: pointerdown followed by a zero-travel move leaves
the separation changed by exactly 0 px on all three pairs and both sides, and
50 px of travel buys exactly `2·50/z` px of separation.

**Both centres come from `bondCentre()`.** The layout and the drag computed the
midpoint separately, and the drag's copy still said `canvas.width/2` after the
bar panel took its slice off the right. On a 1146-wide canvas that put the two
centres 160 px apart, so the first pointer-move snapped the separation to match
the wrong centre — **the larger half of the atom-jumps-out-of-your-hand bug.**
One function now, used by both.

**At an end stop the drag re-anchors.** Without it, pulling 200 px past
`BOND_MAX_MULT` banks 200 px of dead travel that has to be given back before the
bond answers again. Re-anchoring each frame while clamped means reversing
responds on the next pixel: measured, 4.80x pinned, then 10 px back reads 4.637x.

**Where the two discs overlap, nobody owns the pointer.** Bonded atoms overlap —
at the natural length the H–H lens is about 48 px wide — and in it there is no
honest answer to which atom you meant. `bondTabHit` returns `'both'`, the grab is
refused, the cursor goes to `help`, and a tooltip follows the pointer saying
**"grab one atom or the other, drag to separate"**. Guessing a side there would
have made the same gesture do two different things on either side of an
invisible line. There is always room either side of the lens: squeezed to the
0.6x floor, each atom still keeps ~74 px of exclusive grab; pulled past about
2x, the lens is gone entirely.

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

### One energy scale, shared across pairs

```javascript
const SEPARATED_TOTAL = 1000;   // kJ/mol, the common height of free atoms
```

The zero is placed so free atoms sit at `SEPARATED_TOTAL` for **every** pair, so
the reading is an absolute height and what differs between pairs is how far
**down** the bonded state sits:

| | well depth | at rest | fully apart |
|---|---|---|---|
| H–H | 436 | **564** | 1000 |
| O=O | 498 | **502** | 1000 |
| O–H | 463 | **537** | 1000 |

The O=O rests lowest because its well is deepest. That comparison is the whole
point of the scale.

There was briefly a toggle offering the other convention — zero at each bond's
own natural length, so the reading was the work you had done. It is a real way
to measure and answers a different question, but on it **every bond starts at 0
and they all look alike**, which is the one thing this tab should not say. The
shared scale is the one that lets two pairs be compared, so it is now the only
one and the buttons are gone.

**The glow never depended on this.** The lobe's colour follows the work done,
which is the same quantity wherever the zero sits; only the number moves.
Verified the centre pixel at 2x still reads (197,127,253) after the rescale.

### Snapshots

A **Snapshot** button in the energy box stills the bond as it is now and stacks
it in a scrollable column down the right, newest last, so the list reads top to
bottom as the order you took them in. Each row carries the reading as it stood when the shot was taken.

**The crop is a fixed fraction of the canvas, not a box fitted round the two
atoms.** Fitting would scale every thumbnail to fill its frame and so hide the
one thing the series is for: a snapshot at 3x should *look* wider apart than one
at 1x, and it only does if they share a scale. The view zoom already sizes the
scene for the widest separation, so a fixed crop always contains it.

The panel is positioned in JS rather than CSS because its height depends on the
toolbar's, which changes with the tab; it re-fits on resize and hides itself on
the Reactions tab.

Verified: three snapshots at 1x, 2x and 3.5x read 0, 307 and 485 kJ/mol with
thumbnail ink rising 1536 → 2581 → 6126 as the pair spreads; one taken after
switching to the lowest-energy zero reads 102; eight of them overflow 601 px of
panel into 881 px of content and scroll, landing on the newest; clear empties
it; and the panel stays inside the viewport.

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
let BOND_FIELD_WIDTH = 0.22;   // own copy of FIELD_WIDTH
let BOND_FIELD_GAIN  = 0.90;   // own copy of FIELD_GAIN
let BOND_ALPHA_POW   = 1.8;    // opacity taper
let BOND_FIELD_FLOOR = 0.70;   // what a bond at rest looks like
const BOND_FIELD_STAGES = [...];   // its own hues: grey -> purple -> pink
const BOND_COLORMAP  = new Uint8Array(LOAD_BINS * 256 * 4);
```

**Its own colour stages, with a GREY bottom.** The main tab's cold stage is a
blue-white, which is right there — a resting bond in a world of them should
still read as something. Here the bottom of the scale is a bond at its natural
length with *nothing* stored, and that should look like a bond rather than like
an event: a visible glow with no colour in it. The colour is then what
stretching adds, so hue carries the reading instead of decorating it.

**The floor is 0.70, not 0.35.** At 0.35 a bond at rest was a dim smudge. At
0.70 it reads **(169,169,180)** — brightness 518, saturation 11 — about as
present as a 2x stretch used to be. The range above it is then carried by hue
rather than brightness, which is the right division of labour:

| separation | reads | centre | saturation |
|---|---|---|---|
| 1.0x | 0% | (169,169,180) grey | **11** |
| 1.5x | 28% | (218,138,165) | 80 |
| 2.0x | 62% | (197,127,253) purple | 126 |
| 3.0x | 92% | (243,97,217) pink | 146 |
| 3.79x | 99% | (255,92,208) full pink | **163** |

Saturation climbs monotonically while brightness barely moves (518 → 577 →
555).

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
let BOND_FIELD_WIDTH = 0.22;   // own copy of FIELD_WIDTH
let BOND_FIELD_GAIN  = 0.90;   // own copy of FIELD_GAIN
let BOND_ALPHA_POW   = 1.8;    // opacity taper
let BOND_FIELD_FLOOR = 0.70;   // what a bond at rest looks like
const BOND_FIELD_STAGES = [...];   // its own hues: grey -> purple -> pink
const BOND_COLORMAP  = new Uint8Array(LOAD_BINS * 256 * 4);
```

**Its own colour stages, with a GREY bottom.** The main tab's cold stage is a
blue-white, which is right there — a resting bond in a world of them should
still read as something. Here the bottom of the scale is a bond at its natural
length with *nothing* stored, and that should look like a bond rather than like
an event: a visible glow with no colour in it. The colour is then what
stretching adds, so hue carries the reading instead of decorating it.

**The floor is 0.70, not 0.35.** At 0.35 a bond at rest was a dim smudge. At
0.70 it reads **(169,169,180)** — brightness 518, saturation 11 — about as
present as a 2x stretch used to be. The range above it is then carried by hue
rather than brightness, which is the right division of labour:

| separation | reads | centre | saturation |
|---|---|---|---|
| 1.0x | 0% | (169,169,180) grey | **11** |
| 1.5x | 28% | (218,138,165) | 80 |
| 2.0x | 62% | (197,127,253) purple | 126 |
| 3.0x | 92% | (243,97,217) pink | 146 |
| 3.79x | 99% | (255,92,208) full pink | **163** |

Saturation climbs monotonically while brightness barely moves (518 → 577 →
555).

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

## The Sandbox tab

Atoms you place, pull apart and put back together in other combinations, with
the total energy held in every field read out as you go. **No motion at all** —
nothing moves unless you drag it, and energy of motion is not computed.

### The energy

```javascript
const SB_FREE_ATOM = 500;   // kJ/mol carried by one free atom
total = 500 x atoms  -  sum over bonds of  D x (1 - work)
```

Free atoms are the baseline at **half** of One Bond's 1000, because that 1000 is
*two* of them. For two atoms this is exactly One Bond's reading — 1000 apart,
1000 − 436 for an H₂ at rest — so the two tabs cannot disagree. Verified across
the moves the tab is for:

| scene | reads | = |
|---|---|---|
| 2 H₂ + O₂ | 1630 | 6×500 − (436+436+498) |
| pull one H₂ apart | 2066 | 6×500 − (436+498) |
| H₂ + O₂ | 1066 | 4×500 − (436+498) |
| pull the O₂ apart | 1564 | 4×500 − 436 |
| **H₂O + O** | **1074** | **4×500 − 2×463** |
| **H₂CO₃** | **559** | **6×500 − (799 + 2×358 + 2×463)** |
| H₂CO₃ + H₂ + O₂ | 1625 | 10×500 − 3375 |

The sandbox's lobes now go through `bondShownMag` as well, the same compressed
depth scale One Bond uses, because this tab shows the same mixed set of
strengths: a 799 kJ/mol C=O sits in the scene beside a 358 C–O, and against the
old `BOND_ENERGY_MAX` of 498 the double would have been a permanently clipped
white blob while the single went nearly dark. The cost is the same one One Bond
pays — **H₂ and O₂ are a little dimmer than they were.**

### Bonding

**Capped by valence** — H takes one, O two, **C four** — so the sandbox only ever
makes real species and the total cannot be driven down by piling atoms together.
A third H dropped on a finished water is ignored.

The cap counts **bond order, not bonds**: an O=O is a double bond and uses up
both of each oxygen's slots, the way it really does. So an intact O₂ is
saturated and ignores a passing H — you have to pull it apart first, which is the
point of the tab.

**Pairs are ranked by the energy they would release, `D × (1 - work)`, not by
how close they are.** Closeness is the wrong measure and ranking by it was a
real bug: a pair squashed to 0.46 of its natural length is *nearer* than one
sitting exactly right, but it is up the repulsive wall and releases almost
nothing. Ranked by distance, a crushed H–H outranked a perfect O–H and stole its
atom — dropping a third H beside a finished water **pulled one of its bonds
apart**. Ranked by energy released, the water survives.

### Carbonic acid, and the double bond that is not in any table

H₂CO₃ is the first molecule here that needs **two different bonds between the
same two elements**: one C=O and two C–O on the same carbon. No element-pair
table can say which is which, and `BOND_TYPES` cannot either — it holds both
rows, but nothing in it says which row a particular bond in the scene should
take.

So the order is not looked up. **It falls out of what the atoms have left
over.** `sbRebuildBonds` lays the singles down as it always did, and then
`sbPromoteBonds` walks them: any promotable bond whose two atoms *both* still
have a free slot becomes a double. On a carbonic acid that is exactly one bond —
carbon has used 3 of 4, the two hydroxyl oxygens are full (a carbon and a
hydrogen each), and only the bare oxygen still has a slot, so the C=O lands
there and nowhere else. Every atom ends up with a full shell. **That is the
Lewis structure, arrived at rather than hard-coded.**

A pair is promotable only when `BOND_TYPES` holds both a single and a double for
it, which today means C–O alone. O=O has no single in the table, so it is still
made a double up front by `BOND_ORDER` and this pass never touches it; H₂ and
water come through untouched for the same reason.

Because it is a rule and not a template, **it survives being taken apart.**
Measured: pull one H off an H₂CO₃ and the remainder re-promotes a *different*
C–O, giving `C0=O2 C0–O1 C0–O4` — carbon still at 4 of 4. Put the H back and it
is H₂CO₃ again with exactly one double.

### Geometry

Molecules snap into the shape they actually have, but **only when you let go** —
snapping mid-drag fights the hand. The centre is whichever atom holds the most
bonds; neighbours go at the natural length, and two of them open to water's
**104.5°** rather than lying in a line. The arrangement is rotated to sit
closest to where you dragged it, so snapping moves atoms as little as it can,
and neighbours keep the side they were already on so they do not swap.

Verified: a built H₂O lands at **104.5°** with both O–H at **49 px**, the
natural length.

#### Two neighbours: bent or straight, and the atom's own electrons decide

"Two of them open to water's 104.5°" was not a rule, it was water's answer
applied to everything, and **CO₂ is straight**. Water's oxygen keeps two lone
pairs after its two bonds and those pairs take up room — they push the hydrogens
down to 104.5°. Carbon dioxide's carbon has spent all four of its electrons on
the two double bonds, has nothing left over to push with, and is linear.

So the spread is read off the leftover electrons, `VALENCE[el] − Σ bond order`,
rather than by naming CO₂ — the same arithmetic that already decides how many
lone pairs get *drawn* on the atom, so the picture and the shape cannot disagree
about what is left. Any lone pair → bent; none → 180°.

**Both C=O doubles were already right**, which is worth recording because it was
the thing most likely to need fixing and did not. `sbPromoteBonds` places the
singles first and then promotes any bond whose two atoms both still have a slot:
on a CO₂ the carbon has used 2 of 4 and each oxygen 1 of 2, so *both* bonds
promote and every atom ends full. On an H₂CO₃ the same rule promotes exactly one
— the two hydroxyl oxygens are already full with a carbon and a hydrogen each,
and only the bare oxygen has a slot. The Lewis structure arrives rather than
being hard-coded, and it gives a different answer for the two molecules without
being told about either.

Verified on a hand-built CO₂: `C=O` and `C=O`, D **799** each, both at **50 px**,
O=C=O at **180.0°**, carbon holding **0** lone pairs. And nothing else moved —
water still 104.5° with two O–H singles, O₂ still order 2, H₂ still order 1, and
H₂CO₃ still one double with its carbon's three angles at 125.5 / 125.5 / 109.

The world has the same molecule in `MOLECULE_SHAPES.OCO`, and it has to say
*both* things explicitly: a three-atom molecule with no shape of its own gets
water's bend, and `moleculeBonds` would leave the two C–O orders to the
element-pair table, which says single — the same table that could not tell
carbonic acid's C=O from its two C–O. Verified: 180.0°, both orders 2, both
50 px.

**Not yet done, and flagged:** a CO₂ that appears in the *Reactions* tab by
peeling an H₂CO₃ down is assembled from live coordinates, not from this table,
so it keeps the ~125.5° it had inside its parent. Straightening it means
re-snapping a peeled residue to its table shape when one exists, which is a
visible jump at the break and a decision of its own.

**Carbonic acid's angles come off the Lewis structure and VSEPR.** The carbon is
sp² with no lone pair of its own, so its three oxygens are trigonal planar — but
not at a flat 120° each. The double bond takes more room than the two singles,
which opens both O=C–O to **125.5°** and closes the (HO)–C–(OH) between them to
**109°**. Those three are the whole circle, which is the check that they belong
together: 125.5 + 125.5 + 109 = 360. Each hydroxyl's hydrogen then sits at
**106.8°** off its oxygen's O→C direction.

```javascript
const H2CO3_OCO   = 125.5 * Math.PI / 180;   // O=C–O, both of them
const H2CO3_HOCOH = 109.0 * Math.PI / 180;   // (HO)–C–(OH), what is left
const H2CO3_HOC   = 106.8 * Math.PI / 180;   // H–O–C at each hydroxyl
```

Two things the snap had to learn:

**A trigonal centre keeps its double pointing where it points.** The double bond
takes the reference direction and the two singles open `H2CO3_OCO` either side,
so a snap never spins the molecule.

**A second shell.** Carbonic acid is the first molecule here deeper than a
centre and its neighbours — each hydroxyl oxygen carries a hydrogen one bond
further out, and those used to be "left where they are rather than guessed at",
which on a six-atom molecule means they never find their angle at all. Each is
now placed off its oxygen's O→C direction and **turned to the outside**, the
side that oxygen already sits on, so the two hydroxyls open away from one
another. Bending toward the centre instead would fold both hydrogens into the
middle of the molecule.

Measured on a spawned H₂CO₃, after the relax pass has had its say:

| | target | measured |
|---|---|---|
| O=C–O (×2) | 125.5° | 125.5°, 125.5° |
| (HO)–C–(OH) | 109° | 109.0° |
| H–O–C (×2) | 106.8° | 106.8°, 106.8° |
| C=O | 50.0 px | 50.0 |
| C–O (×2) | 58.1 px | 58.1 |
| O–H (×2) | 49.0 px | 49.0 |

Lone electrons, which is the Lewis diagram itself: **C 0, every O 4 (two pairs),
every H 0.** The renderer already derived these from `VALENCE − shared`, so
giving carbon a valence of 4 and the C=O an order of 2 was the whole of it.

### What the ADD buttons place

A species is a **shape** — a list of atoms at offsets from the molecule's own
origin — not a pair of elements. The diatomics were two atoms laid out
horizontally in three separate places (the spawn, the room scan, the button
picture), and a six-atom molecule fits none of them. One list drives all three
now, so the button's picture is necessarily the thing the button places.

That also generalised two things that had the pair baked in: the room scan
insets the legal band by the shape's **own reach on each side** (carbonic acid
hangs further below its carbon than above it), and the thumbnail **fits** the
shape rather than assuming a horizontal pair. A diatomic still comes out at
exactly the old size, because at that size the old 0.8 shrink is what binds.

The thumbnail runs the scene's real bonding rules on the shape in isolation, so
the picture shows the orders the button will actually place — the C=O in the
H₂CO₃ button is a C=O for the same reason it is one in the scene.

Deleting now compares **formulas** against what the ADD buttons make, rather
than asking for "two atoms, the same element". That is what lets a carbonic acid
be deleted too, and it still refuses a lone atom and anything the scene built
for itself.

`formulaOf` has a special case for it: Hill ordering gives CH₂O₃, which is
correct and which nobody writes. It is H₂CO₃ everywhere it is taught, so it is
H₂CO₃ here — the same reason OH is not written HO.

Only **3** carbonic acids fit a 1033-wide canvas before the button greys out,
against 45 H₂, because the molecule is large and the three-radii clearance is
generous. The H₂ and O₂ buttons stay live when the H₂CO₃ one greys: the scan is
per species, and smaller molecules still fit.

### Atoms cannot be squashed

Nothing here has momentum, but a pair closer than its natural length still
pushes apart — `sbRelax()`, three sweeps a frame, each removing 70% of the
overlap — until it sits exactly there. So shoving two atoms together does not
pile them up: they spring back out and settle at the neutral distance, which is
also how a dragged atom ends up bonded at the right length instead of buried in
its partner.

```javascript
const SB_PUSH = 0.35;        // share of the overlap removed per sweep
const SB_SETTLED = 0.05;     // px; below this the pair counts as settled
const SB_RELAX_ITERS = 3;    // sweeps per frame
```

Whatever the pointer holds is **pinned** — the push goes entirely into the free
end, so the hand always wins and the other atom slides away from it. Corrections
are accumulated over a whole sweep and applied together; applying them as they
are found let the order of the pairs decide where a crowded atom ended up.

**A bonded pair's floor is that bond's own `r0`, not the element pair's.** This
was a real bug the moment C=O arrived: a double is genuinely shorter than its
single — 50.0 px against 58.1 — so judged by the element pair it reads as "too
close", and the relax pass pushed it back out to single length on every sweep of
every frame. The snap would place it correctly and the next frame would undo it.
Unbonded pairs keep the element-pair distance, which is the hard-sphere floor
and has nothing to do with what either atom happens to be bonded to.

Two things it needs on top of the plain rule:

- **Exactly stacked atoms** have no direction to separate along, so a pair
  inside 0.001 px is given a random one.
- **An atom shoved into the middle of a held molecule** gets pushes from several
  sides that nearly cancel, and would sit there squashed indefinitely. When the
  net is under `SB_STUCK` = 0.3 px it is ejected away from the centroid of
  whatever is overlapping it, 2.5 px a sweep, which is always a way out.

Measured: an H₂ squashed to 2 px apart is back at **43.17** of its natural 43.20
within two frames, an O₂ at **54.83** of 54.86, and both hold there. An
untouched scene drifts **0.00 px** over 60 frames — no jitter. Worst case, a
water dragged by its bond straight onto a loose O: the O works its way out in
about 25 frames and settles at 62 px, past the 54.9 natural, with the water's
own geometry untouched (104.5°, both O–H at 49.03) and the total back at 1074.

### Dragging: an atom, or a whole molecule

An **atom** is a handle, and so is a **bond** — grab the gap between two atoms
and the molecule they belong to moves as one rigid piece, so you can rearrange a
finished water without pulling it apart one atom at a time. The group is
translation-only and clamped as a whole, so a molecule never deforms against an
edge. A line along the bond under the pointer says the molecule is the handle; a
ring says the single atom is.

The hit test's order of precedence took a fix. Bonded atoms **overlap heavily** —
a bond is 0.72 of the two radii summed — so an atom's grab disc covers the entire
bond, and testing atoms first made bonds unreachable: the midpoint of water's
O–H grabbed the oxygen. Now each nucleus keeps a tight disc that wins outright,
the middle 40% of the bond claims the band between them, and the atom's wide
disc is the fallback:

```javascript
const SB_ATOM_NEAR = 0.45;   // of the valence radius: the nucleus's own disc
const SB_BOND_BAND = 0.3;    // the middle 40% of the bond is the bond's
const SB_BOND_GRAB = 16;     // px from that segment
```

Verified on a water: the O–H midpoint resolves to the **bond**, each nucleus to
its own **atom**. Dragging by the bond moved all three atoms by exactly
(−220, +140) and left the lone O alone, geometry and energy unchanged.

### Range

```javascript
const SB_RANGE = 0.5;
```

The whole interaction profile is half One Bond's, so a pair is fully apart by
about **2.5×** its natural length instead of 4.07× — the span beyond natural,
3.07, halving to 1.54. With a dozen atoms on one screen the long tail meant
everything was faintly bonded to everything; halving it makes a neighbour's
neighbour stop counting.

Scale and rendering are Reactions': same nuclei, same shells, same electron
seating. 0.60 ms/frame with 12 atoms.

### The ADD buttons are pictures

Each button carries a **picture of the molecule it places**, drawn by `sbThumb()`
through the scene's own renderer — field lobe, real nucleus, shells, electrons —
so the button and the thing it makes cannot drift apart. The box needs no
heading once the pictures are there, so the **ADD** label is gone.

An O₂ at rest is 131 px wide including both halos, so the thumbnail is drawn
1:1 into a 164×104 box and shown at 82×52: a 2× asset for free. Everything is
drawn at `SB_THUMB_SHRINK` = 0.8 about the centre, because at full size the
halo was cut off at the edge instead of falling away. The field is blitted with
`drawImage`, so it follows that transform along with the atoms.

Two things it has to be sequenced around: the field buffer's resolution comes
from the quality tier, so the thumbnails are painted **after** the startup
benchmark; and the heatmap resizes that one shared buffer to whatever box it is
handed, so they are painted **before** the world loop starts rather than
between its frames.

`sbDrawValenceElectrons(c, A, bonds)` takes its context and arrays as arguments
for this — it used to read the sandbox's own globals, and the thumbnails are not
the sandbox.

### Electrons: one shared pair per unit of bond order

A bond holds **two electrons per unit of order**, one of each pair from either
atom and drawn in that atom's colour — two dots in an H–H, **four in an O=O, two
from each oxygen**. The pairs are stepped along the bond axis so a double bond
reads as two pairs side by side rather than four dots in a line.

What is left over is lone, and the count is taken from the **order** too, not
from the number of bonds: an electron per unit of order has gone into the gap,
so an O in an O=O keeps four, not five. (It was five — `VALENCE − bonds.length`
is only right for single bonds.)

Counted off the renderer with 1 O₂ + 1 H₂ on screen: 18 electron dots — 4 in the
O=O gap, 4 lone on each oxygen, 2 in the H–H, and 2 inner-shell per oxygen.

### Room to place: three radii, or the button greys out

A new molecule never lands on or beside what is already there. Every one of its
atoms must sit **`SB_ROOM` (3) × the larger of the two valence radii** clear of
every atom in the scene — about 90 px for a pair of hydrogens, twice their own
bond length, and `SB_RANGE` has the interaction profile faded out well inside
that, so a fresh molecule reads as zero against everything else.

`sbRoomFor(els)` **scans the whole legal area** on a 12 px grid rather than
guessing a spot and retrying. That matters for the greying out: "there is
nowhere left" is then a fact about the scene, not about how many times we rolled
the dice. Among the spots that clear, the one nearest a walking ring position
wins, so repeated presses still fan out around the middle instead of filling the
scan order corner-first.

When the scan comes up empty, `sbSyncAddButtons()` **disables that ADD button**
and retitles it. It is greyed, not hidden, so the row never reflows and the
picture still says what the button is for. The two buttons are tested
separately: an O₂ demands a bigger gap than an H₂, so it can grey out first.
Re-tested after every spawn, drag, nudge, delete, clear, tab switch and resize —
a narrower canvas can run the scene out of room on its own.

Measured on a 1242×785 canvas: 45 H₂ fit before both buttons greyed, and the
closest pair in different molecules sat 91 px apart against a 90 px requirement.

### Selecting: a dotted outline, the arrow keys, and the X

A click that does not move the pointer (`SB_CLICK_SLOP`, 4 px) **selects**
instead of dragging. The target follows the same precedence the drag handles
do — the middle band of a bond means the whole molecule, an atom's own nucleus
means that atom — so there is one rule on screen, not two:

- **Molecule** → a dashed rounded box around the group, padded clear of the
  element letters that hang under each atom.
- **One atom** → a dashed ring around that atom. A ring, because a box around a
  single atom reads as "this little molecule".

Either way the **arrow keys** then nudge what is outlined, 4 px a press and 16
with shift, clamped as a group so a molecule keeps its shape against an edge.
That is the only way to place something to the pixel; a drag is as precise as
your hand. Nudging deliberately does **not** snap — putting an atom a few px off
its natural length is a legitimate thing to want, and it is what makes the
energy readout move under your hand. Escape drops the outline; Delete is the X's
keyboard twin.

A drag leaves the selection alone, and the selection is stored as an **anchor
atom, not a list** — the molecule is resolved from it every frame, so pulling an
atom off a selected water re-reads as the smaller molecule instead of outlining
a group that no longer exists.

**The X only appears on a molecule you added and have not changed**: two atoms,
same element, bonded — exactly an intact H₂ or O₂. Products are not deletable,
and neither is a lone atom: a reaction cannot lose atoms halfway through, and
letting one vanish would quietly break the energy accounting this whole tab is
about. Clearing out a reactant you decided you did not want is a different
thing, and that is all this is for. `sbDrawSelection` records the badge's box
where its geometry already is, so the click test cannot drift from what is drawn.

## Recombination — the other half of the reaction

Once the scene has come apart, the atoms find each other again and make
**water**, which is the reaction this whole sim is pointed at: 2 H₂ + O₂ → 2 H₂O.
Verified end to end — a default scene run through breaking and then
recombination finished as **4 H₂O and nothing else**.

**It only opens once every original molecule has come apart**, and the condition
is `world.bodies.every(b => b.els.length === 1)`, latched the first frame it is
true. The latch is the whole trick: forming the first OH makes the condition
false again, and the gate must not slam behind it.

Only the two steps towards water pull on each other:

```
H + O -> OH        OH + H -> H2O
```

Not H + H, not O + O — hydrogen and oxygen going back to H₂ and O₂ is the
reaction running backwards and there is nothing to watch in it. The bonding site
is the oxygen where there is one (in an OH it still has a slot free), otherwise
the atom itself.

### Distances are in contact widths, not bond lengths

This is not a detail, and it is why the first attempt never formed anything in
20 seconds of trying. A bond is 0.72 of the two radii summed, so **bonded atoms
overlap** — and the collision solver treats them as hard spheres right up until
they are one body. An O and an H touch at **68 px** and their bond is **49 px**,
so waiting for bond length waits forever. Capture therefore happens **on
contact**: a slow enough touch sticks instead of bouncing, and the new molecule's
own geometry then pulls the pair in to 49.

```javascript
let recombRange   = 1.6;    // x contact width: where the pull starts
let recombPull    = 420;    // force, px/s^2 at mass 1 — an H feels it 16x an O
const RECOMB_SNAP = 1.03;   // x contact width: close enough to stick
```

The pull is a **constant force inside the range**, which makes the field
conservative: whatever it puts in on the way down it takes back on the way out,
so a near miss costs the books nothing. What it does put in while a pair is
closing is tallied as `pullWork` and handed back at the join, or dropped when the
pair drifts apart.

**Capture needs the bond to be able to hold them**: the pair's relative kinetic
energy must not exceed the bond's own energy. Two atoms crashing together faster
than that do not stick in the real world either — they have nowhere to put the
excess. Measured on a live scene, a typical approach was 53,000 against an O–H
bond's 181,000, so ordinary encounters capture and only the hardest bounce off.

### The pull comes out as spin, a glow, and speed

```javascript
spinE = KE(a) + KE(b) + released + inherited - KE(drift) - pullWork
```

Momentum fixes the drift, so the drift's share is not ours to give. `inherited`
is whatever vibration the two were already holding — see *Nothing is dropped on
the way in* below. What is left is split three ways, in this order:

1. **The tumble**, at the rate the pair's own angular momentum about their common
   centre of mass asks for, capped at `MAX_SPIN` like every other spin in the sim.
2. **The bond's glow** — `joinVibCap` (0.4) of what would break that bond, parked
   in `vib`, so a molecule that has just formed is visibly hot and `VIB_TAU`
   hands it back to motion over the next couple of seconds through the path that
   already exists for it. `fieldsKJ` counts `vib`, so the bars balance either way.
3. **Speed**, for anything past that cap: the velocity is scaled, not added to,
   so the direction the pair was already drifting survives.

It all used to go to the tumble alone, with its direction taken from
`a.spin + b.spin >= 0` — which for two free atoms is zero and so decided nothing,
and at a rate well past the `MAX_SPIN` its own comment calls *a blur, not a
spin*. A water could come out turning the opposite way from the pair that made
it. Measured join spins: **9.6–16.3 rad/s** before, **0.06–9.2** now, never over
the cap.

`makeBody` grew a **bent triatomic** for this: a three-atom chain opens to
`H2O_ANGLE` about its middle atom instead of lying in a row, and the body-local
origin moved from the midpoint of the row to the **centre of mass** — the same
point for an H₂ or an O₂, so nothing that existed before moved, but not the same
for an OH or a water, and the energy split above is about exactly that point.

**Water's pivot is exaggerated on purpose.** Oxygen is 16 of water's 18, so the
true centre of mass sits **3.34 px** from the O's nucleus towards the two
hydrogens — 11% of the way to the midpoint between them — and a spin about it is
indistinguishable on screen from a spin about the oxygen itself. The whole point
of watching a water tumble is that it does *not* turn about the big atom, so
`waterPivot` walks it out to **0.35**, putting the pivot 10.5 px off the nucleus
and sending the oxygen round a **22 px circle** instead of a 6.7 px one. Dev
slider, and `WATER_PIVOT_TRUE` (0.111) is marked `(true)` on it so the honest
value is one drag away. Like the bar chart's raised zero, the number is wrong
and the behaviour it shows is right.

Nothing in the accounting pays for it. `inertia` is summed about whatever origin
the body is built with, so the impulse solver, `energyOfMotion` and the collision
response all use the same reference point and the same quadratic form. Measured:
six waters, strong water on (so every hit is elastic), speed band off, 15 s of
collisions — `energyOfMotion` finished on **exactly** its starting value, pivot
at 0.35. What the offset really costs is subtler and is not energy: a free rigid
body's rotation is only decoupled from its translation about the true COM, so
this is a body spinning about a point it would not quite spin about. That is the
distortion, and it is the one being asked for.

`rebuildWaterPivots()` moves the pivot of every water already in the world when
the slider changes — otherwise nothing happens until the next join and the
slider cannot be judged by eye. The atoms stay exactly where they are on screen:
the body's position absorbs the shift of its own origin, rotated into world
space.

Fields falls and motion rises as it runs, because fields is One Bond's
expression and forming a bond pays part of what the atoms owed: measured
**6,000 → 2,436** over one run while the motion bar rose by the same, with the
bar total constant at 19,158 across 34 joins and 22 breaks. A scene also reaches
a real equilibrium — water forms, gets hit hard, comes apart, re-forms.

Note the straggler problem: `breakSlow` cools the world as it breaks, so the last
intact molecule can take a very long time to go, and until it does the gate stays
shut. Sometimes all six break in 5 s, sometimes five break and the last holds out
past 20 s.

### Where the molecule goes: the placement that moves the atoms least

The new molecule's angle used to be set from the line between the two bonding
sites, which is right for nothing:

* For an **H + O**, the product `['O','H']` has its local O→H along +x, so
  whenever the H happened to come first in `world.bodies` the molecule was built
  with its hydrogen on the **far side** of the oxygen. A 180° flip, decided by
  array order.
* For a **water** it cannot be right at all. The bisector is +x and both
  hydrogens sit `H2O_ANGLE/2` off it, so pointing the bisector at the incoming H
  threw that H 52° from where it was and swung the OH's existing one 104° across
  the molecule.

It is solved for instead. Given the product's local atom coordinates **p**, the
world positions **q** those atoms are actually at, and weights **w**, the
rotation and offset that minimise the weighted sum of squared movement are exact
and closed form — the 2D case of Kabsch, two sums and an `atan2`:

```javascript
theta = atan2( SUM w (px*qy - py*qx), SUM w (px*qx + py*qy) )
```

once both sets are centred on their weighted means. Weighting by **mass** is
what keeps the oxygen still and moves the hydrogens: it is sixteen times harder
to shift. `joinAssignments` enumerates the ways of matching product atoms to
real ones — one for H + O, two for OH + H since either hydrogen could take
either of water's slots — and the closer fit wins.

Measured, head-on H + O at twelve angles in both array orders: the oxygen moves
**1.1 px**, the hydrogen **17.9 px**, the final bond is **49.0 px** and the
final direction is within **0.0°** of the direction the H arrived from, in every
case. Ideal-angle OH + H: existing hydrogen **0.9 px**, new one **17.9**,
oxygen **1.1**, bond angle 104.5°.

### The slide in

Best fit or not, the atoms still have to close from contact to bond length — an
O and an H touch at 68 px and bond at 49 — and **that distance is the jump**. So
the body is born holding each atom's leftover gap as an offset in its own frame,
and the offsets decay to zero over `joinFormSec` (0.18 s, 11 frames).

They are applied by **mutating the atoms' local coordinates**, which is the whole
trick: the bond segments, the valence shells, the collision radius and the
drawing all read those coordinates, so every one of them follows the closing bond
without knowing anything about it. Smoothstep rather than linear, so the atoms
arrive with no velocity rather than stopping dead — frame by frame the arriving
bond runs 68.0, 67.5, 66.3, 64.4, 62.1, 59.6, 56.9, 54.4, 52.2, 50.4, 49.3,
49.0, while the existing O–H sits at exactly 49.0 throughout.

`inertia` is left at the finished molecule's value for the ~0.2 s this lasts. It
is wrong by about a third on the first frame and exactly right on the last, and
correcting it would make the spin the join just handed over wrong instead.
Checked for the obvious symptom — over two 90 s runs, **not one** of the 1,638
frames in which some body exceeded `MAX_SPIN` belonged to a forming molecule;
they were all spawn-time H₂ and band-capped waters, which the baseline does too.

**Measured jump at the moment of joining: 1.3e-13 px**, i.e. floating-point
zero, over 46 runs. Before: 0, 17.9, 19.0, 19.6, 55.5, 56.9, 57.7, 59.7 and
**109.5 px** (that last one the 180° flip).

### The little rotation, so the collision fits

While the pull is drawing a pair together, a molecule that already has a bond
turns its open slot to face what is coming. `slotLocalDirs` gives the two
directions an incoming atom would have to arrive from for the product's shape to
fit — water's bond angle either way round from the bond already there — and
`alignTarget` picks whichever is the shorter turn. A lone atom gets nothing: H +
O makes a straight molecule, so every orientation fits it equally.

It is **kinematic**. The body's `spin` is untouched and only its drawn rotation
is redirected, so a pair that drifts apart again resumes exactly the tumble it
had and nothing has leaked. Treat it like the water pivot: the mechanics are
nudged so the picture reads.

`recombAlign` is measured, not guessed — the angle between an OH's open slot and
the hydrogen arriving at it, over ten 90 s runs (40 water joins) each:

| rad/s | mean | worst |
|---|---|---|
| off | 35.4° | 80.7° |
| 14 | 8.4° | 38.2° |
| 20 | 5.6° | 44.7° |
| **32** | **3.7°** | **9.2°** |
| 40 | 5.7° | 72.7° |
| 60 | 3.3° | 7.2° |

The mean is flat past about 20 and the worst case is noise: it is the occasional
hydrogen that enters the range already touching, which has no time to be turned
whatever the rate. So the rate is chosen on how it **looks** — at 32 a full 104°
turn takes 57 ms, three and a half frames, which reads as a molecule turning. At
60 it is under two frames and the turn is no more visible than the jump it
replaced. Every one of those 46 runs still ended with four waters.

### Nothing is dropped on the way in

Two accounting holes surfaced once the join started handing energy over. Both
predate this work; both were invisible until it did.

**The pull's work was a path integral that never gave anything back.**
`pullWork` was accumulated as `F × (distance closed)` every frame a pair was
approaching, and only cleared when the pair left the range entirely — so a pair
that came in, drifted out a little, came in again and finally stuck was charged
for the same ground twice. The surplus ran `spinE` down to its `max(0, ...)`
floor and the difference left the world. Harmless while pairs flew straight
through; not harmless once `recombAlign` keeps them in range long enough to
wander. A constant inward force is conservative, so its work is now read off as
a **potential difference** — `F × (entry separation − separation now)`, kept per
pair in `pullEnter` — which is what the tally was always described as being.

**The molecules' own `vib` was thrown away.** Nothing used to put vibration into
an OH, so dropping it at the water step cost nothing and nobody noticed. Now
that a join hands some over, the OH arrives holding about **185 kJ/mol** of its
own, and that was going out of the world every time.

With both fixed, a join's books close exactly. Measured over three 90 s runs,
the energy a join does not grant is **the pull's own work and nothing else** —
`join loss + pullWork = 0` to the round, every run, where it had been −1,113 to
−1,247 kJ/mol a run unexplained. Across a whole run the anchored total and the
measured one now disagree by 4,900–6,500 kJ/mol where the old tally put that at
6,900–10,200.

### What the sliders are

All three are in the dev panel under Recombination:

| slider | variable | default |
|---|---|---|
| Snap glides in | `joinFormSec` | 0.18 s |
| Turns to fit | `recombAlign` | 32 rad/s |
| New bond keeps | `joinVibCap` | 40% of break |

`joinVibCap` is well under 1 on purpose: a freshly made OH holding its whole
bond energy would be one ordinary collision from coming apart again, and the
reaction would never reach water.

### Strong water

A toggle in the Reactions tab's pink box, **on by default** (see Temperature for
why it ships on alongside Low). With it on, a water molecule is the
end of the line: **it cannot break, and it cannot take energy into its fields
either**, so a hit on one is perfectly elastic and bounces away with everything
it arrived with. Same button shell as the temp buttons, so nothing in that box
shifts when it turns on.

Those two halves are **one rule, not two**, and the implementation is one line:
water is left out of the collision's `storers` count in exactly the way a free
atom already is.

- A bond only ever breaks because its field filled past the threshold, so a
  molecule that stores nothing can never break. The break half comes free.
- The converse matters more: a molecule that could not break but still stored
  would be a **sink that swallows motion and never gives it back**, dragging the
  whole world cold. With no storer on either side the restitution stays at 1, so
  the energy never leaves the motion in the first place — nothing is deleted and
  the bar chart's total still holds.

Flipping the toggle on hands back whatever the waters are already holding, via
`returnFieldEnergy`, rather than stranding it in a field that can no longer be
refilled or released. Measured on a mid-run flip: fields 2,992 → 2,810 and
motion 17,113 → 17,295, total unchanged at 20,106.

**The flash is a flash.** An ordinary bond's flare fades exponentially
(`FLARE_TAU`, ~1.2 s to 2%), which is right when the colour is also reporting
the energy the bond took in — the two tail off together. A strong water took
nothing in, so its colour has nothing to report once the contact is over, and a
slow exponential tail would read as a bond still holding something. It goes out
**linearly inside a fixed window** instead, so `strongFlareSec` (dev slider,
default 0.5 s) is literally how long the pink can last. Measured by driving the
clock a frame at a time: 0.25 → 0.267 s, 0.5 → 0.517 s, 1.0 → 1.000 s, 1.5 →
1.517 s, half gone at 0.267 s of a 0.5 s flash. The same water with the toggle
off takes 1.183 s to fall to 2%.

Measured over a 45 s run with it on: 8 joins, 4 waters, **not one water lost**,
every water's `vib` exactly 0 throughout, bar total constant at 18,360. The same
scene with it off broke a water 5 times and ran waters up to 1.22 doses of field
energy.

## The energy bar chart

Two bars down the right-hand side of the Reactions tab — **Energy of Motion**
and **Energy in Fields** — ported from fusion's panel: same dark panel, same
left-edge glow, same dim fill under a glow under a crisp neon outline, dashed on
the field bar, labels in two lines below the baseline. What changed is the
contents: **two bars, never three.** Fusion could split its field bar into
Electric and Nuclear; a bond's field is one thing, so there is nothing here to
split and no toggle for it. And the units are **kJ/mol**, not keV.

The panel is 320 px and `syncWorldBox` takes that off `world.w`, so the walls
stop at its left edge and nothing can be hidden behind it — the same rule that
keeps the box above the energy bar.

### It is on all three tabs

The panel is permanent now, and `barChartSpec()` is the one place that says what
it shows. Every tab reads the same quantity in the same units — kJ/mol, One
Bond's scale, 500 to a free atom — so a bar of a given height means the same
thing wherever you are, and switching tabs does not switch rulers. What differs
is the full scale, and in each case it is **the state that tab measures
against**:

| tab | bar | full scale | reads |
|---|---|---|---|
| One Bond | Fields | `SEPARATED_TOTAL`, 1000 | two free atoms. The pair sits down from the top by the depth of the well you picked — 564 of 1000 for a resting H–H, up to 874 at 2.2× natural |
| Sandbox | Fields | 500 × atoms in the scene | every atom free. 1,066 of 2,000 for an H₂ + an O₂; the gap to the top is what the bonds have discounted |
| Reactions | Motion + Fields | the run's constant total | unchanged |

Only Reactions gets a motion bar, because only Reactions has motion. The other
two are static instruments — nothing moves unless you drag it — so a second bar
there would be a permanent zero. A lone bar keeps the width it would have in the
two-bar layout and is centred, so the Fields bar is visibly the same object on
all three tabs rather than a fat slab on two of them.

Scale is per tab, but never per MOMENT: nothing here rescales as the value
moves. The One Bond bar is always a fraction of 1000, so pulling a bond apart
raises the bar instead of redrawing the axis under it. The Sandbox ceiling does
move, but only when you add or delete an atom, because the ceiling IS the atom
count.

**`sceneW()`** is what made this cheap. The canvas is still the whole window —
the background and the field buffer fill it — but everything that places
something in the *scene* now measures `canvas.width − BAR_PANEL_W`: the walls,
the One Bond pair and its zoom, its snapshot crop, where a sandbox molecule may
be dropped, and every sandbox clamp. The snapshots column moves left of the
panel for the same reason. Before this, that subtraction existed only inside
`syncWorldBox` and only on Reactions.

### The zero is raised, and it says so

Honest bars are honestly useless here: at High a molecule carries about
2990 kJ/mol of motion and its bond is worth 436, so breaking every bond in the
scene moves 2740 of 20,649 — the motion bar twitches by a tenth and the story is
lost. So **both bars are drawn from a raised zero**, and the panel prints
`zero raised N kJ/mol — this is the change, not the total` under the title. A
distorted axis that does not admit it is a lie; one that does is a magnifying
glass.

```javascript
let barZeroAt    = 0.96;   // of the all-bonds-broken motion
let fieldsZeroKJ = 0;      // the field bar reads big already
motionZero = barZeroAt x (totalKJ - SB_FREE_ATOM x atoms)
```

The motion zero is a fraction of the motion the run would **still** have with
every bond broken — `totalKJ` less a free atom each — which is the lowest the bar
can go while bonds are the only thing spending. So the fraction doubles as a
promise about what survives: at 0.96 the bar keeps 4% of that floor, and no
composition or temperature can push it through zero. Defined off `totalKJ`, it
needs no anchoring of its own; the temperature carries it. Both are dev sliders
("Motion zero at", "Fields zero") because they are tuned against the picture.

Measured, default scene at High:

| | raw | with the raised zero |
|---|---|---|
| motion, start → all broken | 17,389 → 14,649 (−16%) | **3,362 → 622 (−82%)** |
| fields, start → all broken | 3,260 → 6,000 | unchanged, it needs no help |
| start ratio motion : fields | 5.3 : 1 | **1.03 : 1** |

The shown sum stayed at **6,622 every sample** across a whole run: what is taken
off is a constant, so the total still is.

### The exchange rate

Sim units are pixels and masses where H = 1; One Bond's scale is kJ/mol. They are
pinned together at the one place they actually touch: **a collision that just
breaks a bond has, by definition, delivered exactly that bond's dissociation
energy.**

```javascript
D = kJperSim x breakEnergy x (D / BOND_ENERGY_MAX) x doseUnit()
  =>  kJperSim = BOND_ENERGY_MAX / (breakEnergy x doseUnit())      // = 0.00256
```

D cancels, which it has to — otherwise an O=O and an H–H would disagree about
what a kJ is. `launchSpeed()` carries no temperature, so `doseUnit()` and this
rate are both constants: changing the temperature moves energy about, never the
exchange rate.

Verified by loading one bond to its break threshold and reading the total:

| scene | at rest | half loaded | at the threshold |
|---|---|---|---|
| one H₂ | 564 | 782 | **1000** |
| one O₂ | 502 | — | **1000** |

1000 is two free atoms, so a bond loaded to breaking has had its whole
dissociation energy put in — for both pairs, off one rate.

### Fields is One Bond's number, exactly

```javascript
fields = SB_FREE_ATOM x atoms  -  sum over bonds of D  +  stored
```

Every atom carries 500 on its own and each bond discounts its dissociation
energy by what it is still owed, so **breaking a bond makes fields go UP** — the
remainder gets paid. Cross-checked against One Bond: a lone H₂ reads **564**, a
lone O₂ **502**, the default scene (4 H₂ + 2 O₂) **3260**, all exact.

### The total is a constant, and motion is the remainder

`totalKJ` is anchored once per spawn at `fields + kJperSim x energyOfMotion()`,
and moves only when the temperature buttons genuinely put energy in or take it
out (`stepTemperature` adds `kJperSim x ΔKE`). `motionKJ()` is then
`totalKJ - fieldsKJ()`. So for a given collection of molecules at a given
starting speed the two bars always add to the same number. Measured over a run
at High: the sum sat at **20,649 every sample** while fields climbed 3260 → 5565
across five breaks; held constant again at 16,683 after switching to Low, and
tracked back up on the way to High.

**Motion is derived rather than measured, and that is the trade-off to know
about.** Measured, it does not balance, because two things in the sim are
deliberately not energy accounting:

| | measured | what the bars say |
|---|---|---|
| dissociation, 5 breaks | 11,626 kJ/mol spent | 2,180 (5 bonds' worth) |
| speed band floor top-ups | +1,743 kJ/mol from nowhere | not counted |

`breakSlow` drops a fragment to a quarter of its speed — about 94% of its
energy, where the bond it lost was worth 436 of the ~2990 kJ/mol an H₂ carries
at High. That dramatisation was asked for and the molecules on screen still do
it; what it is not is accounting. So **after bonds start breaking the molecules
hold less energy than the motion bar says.**

`exactBreakCost` (dev panel, "Dissociation costs exactly D") is the switch that
makes them agree: the fragments keep their inherited velocities, the collision's
over-delivery `(vib - threshold)` comes straight back out as motion, and only
the velocities *relative* to the pair's centre of mass are scaled to land on the
figure — scaling the whole velocities would undo the momentum the collision just
settled. It also has to pin each fragment's `vTarget` to the speed it actually
leaves with: left at its species' nominal, the speed band treats every free atom
as under-speed and tops it up, measured at **28,000 kJ/mol across one run** —
worse than the dramatisation the mode exists to remove.

With it on, measured motion came within **28%** of the bar (10,645 vs 14,789)
against **84%** off; the rest is the speed band. **Default off** — it halves the
visible slowdown, which was tuned on purpose.

## Energy graph

**Hidden by default** — SHOW LINE GRAPH on the energy bar brings it back,
along with the key and the percentage readout, which share its `.visible`
classes.

Showing or hiding it changes the energy bar's height, which moves the bottom
wall, so `setGraphShown` calls `restretchWorld()` — the same re-fit the window
resize uses. Without it the world keeps its old box and leaves a gap between
the bottom wall and the bar.

### The x axis is not time

It was, and a scrolling time trace was the wrong picture. What the two bars are
doing is a **reaction**, and a reaction's natural axis is how far along it is —
reactants on the left, products on the right, with the energy hump in between.
The sim produces that diagram for free, because `fieldsKJ` really does rise as
bonds come apart and fall as they go back together.

The time trace is **gone**, and with it the energy-conservation check it used to
serve (a flat line meant the solver did not leak). That check lives on in
`energySplit()` in the dev diagnostics, which is where measurement belongs.

**Progress is in two halves, with ALL ATOMS FREE pinned at the middle**, because
that is genuinely the shape of the thing: the recombination gate will not open
until every molecule has come apart, so the breaking finishes before the joining
starts.

| | coordinate | counts |
|---|---|---|
| left, 0 → 0.5 | `0.5 × free / atoms` | the share of the world's atoms that are loose |
| right, 0.5 → 1 | `0.5 + 0.5 × bonds / productBonds` | the share of the products' bonds that have formed |

The left half counts **atoms**, not break events, on purpose: a peel frees one,
a shatter frees several, a double bond takes two hits, and counting atoms gets
all of that right with no table of how many breaks a molecule is supposed to
take. The right half counts **bonds** for the same reason — every bond in the
world after the latch is a product bond, since everything was a free atom a
moment earlier.

Measured, one water run end to end: 14 landmarks at 0, 0.083, 0.25, 0.333,
0.417, **0.5** (12 free atoms, `recombLatched` true), 0.563 … 1.0 (4 waters).
The fields fraction climbs 0.658 → **0.930** at the midpoint and falls to 0.394,
monotone on both sides — a clean hump whose products sit lower than its
reactants, which is what exothermic looks like.

### A reaction with no product half yet

`REACTIONS[*].productBonds` is how many bonds the finished products hold.
Carbonic's is `null` because its product half is **not coded yet**, not because
it does not have one — carbonic acid will recombine. Until it does, progress
stops at 0.5: the curve climbs to the peak and ends there, markers and all, with
the right half of the axis empty under PRODUCTS. That is the honest picture of a
model that knows how to take the molecule apart and not yet how to rebuild it,
and the caption under the graph says so. Filling in `productBonds` alongside
`recombine: true` is all the graph needs to run the second half.

### One sample per step, taken the first time the reaction gets there

Progress moves in steps, so the trace is a short list of landmarks, not a
stream. Each is recorded **once**, when the reaction first reaches it, and never
rewritten: the curve is the path the reaction took, and a path does not change
behind you. Nothing is ever removed, so the drawn curve only grows rightward.

When the reaction runs **backwards** — strong water off, a finished water
knocked apart — the curve stands and the marker slides back down it. Measured:
a run reached 1.0 at 14.9 s with 13 landmarks, fell back as far as 0.813 and
spent 5,484 frames below 1.0, and finished with the same 13 landmarks and a
curve still reaching 1.0.

`traceAt()` interpolates between landmarks, because a shatter can free three
atoms at once and the levels it skipped were never visited.

### Stored as bar fractions, not kJ/mol

A landmark holds each bar's **fraction of its own chart**, not its kJ/mol. The
two are the same picture only while the scale holds still, and it does not: the
temperature buttons move `totalKJ` for the whole world at once, so a landmark
recorded at Low and one recorded at High are drawn against different scales, and
absolute kJ puts a step in the line that is about the thermostat rather than
about the reaction. A fraction is also what *"follows the bar chart positions"*
actually means — a point on the line sits at the height of the bar beside it.

A consequence worth knowing: `motionKJ = totalKJ − fieldsKJ` and the chart's
full scale is `totalKJ` less the two raised zeros, so **the two lines are exact
mirrors** — verified, they sum to 1.000 at every landmark. That is inherent to
the accounting, not a tuning choice.

### Drawing

A **Catmull-Rom spline** laid down as beziers, tension 1/6, end segments reusing
their own endpoint for the missing neighbour. The curve passes through every
landmark with no corner at any of them, which is the point: the reaction moves
in steps and the line that shows it should not look like it does.

Labels: `REACTANTS` left, `PRODUCTS` right, `REACTION PROGRESS` centred under
the axis — and **only that one is centred**. The halfway mark carries no words
at all, just its dashed line. Three versions of naming it were built and all
three came off: stacked under the title it read as a two-line caption, running
up the dashed line it clipped, and sitting on the line in a backed pill it was a
label looking for somewhere to not be in the way. The dashed line says where the
midpoint is on its own.

### Smoothing is done to the data, not to the spline

The spline already has no corners — it is C1 through every landmark by
construction — so the wiggles left in the line are not a drawing problem. They
are in the numbers: each landmark is a single snapshot of a noisy world, taken
at the one frame the reaction crossed that step, so neighbouring landmarks
disagree by more than the reaction itself moved. Raising the spline's tension
would only bulge those wiggles, not remove them.

So a **Laplacian pass** over the landmark heights — each pulled towards the
average of its two neighbours — run more times the further `graphSmooth` goes.
Dev panel, under Line graph, default **0.35**.

**Smoothing harder means more passes, never a heavier one.** The weight per pass
is fixed at a half, because at a full weight a point is not pulled towards its
neighbours' average, it is *replaced* by it: the odd and even landmarks stop
talking to each other and the pass oscillates between two interleaved sets
instead of diffusing. The first build had the slider set the weight, and the
measured wiggle fell 36.7 → 14.0 → 12.2 → 11.6 → 11.0 as it went up and then
**jumped back to 22.6 at the very top**. That last number is the oscillation and
nothing else.

The trade is hump height against wiggle — measured on one finished water run,
wiggle as mean |second difference| × 1000, hump as the peak's rise above the
reactants:

| `graphSmooth` | hump kept | wiggle |
|---|---|---|
| 0 (raw) | 100% | 35.3 |
| 0.2 | 86% | 15.4 |
| **0.35** | **80%** | **14.4** |
| 0.6 | 72% | 13.0 |
| 1 | 60% | 11.4 |

The knee is at about 0.2 — nearly all the noise goes for 14% of the hump, and
past that you pay hump for very little. The wiggle never reaches zero because
the hump itself has curvature; 11 is the shape, not noise, which is why even the
top of the slider leaves a reaction curve rather than a straight line.

Three things the pass does not touch:

- **The ends are pinned.** The reactants' energy and the energy at the furthest
  point reached are real readings about real states. Smoothing is here to take
  the noise out of the path between them, not to move them.
- **Heights only, never x.** Those are the progress levels, and 0.5 means every
  molecule is in pieces. That one is not up for averaging — verified, the peak
  stays at p = 0.5 at every setting.
- **The mirror survives.** Motion and fields sum to 1 at every landmark and the
  pass is linear and identical on both, so the smoothed pair still sums to 1.

The markers read the smoothed heights too, so they sit on the line rather than
beside it.

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

GitHub Pages **is** enabled — legacy build from `main` at `/`, status `built` —
so a push publishes to **https://ghostoutfit.github.io/reactions/**. Check it
with `gh api repos/ghostoutfit/reactions/pages` rather than assuming; this file
said it was off for a while after it had been switched on.

This sim is deliberately **not** on the SSP site. It was briefly built to
`sim.ssp.org/reactions/` and that was removed: reactions lives only in
ghostoutfit.
