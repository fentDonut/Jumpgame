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

**The arc is asymmetric** (Numbers §1): it rises at 392.4 and falls at 147.2, the difference held
up by a client-side `VectorForce`. There is no single gravity value, so the old symmetric
`reach = Vx·(Vy + √(Vy² − 2g·rise))/g` no longer applies — **derive reach from the two halves
separately, or every gap in the level will be wrong.**

Launch is derived from a target *peak height*, so peak is fixed by S and does not move when
gravity does:

```
peak   = 72 · S²            g_rise = 392.4      g_fall = 147.2
Vy     = √(2 · g_rise · peak)
t_rise = Vy / g_rise
Vx     = 31.14 · S          -- calibrated so flat reach reproduces Numbers' 49.7·S²

reach(rise) = Vx · ( t_rise + √( 2·(peak − rise) / g_fall ) )     for rise < peak
```

| Rise | Level 1 reach (S = 1.00, peak 72) | Level 2 reach (S = 1.11, peak 88.7) |
|---|---|---|
| 20 | 46.7 | 56.6 |
| 30 | 43.7 | 54.1 |
| 41 | 39.8 | 51.9 |
| 48 | 36.9 | 48.9 |
| 65 | 26.9 | 42.9 |
| 76 | — | 37.6 |

A hop is legal when `horizontalDistance − padSlack ≤ reach(rise) − margin`, with `margin = 5`
studs demanded on every hop, and `rise` constrained to `[minRise, maxRise]` — below `minRise` it
isn't a climb, and `maxRise` must sit clearly under `peak` (level 2 uses 20–76 against a peak
of 88.7).

**Use the S the player is expected to hold on that level**, from Numbers §2.4 — level 1 is
S = 1.00 (Starter Legs, nothing bought), level 2 is S = 1.11 (trained at the Packed Earth pit
and wearing Cardboard Braces).

---

## 4.1 Gating a level against an under-levelled player

A level that can be climbed without the upgrade it is meant to gate is the failure mode this
section exists to prevent. Two facts make it easy to get wrong:

- **Peak height is a hard ceiling.** `peak = 72·S²`, so an untrained S = 1.02 arrival tops out at
  **74.9 studs** and a trained S = 1.11 at **88.7**. A hop that rises more than 74.9 is not "hard"
  for the untrained player — it is *impossible*, at any horizontal distance.
- **Horizontal gaps gate weakly.** At the same rise, untrained reach is about 80% of trained, so a
  gap tuned to 80% of a trained Perfect only forces the untrained player into a flawless Perfect.
  That is a soft gate, and a long field gives them many chances to find the one easy way through.

**So gate on the two hops nobody can avoid: getting on, and getting off.** Put the lowest band a
rise of ~80 above the lower island and the highest band ~77 below the upper one. Both sit in the
window between the two peaks, so the untrained player cannot start *or* finish, whatever the
middle of the field looks like. Everything between is then free to be as generous as it needs to be.

Two things will quietly defeat this:

- **Anything standable near an entry cloud is a free step up.** Island 2's tree canopies reach 431
  — 31 studs above the grass — which turns an 80-stud entry hop into a 49-stud one. The canopies
  are now non-collidable, which drops the island's highest standable point from 431 to 407.
  Check every entry against a *fine* grid of the island's real surface (4-stud spacing); sampling
  a dozen rays per candidate lets a canopy slip between them.
- **The gate must be tested as reachability, not height.** A low crate 40 studs away is irrelevant,
  because the rise from it is still beyond the untrained peak. Test whether an untrained player
  standing there could actually make the jump.

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
- **Building level N+1 puts its playable volume where level N's scenery already is.** Level 1's
  scenery pass ran with nothing above it and filled `r < 272, y 402–900` — exactly where level 2
  goes. Clear the lower level's decor out of the new field before generating, or you ship
  non-collidable clouds that look identical to landable ones. Both fields now assert zero
  intrusions in either direction.
- **Entries onto a sky island must be measured from its grass, not from a raycast.** A ray down
  from a low cloud hits treetops and roofs, which are not places you can stand and jump from.
  Level 2 treats a cloud as a departure point only if it is inside the island's rim radius and
  within `maxRise` of its *surface altitude*.
- **A `minRise` in the link model does not constrain the geometry.** Level 2's first build set
  `minRise = 20` for its own reachability test, then let the fill drop clouds a few studs above
  each other — so the *real* graph, which has no minimum, was a staircase of 15-stud steps that
  anyone could walk up. Measured after the fact: 56% of hops rose under 40 studs and the median
  easiest onward rise was **16**. Constrain the placement, not just the model.
