# Leg Day (working title) — Design Doc

Roblox game: a vertical climbing game where the only way up is a timing-based jump. Players train and upgrade their legs to jump farther, and climb an endless sky of islands and clouds that get sparser — and periodically rearrange — the higher you go.

> **Concrete values live in [Leg Day - Numbers.md](Leg%20Day%20-%20Numbers.md)** — jump-stat curve, per-island gravity, Leg Level bands, Medal and Coin economies, gap distances and object sizes, calibrated against real Roblox physics and shipped-game economies. Several open questions below now have proposed numeric answers there, marked as such; they're proposals awaiting a feel test, not settled decisions.

## Core loop

1. Stand at the edge of a platform.
2. **Tap Space** for the safe jump: instant, a guaranteed **Okay**, no bar drawn at all. No risk — and no reach. It's a repositioning jump, not a climbing one.
3. **Hold Space** instead and the crouch opens the jump bar — an indicator sweeps across it. **Release** to lock in. Distance from center determines jump quality: Perfect / Good / Okay / Weak.
4. Land on the next island or cloud. Miss, and you fall to the last checkpoint.
5. Currency and height records accumulate as you climb; spend currency on leg upgrades to reach higher.

## The jump bar

- **Taking the bar is a choice, not a toll.** Tapping Space takes the safe Okay jump and skips the bar entirely; holding opens it. This matters because Okay is 67% of a Perfect's distance while a main-line gap is 80% of one — so the safe jump *cannot clear the route*. It gets you around an island, onto a filler cloud, back from an overshoot. Every jump that actually gains you altitude is one you chose to hold for. The floor is generous; the ceiling is earned.
- The bar is also a real risk: 38% of it is Weak, which is *worse* than the tap you gave up. Holding is only correct if you can hit the middle 62%, and only *profitable* if you can hit the middle 30% — which is the skill curve the whole game rests on.
- Perfect: full jump arc (height + distance), plus a small bonus — e.g. a brief speed boost or a combo counter for consecutive perfects (cosmetic flex / leaderboard bragging rights, not a power spiral).
- Good/Okay: reduced arc, still enough to reach *nearby* platforms, not the next tier up. Bad timing should be forgiving at low altitude and punishing at high altitude, by virtue of the platforms themselves being farther apart — not by changing the bar.
- Weak/miss: fall.
- **Takeoff presentation scales with strength**: higher Leg Level/Tier gives a more superhero-style launch — crouch wind-up, ground crater/shockwave, dust burst — instead of a plain jump. Purely cosmetic (doesn't affect jump quality or arc), but makes strength visibly readable to other players in the shared world, same as the leg-growth signal above.

**Server authority (non-negotiable, same principle as prior projects):** the jump bar's sweep must be deterministic and tracked server-side per attempt (seeded, timestamped), and the server judges jump quality from *when* the client's input arrived — never from a client-reported accuracy value. This is the entire scoring mechanic for the game; if it's client-trusted, the game is trivially exploitable.

## The tutorial, and Starter Legs

Three jumps long, and its only real job is to hand over **Starter Legs** — a free pair that puts your jump height up **×10** (and your distance ×3.16; see Numbers §2 for why those differ).

1. *Tap Space to jump.*
2. *Now hold Space instead* — the bar opens, release to lock in.
3. *Land a Perfect.* After five honest attempts it'll settle for a Good, because nobody should be stuck behind a 140 ms window before the game has started.

Then the legs land, with a title card and a jump that suddenly goes ten times higher.

**The point is that everything before the handover is deliberately feeble.** For those thirty seconds you jump like a vanilla Roblox character — 7 studs, barely off the ground, on an island 280 studs across. That's not a placeholder, it's the setup: the ×10 has to be something the player *felt* arrive, not a number they read. It also means the three things they need to know — tap is safe, hold is the bar, the middle is where the game is — get taught while the stakes are nothing.

The steps advance on jumps **the server judged**, never on the client reporting progress. The reward is real power, so the steps are worth faking.

## Legs: two progression tracks

- **Training** (skill, free, repeatable): a **high-gravity training area** on each island where players raise a "Leg Level" stat by doing the one thing the game is already about — jumping. No gym, no machines, no separate verb to learn: training is jumping under resistance.
  - **The resistance is local gravity.** Inside a training area, gravity is simulated higher than the rest of the world, so jumps are shorter, heavier, and slower to recover from. Each completed jump inside the area grants Leg Level, scaled by how heavy that area's gravity is.
  - **The starting island's area is a sand pit** — loose sand that swallows the push-off (as it does in real life), the game's gentlest resistance and the tutorial for the whole mechanic. It reads visually as "this is where jumping is hard" before the player has any numbers to interpret.
  - **Gravity scales with altitude.** Low islands add only a small amount over normal; higher islands stack it far more aggressively. So training is gated behind having climbed there first, not just behind currency, and each area's gravity doubles as its Leg Level cap band.
  - **Each area's gravity is tuned against the strength a player is expected to have when they arrive, so a training jump looks and feels roughly the same height at every altitude.** This is the point of the whole system: the training area is a treadmill that always feels like effort, because your growing legs are cancelled out the moment you step into it. The payoff is visible *outside* the pit — same legs, normal gravity, much bigger arc. It also means heavy gravity can never become unfun-sluggish at the top, since nobody experiences the top island's gravity with bottom island legs.
  - Because heavier gravity grants more Leg Level per jump, a player who has climbed higher trains faster — the reward for pushing your altitude is that everything below it becomes cheap.
  - **Training gain scales with jump quality, not just jump count.** A pit's listed gain is what a *Perfect* earns; a tap earns a tenth of it. Since a tap skips the crouch and a played bar doesn't, this is what stops mindless spamming from being the optimal way to train — playing the bar well is the fastest route through a band, spamming is a slower but valid low-attention one, and half-playing it is worse than either. Same shape as the climb: the bar pays only if you can hit the middle.
  - Pacing target: each island should take roughly the same 3-5 minutes of active training to reach the next island's strength threshold. Achieved by keeping the jump count needed roughly constant per island (~40-60 training jumps) and scaling strength-per-jump with the area's gravity tier, rather than demanding more jumps at higher islands.
  - Same server-authority principle as the jump bar: the server owns which area a player is standing in, what gravity multiplier applies, and whether a training jump actually completed (left the ground, landed inside the area) — never a client-reported count or multiplier. A client that claims it's in the top island's pit while standing in the sand pit is the obvious exploit to close.
  - Implementation note (Roblox): `Workspace.Gravity` is global, so per-area gravity has to be faked per character — a downward `VectorForce`/`LinearVelocity` on the humanoid root plus a reduced `JumpPower` while inside the zone — applied server-side, cleared on exit and on death.
  - **Climbing jumps (outside a training area) also grant a secondary strength trickle**, weighted by jump quality so perfect jumps grant more than weak ones. It's smaller per jump than a training-area jump — normal gravity is the easy setting — so climbing contributes to progress without making the training areas redundant.
  - **Legs visibly grow (bulk up) as Leg Level rises**, then reset to baseline size when the player upgrades their Leg Tier (gear). This makes visible size a live status signal within a tier — since the game is a shared world where players see each other, a visibly bulked-up player reads as "close to their next gear upgrade" — and makes the upgrade decision feel like a real trade: you cash in your visible progress for a higher power ceiling.
- **Upgrading** (currency-bought gear tiers): raises base jump height/distance, and can unlock perks — double jump, fall-damage resist, air dash.
  - **Ten tiers, one per island** — every island you reach unlocks its own tier, sold by a vendor on that island. Junk → mechanical → powered → sci-fi: bare legs, cardboard braces, duct-tape wraps, wooden stilts, coil springs, shock absorbers, hydraulic pistons, turbine calves, rocket boosters, antigrav struts.
  - A tier per island means gear and training each contribute one step per island, so the power curve is smooth. Fewer, larger tiers made it a sawtooth — an island carried by a purchase, then an island carried only by grinding.
  - Each tier costs about three cash-out runs, so arriving somewhere new always sets up a concrete, nearby goal.
- Effective jump stat = Leg Level (skill) × Leg Tier (gear), so neither pure grinding nor pure paying maxes a player out alone.

## World structure

- Players start on a main island with the sand pit — the game's lightest training area. Low islands beyond it are close together and forgiving; higher up, clouds get progressively sparser and training-area gravity gets heavier, gating progress behind both stat upgrades and player skill.
- **Clouds reposition (X/Z) roughly every 30 minutes.** This is the game's "the world changes under you" hook. A reposition for a given level (the band of clouds between two islands) only fires if no player is currently in that level — so nobody gets stranded or repositioned out from under mid-climb. Fixed islands with training areas don't move, only the connecting clouds do. If a level stays occupied for a full hour, the server forces a reposition anyway (see below) rather than letting one player block it forever.
- Checkpoints: on reaching a new cloud/island, it becomes the respawn point. Falling sends the player back to their last checkpoint, not to the ground. No fall damage — a miss costs progress (back to checkpoint), not health. **The one exception is cashing out a medal cache, which deliberately wipes the whole chain back to the bottom** — see Progression & currency.
- **No fast travel.** There is no teleport back to a previously-landed island — every return trip, including going back down to retrain at a lower island's training area, is a real physical climb. This makes altitude genuinely hard-won (you can't cheaply revisit it), but it also puts a real time-cost on the free Training track that didn't exist before — see open questions.

