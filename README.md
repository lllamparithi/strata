# strata

A scroll-driven 3D web experiment, kept as a technique reference.

**[LESSONS.md](LESSONS.md) is the point of this repo** — a line-by-line teardown of
[MengTo/kage](https://github.com/MengTo/kage) and the patterns worth reusing in future frontends.
`index.html` is the working proof that those patterns hold.

## Run it

No dependencies, no build step.

```bash
python -m http.server 4173 --bind 127.0.0.1
```

Then open <http://127.0.0.1:4173/>.

## What it is

A 300-metre vertical descent through rock strata. Scrolling down goes deeper underground; the rock
changes character, colour and lighting with depth — ochre soil, grey limestone, red beds, black
shale, then a hot basement that lights itself.

Deliberately built as the inverse of Kage on every axis:

| Kage | strata |
|---|---|
| Walks forward through space | Falls downward |
| Polygon meshes + WebP cutouts | Raymarched SDF — no meshes, no buffers, no vertex data |
| 594 KB Three.js vendored | **Zero dependencies** — raw WebGL2, one fullscreen triangle |
| Poetic art book | Instrumental — a core-sample log |
| Hand-authored waypoints | Procedural bands — the shaft is a function, so it never ends |

The whole world is one distance function:

```glsl
float shaft(vec3 p){
  float dep = -p.y;
  float b   = floor(dep / BAND);          // which stratum
  float h   = h1(b * 1.7 + 3.1);          // this band's character
  float r   = 3.5 + h * 1.2 + sin(dep * 0.031) * 0.75;
  float bed = sin(dep * (1.7 + h * 2.0)) * (0.09 + h * 0.15);   // bedding planes
  float er  = fbm(vec3(p.x, dep * (0.6 + h * 0.8), p.z) * (0.30 + h * 0.22)) * (0.9 + h * 0.8);
  return r + bed - er - length(p.xz);
}
```

Because the geometry *is* a function, there is nothing to upload to the GPU — which is why it needs
no library at all.

## Verified output

Sampled rendered pixels at four depths; the descent reads in the numbers:

| Depth | Mean RGB | Reads as |
|---|---|---|
| 4 m | `113/103/84` | warm ochre soil |
| 90 m | `96/89/87` | neutral grey limestone |
| 200 m | `67/56/51` | dark shale, dimmest point |
| 296 m | `89/52/32` | hot basement, red ≈ 3× blue |

## Honest provenance

This started as a misread. The brief was *"recreate this kind of site with a different ideology or
concept"* — a creative brief only its author can fill — and the geological concept was invented
unilaterally rather than asked for. It is kept because the **technique** is sound and reusable, not
because the concept was ever chosen.

Treat it as a reference implementation, not a design direction.

## Licence

All code here is original. No Kage code or artwork is included or redistributed — that project
ships without a redistribution licence, and only its ideas are discussed, in `LESSONS.md`.
