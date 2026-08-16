# Scroll-driven 3D on the web — a teardown of Kage, and a playbook

Notes taken while reading [MengTo/kage](https://github.com/MengTo/kage) line by line, plus the
patterns worth carrying into future frontends.

---

## 1. What Kage actually is

The first thing to get straight, because it changes how you'd plan a project like it:

**It is not React Three Fiber. There is no React, no npm, no bundler, no build step.**

| | |
|---|---|
| Structure | One `index.html`, 4,821 lines (238 KB) |
| Library | `three.min.js` r149, **vendored as a file** |
| Repo | 22 files, 3.46 MB total |
| Build | None. `python -m http.server` and it runs |
| Breakdown | ~875 lines CSS · ~320 HTML · ~3,600 JS |

Where the JavaScript goes:

| Lines | What |
|---|---|
| ~1,000 | Procedural canvas textures — `texWall`, `texRoof`, `texMoon` (crater fields + maria), `texShoji`, `texLacquer`, `texStone` |
| ~900 | Procedural geometry — `buildTorii`, `buildTemple` (bracket sets), `roofGeo` (flared curve), recursive `buildMaple` |
| ~400 | Hand-rolled bloom — mip pyramid, separable blur, additive upsample. No `EffectComposer` |
| ~300 | A GPU cloth simulation — verlet nodes, custom shader programs |
| ~250 | Performance governor, DPR scaling, aspect correction, reduced-motion |
| ~40 | **The entire scroll-camera engine** |

That last row is the important one.

---

## 2. The engine is 40 lines

Everyone assumes the scroll-camera rig is the hard part. It isn't:

```js
const CAM = [
  { p: [ 0.0, 4.05,  13.6 ], t: [ 0.0,  6.60, -18.0 ], fov: 36 },  // hero
  { p: [-5.6, 2.35,  11.6 ], t: [ 1.2,  5.60, -14.0 ], fov: 48 },  // the gate
  { p: [ 1.2, 3.60,   2.2 ], t: [-0.6,  7.50, -22.0 ], fov: 40 },  // gardens
  { p: [ 5.2, 2.10,  -3.4 ], t: [-2.6,  7.00, -20.0 ], fov: 46 },  // craft
  { p: [ 0.0, 7.60, -16.0 ], t: [ 0.0, 13.00, -40.0 ], fov: 42 },  // afterlight
  { p: [ 0.0, 10.5, -20.0 ], t: [ 0.0,  3.00, -34.0 ], fov: 46 }   // footer
];

// two splines: where the camera is, and what it looks at
curveP = new THREE.CatmullRomCurve3(CAM.map(c => new THREE.Vector3(...c.p)), false, 'catmullrom', .42);
curveT = new THREE.CatmullRomCurve3(CAM.map(c => new THREE.Vector3(...c.t)), false, 'catmullrom', .42);

// scroll → float chapter index (2.4 = 40% from waypoint 2 to 3)
RIG.prog   = progressFor(scrollY);
RIG.smooth = damp(RIG.smooth, RIG.prog, 5.2, dt);

curveP.getPoint(u, _p);
curveT.getPoint(u, _t);
camera.position.copy(_p);
camera.lookAt(_t);
```

**Separating position from look-at is what makes it feel filmed rather than driven.** A single
curve with a forward-facing camera reads like a rollercoaster. Two curves let the camera track a
subject while it moves — the language of a dolly shot.

The remaining 3,500 lines are *content*. Framework choice does not save you any of it.

---

## 3. Patterns worth stealing

### 3.1 Damping is the whole feel

```js
const damp = (a, b, l, dt) => b + (a - b) * Math.exp(-l * dt);
```

Frame-rate independent exponential smoothing. Note this is **not** `lerp(a, b, 0.1)` per frame —
that naive version runs at different speeds on 60 Hz and 144 Hz displays. The `exp(-l * dt)` form
is correct at any refresh rate.

Kage damps everything: scroll progress (λ 5.2), pointer parallax (λ 2.6), focus states. This single
function is most of why the site feels expensive. It is the highest quality-per-line item in the
entire codebase — put it in every project.

### 3.2 Scroll position → narrative progress

Don't map `scrollY / maxScroll` directly. Anchor to the DOM so copy and camera stay in sync when
text reflows:

```js
anchors = SECS.map((el, i) => {
  if (i === 0) return 0;
  if (i === SECS.length - 1) return maxScroll;
  return clamp(el.offsetTop + el.offsetHeight * .5 - vpH() * .5, 0, maxScroll);
});

function progressFor(y) {
  for (let i = 0; i < anchors.length - 1; i++)
    if (y <= anchors[i + 1]) return i + (y - anchors[i]) / (anchors[i + 1] - anchors[i]);
  return anchors.length - 1;
}
```

Each section centre-lines up with a camera waypoint. Change the copy, the rig follows.

### 3.3 Governor, not budget

Kage does **not** sniff the user agent to guess device capability. It measures real frame time and
adjusts render resolution:

```js
PERF.acc += raw; PERF.n++;
if (PERF.n >= 40 || PERF.acc > .9) {
  const avg = PERF.acc / PERF.n; PERF.acc = 0; PERF.n = 0;
  if      (avg > .0230 && PERF.scale > .55) PERF.scale = Math.max(.55, PERF.scale * (avg > .05 ? .64 : .85));
  else if (avg < .0138 && PERF.scale < 1)   PERF.scale = Math.min(1, PERF.scale + .08);
  resize();
}
```

~10 lines, and it self-tunes on hardware that didn't exist when you shipped. Thermal throttling on
a phone gets handled for free. **Adopt this by default in anything using WebGL.**

### 3.4 Waypoints break on mobile — fix it in the rig

Non-obvious production problem, cleanly solved. Camera positions composed on a wide monitor crop
badly on a 390×844 phone. Rather than authoring a second set of waypoints, the rig steps backward
along its own view axis and widens the lens:

```js
function aspectFix() { return clamp((1.62 - vpW() / vpH()) / 1.05, 0, 1); }

function fitAspect(p, t, fov) {
  const nf = aspectFix();
  if (nf <= 0) return fov;
  _d.subVectors(p, t).normalize();
  p.addScaledVector(_d, nf * 8.2);   // dolly back
  p.y += nf * 1.1;                   // lift slightly
  return fov * (1 + nf * .40);       // and open up
}
```

One set of waypoints, both form factors. Most sites just let mobile break.

### 3.5 Fake the depth you don't need to touch

Kage is a **hybrid**, and this is its biggest cost saving. Alpha-cut WebP images sit in ordinary
HTML layers *and* as in-scene planes, dissolving as the camera reaches them:

```js
WORLD.fg.forEach(m => {
  const a = smooth(.9, 4.6, camera.position.z - m.position.z);
  m.material.opacity = a;
  m.visible = a > .006;
});
```

Grass, pines, sakura, ruins — none of it is modelled. It's a parallax collage. **Only build real
geometry for what the camera must move around.** Everything else is a card.

### 3.6 Typography carries the piece

875 lines of CSS versus a WebGL scene. If you stripped the canvas out, Kage would still read as a
beautiful editorial site. If you stripped the type out, it would read as a tech demo.

**Budget accordingly.** The most common failure in this genre is spending everything on the 3D and
setting the words in default Helvetica.

### 3.7 Reduced motion as a code path, not a bolt-on

```js
RIG.smooth = REDUCE ? RIG.prog : damp(RIG.smooth, RIG.prog, 5.2, dt);
```

One ternary. The narrative stays complete, the motion doesn't. Handle it at the source and it costs
nothing; retrofit it later and it costs a rewrite.

---

## 4. What *not* to copy

Honest engineering notes:

- **The single 4,800-line file.** Excellent for a solo art piece and for longevity. Brutal on a
  team — unreviewable diffs, guaranteed merge conflicts, no code splitting.
- **Vendoring all 594 KB of Three.js** when you use maybe 5% of it. Justified here because it buys
  zero-toolchain permanence; not justified if you already have a bundler that can tree-shake.
- **Procedural textures at boot.** Great at this scale (no asset pipeline, no 404s, infinite
  variation). It does not scale — canvas texture generation is CPU work on the main thread, and it
  grows linearly with scene complexity.
- **No types, no tests.** Correct for an art piece with one author. Not correct for a product.

---

## 5. Choosing a concept

The strongest lesson isn't technical.

> **Kage works because scrolling *is* walking.** The input and the idea are the same gesture.

There's no translation layer, no "and this button rotates the model." So the design question for
any site in this genre is not *"what should the 3D be?"* but:

> **Scroll is one dimension. What is that dimension in your story?**

| Scroll becomes | Feeling | Suits |
|---|---|---|
| **Distance** — walk a path | Arrival, pilgrimage | Places, journeys, brand stories *(Kage)* |
| **Time** — past to future | Inevitability | Company history, roadmaps, process |
| **Scale** — zoom in or out | Awe, revelation | Science, data, magnitude |
| **Assembly** — parts to whole | Craft, clarity | Products, engineering, how-it-works |
| **Transformation** — state to state | Change | Case studies, before/after, sustainability |

Pick the row **before** opening an editor. If the mapping is arbitrary, no amount of shader work
will rescue it.

---

## 6. A build order that works

1. **Grey-box the rig first.** Six `BoxGeometry` cubes as waypoint markers, splines, scroll
   mapping, damping. Half a day. If the *movement* isn't good with cubes, better art won't fix it.
2. **Set the type second.** Real copy, real hierarchy, real spacing — before any modelling. It will
   change the camera positions.
3. **Add the governor and reduced-motion now**, while they're 10 lines each.
4. **Then** spend the remaining 80% of your time on content, which is where it always goes.
5. **Test at 390×844 continuously**, not at the end. Aspect problems are structural.

---

## 7. Cost, honestly

A Kage-quality piece is **4–8 weeks of focused work** for someone already fluent in WebGL, and most
of it is art direction rather than programming. The technical scaffolding — rig, scroll, governor,
post-processing — is maybe a week.

It suits brand moments: launches, portfolios, campaign sites, anything whose job is to be
*remembered*. It's a poor fit for interfaces with tasks to complete, where a heavy 3D hero usually
costs more in load time and cognitive friction than it earns.

Decide which one you're building before you start.