- **Pads make hops easier than a point-to-point test suggests.** Reachability subtracts ~20 studs
  of pad slack at the two ends, so an exclusion rule that ignores it will happily place a pair the
  player can actually jump between.
- **Raycasts hit non-collidable parts.** Setting `CanCollide = false` on the tree canopies did not
  remove them from the perch scan — `RaycastParams.RespectCanCollide` must be set to `true`, or
  the probe keeps seeing geometry the player can no longer stand on.
- **Training-pit rims — and anything else you walk over — must top out ≤ 1.6 studs above the
  surrounding surface.** The humanoid step limit is 2. Island 1's sand pit is the reference.
- **A field can pass every connectivity rule and still be useless.** 35 clouds in a line satisfies
  "no traps" trivially. The report gates on cloud count, entry count, ways onto the upper island,
  and branching fraction as well.
- **The gate hop onto the upper island is Perfect-only by construction, so the fault test must
  exempt it.** §4.1 puts that hop above the untrained peak, which also puts it above the *trained*
  player's Good (a Good peaks at `72·(0.92·S)²`, ~15% under a Perfect). A repair pass that treats
  "easiest exit demands a Perfect" as a fault will therefore condemn every top-band cloud, delete
  them, and cascade: on level 3 that took the field from 131 clouds to 38 in one pass. The
  Perfect-only rule is about *cloud-to-cloud* progression only. Same for the entry hop.
- **The repair pass must not move clouds vertically.** Level 3's first repair searched
  `dy ∈ {0,−8,−16,+8,−24}` as §12 describes, which widened band 9's spread from 6 studs to 13 and
  produced same-band pairs 10.7 studs apart — a staircase, exactly what the bullet above this one
  warns about, reintroduced by the fix rather than the fill. Repair horizontally, re-level any band
  that drifts, and re-run the shortest-climbing-hop check afterwards.
- **The shortest-climbing-hop check is against `band spacing − 2 × jitter`, not the spacing.** With
  ±3 jitter on a 57.5 spacing the tightest legal adjacent-band hop is 51.5. Level 2 reports 39
  against a 45–53 spacing for the same reason. Measure same-band links separately; they are lateral
  moves, not climbs.
- **Check the lower island's collidable props before trusting the gate, not just its canopies.**
  Island 3's tree canopies were still collidable and stood 27.3 studs above its grass — the island 2
  fix from this section had never been applied to island 3 — which would have turned level 3's
  92-stud entry hop into 65 and opened the gate completely. After de-colliding them the island still
  tops out at 956.1 on a sign board, a knoll and the cache plinth, all legitimately walkable, so the
  gate is computed against 956.1 rather than the 950 grass. **Measure on a Cartesian grid**: the
  polar scan used first under-samples the rim and missed the sign board entirely.

---

## 10. Current state (2026-08-11)

| | Level 1 (island 1 → 2) | Level 2 (island 2 → 3) | Level 3 (island 3 → 4) |
|---|---|---|---|
| Altitudes | 0 → 400 | 400 → 950 | 950 → 1600 |
| Design S | 1.00 | 1.11 | 1.20 |
| Layout | free scatter | **banded** (see §4.1) | **banded**, 9 bands |
| Landable clouds | 125 | 154 | 129 |
| Scenery clouds | 472 | 600 | 430 |
| Traps / orphans | 0 / 0 | 0 / 0 | 0 / 0 |
| Reachable from below | 125 of 125 | 154 of 154 | 129 of 129 |
| Departure points | 25 | 6 | 12 |
| Clouds reaching the island above | 9 | 12 | 11 |
| Distinct routes | 275 | >1,000,000 | ~479,000,000 |
| Clouds offering a choice | 38 | 117 | 115 (89%) |
| Climbing hop: shortest / median / longest | — | 39 / 48 / 58 | 51.9 / 57.5 / 62.6 |
| Vertical / radial span | y 45–381, r 34–211 | y 477–876, r ≤300 | y 1046–1511, r 36–221 |
| **Gated against an untrained arrival** | no, by design | **yes — 0 clouds reachable at S = 1.02** | **yes — 0 clouds reachable at S = 1.11** |

**Level 3 is sparser than level 2 on purpose** — 129 clouds against 154, over a taller span
(650 studs against 550) and with hops half again as long (median rise 57.5 against 48). That is
the design doc's "sparser the higher you go" showing up as a measured number for the first time.

