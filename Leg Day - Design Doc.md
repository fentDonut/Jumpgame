# Leg Day (working title) — Design Doc

Roblox game: a vertical climbing game where the only way up is a timing-based jump. Players train and upgrade their legs to jump farther, and climb an endless sky of islands and clouds that get sparser — and periodically rearrange — the higher you go.

## Core loop

1. Stand at the edge of a platform.
2. Press **Space** to open the jump bar — an indicator sweeps across it.
3. Press **Space** again to lock in. Distance from center determines jump quality: Perfect / Good / Okay / Weak.
4. Land on the next island or cloud. Miss, and you fall to the last checkpoint.
5. Currency and height records accumulate as you climb; spend currency on leg upgrades to reach higher.

## The jump bar

- Perfect: full jump arc (height + distance), plus a small bonus — e.g. a brief speed boost or a combo counter for consecutive perfects (cosmetic flex / leaderboard bragging rights, not a power spiral).
- Good/Okay: reduced arc, still enough to reach *nearby* platforms, not the next tier up. Bad timing should be forgiving at low altitude and punishing at high altitude, by virtue of the platforms themselves being farther apart — not by changing the bar.
- Weak/miss: fall.
- **Takeoff presentation scales with strength**: higher Leg Level/Tier gives a more superhero-style launch — crouch wind-up, ground crater/shockwave, dust burst — instead of a plain jump. Purely cosmetic (doesn't affect jump quality or arc), but makes strength visibly readable to other players in the shared world, same as the leg-growth signal above.

**Server authority (non-negotiable, same principle as prior projects):** the jump bar's sweep must be deterministic and tracked server-side per attempt (seeded, timestamped), and the server judges jump quality from *when* the client's input arrived — never from a client-reported accuracy value. This is the entire scoring mechanic for the game; if it's client-trusted, the game is trivially exploitable.

## Legs: two progression tracks

- **Training** (skill, free, repeatable): a gym on each island, with leg machines (squat rack, calf raise, leg press, etc.) that raise a "Leg Level" stat. Rewards playtime and skill, not just spend.
  - The starting/main island has only basic equipment (light weights, low Leg Level cap).
  - Every island higher up has heavier weights than the one before, raising the Leg Level cap further — so training itself is gated behind having climbed there first, not just behind currency.
  - Mechanic: click the screen to do reps. A rep has a 2-second charge-up during which sustained clicking raises the gain multiplier, scaling with click speed up to 5 clicks/sec, where it locks at a 2x cap (going faster than 5 cps grants nothing further). Rewards active engagement over walking away and idling, without turning training into an unbounded numbers race.
  - Same server-authority principle as the jump bar: the server should count/validate click timestamps itself (and can rate-limit or flag inhuman click rates) rather than trusting a client-reported multiplier — otherwise an autoclicker/macro trivially sits at the 2x cap forever.
  - Pacing target: each island should take roughly the same 3-5 minutes of active training to reach the next island's strength threshold — long enough to feel like progress, short enough to not lose players. Achieved by keeping the rep count needed roughly constant per island (~40-60 reps) and scaling strength-per-rep with the machine's weight tier, rather than requiring more reps at higher islands.
  - **Jumping itself also grants a secondary strength trickle** (smaller than dedicated gym training), weighted by jump quality so perfect jumps grant more than weak ones — climbing contributes to progress without making the gym redundant.
  - **Legs visibly grow (bulk up) as Leg Level rises**, then reset to baseline size when the player upgrades their Leg Tier (gear). This makes visible size a live status signal within a tier — since the game is a shared world where players see each other, a visibly bulked-up player reads as "close to their next gear upgrade" — and makes the upgrade decision feel like a real trade: you cash in your visible progress for a higher power ceiling.
- **Upgrading** (currency-bought gear tiers: cardboard → springs → pistons → rockets, etc.): raises base jump height/distance, and can unlock perks — double jump, fall-damage resist, air dash.
- Effective jump stat = Leg Level (skill) × Leg Tier (gear), so neither pure grinding nor pure paying maxes a player out alone.

## World structure

- Players start on a main island with basic gym equipment. Low islands beyond it are close together and forgiving; higher up, clouds get progressively sparser, gating progress behind both stat upgrades and player skill.
- **Clouds reposition (X/Z) roughly every 30 minutes.** This is the game's "the world changes under you" hook. A reposition for a given level (the band of clouds between two islands) only fires if no player is currently in that level — so nobody gets stranded or repositioned out from under mid-climb. Fixed islands with gyms don't move, only the connecting clouds do. If a level stays occupied for a full hour, the server forces a reposition anyway (see below) rather than letting one player block it forever.
- Checkpoints: on reaching a new cloud/island, it becomes the respawn point. Falling sends the player back to their last checkpoint, not to the ground. No fall damage — a miss costs progress (back to checkpoint), not health.
- **Fast travel**: free and instant, once a player has physically landed on an island. This means the climb only has to be earned once per island — teleporting is for returning to train at a gym you've already unlocked or for retrying the climb from a high point, not a way to skip the jump challenge itself (you still have to physically reach an island the first time).

## Visual style reference

**Environment art — low-poly, faceted, storybook-cute.** Reference images: a beach island with wooden docks/bridges, chunky blue-grey faceted boulders, rounded low-poly tree canopies, colorful mushroom and flower props, and triangular bunting flags on poles; a close-up of a wooden bench and layered canopy trees with a hanging lantern, wooden crates, and a striped tree trunk; and the sky reference (floating brown rock islands, puffy faceted clouds with soft pastel-blue-to-grey shading, warm sun bloom). Key traits to keep consistent across builders/assets:
- **Visible facets, not smooth shading** — geometry reads as flat-shaded low-poly planes (trees, rocks, clouds, terrain), not sculpted/high-poly or painted-texture smooth.
- **Saturated but soft palette** — bright, cheerful colors (turquoise water, green foliage, pastel sky) rather than gritty/realistic tones; shading is soft gradient blocks, not harsh contrast.
- **Chunky, rounded silhouettes** — foliage and rock formations are blobby/rounded clusters of low-poly facets, not sharp/angular or spindly.
- **Warm wood + whimsical nature props** for built structures and set dressing: docks, benches, crates, lanterns, mushrooms, flowers, bunting flags — this is the palette to draw island/gym set-dressing props from.
- **Clouds and floating islands** follow the same faceted-low-poly language as the ground-level references, just recolored for sky (white/pale-blue-grey clouds, warm brown/tan rock islands), with soft bloom around light sources.

**UI/HUD layout reference (functional convention only, not art style).** A separate reference image shows a typical Roblox simulator/tycoon HUD: stacked resource counters top-left, each with an icon and a `+` button (e.g. gems, a leveled stat bar, multiple currencies), and a vertical column of action buttons top-right/side (Settings, Help, Codes, Invite, Upgrade, Sell, Packs, Pets, Trade). That reference game's actual theme/art (anime effects, ninja icons) doesn't apply here — only the layout convention does. For Leg Day, adapt the same structure: a compact top-left stack for currency + Leg Level/current altitude, and a right-side vertical rail for menu actions (Upgrade legs, Fast Travel list, Leaderboard, Settings), styled in the low-poly/pastel palette above rather than the reference's anime look.

## Progression & currency

- Currency sources: passive trickle for climbing, plus a one-time bonus the first time a player reaches a new altitude band (encourages pushing your personal best, not just farming a comfortable low floor).
- Leaderboard for max height reached — cheap, strong retention hook for the genre.

## Monetization (proposed, not settled)

- Cosmetic leg skins: fine, no gameplay impact.
- Gamepasses that affect jump power: risky — undercuts the skill-based hook of the whole game if players can pay past the timing mechanic. Lean toward monetizing training speed (e.g. a "double training XP" pass) rather than raw power, so payment saves time but doesn't replace skill.

## Decisions that are settled — don't relitigate without discussing first

- Jump quality is judged server-side from input timing, never trusted from the client.
- Falling returns the player to their last checkpoint, not to the ground.
- Two separate progression tracks (skill-based training, currency-based upgrading) rather than one.
- Training happens at gyms located on islands, not on a separate ground-level course. Weights (and the Leg Level cap they unlock) get heavier at higher islands.
- Training is click-based: clicking faster multiplies gains, capped at 2x.
- Players start on a main island with basic equipment.
- Fast travel to any previously-landed island is free and instant; this doesn't bypass the jump challenge for reaching it the first time.
- A cloud level only repositions when no player is currently in it; islands themselves never move. Exception: if a level has stayed occupied for a full hour (someone parked in it blocking the reshuffle), the server forces it anyway — a 15-second warning fires, then the new cloud set fades in while the old set fades out, with both sets solid/collidable during the overlap so nobody falls through the transition, before the old set fully disappears.
- No fall damage. Missing a jump costs a return trip to the last checkpoint, not health.
- **Cooperative, not competitive.** No racing/PvP framing — but it's a shared world: players can see each other climbing, training, and jumping, even though there's no head-to-head win condition.
- Servers are capped at 16 players, with 32 as a stretch target — worth raising if a prototype shows no major performance hit, but 16 is the safe baseline to ship with if it doesn't. Overflow beyond the cap routes players to a new server instance (standard Roblox behavior), not a queue.
- If a new player enters a level during its forced-reposition warning or fade transition, they're caught in the swap along with everyone already there.
- Click-training: gains scale with click speed up to 5 clicks/sec, at which point the player is locked at the 2x multiplier (going faster than 5 cps grants nothing further). There's a 2-second charge-up before the multiplier kicks in, so a rep needs sustained clicking, not a single burst.
- The 1-hour occupied-timer for a stuck level resets instantly if the level becomes fully empty (all players leave); it only keeps counting up while at least one player remains in it continuously.
- Legs visibly bulk up as Leg Level rises, and reset to baseline size on Leg Tier upgrade.
- Jump takeoffs get progressively more superhero-style (crouch wind-up, crater/shockwave, dust) as strength rises. Cosmetic only — doesn't affect jump quality or arc.
- Environment art style is low-poly/faceted with a saturated-but-soft palette and chunky rounded silhouettes (see Visual style reference). HUD layout follows the standard Roblox simulator convention (top-left resource stack, right-side action rail) restyled to match.

## Open questions (need a prototype, not more design docs)

- **Exact rep count and per-island weight scaling**: the ~40-60 rep / 3-5 min target is a design goal, not a tuned number — needs actual playtesting to land the weight-per-rep curve and confirm the pacing holds up in practice.
- **Jump-strength trickle balance**: how much strength should a perfect jump grant relative to a full gym rep, so climbing feels rewarding without letting players skip the gym entirely by just climbing more?
- **Leg-growth and takeoff-effect tiers**: how many visible size/effect stages per Leg Tier (continuous scaling vs. a handful of discrete stages), and whether the reset-on-upgrade should be instant or a small "shrink" animation/moment of its own — needs an art/animation pass, not just a design call.
- **16 vs. 32 player cap**: needs an actual server-performance prototype (client physics/rendering load with that many players jumping and clicking at once) to decide which one ships.

## Working together

(Carried over convention from prior project — update once collaborators/tooling for this project are confirmed.)

- Roblox Studio's Team Create for building the same place file together, if this ends up multi-collaborator.
- Rojo recommended for syncing Luau scripts to text files in this repo for real git diffs/history.
