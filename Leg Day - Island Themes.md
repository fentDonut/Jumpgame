# Leg Day — Island Themes (islands 4–10 and the Summit)

What each island above island 3 *looks like*, and why. Companion to [Leg Day - Design Doc.md](Leg%20Day%20-%20Design%20Doc.md) (owns the design), [Leg Day - Numbers.md](Leg%20Day%20-%20Numbers.md) (owns the values) and [Leg Day - Cloud Field.md](Leg%20Day%20-%20Cloud%20Field.md) (owns the sky between them). This doc owns the *art brief*: theme, palette, hero prop, and the pit's read.

Nothing here changes a number or a mechanic. Where a theme brushes against one, it's flagged at the bottom under [Open calls](#open-calls).

Islands 1–3 are already built and aren't re-specified here.

---

## 1. The spine: three ladders already in the docs

The Numbers doc names every island three separate times, and the three sequences point in different directions. That's the whole theme, and it was already there:

| Ladder | Islands 4 → 10 | Direction |
|---|---|---|
| **Pit names** (§4.3) | Clay Flat → Iron Sand → Basalt Pan → Slag Bed → Leadfield → Deep Well → Coreground | downward, geological, heavier |
| **Tier names** (§2.1) | Wooden Stilts → Coil Springs → Shock Absorbers → Hydraulic Pistons → Turbine Calves → Rocket Boosters → Antigrav Struts | junk → machine → powered → sci-fi |
| **Egg names** (§6.3) | Breeze → Storm → Aurora → Thunder → Nebula → Void → Summit | upward, atmospheric → cosmic |

So each island reads as three layers stacked:

- **The sky between islands is the weather ladder.** The cloud field's palette and effects come from the egg name of the island above it.
- **The island surface is the tech ladder.** Each island is the abandoned workshop where its Leg Tier was made. Reaching island N unlocks tier N, so if the island visibly *is* the source of that gear, the unlock reads diegetically rather than as a menu event.
- **The pit is the geology ladder.** A slab of something dragged up from deep underground and parked in the sky. The higher you climb, the deeper the thing you're standing on came from.

**Ground looks down, sky looks up.** Same island, two opposite directions — that tension is the identity of the back half of the game.

---

## 2. Island 4 — The Stilt Flats — **BUILT 2026-08-11**

| | |
|---|---|
| Tier / Pit / Egg | Wooden Stilts · Clay Flat · Breeze |
| Diameter / Altitude | 140 / 1600 |
| Pit gravity | ×2.3 |
| Palette | Terracotta, warm wood, pale blue |

> **As built:** 551 parts. Ground and keel cloned from island 3 and rescaled ×0.875, restratified
> in clay. 27 cracked plates on a jittered lattice (cell 17, ~40% merged into longer plates, three
> top heights 0.08 apart so no pair can z-fight). 7 stilt frames with decks at +2.6 to +8.4 and
> 3 rope bridges; 5 windsocks, bunting, laundry and clutter, all on one wind bearing of 52°.
> Clay Flat pit at (−26, −18), Ø42 zone, rim top +1.35. Medal cache at (14, 34) under a stilt
> pavilion. Reached by the [level 3 cloud field](Leg%20Day%20-%20Cloud%20Field.md) (129 clouds).

A cracked terracotta plateau split into big flat plates. Everything built on it stands on wooden legs — a rickety stilt village of platforms and rope bridges, which is the game's first joke about legs. Windsocks, bunting, laundry lines at a permanent lean, all reading the breeze.

**The last warm, inhabited-looking island.** Islands 1–4 are one visual family (sun, wood, colour); everything above is a different game. Play that for all it's worth here — this is the last postcard before the light changes.

- **Hero prop:** wooden stilt frames, repeated at varying heights.
- **Pit read:** the clay flat itself. A wet rust-orange basin, low rim, footprints that hold their shape and slowly fill back in.
- **Cache:** under a stilt pavilion with a hanging lantern.
- **Underside:** clay strata and dangling root mats.