**Level 3's difficulty histogram**, by the easiest onward *cloud* hop: Weak 19, Okay 90, Good 16,
Perfect 0, and 4 clouds whose only exit is the island hop itself. Compare level 1's "mostly Weak
and Okay" — level 3 has moved up a notch without any cloud being stranded behind a Perfect.

Plus 85 small scenery clouds in `World.Clouds` around island 1.

**Level 1 is deliberately ungated** — a player with Starter Legs and nothing else must be able
to climb it. Level 2 onward should be gated, and §4.1 is how.

**Deviations from Numbers §7.2 worth knowing.** That table specifies level 1 as ten hops of 41
studs rise and 25 studs gap, with clouds Ø38 (150% of the gap). The field departs from it: rises
vary 16–68, gaps 17–35, cloud diameters 19–29. The doc's shape assumed a single main line, which
no longer exists. What is preserved is the thing the numbers were protecting — that level 1 is
forgiving — now measured directly as the difficulty histogram instead of as a Ø-to-gap ratio.

Level 3 departs the same way. §7.2 gives it 12 hops rising 57 with a 35-stud gap and Ø38 clouds;
built, it is **10 hops** (island → 9 bands → island) rising 51.9–62.6, with clouds Ø22–32 whose
*measured* usable radius is 6–12. The hop count is lower because the two gate hops of ~90 studs
eat 180 of the 650-stud climb on their own, which §7.2 does not model — it assumes uniform hops.
Worth reconciling in Numbers when that table is finally rebuilt for the 1.32× airtime stretch,
since gate hops will only get more expensive as the peaks rise.

---

## 11. Extending to level 4 and beyond

Levels 1, 2 and 3 are built; §12 lists all three side by side, which is the easiest way to see
which knobs move with altitude. Everything is parameterised on the two islands the level spans.
For level *N*:

- **Upper island altitude, rim radius and keel profile** — from the island above
- **`yMin`** — just above the island below
- **`Vy`** — **rescale for the expected jump stat at that altitude.** `Vy = 53.2 · S · √10` with S
  from Numbers §2.4. Everything else follows from `reach()`, so gaps widen automatically. This is
  the single most important line to change; forgetting it makes a high level trivially easy.
- **Band quotas** — scale with the field radius
- **Field radius** — the upper island's keel sets the minimum; sparser higher up per the design doc
- **The gate window, and how much of it the lower island's own relief eats.** The two gate hops
  have to land between the untrained peak and the trained one. On level 3 that window is
  88.71 → 103.68, only **15 studs**, and island 3's surface relief (6.1 studs between its grass and
  the top of a knoll) consumes 40% of it before any cloud is placed. The window is a fixed ~17% of
  the trained peak, so it *widens* in absolute terms with altitude — 25 studs by level 6, 50 by
  level 9 — while island relief stays roughly constant. **Level 3 is therefore the tightest gate in
  the game**, and the lower island's tallest walkable prop is a real parameter, not set dressing.
  Measure it before choosing the entry band, and re-measure after any set-dressing pass.

The design doc calls for clouds to get sparser with altitude and to reposition every 30 minutes.
Repositioning will need the reachability graph re-validated after every shuffle — §13 is the piece
to reuse, and it's why `_RouteData` stores each cloud's position and usable radius rather than
just its position.

---

## 12. Every parameter, as built

Physics is in §4, lobe geometry in §7. The rest:

| Group | Level 1 | Level 2 | Level 3 |
|---|---|---|---|
| Lower / upper altitude | 0 → 400 | 400 → 950 | 950 → 1600 |
| Upper rim radius | 82 | 82 | **71.9** (island 4 is Ø140) |
| Upper keel `{y, radius}` | `{400,82} {391,83} {375,77} {355,65} {333,49} {311,31} {295,13} {286,0}` | same shape at +550 | `{1569,75} {1551,62} {1532,48} {1516,33} {1505,24} {1494,4} {1483,2}` — island 3's keel scaled ×0.875 |
| Play volume | radius 252, y 46–382 | radius 300, y 477–876 | radius 300 (occupied 36–221), y 1046–1511 |
| Hop limits | rise 16–68, margin 5 | margin 5; band spacing sets the rise | same as level 2 |
| Bands | none (free scatter) | 9, at y 480, 525, 577, 623, 672, 722, 768, 816, 873; spacing 45–53, ±3 jitter | 9, at y 1048, 1105.5, 1163, 1220.5, 1278, 1335.5, 1393, 1450.5, 1508; spacing 57.5, ±3 jitter |
| Band populations | — | 6 / 12 / 18 / 24 / 26 / 24 / 18 / 14 / 12 | target 12/16/18/20/20/18/16/14/12; **built 12/16/17/17/17/14/11/13/12** |
| Gate hops | none | entry 80 studs, island 77 — both above the 74.9 untrained peak | entry **92**, island **89** — both above the **88.71** untrained peak (S = 1.11) |
| Separation | 28 (24 when repairing) | 34 | 38 (30 when repairing) |
| Keel clearance | 20 studs outside `keelRadius(y)` | same | top band pinned to r ≥ 76, i.e. outside island 4's plan footprint entirely |
| Skeleton entries (bearing°, radius) | (6,74) (87,74) (126,86) (219,90) (276,90) | (25,52) (100,44) (170,60) (245,50) (315,58) | (15,54) (88,46) (162,58) (238,50) (310,60) |
| Skeleton walk | 11 steps, rise 30–46, gap 17–24 | 11 steps, rise 38–56, gap 26–38 | 8 steps (one per band gap), gap 26–46 |
| Walk shape | heading turn ±1.15 rad, fork p = 0.4 to step 8, max 10 strands, 40 attempts per step | same, fork p = 0.42, turn ±1.1 | same as level 2, 60 attempts per step, fork to band 7 |
| Skeleton bound | r ≤ 0.78 × field radius; last two steps steer to 112 then 100 | same; steer to 126 then 108 | last two steps steer to 118 then 96 |
| Random fill | cap 288, 6 × 900 attempts, uniform over the whole disc | cap 300, 6 × 1100 | per-band targets, 6 passes × 900 attempts per band |
| Density neighbourhood | 62 | 75 | — (radial quotas applied at sampling, no separate thinning pass) |
| Band quotas (radius : keep) | 0–70:11, 70–110:26, 110–165:76, 165–195:11, 195+:1 | 0–70:10, 70–115:24, 115–175:78, 175–210:18, 210+:4 | 0–75:9, 75–125:24, 125–195:74, 195–240:16, 240–300:4 |
| Floors | ≥ 9 tops, ≥ 16 entries | ≥ 9 tops, ≥ 12 entries | ≥ 9 tops, ≥ 9 entries |
| Cloud diameter | 19–29 landable, 10–40 scenery | 20–30 landable, 10–40 scenery | 22–32 landable, 10–40 scenery |
| Surface probe | 14 rings × 12 azimuths, spacing 1.8; standable = within 2.5 of the crown | same, spacing 1.9 | same, spacing 2.0 (measured usable radius 6–12, mean 8.2) |
| Repair search | dy ∈ {0,−6,−12,+6,−18} × 12 azimuths × dr ∈ {4,8,12,16}, 3 rounds | dy ∈ {0,−8,−16,+8,−24} × 12 × dr ∈ {6,12,18,24} | **horizontal only** — dr ∈ {6,10,14,18,22,26,30} × 16 azimuths, 4 rounds (see §9) |
| Scenery | 780 attempts, r 150–1250, y −120–900, keep 52% inside r 420 / 74% beyond | 520 attempts, r 255–1150, y 410–1010 | 5200 attempts, r 276–1150, y 1000–1780, keep 52% inside r 520 / 74% beyond |
| Ground clearance | 14 studs, checked below y = 120 | 14 studs, checked below y = 500 | n/a — nothing below level 3 but island 3 |

Level 2 needed four extra departure clouds hand-placed over island 2's open ground after the
main pass came up one short of the entry floor — the entry test is strict (inside the rim
radius, on grass rather than a treetop) so the random fill rarely satisfies it.

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
| **Shortest *climbing* hop** | **≥ band spacing − 2 × jitter**, counting only hops between different bands |
| **Clouds reachable by an under-levelled player** (gated levels) | **0** |

"Clouds whose easiest onward hop demands a Perfect" counts **cloud-to-cloud** hops only. On a gated
level the hop onto the upper island is Perfect-only by design (§4.1), so a cloud whose sole exit is
that hop is correct, not broken — see the §9 bullet, which exists because treating it as a fault
deleted three quarters of level 3's field.

The last two are the ones that were missing, and both need measuring against the *real* graph
with no artificial minimum rise. For the gate, run the whole reachability search a second time
with the untrained model and confirm it reaches nothing: entries, clouds and launch points all zero.

Also report, for eyeballing rather than gating: distinct route count, the difficulty histogram
(Weak / Okay / Good / Perfect), mean standable radius, and x/z spread.
