# Leg Day — Cloud Field

How the sky between islands is built, and why it's built that way. Companion to
[Leg Day - Design Doc.md](Leg%20Day%20-%20Design%20Doc.md) (the *what*) and
[Leg Day - Numbers.md](Leg%20Day%20-%20Numbers.md) (the *how much*). This doc owns the *how*.

There is no generator checked in. This doc is the spec: write the Luau from it and run it through
the Roblox Studio MCP with `datamodel_type = "Edit"`, building `Workspace.World.SkyRoute`. §5 is
the pipeline, §12 has every parameter, §9 lists the bugs that will otherwise be rediscovered, and
§13 is the validation the build must pass before it is called done.

> **The build lives in the place file, not in this repo.** Nothing here is synced by Rojo yet, so
> a rebuild leaves no git diff and is lost unless the place is saved in Studio. Save after building.

---

## 1. Settled decisions — don't relitigate these

These were each arrived at by trying the alternative and rejecting it. The rejected version is
recorded so we don't circle back to it.

| Decision | Rejected alternative | Why |
|---|---|---|
| **Islands stack vertically**, island 2 centred at (0, 400, 0) directly over island 1 | Island 2 offset ~460 studs to the north-east | Offsetting made the climb read as a diagonal staircase to a separate place, not as a climb |
| **No built launch structure** | A wooden deck and trestle stair on island 1's mountain ring | A single marked departure point makes one route the obvious route |
| **Cloud positions are uniform random in x, z and y** | Nine flat layers on a ring; then a radial spiral; then six braided strands | Every version with a radial or layered rule read as an obvious path from a distance |
| **Small, flat, well-separated clouds** | Big chunky cumulus blobs | Matched to a reference screenshot the user supplied |
| **The lobe meshes carry the collision. There is no pad part.** | A visible cylinder pad (read as a "flat top"); then the same pad made invisible | "Land on the part you see" |
| **Every cloud inside the play volume is landable** | Mixing landable and decorative clouds in the same airspace | Decorative clouds that look identical to landable ones are a trap dressed as scenery |

---

## 2. The one idea the whole thing rests on

**Connectivity is an acceptance filter, not a layout.**

Scatter candidate positions at random. Keep a candidate only if *something below can jump to it*
**and** *it can jump to something above* (or straight onto the upper island). Throw the rest away.

This is what lets the field be genuinely random and still be playable. Every earlier attempt tried
to *lay out* a route — layers, spirals, braided strands — and every one of them was visible as a
route. Randomly placing and then filtering produces a field with no geometric rule in it at all.

Two supporting pieces make it work:

- **Seed it with a skeleton first.** A handful of random-walk strands (free heading, forking) are
  placed before any filtering, which guarantees at least one complete ground-to-island route exists.
  Without this, a sparse random field can filter down to nothing.
- **Because every edge points upward, "no traps and no orphans" is a complete proof of connectivity.**
  From any cloud, follow a down-edge repeatedly: height strictly decreases, the set is finite, so it
  terminates — and it can only terminate at a cloud with no down-edges, which by the no-orphan rule
  is a ground entry. Same argument upward. No global search needed.

---

## 3. Vocabulary

| Term | Meaning |
|---|---|
| **Trap** | A cloud you can land on but can't leave upward, and that can't reach the upper island. Must never exist. |
| **Orphan** | A cloud nothing below can reach and that isn't a ground entry. Unreachable decoration pretending to be a platform. |
| **Entry** | A cloud low enough to be reached from island 1's surface directly. |
| **Top** | A cloud from which the upper island's rim is within a Perfect's reach. |
| **Choice** | A cloud with more than one onward cloud in reach. The measure of whether the field is a mesh or a corridor. |
| **Reach** | Horizontal distance a Perfect jump covers while climbing a given rise. |

---

## 4. Physics

Everything derives from one formula (Numbers §2, Starter Legs at S = 1.00):

```
Vy = 53.2 × √10 = 168.2      Vx = 22.0      g = 196.2

reach(rise) = Vx · (Vy + √(Vy² − 2g·rise)) / g
```

| Rise | Perfect reach |
|---|---|
| 16 | 35.3 |
| 30 | 33.5 |
| 41 | 31.3 |
| 55 | 28.2 |
| 68 | 24.6 |

A hop is legal when `horizontalDistance − padSlack ≤ reach(rise) − margin`, with `margin = 5`
studs demanded on every hop and `rise` constrained to 16–68 (below 16 it isn't a climb; the
Perfect peak is 75, so 68 leaves headroom).

**Difficulty is reported as the easiest onward hop from each cloud, expressed as the jump quality
it demands** — Good reaches 85% of a Perfect, Okay 67%, Weak 42% (Numbers §2.3). This is the
number that catches a field quietly becoming harder, e.g. after thinning. Level 1 should be
mostly Weak and Okay, and must never contain a cloud whose only exit demands a Perfect.

---

## 5. The pipeline

Nine phases, in order.