## 3. Island 5 — The Springworks — **BUILT 2026-08-12**

| | |
|---|---|
| Tier / Pit / Egg | Coil Springs · Iron Sand · Storm |
| Diameter / Altitude | 140 / 2500 |
| Pit gravity | ×2.8 |
| Palette | Black glitter sand, storm grey-blue, one saturated paint colour on the springs |

> **As built:** 904 parts. Ground and keel cloned from island 3 and rescaled ×0.875 into cold iron
> strata, with a `KeelFill` from the outset (0% of radial sightlines escape). Surface is `Sand`
> material rather than the clay's `SmoothPlastic`, with 7 low wind drifts and 131 near-black
> metal glints at 0.55 reflectance — the glints are what make it read as iron sand rather than ash.
> **9 coils** generated as helices of chord segments: one Ø25 laid on its side and half-sunk at
> (16, 32), 2 medium tilted like fossils, 6 small upright and mostly buried. Iron Sand pit at
> (−26, −18), Ø40 zone, with 8 more coils standing upright around the rim leaning inward. Medal
> cache nested inside the big coil, so you step through the spring to reach it. Reached by the
> [level 4 cloud field](Leg%20Day%20-%20Cloud%20Field.md) (113 clouds, 8 bands).
>
> The theme's "one saturated paint colour" is doing real work: at 43% surface occupancy the island
> would read as an empty black disc without the orange, which is the only warm hue above island 4.

Black magnetic iron sand, glittering where the light catches it. Giant painted coil springs lie half-buried across the island like fossils; a few still bounce gently on their own. Tarpaulin windbreaks pegged into the sand, wire spools, everything lashed down against weather.

**The light goes flat here.** First island under an overcast storm bank — no direct sun, no shadows with edges. The springs telegraph what your new legs do before you've bought them.

- **Hero prop:** the half-buried coil, at three scales.
- **Pit read:** a bowl of black sand that stands the springs upright around its rim, as if magnetised by it.
- **Cache:** nested inside the largest coil, so you climb through the spring to reach it.
- **Underside:** iron sand streaming off the edges in thin falls.

## 4. Island 6 — The Dampening Field

| | |
|---|---|
| Tier / Pit / Egg | Shock Absorbers · Basalt Pan · Aurora |
| Diameter / Altitude | 140 / 3700 |
| Pit gravity | ×3.4 |
| Palette | Black basalt, aurora green and violet, wet reflections |

Columnar basalt — hexagonal prisms, which is free low-poly geometry and exactly the faceted language the art brief asks for — laid out as a flat black pan. A ring of shock-absorber pylons stands around it like standing stones, each one slowly compressing and rebounding on its own rhythm. Overhead, the aurora: the first non-blue sky in the game, ribbons of green and violet reflecting off wet black rock.

**This is the midpoint and it should be the postcard.** The first island that looks like nowhere on Earth. Put the warm→cold palette turn here, and make it abrupt — one island, no gradient — so players remember arriving.

- **Hero prop:** the shock-absorber pylon, ring-arranged, animated on a slow offset cycle.
- **Pit read:** a sunken hexagonal well cut into the basalt, floor a shade darker than the pan.
- **Cache:** at the centre of the pylon ring, lit from below.
- **Underside:** basalt columns hanging like an organ pipe rank.

## 5. Island 7 — The Pump House

| | |
|---|---|
| Tier / Pit / Egg | Hydraulic Pistons · Slag Bed · Thunder |
| Diameter / Altitude | 120 / 5250 |
| Pit gravity | ×4.2 |
| Palette | Black slag, orange glow seams, cold blue-grey sky |

Slag is smelting waste, so this island is the foundry that produced it. A squat, half-buried machine hall with pistons driving up through its roof, orange seams glowing through the black slag ground, heat shimmer over the hot patches.

**The first island that makes its own light** rather than borrowing the sun's — which is the point, because the sky is dark from here up. Sync the piston strokes to the thunder flashes in the cloud field below so the island and the weather read as one system.