## Visual style reference

**Environment art — low-poly, faceted, storybook-cute.** Reference images: a beach island with wooden docks/bridges, chunky blue-grey faceted boulders, rounded low-poly tree canopies, colorful mushroom and flower props, and triangular bunting flags on poles; a close-up of a wooden bench and layered canopy trees with a hanging lantern, wooden crates, and a striped tree trunk; and the sky reference (floating brown rock islands, puffy faceted clouds with soft pastel-blue-to-grey shading, warm sun bloom). Key traits to keep consistent across builders/assets:
- **Visible facets, not smooth shading** — geometry reads as flat-shaded low-poly planes (trees, rocks, clouds, terrain), not sculpted/high-poly or painted-texture smooth.
- **Saturated but soft palette** — bright, cheerful colors (turquoise water, green foliage, pastel sky) rather than gritty/realistic tones; shading is soft gradient blocks, not harsh contrast.
- **Chunky, rounded silhouettes** — foliage and rock formations are blobby/rounded clusters of low-poly facets, not sharp/angular or spindly.
- **Warm wood + whimsical nature props** for built structures and set dressing: docks, benches, crates, lanterns, mushrooms, flowers, bunting flags — this is the palette to draw island and training-area set-dressing props from.
- **Training areas must read as heavy at a glance**, before any UI explains them: the starting island's sand pit is pale loose sand with a wooden rim and scattered footprint dents; higher islands escalate the same idea (denser, darker, more compressed ground, heavier framing, sagging/strained props around the rim). The visual weight of the ground is the player's cue for how much gravity is waiting in it.
- **Clouds and floating islands** follow the same faceted-low-poly language as the ground-level references, just recolored for sky (white/pale-blue-grey clouds, warm brown/tan rock islands), with soft bloom around light sources.