1. **Skeleton.** Five entries above verified-open ground on island 1. Each is a random walk:
   heading turns freely each step (±1.15 rad), rise 30–46, gap 17–24, forking with p = 0.4 up to
   ten concurrent strands. The last two steps steer toward radius ~100 because the upper island's
   rim is the only possible target up there.
2. **Random fill.** Uniform over the whole disc — *not* an annulus, or the core where the skeleton
   lives gets excluded and nothing has a neighbour to connect to. Accepted only if it connects
   both ways. Six passes, because each acceptance enables later ones.
3. **Prune** anything the fill order left as a trap.
4. **Thin** by local density with per-radius-band quotas (see §6).
5. **Build** the cloud models (see §7).
6. **Measure** the real collision surface by raycasting (see §8).
7. **Repair** against the measured geometry — search offsets for each broken cloud, accept the
   first that fixes it without breaking a neighbour.
8. **Scenery** — same geometry, no collision, kept outside the play volume.
9. **Validate and report.**

---

## 6. Thinning

Clouds are removed greedily, **densest neighbourhood first** (neighbours within 62 studs), so
clumps thin out while isolated clouds survive. Never remove at random.

A removal is refused if it would leave any cloud below with no onward hop, or any cloud above with
no way in, or drop the field below the minimum number of entries or tops.

Per-band keep quotas shape where the density goes:

| Radius | Keep | Why |
|---|---|---|
| 0–70 | 11 | The core column between the islands, which reads as too dense |
| 70–110 | 26 | |
| 110–165 | 76 | The middle band carries most of the branching — thin it least |
| 165–195 | 11 | |
| 195+ | 1 | Outer fringe |

**Thinning makes the climb harder** — fewer clouds means longer forced hops. Always re-run the
difficulty histogram afterwards; going from 288 clouds to 125 moved a meaningful number of clouds
from a Weak onward hop to a Good one.

---

## 7. Cloud geometry

One cloud is 4–6 flattened lobe MeshParts around a nominal diameter **D** (19–29 for landable,
10–40 for scenery), with `baseY = pos.Y − 0.14·D`:

| Lobe | Size | Position |
|---|---|---|
| Centre | `0.80D × 0.32D × 1.10D` | at `baseY` |
| Sides (2–4) | `0.56D × 0.26D × 0.74D`, each × 0.72–1.0 | radius `0.36D`, yaw facing outward |
| Underside | `0.80D × 0.20D × 1.02D`, shaded | `baseY − 0.17D` |

Colours: `253,253,255` / `246,249,253` / underside `226,235,246`. Material `SmoothPlastic`.

**Landable clouds set `CollisionFidelity = Hull` and `CanCollide/CanQuery/CanTouch = true` on
every lobe. Scenery sets all three false.** That is the *only* difference between them.

Hull was chosen over Box: Box would give flat tops but overhangs the blob at the corners, so you'd
stand on visible nothing. Hull hugs the mesh, and a Roblox humanoid doesn't slide on slopes under
`MaxSlopeAngle`, so a rounded crown is perfectly standable. Measured crown-to-edge slope stays
under 45° on every cloud.

---

## 8. Measure, never assume

After building, probe each cloud's actual collision surface: raycast down on a polar grid
(14 rings × 12 azimuths) and record

- **`SurfaceTop`** — the crown height
- **`UsableRadius`** — the radius out to which the surface is still within 2.5 studs of the crown

Then rebuild the reachability graph from *those* numbers.

This is not ceremony. When the invisible pads were replaced with mesh collision, re-deriving the
graph from the measured surface exposed **2 traps and 1 orphan** that the idealised pad geometry
had been hiding. Authored position ≠ landing surface.

---

## 9. Traps that cost time (read before writing any of this)

Every one of these was hit for real. They fail *quietly* — the build completes and looks plausible.

- **`MeshId` on a fresh `Instance.new("MeshPart")` does not resolve.** It renders as a checkerboard.
  You must `:Clone()` a MeshPart already in the place. Match on the asset id
  (`rbxassetid://5515652172`), never on shape — island 1's tree canopies are also wide, flattish
  MeshParts and a loose heuristic picks one of those.
- **The neighbour search radius must cover the longest hop in 3D**, not just its horizontal part:
  68 up and ~35 across is a ~76-stud separation. Searching a 44-stud radius silently reports
  "no connection" and mass-prunes a perfectly good field. Derive it —
  `ceil(√(maxRise² + reach(minRise)²)) + 10` — rather than picking a number.
- **Sample the random fill uniformly over the whole disc.** Sampling an annulus (`0.18–1`) excludes
  the core the skeleton lives in, and the fill accepts literally nothing.
- **Raycast terrain clearance from y = 300 downward**, never from just under the cloud. A ray
  starting inside a tree canopy skips it and reports the ground far below as clearance.
- **Terrain probes must exclude every cloud folder in the place**, or a leftover field is mistaken
  for ground.
- **Training-pit rims — and anything else you walk over — must top out ≤ 1.6 studs above the
  surrounding surface.** The humanoid step limit is 2. Island 1's sand pit is the reference.