- **Hero prop:** the piston, driving through a roof or straight out of the ground.
- **Pit read:** the slag bed — matte black crust with orange cracks, the heaviest-looking ground so far. Heat haze above it.
- **Cache:** on the machine hall's loading step, lit by the seams.
- **Underside:** cooling slag, dull orange fading to black at the rim.

## 6. Island 8 — The Jetstream Farm

| | |
|---|---|
| Tier / Pit / Egg | Turbine Calves · Leadfield · Nebula |
| Diameter / Altitude | 120 / 7350 |
| Pit gravity | ×5.3 |
| Palette | Matte grey lead, deep indigo sky, first stars, faint nebula wash |

You're in the jet stream. A field of huge turbines all leaning the same way, blades turning fast, guy-wires humming, flags stretched flat. The ground beneath them is dull matte lead — completely inert, absorbing all the light that hits it.

**The contrast is the whole island: a screaming sky over a dead floor.** Everything above ground level is in violent motion and nothing at ground level moves at all. That reads instantly and it sells the Leadfield's weight without a word of UI.

- **Hero prop:** the turbine, repeated at three sizes, all aligned.
- **Pit read:** the Leadfield — a flat grey plate, no texture, no sparkle, visibly heavy. The one place on the island where the wind doesn't reach.
- **Cache:** at the base of the largest turbine, in its shadow.
- **Underside:** flat grey, featureless, with the guy-wire anchors punching through.

## 7. Island 9 — The Gantry

| | |
|---|---|
| Tier / Pit / Egg | Rocket Boosters · Deep Well · Void |
| Diameter / Altitude | 120 / 10150 |
| Pit gravity | ×6.7 |
| Palette | Near-black, hard star field, no atmospheric haze |

No haze at all — the air has run out, so shadows are hard and the horizon is a knife edge. One launch gantry and a scatter of spent booster casings repurposed as huts. Nominally 120Ø but should *feel* cramped: mostly structure, very little ground.

The Deep Well is literally that. A shaft in the middle of the island with a light somewhere far below — built as a 2-stud depression with a painted void floor and a point light, so it reads bottomless while the rim still obeys the ≤1.7-stud step rule.

- **Hero prop:** the gantry, and the spent-booster hut.
- **Pit read:** the well mouth — the only pit in the game you look *into* rather than step down onto.
- **Cache:** on the gantry's launch platform, at the top of a short ramp.
- **Underside:** the well's shaft protruding through the island's belly, lit from within. Visible for the whole climb of level 8, which is the point.

## 8. Island 10 — The Coreground

| | |
|---|---|
| Tier / Pit / Egg | Antigrav Struts · Coreground · Summit |
| Diameter / Altitude | 100 / 14100 |
| Pit gravity | ×8.8 |
| Palette | Black rock, glowing veins, void |

A chunk of planetary core hauled into the void. Glowing veins run through black rock, and the broken pieces around the edge don't fall — they hang and slowly orbit. Antigrav struts are the scaffolding holding the whole thing together.

**Gravity here is 8.8×, so everything visible reads as strained** — this is the design doc's "sagging/strained props" note taken to its endpoint. Bowed struts, cables under visible tension, one loose rock hanging at an angle it has no business holding. The smallest island in the game and the pit takes most of it, which is correct: by island 10 there is nothing here but the work.