**UI/HUD layout reference (functional convention only, not art style).** A separate reference image shows a typical Roblox simulator/tycoon HUD: stacked resource counters top-left, each with an icon and a `+` button (e.g. gems, a leveled stat bar, multiple currencies), and a vertical column of action buttons top-right/side (Settings, Help, Codes, Invite, Upgrade, Sell, Packs, Pets, Trade). That reference game's actual theme/art (anime effects, ninja icons) doesn't apply here — only the layout convention does. For Leg Day, adapt the same structure: a compact top-left stack for both currencies (Medals, Coins) + Leg Level/current altitude, and a right-side vertical rail for menu actions (Upgrade legs, Pets, Auras, Leaderboard, Settings), styled in the low-poly/pastel palette above rather than the reference's anime look.

## Progression & currency

Two currencies, kept deliberately separate so power and vanity never compete for the same wallet:

- **Medals — spent on Leg Tier upgrades (gear/power). Earned by cashing out a run.** Each island holds a single **medal cache**. Touching it banks the medals and **teleports you back to the bottom**, so you get one cache per climb and every run is the same decision: *how high do I dare go before cashing out?* Higher caches are worth more, so the ceiling of your legs sets your earn rate, and a Leg Tier costs several full round trips. Reaching an island for the first time also pays a one-time arrival bonus.
  - **Collecting is voluntary and arrival never triggers it.** You walk into the cache deliberately. If landing on a new island ejected you automatically, pushing for a personal best would punish you and the risk/reward decision would evaporate.
  - **It takes a deliberate confirm, not a touch.** Proximity opens a prompt and collection fires on a short hold. Ending a twelve-minute climb is far too expensive to hang off a collision check — especially on a small high island with other players jostling around the cache.
  - **Cashing out clears your checkpoint chain back to the first island.** Without this the loop is trivially broken: bank the cache, jump off deliberately, and respawn at your top checkpoint with the medals already paid. Award, wipe checkpoints, and teleport must be one server-side transaction.
  - Medals stay scarce and altitude-gated — you can't grind them in place, only by climbing. The cost is measured in *climbs*, which is why the numbers stay small (three digits all game).
