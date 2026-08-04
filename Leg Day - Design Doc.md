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

**Server authority (non-negotiable, same principle as prior projects):** the jump bar's sweep must be deterministic and tracked server-side per attempt (seeded, timestamped), and the server judges jump quality from *when* the client's input arrived — never from a client-reported accuracy value. This is the entire scoring mechanic for the game; if it's client-trusted, the game is trivially exploitable.

## Legs: two progression tracks

- **Training** (skill, free, repeatable): a ground-level practice course or timing drill that raises a "Leg Level" stat. Rewards playtime and skill, not just spend.
- **Upgrading** (currency-bought gear tiers: cardboard → springs → pistons → rockets, etc.): raises base jump height/distance, and can unlock perks — double jump, fall-damage resist, air dash.
- Effective jump stat = Leg Level (skill) × Leg Tier (gear), so neither pure grinding nor pure paying maxes a player out alone.

## World structure

- Low islands are close together and forgiving. Higher up, clouds get progressively sparser, gating progress behind both stat upgrades and player skill.
- **Clouds reposition (X/Z) roughly every 30 minutes.** This is the game's "the world changes under you" hook, but it's also the biggest fairness risk — see Open Questions.
- Checkpoints: on reaching a new cloud/island, it becomes the respawn point. Falling sends the player back to their last checkpoint, not to the ground — full resets would make high-altitude play too punishing to be fun; zero penalty removes the tension the game is built on.

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

## Open questions (need a prototype, not more design docs)

- **Cloud reposition fairness**: what happens to a player mid-air toward a cloud, or standing on one, when a shift fires? Candidates: (a) shifts only ever apply to clouds above the highest point any current player has reached; (b) a visible telegraph/warning a few seconds before a shift; (c) the server simply defers a cloud's shift until no player is airborne toward it or standing on it. Needs picking before building the reposition system.
- **Competitive or cooperative?** Solo race to personal best, or a shared world where players see each other climbing? Affects server/instance design significantly.
- **Fall damage**: instant "you missed, respawn at checkpoint," or a health/fall-damage system with some margin for near-misses?
- **Training minigame design**: what does the ground-level practice course actually look like, and how much Leg Level can it grant vs. how much must come from Leg Tier purchases?

## Working together

(Carried over convention from prior project — update once collaborators/tooling for this project are confirmed.)

- Roblox Studio's Team Create for building the same place file together, if this ends up multi-collaborator.
- Rojo recommended for syncing Luau scripts to text files in this repo for real git diffs/history.