- **Hero prop:** the antigrav strut, and free-floating orbiting shards.
- **Pit read:** the Coreground — glowing rock, the pit floor brighter than its rim, the heaviest place in the game and lit like a forge.
- **Cache:** on a shard that orbits back to the island periodically. (See [Open calls](#open-calls) — a moving cache may be a bad idea.)
- **Underside:** the brightest thing in the game. Vein-glow visible from island 9, 3,950 studs below, as the whole level's landmark.

## 9. The Summit — altitude 20100

6,000 studs above island 10, the largest gap in the game by a factor of two, and there is no island 11.

**Let the clouds run out.** The last stretch has no filler and no catch ratio — bare sky, a handful of hops, nothing to save you. It's the only place in the game where that's true, and it's earned by then.

What's at the top should pay off the title. The proposal: a monumental plinth holding the original Starter Legs at fifty times scale, the free pair the tutorial gave away four hours earlier, treated as a monument. That joke is worth the climb.

---

## 10. Cross-cutting rules

These do more work than any individual island theme.

**The light ladder.** The cheapest and strongest altitude signal available — lighting and material choices only, no extra geometry:

| Islands | Lit by |
|---|---|
| 1–4 | the sun, warm, hard shadows |
| 5–6 | flat overcast, then aurora |
| 7 | its own foundry |
| 8–10 | stars, and the island's own glow |

**One hero silhouette per island, repeated.** Stilts, coils, pylons, pistons, turbines, gantry, shards. One asset to build and scatter per island rather than a full set-dressing kit each — which is what makes seven islands affordable.

**The pit gets more contained as it gets heavier.** Island 1's sand pit is open ground you walk onto; island 6's is a cut well; island 9's is a shaft you look into. Free visual escalation that tracks §4.3's gravity ramp exactly.

**Undersides are themed — and they have to be solid.** Islands stack vertically (Cloud Field §1), so an island's belly is what the player stares at for the entire level below it. It's the destination made visible from the start of the climb, and it's free real estate — one material and one light per island.

*Islands 2, 3 and 4 shipped hollow.* The keel is a ring of angled facets, and the wedge slits between adjacent plates showed open sky from below for the whole climb. Each now carries a `KeelFill` — a stack of axis-aligned discs sized from the narrowest petal at each height, so the slits read as shadow instead of holes. The fill is non-collidable and non-queryable, so it changes nothing about the cloud field's gate. Build it for every island from 5 up as part of the island, not as a later fix; see the see-through-shells entry in the build checklist for the three ways to get it wrong.

**Everything above island 6 is under strain.** Sagging, bowing, tension. The design doc already asks for this around pit rims; above the midpoint it should be the whole island's posture.

---

## 11. Open calls

Four things to settle before island 4 is built, not after.

1. **Man-made structures vs. the storybook art brief.** The design doc's visual reference is beach islands, mushrooms, bunting, warm wood. Seven abandoned industrial workshops is a different register. The mitigation is that everything is *abandoned, overgrown, and whimsically proportioned* — painted machinery, chunky rounded silhouettes, no gritty realism, no rust-and-grime texturing. But it's a real tension and it wants a decision rather than a hope.

2. **Tall props are a documented cloud-field hazard.** Cloud Field §4 already caught this once: island 2's tree canopies reach 431, which turned an 80-stud entry hop into a 49-stud one and had to be made non-collidable. Turbines, gantries and pylons are exactly that failure mode, and they're the hero props for islands 6, 8 and 9. Either make them non-collidable or site them clear of entry clouds — and **run the entry-clearance check after set dressing, not before**, against the 4-stud grid the cloud doc specifies.

   *Confirmed twice while building island 4.* Island 3's canopies were still collidable at +27.3 and would have collapsed level 3's gate; they are now non-collidable, matching island 2. And island 4's own stilt decks are collidable at up to **+8.4 above its clay**, which is deliberate — they are meant to be jumped onto — but it means **level 4's entry band must be measured from 1608.4, not 1600**. Every island from here up should record its highest standable point alongside its altitude, because the next level's gate is computed from it.

3. **Nothing may read as a shopfront.** Islands 2–10 carry no vendor (settled decision); every shop is on island 1. The Pump House and the Springworks are exactly the kind of building a player walks up to expecting to buy pistons. Keep them sealed, derelict, or clearly non-interactive.

4. **The orbiting cache on island 10.** A cache that moves is a nice image and a bad interaction: collection needs a 0.75-second hold within proximity (Numbers §5.1), and a moving target can drift out of range mid-hold. Either pin it to the island or drop the idea.