- **Coins — spent on pets and auras (cosmetics only, no stat effect). Earned on every jump, scaled by how high that jump peaked.** No altitude term and no island multiplier: the payout is a function of the arc itself, so stronger legs and better timing both pay, and the same upgrade that doubles your reach multiplies your income far more. Coins are abundant and farmable by playing a lot, which is exactly right for a vanity sink — a grinder inflating their coin total only buys flair, never power.
  - **This makes training a deliberate coin desert.** Pit jumps are pinned to a constant low height by design, so they pay a flat trivial amount at every altitude and never improve. Coins come from climbing tall.
  - **The two loops interlock.** The re-climbs the medal loop forces on you are exactly when the coin loop pays out, so the repetition is never unpaid.
- Leaderboard for max height reached — cheap, strong retention hook for the genre.

## Monetization (proposed, not settled)

- Cosmetic leg skins, pets, and auras: fine, no gameplay impact. Since pets/auras are the deep, ever-expandable side of the economy (new pets, new aura rarities, duplicates), they're also the coin sink that's meant to keep absorbing the currency's unbounded, playtime-driven supply — don't let this catalog ship shallow, or coins pile up with nowhere to go.
- Gamepasses that affect jump power: risky — undercuts the skill-based hook of the whole game if players can pay past the timing mechanic. Lean toward monetizing training speed (e.g. a "double training XP" pass) rather than raw power, so payment saves time but doesn't replace skill.
- A direct real-money Medal purchase is the highest-risk gamepass to ever consider: Medals buy Leg Tier (power), so selling them directly would let players pay past the climb itself, not just past the jump-bar timing. If Medals are ever monetized, do it indirectly (e.g. a small bonus-medal pass tied to reaching a *new* personal-best altitude, never a flat currency purchase).

## Decisions that are settled — don't relitigate without discussing first

- Jump quality is judged server-side from input timing, never trusted from the client.
- **A three-jump tutorial hands over free Starter Legs worth ×10 jump height.** Before it you jump like a vanilla character, on purpose, so the handover is felt rather than read. The lift is vertical only — height ×10, distance ×3.16 — which is what keeps the built island the right size to walk around.
- Horizontal launch is damped to 35% inside a training pit: training is jumping on the spot, so a pit never has to be as wide as your reach.
- **Tap to jump, hold for the bar.** A tap is a guaranteed Okay, fired instantly on release with no bar drawn; holding past the crouch opens the bar and releasing locks it in. The bar is opt-in, and taking it risks a Weak that's worse than the tap. The crouch wind-up is purely the threshold between the two, so the input has one timing rule, not two — and that holds inside training pits as well, where the crouch is now paid only by players who choose the bar. This means the pit crouch no longer paces training; see Numbers §4.2, which is an open balance question rather than a settled answer.
- Falling returns the player to their last checkpoint, not to the ground.
- Two separate progression tracks (skill-based training, currency-based upgrading) rather than one.
- Training happens in high-gravity training areas located on islands, not on a separate ground-level course. Training is jumping under increased simulated gravity — there is no gym, no weight machines, and no click-to-do-reps mechanic.
- Gravity (and the Leg Level cap it unlocks) scales with altitude: small increases on low islands, much larger ones higher up.
- Training-area gravity is tuned against expected leg strength at that altitude, so training jumps stay roughly the same height everywhere. The area always feels like effort; the strength gain shows up outside it, under normal gravity.
- The starting island's training area is a sand pit — the lightest resistance in the game and the tutorial for the mechanic.
- Players start on a main island with the sand pit.
- No fast travel. Every return trip, including retraining at a lower island, is a physical climb.
- Two currencies, split by purpose: Medals spend on Leg Tier upgrades only; Coins spend on pets and auras only. Power and cosmetics never share a wallet.
- Medals are earned by **cashing out a run**: one voluntary medal cache per island, collecting it teleports you to the bottom, so a Leg Tier costs several full climbs. Higher caches pay more. Plus a one-time arrival bonus per island.
- Cashing out **resets the checkpoint chain to the bottom**. Arriving on an island never auto-triggers the cache, and collecting takes a deliberate confirm rather than a touch.
- Coins are earned **per jump, scaled by the height of that jump** — not by altitude or island. Training-pit jumps are deliberately low, so training is the worst coin rate in the game and never improves.
- **Ten Leg Tiers, one unlocked per island**, each costing roughly three cash-out runs.
- A cloud level only repositions when no player is currently in it; islands themselves never move. Exception: if a level has stayed occupied for a full hour (someone parked in it blocking the reshuffle), the server forces it anyway — a 15-second warning fires, then the new cloud set fades in while the old set fades out, with both sets solid/collidable during the overlap so nobody falls through the transition, before the old set fully disappears.
- No fall damage. Missing a jump costs a return trip to the last checkpoint, not health.
- **Cooperative, not competitive.** No racing/PvP framing — but it's a shared world: players can see each other climbing, training, and jumping, even though there's no head-to-head win condition.
- Servers are capped at 16 players, with 32 as a stretch target — worth raising if a prototype shows no major performance hit, but 16 is the safe baseline to ship with if it doesn't. Overflow beyond the cap routes players to a new server instance (standard Roblox behavior), not a queue.
- If a new player enters a level during its forced-reposition warning or fade transition, they're caught in the swap along with everyone already there.
- Training gains scale with the training area's gravity tier, so climbing higher makes training faster — not with input speed. The server decides which area a player is in and whether a training jump completed.
- Training gains also scale with **jump quality**: a pit's listed gain is what a Perfect earns, a tap earns a tenth. Weights are steeper than the climbing trickle's (Numbers §4.2.1) because a tap skips the crouch and a played bar doesn't — at the climbing weights, spamming would out-train skilled play three to one.
- The 1-hour occupied-timer for a stuck level resets instantly if the level becomes fully empty (all players leave); it only keeps counting up while at least one player remains in it continuously.
- Legs visibly bulk up as Leg Level rises, and reset to baseline size on Leg Tier upgrade.
- Jump takeoffs get progressively more superhero-style (crouch wind-up, crater/shockwave, dust) as strength rises. Cosmetic only — doesn't affect jump quality or arc.
- Environment art style is low-poly/faceted with a saturated-but-soft palette and chunky rounded silhouettes (see Visual style reference). HUD layout follows the standard Roblox simulator convention (top-left resource stack, right-side action rail) restyled to match.