- **A field can pass every connectivity rule and still be useless.** 35 clouds in a line satisfies
  "no traps" trivially. The report gates on cloud count, entry count, ways onto the upper island,
  and branching fraction as well.

---

## 10. Current state (2026-08-06)

| | |
|---|---|
| Landable clouds | 125 (632 solid lobes) |
| Scenery clouds | 501 in `SkyRoute.Level1.Decor`, plus 85 in `World.Clouds` around island 1 |
| Traps / orphans | 0 / 0 |
| Reachable from the ground | 125 of 125 |
| Ground entries | 25 |
| Clouds reaching island 2 | 9 |
| Distinct routes | 275 |
| Clouds offering a choice | 38 |
| Easiest onward hop | Weak 81 · Okay 31 · Good 13 · Perfect 0 |
| Mean standable radius | 13.7 studs |
| x/z spread | radius 34–211 |

**Deviations from Numbers §7.2 worth knowing.** That table specifies level 1 as ten hops of 41
studs rise and 25 studs gap, with clouds Ø38 (150% of the gap). The field departs from it: rises
vary 16–68, gaps 17–35, cloud diameters 19–29. The doc's shape assumed a single main line, which
no longer exists. What is preserved is the thing the numbers were protecting — that level 1 is
forgiving — now measured directly as the difficulty histogram instead of as a Ø-to-gap ratio.

---

## 11. Extending to level 2 and beyond

Everything is parameterised on the two islands the level spans. For level *N*:

- **Upper island altitude, rim radius and keel profile** — from the island above
- **`yMin`** — just above the island below
- **`Vy`** — **rescale for the expected jump stat at that altitude.** `Vy = 53.2 · S · √10` with S
  from Numbers §2.4. Everything else follows from `reach()`, so gaps widen automatically. This is
  the single most important line to change; forgetting it makes a high level trivially easy.
- **Band quotas** — scale with the field radius
- **Field radius** — the upper island's keel sets the minimum; sparser higher up per the design doc

The design doc calls for clouds to get sparser with altitude and to reposition every 30 minutes.
Repositioning will need the reachability graph re-validated after every shuffle — §13 is the piece
to reuse, and it's why `_RouteData` stores each cloud's position and usable radius rather than
just its position.

---

## 12. Every parameter, as built for level 1

Physics is in §4, band quotas in §6, lobe geometry in §7. The rest:

| Group | Value |
|---|---|
| Upper island | altitude 400, rim radius 82 |
| Upper island keel `{y, radius}` | `{400,82} {391,83} {375,77} {355,65} {333,49} {311,31} {295,13} {286,0}` |
| Play volume | field radius 252, y from 46 to 382 |
| Hop limits | rise 16–68, reach margin 5 studs |
| Separation | 28 studs during generation, 24 when repairing |
| Keel clearance | 20 studs outside `keelRadius(y)` |
| Skeleton entries (bearing°, radius) | (6, 74) (87, 74) (126, 86) (219, 90) (276, 90) |
| Skeleton walk | 11 steps, rise 30–46, gap 17–24, heading turn ±1.15 rad, fork p = 0.4 up to step 8, max 10 strands, 40 placement attempts per step |
| Skeleton bound | radius ≤ 0.78 × field radius, except the last two steps which steer to radius 112 then 100 |
| Random fill | cap 288, 6 passes × 900 attempts, uniform over the whole disc |
| Density neighbourhood | 62 studs |
| Floors | ≥ 9 clouds reaching the upper island, ≥ 16 ground entries |
| Cloud diameter | 19–29 landable, 10–40 scenery |
| Surface probe | 14 rings × 12 azimuths, ring spacing 1.8 studs; standable = within 2.5 studs of the crown |
| Repair search | dy ∈ {0, −6, −12, +6, −18} × 12 azimuths × dr ∈ {4, 8, 12, 16}, 3 rounds |
| Scenery | 780 attempts, radius 150–1250, y −120 to 900, keep 52% inside radius 420 and 74% beyond, excluded from the play volume and from under the upper island |
| Ground clearance | 14 studs, checked for any cloud below y = 120 |

---

## 13. The build is not done until this passes

Compute all of it from the **measured** surface (§8), not from authored positions, and print it.
The first four are correctness; the last four exist because a field can satisfy every connectivity
rule and still be useless — 35 clouds in a line passes "no traps" trivially.

| Check | Requirement |
|---|---|
| Traps | 0 |
| Orphans | 0 |
| Reachable from the ground | all of them |
| Clouds whose easiest onward hop demands a Perfect | 0 |
| Cloud count | within ±20% of the sum of the band quotas |
| Ground entries | ≥ the configured floor |
| Clouds reaching the upper island | ≥ the configured floor |
| Clouds offering a choice | ≥ 10% of the field |

Also report, for eyeballing rather than gating: distinct route count, the difficulty histogram
(Weak / Okay / Good / Perfect), mean standable radius, and x/z spread.