## Open questions (need a prototype, not more design docs)

- ~~**How does a player retrain after climbing higher, now that fast travel is gone?**~~ **Resolved by the medal cash-out loop.** The cash-out *is* the ride down, and it's opt-in — so retraining stops being a dedicated trip. You bank at a high island, land at the bottom, and the re-climb passes every pit on the way up, topping up each band in passing on a journey you were making anyway. No recall mechanic needed.
- **Does removing fast travel create backtracking chokepoints?** *Sharper now, not softer.* The cash-out loop funnels every player through the bottom island constantly — its spawn, arrival pad, sand pit entrance and first ledges are now the highest-traffic geometry in the game, and a full playthrough runs them ~22 times. Oversizing the first island (see Numbers) is the proposed mitigation, but this needs a prototype with more than one player in it before it's trusted.
- **Does the reshuffle interval fight the grind?** Cloud repositioning bites harder under this loop: a player re-runs a memorised route ~22 times, and a reshuffle invalidates that muscle memory. That's the "world changes under you" hook working as intended — but 30 minutes may be the line between "the world changed" and "my grind route got deleted." Worth watching.
- **Exact gravity curve per island**: since gravity tracks expected leg strength, the real question is how tightly it tracks — whether it's a straight formula off the island's Leg Level band (so training height is near-identical everywhere) or deliberately runs slightly ahead of it, so each new area feels heavy for a few jumps before you settle in. The second is more interesting but needs a feel test.
- **What happens to an under-levelled player in a high area?** Gravity is tuned for the strength you're *supposed* to have on arrival, so someone who fast-travels or gets carried past their training will find that pit disproportionately brutal. Is that a fine, self-correcting "go train lower first" signal, or does it need a floor on air-time so the area is never unusable?
- **Exact training-jump count and strength-per-jump scaling**: the ~40-60 jump / 3-5 min target is a design goal, not a tuned number — needs playtesting to land the curve and confirm the pacing holds.
- **Jump-strength trickle balance**: how much strength should a perfect climbing jump grant relative to a training-area jump, so climbing feels rewarding without letting players skip training entirely by just climbing more?
- **Leg-growth and takeoff-effect tiers**: how many visible size/effect stages per Leg Tier (continuous scaling vs. a handful of discrete stages), and whether the reset-on-upgrade should be instant or a small "shrink" animation/moment of its own — needs an art/animation pass, not just a design call.
- **16 vs. 32 player cap**: needs an actual server-performance prototype (client physics/rendering load with that many players jumping at once, including several running custom per-character gravity forces in training areas) to decide which one ships.

## Working together

(Carried over convention from prior project — update once collaborators/tooling for this project are confirmed.)

- Roblox Studio's Team Create for building the same place file together, if this ends up multi-collaborator.
- Rojo recommended for syncing Luau scripts to text files in this repo for real git diffs/history.
