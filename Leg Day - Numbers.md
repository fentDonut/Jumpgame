# Leg Day — Numbers (first tuning pass)

Concrete values for jump strength, gravity, Leg Level, Medals, and world geometry. Companion to [Leg Day - Design Doc.md](Leg%20Day%20-%20Design%20Doc.md) — that doc owns the *design*, this one owns the *numbers*. Where a number answers an open question, it's marked **[answers OQ]**.

All numbers are a first pass calibrated against real Roblox physics and shipped-game economies (see [Reference data](#reference-data-what-these-are-calibrated-against) at the end). They're meant to be tuned, not treated as final.

---

## 0. The two loops these numbers serve

**Jumping — the power loop. Jump power is earned by jumping, and by nothing else.** Every jump the server judges grants **Leg Level**, and Leg Level is the *only* input to your jump stat. There is no purchasable power anywhere in the game: a player who never buys anything still reaches the summit, just slowly. Pits (§4) are the concentrated form of this; ordinary climbing jumps pay a smaller trickle (§4.5).

**Medals — the run loop, and the accelerator.** Each island holds a **medal cache**. Touching it banks the medals *and teleports you to the bottom*, clearing your checkpoint chain. One cache per run, so every run is a single decision: **how high do I dare climb before cashing out?** Higher caches are worth more per minute, so the ceiling of your legs sets your earn rate. Medals are the game's **only** currency and they buy two things off the same shelf: **Leg Tiers**, which multiply how much Leg Level every jump grants, and **cosmetics** (eggs, pets, auras), which do nothing at all.

The two loops interlock, and more tightly than the old two-currency version did: the re-climbs the medal loop forces on you are exactly when the power loop pays out. **Repetition is never unpaid — it pays in strength, not in a vanity token.**

> **The single-wallet consequence, stated plainly.** Medals now fund power-rate and flair from the same pot, which the previous design deliberately avoided. What keeps this from being pay-to-win is that a tier buys *rate*, never reach: no amount of medals raises your jump stat by a single stud. What it does cost is the old clean split — buying an aura genuinely delays your next tier, and that tension is now the economy's main decision. Priced accordingly in §6.

---

## 1. Physics baseline (real Roblox constants)

**The place runs at 3× gravity: `Workspace.Gravity = 588.6`.** With heights as large as Starter Legs make them, vanilla gravity leaves you hanging — a 48-stud jump takes 1.41 s to come down, and a jump you wait out is not a jump that feels good. At 3× the same jump takes 0.81 s.

**Nothing else moved.** Both launches are derived from a target *height* against whatever gravity is in force (`Config.launchVy`), and horizontal scales with it (`Config.launchVx`), so every height and reach in this document holds exactly as written. Raising gravity buys air *time* and costs nothing else. The 196.2 below stays as the **reference** the design is defined against, not the value the game runs at.

| Constant | Value | Note |
|---|---|---|
| Gravity (reference) | 196.2 studs/s² | `Workspace.Gravity` default — what the baseline heights are defined against |
| Gravity, rising | **392.4 studs/s²** | `Workspace.Gravity`, 2× vanilla |
| Gravity, falling | **147.2 studs/s²** | 0.75× vanilla — the descent hangs |

**The arc is asymmetric, and both halves are set independently.** No single gravity value gives this, because a symmetric arc falls exactly as fast as it rises: lowering gravity to float the descent also makes the take-off mushy, and raising it to sharpen the take-off drops you like a stone. So the fall is lightened separately, by a `VectorForce` holding up the difference between the two while descending (`MoonFallClient`).

Both are **absolute** values, deliberately — the fall was briefly expressed as a fraction of the rise, which meant retuning the ascent silently moved the descent with it. A jump now rises in 0.50 s and takes 0.81 s to come down. **Measured in-game: 0.46 s up, 0.79 s down** (the recorder starts the frame the feet clear the floor, so "up" reads slightly short).

Two things fall out of this and are easy to forget:
- **It's client-side, and it has to be.** Written server-side first, it only reached a 1.2× ratio — the server reads a replicated velocity that lags, so "am I descending yet" arrived late and the force switched on after much of the fall was over. The client owns the character's physics and knows its velocity exactly. Nothing here is worth defending: disabling it only makes you fall faster.
- **It doesn't apply inside a training pit**, where a heavy fall is the entire point of the place.
| Default `JumpPower` | 50 | → peak height 6.37 studs |
| Default `JumpHeight` | 7.2 studs | ≡ `JumpPower` 53.2 — **we use 53.2 as our S=1 baseline** |
| Default `WalkSpeed` | 16 studs/s | |
| Vanilla horizontal jump | ~10–12 studs | community-measured |
| Character height (R15) | ~5 studs | ~2 studs radius |
| Stud → metre | 1 stud = 0.28 m | see §9 on what to *display* |

---

## 2. Jump Stat (S) — the single power number

Everything about jump power collapses into one scalar, **S**, where **S = 1.00 is a vanilla Roblox character**.

```
S    = LevelMult × QualityMult
Lift = √10 once the tutorial hands over Starter Legs, otherwise 1
```

**There is no TierMult any more, and its absence is the whole point of the economy.** Leg Level — earned by jumping — carries the entire 1.00 → 2.75 range on its own. Leg Tier does not appear in this formula at all; it multiplies how fast Leg Level accrues (§2.1), never how far you jump. Two players at the same Leg Level jump identically whether one of them owns Antigrav Struts and the other is barefoot.

Applied on takeoff, server-side, as a single velocity write:

```lua
local Vy = 53.2 * S * Lift   -- Starter Legs lift the launch...
local Vx = 22.0 * S          -- ...and nothing else
hrp.AssemblyLinearVelocity = aim * Vx + Vector3.new(0, Vy, 0)
```

**Lift is vertical only, and that is the whole design of it.** Height goes as `Vy²`, so ×√10 on Vy is exactly ×10 on height. Reach is `Vx × airtime` and only the airtime moved, so reach goes ×3.16. The jump becomes a *moon jump* rather than a long jump — which is what keeps a 280-stud island somewhere you walk around instead of clear in two hops.

Derived arc (gravity 196.2), with Starter Legs:

| Quantity | Formula | S=1.00 | S=2.75 (max) |
|---|---|---|---|
| Rise time | `√(2h / g_rise)` | 0.61 s | 1.67 s |
| Fall time | `√(2h / g_fall)` | 0.99 s | 2.72 s |
| **Airtime (flat)** | rise + fall | **1.60 s** | **4.38 s** |
| **Peak height** | `Vy² / 2g` = `72·S²` | 72.1 studs | 544.2 studs |
| Flat reach | `Vx · airtime` = `49.7·S²` | 49.7 studs | 375.0 studs |
| **Rising reach** (see §7.1) | `0.835 × flat` | 41.5 studs | 313.1 studs |

Reach is **1.32× the symmetric figure**, because a lighter fall is a longer flight and horizontal speed doesn't change during one. That factor is `Config.airtimeStretch()`, and it moves whenever either gravity does.

Without the legs — which is only ever the ~30 seconds of the tutorial — the same table reads 0.54 s / 7.2 / 11.9 / 9.9. That 11.9-stud flat reach lands inside the measured 10–12 stud vanilla range, so the model is still anchored to real Roblox behaviour; Starter Legs are a deliberate step off that anchor, not a drift from it.

> **The float at the top of the curve is what 3× gravity is for.** Endgame legs peak at 544 studs, which at vanilla gravity meant 4.7 seconds of hang time per hop, twenty times a run. At 3× it's 2.7 s — still substantial, and still the first thing to feel at island 9–10 before trusting §7's spacing, but no longer a wait.

Distance and height both scale with **S²**, so power feels superlinear: doubling S nearly quadruples your reach. That's the whole "superhero legs" payoff, and it's why the power number itself can stay small.

**Full range: S = 1.00 → 2.75.** Deliberately small and readable. What inflates instead is **Leg Level** (§4.1), which runs 0 → 51,000 across the arc — that's the simulator-sized number the HUD gets to count up.

### 2.1 Tier multipliers — ten tiers, one per island

Every island unlocks its own tier, so gear is a milestone you hit on arrival rather than a rare event. **A tier multiplies the Leg Level every jump grants — it does not touch S.** Spanning **×1.0 → ×8.0**.

**"Unlocks at" is a gate, not a location.** All ten tiers are sold from the one shopfront on island 1; islands 2–10 have no vendor. Reaching island N adds tier N to that shelf, and you buy it next time a cash-out drops you at the bottom — which is every run.

| # | Leg Tier | **TierGain** | Unlocks at |
|---|---|---|---|
| 1 | Bare Legs | **1.0** | spawn |
| 2 | Cardboard Braces | **1.4** | Island 2 |
| 3 | Duct-Tape Wraps | **1.6** | Island 3 |
| 4 | Wooden Stilts | **2.0** | Island 4 |
| 5 | Coil Springs | **2.5** | Island 5 |
| 6 | Shock Absorbers | **3.0** | Island 6 |
| 7 | Hydraulic Pistons | **4.0** | Island 7 |
| 8 | Turbine Calves | **4.5** | Island 8 |
| 9 | Rocket Boosters | **6.0** | Island 9 |
| 10 | Antigrav Struts | **8.0** | Island 10 |

Junk → mechanical → powered → sci-fi, so the silhouette escalates visibly across the climb.

**The column is exactly the old `Pit gain / jump` column** (10, 14, 16, 20, 25, 30, 40, 45, 60, 80), divided by 10 and moved from the island to the gear. That is the whole restructure in one line: the escalation that used to be a property of *where you were standing* is now a property of *what you bought*, and the base rate is flat everywhere (§4.1). It also means a player on schedule — tier N held at island N — sees exactly the pacing the old table was tuned for.

**×8 top to bottom is chosen against the "never buy anything" path, not against the schedule.** A player who stays on Bare Legs forever still reaches the summit; §8 puts it at roughly **5¾ hours of pit training** against 1¾ on schedule. Steeper tiers would make each purchase feel more decisive but would push the barefoot route from *slow* to *fictional*, and the barefoot route being genuinely walkable is the design.

### 2.2 Level multiplier — the only thing that moves S

```
LevelMult = 1 + 1.75 × (LegLevel / 51000)     -- 1.00 at L0, 2.75 at L51000 (cap of arc 1)
```

Training carries the full **2.75×** now that gear carries none of it. The coefficient (0.72 → 1.75) and the cap (17,000 → 51,000) both moved, and the two moves are unrelated: the coefficient absorbs the tier multiplier that was removed, the cap triples the track so the tier *rate* multiplier has enough runway to matter (§4.1).

**The per-island S values are preserved** — within 3% from island 3 up, and +6% at island 1 (in the player's favour, on the island that was already impossible to fall off). This is deliberate and load-bearing: every gap, cloud diameter, altitude and pit-gravity figure in §4.3 and §7 is derived from that curve, and **none of them has to move**. See §2.4.

### 2.3 Jump-bar quality multipliers

| Result | QualityMult (S) | Distance | Leg Level gained |
|---|---|---|---|
| Perfect | 1.00 | 100% | 100% (§4.2.1 / §4.5 weights) |
| Good | 0.92 | 85% | 60% pit / 70% climb |
| Okay | 0.82 | 67% | 20% pit / 40% climb |
| Weak | 0.65 | 42% | 3% pit / 15% climb |
| Miss (no lock-in) | 0.45 | 20% | 0 |

Timing now pays twice and in the same direction: a Perfect goes further *and* trains you faster. The gain weights are their own table rather than a function of S, because gain must not compound with power — see §6.2.

### 2.4 The resulting power curve

Peak and reach below are **with Starter Legs**, i.e. what a player actually has from the end of the tutorial onward. **The tier column is gone from this table on purpose** — what tier you hold has no bearing on any number in it; it only sets how long you spend getting to each row.

| Level | Leg Level | **S** | Peak height | Rising reach | *(old S, for comparison)* |
|---|---|---|---|---|---|
| 1 → 2 | 1,500 | 1.05 | 80 | 35 | *1.02* |
| 2 → 3 | 3,600 | 1.12 | 91 | 40 | *1.11* |
| 3 → 4 | 6,000 | 1.21 | 105 | 46 | *1.20* |
| 4 → 5 | 9,000 | 1.31 | 123 | 54 | *1.32* |
| 5 → 6 | 12,750 | 1.44 | 149 | 65 | *1.45* |
| 6 → 7 | 17,250 | 1.59 | 182 | 80 | *1.61* |
| 7 → 8 | 23,250 | 1.80 | 233 | 102 | *1.81* |
| 8 → 9 | 30,000 | 2.03 | 296 | 130 | *2.05* |
| 9 → 10 | 39,000 | 2.34 | 394 | 173 | *2.35* |
| 10 → summit | 51,000 | 2.75 | 544 | 239 | *2.75* |

The last column is the check that matters: the curve a barefoot-forever player walks is the same curve the geared player walks, and it is the same curve the world was built against. **Nothing in §7 needs rebuilding.**

### 2.5 Starter Legs and the tutorial

The free pair the tutorial hands over, and the only multiplier in the game nobody has to earn.

| Parameter | Value |
|---|---|
| Lift | **√10 = 3.1623**, applied to `Vy` only |
| Effect | **height ×10**, distance ×3.16, airtime ×3.16 |
| Cost | free |
| Granted | on completing step 3 below |
| Stacks with | Level, multiplicatively — it sets the baseline the whole §2.4 curve sits on. (Not Tier: Tier isn't in S.) |
| Lost | never |

**Before the handover you are a vanilla Roblox character:** 7.2 stud peak, 11.9 stud reach, 0.54 s of airtime, on an island 280 studs across. That is the setup, not a placeholder — the ×10 has to be something the player felt arrive.

| Step | Completes on | Teaches |
|---|---|---|
| 1 | any **tapped** jump | tap is the safe jump |
| 2 | any **held** (judged) jump | holding opens the bar, releasing locks it |
| 3 | a **Perfect** | the middle of the bar is the whole game |

| Parameter | Value |
|---|---|
| Steps | 3 |
| Mercy rule | after **5** judged jumps in total, a Good also clears step 3 |
| Expected duration | ~30 s |
| Advances on | jumps **the server judged** — never a client-reported step |
| Persistence | **none yet.** No DataStore in the project, so the tutorial re-runs every join. Needs fixing before ship. |

**Starter Legs are now the only ×10 in the game nobody earns, and the only one nobody can buy either.** Every other multiple of your jump comes out of the §2.4 track, one judged jump at a time. That makes the tutorial handover the single sharpest power moment in the whole arc — worth protecting, and worth keeping short.

**The curve is smooth by construction now.** `LevelMult` is linear in Leg Level and Leg Level is granted per jump, so S is a continuous ramp with no steps in it at all — the sawtooth that five tiers produced, and that ten tiers merely smoothed, is now structurally impossible. What ten tiers buy instead is ten *pacing* milestones: an island's tier is the thing that keeps its band at the same ~150 jumps as every other band (§4.1).

---

## 3. The jump bar

Sweep speed is **identical at every altitude** — per the settled decision, bad timing is punished by platform spacing, never by a faster bar.

**Tap vs. hold — two separate jumps sharing a key.** They have no code in common, which is deliberate: an ordinary jump should be impossible to break by changing the scored one.

| Input | Result |
|---|---|
| Release **before 0.20 s** | **A plain Humanoid jump.** `Humanoid.Jump = true` on the client, and that's all. The server is never told, the bar never appears, nothing is scored. |
| Release **after 0.20 s** | Judged by the bar: Perfect / Good / Okay / Weak |
| Never release | Miss, after the timeout below |

**The tap is vanilla except for its height.** `JumpHeight` is set server-side to the peak an Okay-quality jump would reach, so a tap sits exactly where the safe jump always sat on the curve. Everything else is stock Roblox: you keep your run momentum, so its distance is `WalkSpeed × airtime` rather than anything tuned.

| Legs | JumpHeight | Rise | Fall | Airtime | Run-jump reach |
|---|---|---|---|---|---|
| Tutorial (no legs) | 4.8 | 0.16 s | 0.26 s | 0.42 s | 7 |
| Starter Legs | **48.5** | 0.50 s | 0.81 s | **1.31 s** | 21 |
| Island 5 | 102.3 | 0.72 s | 1.18 s | 1.90 s | 30 |
| Endgame | 365.9 | 1.37 s | 2.23 s | 3.60 s | 58 |

`JumpHeight` is a height, so the engine derives the velocity itself and the tap is correct at any gravity without us touching it. The light fall applies to it exactly as it does to a bar jump — the force doesn't know or care how the jump started. **Measured in-game:** 48.50 set, 41.8 studs of rise from the frame the feet leave the floor, 0.46 s up and 0.79 s down against 0.50/0.81 predicted.

**Fire it with `Humanoid:ChangeState(Jumping)`, not `Humanoid.Jump = true`.** Measured: setting the flag on key-up does nothing whatsoever — it still reads `true` a frame later and the character never leaves the ground, because the humanoid only converts the flag into a jump off the input transition it has just seen end, and ours ended the instant we set it. Entering the state directly is unambiguous and still uses `JumpHeight`. This cost an afternoon; don't switch it back.

**Why it's built this way.** The tap used to be a scripted velocity write, judged server-side like the bar. Three separate faults came out of that — a round trip of input lag, a Humanoid in its Running state driving over the velocity before the character left the floor, and a client/server split in deriving the launch that silently produced jumps a tenth of the right height. The engine's own jump has none of those failure modes, because it isn't ours.

**The bar wind-up is now zero outside a pit** — the 0.20 s hold threshold is already the gate, and a second one would only open a window where letting go does nothing. Inside a pit the 2.5 s crouch remains, and it's the one place letting go early is possible: the attempt is simply dropped, no jump and no award, and you're free to tap or hold again.

> **Resolved: what an unjudged tap earns.** This used to be an open coin question. It isn't one any more — the only thing a jump can pay is Leg Level, and §4.2.1 already fixes the answer: a tap has no recorded quality, so it is awarded at the **Okay** weight, in a pit and on the climb alike. The server watches for the Humanoid's Jumping state and awards the Okay rate; nothing about the tap's own physics has to come back under our control to do it.

The tap's expected value is 0.82. The bar's, for a player releasing at random, is **0.793** — so guessing is worse than tapping, and the bar only pays from the moment you can actually read it. That gap is the entire skill curve, and it's free: it falls straight out of the zone widths.

| Parameter | Value |
|---|---|
| Sweep period (one-way pass) | **1.40 s** |
| Sweep pattern | left → right → left, ping-pong; centre = Perfect |
| Attempt times out after | 2 full passes (2.80 s) → Miss (§2.3) |

Zones, as half-width fraction from centre on a bar normalised to [−1, +1]:

| Zone | Half-width | Share of bar | Time window |
|---|---|---|---|
| Perfect | ≤ 0.10 | 10% | **140 ms** |
| Good | ≤ 0.30 | 20% | 280 ms |
| Okay | ≤ 0.62 | 32% | 448 ms |
| Weak | > 0.62 | 38% | 532 ms |

140 ms is a comfortable rhythm-game window that survives normal Roblox latency.

### 3.1 Server-authority parameters

| Parameter | Value |
|---|---|
| Attempt timebase | `workspace:GetServerTimeNow()`, recorded at attempt open |
| Sweep seed | per-attempt `Random.new()` seed → sent to client, drives phase offset |
| Latency compensation | subtract measured RTT/2, **clamped to 250 ms** |
| Reject attempt if | lock-in timestamp outside `[open, open + 2.80 s]` |
| Reject attempt if | player RTT > 400 ms (fall back to auto-Good, don't punish lag) |
| Client never sends | accuracy, zone name, resulting S, or **Leg Level gained** — only "I pressed, now" |
| Cooldown between attempts | **none.** The ground check is the only gate, and it's enough: finishing an attempt launches you off the floor. |
| Client-side prediction | **none, and none needed.** A tap doesn't need predicting because it isn't scripted — it's the Humanoid's own jump, already local and already instant. A hold can't be predicted, because its quality isn't knowable client-side. This is what the separation buys: the responsive jump has no server in its path, and the authoritative jump has no client in its path. |

**This matters more than it did under two currencies.** Leg Level gain is derived from the judged quality of a jump, and Leg Level is now the *only* source of jump power — so a client that could report its own gain could mint jump strength directly, not just vanity. Server computes the award, and the pit-zone check (§4) is part of the same transaction.

### 3.2 Combo

Consecutive Perfects. **Gain rate only — never S.**

```
GainMult = 1 + 0.05 × comboCount     -- capped at ×2.0 (20 consecutive perfects)
```

Breaks on any non-Perfect, on a fall, and on cashing out.

Combo sits in the same category as a Leg Tier: it multiplies what a jump *grants*, never what a jump *does*. That's what keeps it from being a power spiral — twenty perfects in a row make you progress twice as fast, they do not make you jump one stud further. The cap came down from ×3.0 to **×2.0** and the coefficient from 0.10 to 0.05 for exactly that reason: a ×3 on the only progression axis in the game is a bigger lever than a ×3 on a cosmetic wallet was, and it would out-earn the entire ten-tier track's ×8 with two islands' worth of skill.

---

## 4. Training: Leg Level and gravity

### 4.1 Leg Level bands

**The base pit rate is a flat 10 per Perfect at every island, from the sand pit to Coreground.** The escalation that used to live in this column now lives in `TierGain` (§2.1), which is the entire economy change in one sentence: *the sand doesn't decide what a rep is worth, your legs do.*

```
Leg Level gain (pit) = 10 × TierGain × qualityWeight × GainMult(combo)
```

Design target is **~150 pit jumps ≈ 10 min per island on schedule**, of which the climbing trickle (§4.5) covers about a fifth in practice, so the pit itself sees ~120 jumps and ~8 min. Held constant across islands by the band widening in exact step with the tier that island unlocks — so a player who buys on schedule feels a completely flat pace, and everyone else feels the gap.

> **Why the whole track tripled (17,000 → 51,000).** Under the old split, gear supplied half the power and training only had to fill 33 minutes of the arc. Gear supplies none of it now, so training *is* the arc, and it needs enough runway for a rate multiplier to be worth buying. At the old 500-jump track, the entire ten-tier ladder would have saved a player about 83 minutes for 167 minutes of medal farming — a purchase that costs more than it saves. At 1,500 jumps it saves ~250 minutes for ~80 (§5.4 dropped the price too), which is the right side of the line by a clear margin.

**Pit gain is per *Perfect* jump** — lesser results earn a fraction of it, see §4.2.1.

| Island | Base gain | TierGain *on schedule* | **Gain / Perfect** | Band width | Leg Level to leave | Jumps |
|---|---|---|---|---|---|---|
| 1 (Sand Pit) | 10 | ×1.0 | 10 | 1,500 | 1,500 | 150 |
| 2 | 10 | ×1.4 | 14 | 2,100 | 3,600 | 150 |
| 3 | 10 | ×1.6 | 16 | 2,400 | 6,000 | 150 |
| 4 | 10 | ×2.0 | 20 | 3,000 | 9,000 | 150 |
| 5 | 10 | ×2.5 | 25 | 3,750 | 12,750 | 150 |
| 6 | 10 | ×3.0 | 30 | 4,500 | 17,250 | 150 |
| 7 | 10 | ×4.0 | 40 | 6,000 | 23,250 | 150 |
| 8 | 10 | ×4.5 | 45 | 6,750 | 30,000 | 150 |
| 9 | 10 | ×6.0 | 60 | 9,000 | 39,000 | 150 |
| 10 | 10 | ×8.0 | 80 | 12,000 | **51,000** (arc 1 cap) | 150 |

**1,500 training jumps total** across the arc, 150 per island exactly, *if you buy every tier on the island that unlocks it*. Because every medal run passes every pit below your cash-out point, training and medal-farming happen on the same trip — you top up each band in passing rather than making dedicated training journeys.

**The same table, for a player who buys nothing** (Bare Legs, ×1.0 forever) — the route the design exists to keep open:

| Island | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|
| On schedule | 150 | 150 | 150 | 150 | 150 | 150 | 150 | 150 | 150 | 150 | **1,500** |
| Barefoot | 150 | 210 | 240 | 300 | 375 | 450 | 600 | 675 | 900 | 1,200 | **5,100** |

3.4× the jumps overall, 8× on the last island — **slow, tedious, and entirely possible.** That ratio is the number to move if the barefoot route reads as either a trap or a free ride; it's set by the `TierGain` spread in §2.1 and nothing else.

### 4.2 Training-jump cadence **[answers OQ: exact jump count / pacing]**

Under heavy gravity airtime collapses to ~0.35 s, so raw jump spam would finish a band in one minute. The pacing knob is the **crouch wind-up** already in the design doc as a cosmetic — in a training area it becomes a real duration:

| Phase | Outside a pit | Inside a pit |
|---|---|---|
| Crouch wind-up | 0.20 s | **2.50 s** |
| Jump bar | 1.40 s avg | 1.40 s avg |
| Airtime | 0.54–1.49 s | ~0.35 s |
| **Cycle** | ~4.5 s | **~4.1 s** |

150 jumps × 4.1 s = **10.3 minutes per island**, or **~8.2 min** once the climbing trickle (§4.5) has covered its usual fifth of the band. Crouch duration is the single number to tune if pacing feels wrong; it's the same at every altitude, which is what keeps the pit feeling identical everywhere.

> **The 3–5 min target has been retired, not missed.** It made sense when gear supplied half the power curve and the pit was a side dish. With jumping as the only progression axis, ~8 min an island is the pit carrying its new weight — and it is not 8 minutes *in one sitting*, because a band is topped up in passing across several medal runs. If it plays as a slog, the band widths in §4.1 are the knob, not the crouch.

### 4.2.1 Pit gain scales with jump quality **[settled]**

Instant taps (§3) skip the crouch, so the two routes through a pit cost very different amounts of time — in the sand pit, **~1.07 s for a tap against ~4.27 s for a played bar, a 4× gap**. Flat gain per jump therefore made tap-spam strictly optimal and left the 2.5 s crouch as a tax on the only skilful option. So **the `Gain / Perfect` column above is what a *Perfect* earns**, and everything else is a fraction of it:

| Result | Weight | Gain in the sand pit | Band time |
|---|---|---|---|
| Perfect | **1.00** | 10.0 | **10.8 min** |
| Good | **0.60** | 6.0 | 17.8 min |
| Okay | **0.20** | 2.0 | 53.4 min |
| Weak | **0.03** | 0.3 | 356 min |
| Miss | 0 | — | never |
| *Tap (spammed)* | *0.20* | *2.0* | ***11.0 min*** |

Band times are ×3 what they were, because §4.1's bands are; the *weights* and their relative ordering are untouched.

A tap has no recorded quality — it never reaches JumpService — so the pit award treats it as an Okay, which is what it is. A native tap in the sand pit peaks at 30 studs rather than the bar jump's 45, and carries well inside the 21-stud radius.

> ⚠️ **These band times predate 3× gravity and are now wrong in the lazy player's favour — and the stake is higher than it was.** Heavier gravity shortens every airtime, but the bar's cycle is dominated by the fixed 2.5 s crouch while a tap's cycle is *nothing but* airtime — so spam gained far more than skill did. At 3× gravity a pit tap cycles in ~0.51 s against a played bar's ~3.8 s, which puts a band at **~6.4 min spammed against ~9.5 min played well**: spamming is the faster route, which is precisely what this section exists to prevent. Dropping the Okay weight from 0.20 to **≈0.13** levels them again. Not changed yet, because gravity isn't settled — but it now needs a pass more urgently than it did, because this is no longer a race for a cosmetic wallet: whichever route is faster here is the fastest route to jump power itself.

These are far steeper than the §4.5 climbing weights on purpose — at those values tapping would win outright and the scaling would achieve nothing. At these:

- **Playing the bar well is the fastest route** through the sand pit (10.8 min a band) — but only just: spam comes out at 11.0 min, because a native tap in a pit cycles in 0.88 s. The two are effectively level at island 1.
- **Good is worse than tapping** (17.8 min). Deliberate, and it mirrors the climb: the bar pays only if you can actually hit the middle. Half-playing it is the worst of both.

**Okay is the number to move if this drifts**, because the cycles it's balanced against move whenever pit gravity or Lift does — it was 0.10 against a 0.35 s pit airtime, and Starter Legs tripled that airtime to 1.07 s. Note it is also what an unjudged tap earns *on the climb* (§4.5), so moving it touches both loops.

> ⚠️ **The skill advantage doesn't hold at altitude, and the 2026-08-11 gravity change made it worse.** Pit peak is a constant 26 studs, so heavier pits jump *quicker* as well as shorter: airtime now runs **0.62 s** at island 1, **0.47 s** at island 5, **0.26 s** at island 10 (was 1.07 / 0.81 / 0.46 at the old 45-stud peak). The crouch stays 2.5 s throughout, so the bar's overhead is a fixed cost against a shrinking one and spam catches up — and it now catches up sooner, because every airtime above shrank by a further 42% while the crouch did not move.
>
> The old arc comparison — **~99 min played well against ~96 min spammed** at the new band widths, and ~99 vs 129 with Okay dropped to 0.15 — was computed against the 45-stud peak and **no longer holds**. Redo it as part of the §4.2.1 weight re-tune; Okay is still the number to move, and the case for dropping it below 0.20 is now stronger than it was.

The award still fires on landing inside the zone, weighted by the quality of the jump that started it — so jumping *out* of a pit never pays, and falling in over the rim pays at the Okay weight rather than full price.

### 4.3 Pit gravity

Tuned so a training jump peaks at a constant **26 studs** (vs. 72 outside the pit) at the strength you're expected to have on arrival. What the number encodes is the *ratio* — a pit jump is **2.8× shorter** than the one outside it. The air-time floor is likewise now 13 studs, not 1.3.

> **Changed 2026-08-11, from 45 studs and a 1.6× ratio.** A training zone is a trigger volume, and the ones in the build are only ~36 studs tall — every pit short by the same 16 studs. A 45-stud jump left through the ceiling, at which point the server dropped the pit state, and the landing that followed paid nothing: a well-timed jump earned *less than a tap*, the exact inversion §4.2.1's weights exist to prevent. Raising the gravity curve ×1.73 brings the arc back inside the box. The alternative was raising the four ceilings and leaving every number below as written; the gravity route was chosen deliberately, and §6.1 moved with it.

**Horizontal launch is damped to 0.35 inside a pit.** Training is jumping on the spot: without this, a full-power pit jump would carry 24 studs against a 42-stud pit and land outside the zone, earning nothing. It also means a pit never has to be as wide as your reach — which matters, because reach is exactly what Starter Legs tripled. Fictionally it's the sand swallowing the push-off, which is where the pit came from in the first place.

```
g_pit = 196.2 × (7.2 · S_expected²) / 2.6    →   gravityMult = 2.77 × S_expected²
```

| Island | Pit | S on arrival | Gravity × | Effective g |
|---|---|---|---|---|
| 1 | Sand Pit | 1.00 | **2.77** | 543 |
| 2 | Packed Earth | 1.05 | **3.06** | 601 |
| 3 | Gravel Bed | 1.12 | **3.50** | 686 |
| 4 | Clay Flat | 1.21 | **4.03** | 790 |
| 5 | Iron Sand | 1.31 | **4.75** | 931 |
| 6 | Basalt Pan | 1.44 | **5.72** | 1123 |
| 7 | Slag Bed | 1.59 | **7.02** | 1377 |
| 8 | Leadfield | 1.80 | **8.95** | 1757 |
| 9 | Deep Well | 2.03 | **11.41** | 2238 |
| 10 | Coreground | 2.34 | **15.14** | 2971 |

Small increases low down, aggressive escalation high up ✓ (matches the settled decision) — and a smooth ramp, because `LevelMult` is linear and this table is a function of it.

**Gravity still tracks altitude; it just no longer decides what a rep is worth.** Under the old design these two jobs were fused — a heavier pit both *felt* heavier and *paid* more. They're separated now: gravity is purely the feel-and-fairness curve above (a pit jump peaks at 26 studs wherever you are), and the payout rides on `TierGain` instead. That's what makes the barefoot route possible at all — a barefoot player at Coreground would otherwise be earning island-10 rates by standing in island-10 sand.

**[answers OQ: does gravity run ahead of expected strength?]** — Ship it running **+8% ahead** of the table for a player's **first 15 jumps** in a newly unlocked pit, then relax to the table value. New areas feel heavy for about a minute, then you settle in. One boolean and one float to revert.

**[answers OQ: under-levelled player in a high pit]** — a hard air-time floor rather than a strength gate:

```lua
-- never let a pit jump peak below 1.3 studs, whoever you are
local gEff = math.min(gNominal, (53.2 * S_actual)^2 / (2 * 1.3))
```

At 1.3 studs the pit is miserable and clearly signals "go train lower," but is never literally unplayable. The floor moved with the target above and still sits at 49% of it — left at 2.2 it would have been so close to the new 2.6-stud target that an under-levelled player would barely feel the difference, and the signal would be gone.

### 4.4 Implementation (per-character gravity)

`Workspace.Gravity` is global, so fake it per character, server-side:

```lua
-- Attachment on HumanoidRootPart, VectorForce with RelativeTo = World
force.Force = Vector3.new(0, -hrp.AssemblyMass * (gravityMult - 1) * 196.2, 0)
force.ApplyAtCenterOfMass = true
```

Clear on zone exit, on death, **on cash-out teleport**, and on `CharacterRemoving`. `JumpPower` is left alone — the launch feels the same, the fall is what's heavy.

### 4.5 Climbing trickle — "every jump makes you stronger" **[answers OQ: trickle balance]**

This is the clause that makes the headline true. A pit is the efficient way to earn Leg Level; the climb is the *unavoidable* way, and it pays whether you meant to train or not.

```
LegLevel gain (climb) = 0.25 × 10 × TierGain × qualityWeight × GainMult(combo)
                      = 2.5 × TierGain × qualityWeight
qualityWeight: Perfect 1.0 | Good 0.7 | Okay 0.4 | Weak 0.15 | Miss 0
```

**Same structure as the pit award, at a quarter the base rate**, and it no longer reads off "the pit gain of the island below you" — there is no such number any more, the base is a flat 10 everywhere. `TierGain` multiplies this exactly as it multiplies a pit jump, so buying legs speeds up *every* jump you make, which is the point.

**The coefficient went 0.15 → 0.25.** The old 0.15 was set to stop the trickle making pits optional back when it was one of two ways to buy power; now that jumping is the *only* way, the climb has to visibly pay or the design's central promise is a lie. 0.25 keeps a climbing jump worth a quarter of a pit jump — clearly worse per rep, and clearly not nothing.

| Full climb to | Island 6 | Summit |
|---|---|---|
| Leg Level from the climb (on schedule, avg quality) | ~340 | ~2,190 |

A summit run grants **4.3%** of the whole 51,000-point track. Across the ~14 ladder runs plus discovery pushes, climbing supplies **roughly 15–20% of total Leg Level** — meaningful, never sufficient. This is still the first number to move if training feels like a chore.

> **A barefoot player earns this at ×1.0 and has almost nothing to spend medals on**, so their loop is mostly pit-and-climb with the occasional cosmetic. That's a real playstyle now rather than a rounding error, and it's the one to watch in testing: if it turns out to be *comfortable* rather than merely possible, `TierGain`'s spread (§2.1) is too flat.

---

## 5. Medals — the cash-out loop

### 5.1 Rules

| Rule | Value |
|---|---|
| Caches per island | 1 |
| Caches collectable per run | **1** (it teleports you) |
| Cache respawns | every run, for every player |
| Interaction | **voluntary** — walk into it. Landing on an island never triggers it. |
| Confirm step | **required.** Proximity opens a prompt; collection fires on an explicit input, never on touch alone. |
| Confirm hold | 0.75 s hold (not a tap) — cheap to do on purpose, impossible to fumble |
| Cancel | walking out of range, or any movement input during the hold |
| On collect | bank medals → 2 s fanfare → teleport to Island 1 |
| On collect | **checkpoint chain resets to Island 1** |
| First-arrival bonus | one-time per island, automatic, **does not teleport** |

The voluntary-interaction and no-teleport-on-arrival rules are load-bearing: without them, pushing to a new personal best would punish you the moment you landed, and the risk/reward decision would evaporate. The checkpoint reset is equally load-bearing — see §5.5.

### 5.2 Cache values and run economics

Effective hop time is **5.6 s** (4.5 s cycle + ~25% miss overhead).

| Cash out at | Hops from bottom | Run time | Cache | **Medals/min** |
|---|---|---|---|---|
| Island 2 | 10 | 0.9 min | **20** | 21.5 |
| Island 3 | 21 | 2.0 min | **50** | 25.5 |
| Island 4 | 33 | 3.1 min | **80** | 26.0 |
| Island 5 | 46 | 4.3 min | **130** | 30.3 |
| Island 6 | 60 | 5.6 min | **200** | 35.7 |
| Island 7 | 75 | 7.0 min | **280** | 40.0 |
| Island 8 | 91 | 8.5 min | **400** | 47.1 |
| Island 9 | 108 | 10.1 min | **550** | 54.6 |
| Island 10 | 126 | 11.8 min | **750** | 63.8 |

Rate rises monotonically, ~3× from bottom to top. Climbing higher is always strictly better per minute — but never *so* much better that being capped at island 6 for a while feels pointless.

### 5.3 First-arrival bonuses

One-time, ≈ 3× that island's cache, so a new personal best is worth about three farm runs — a real burst that funds that island's tier and rewards pushing over grinding.

| Island | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| Bonus | 50 | 150 | 240 | 400 | 600 | 850 | 1,200 | 1,650 | 2,250 |

### 5.4 Tier costs

One clean rule: **each tier costs exactly 1.5× its island's arrival bonus**, which lands at ~1.5 cache runs every time.

> **Halved from 2× (≈3 runs), because what a tier buys got weaker and what it competes with got stronger.** A tier no longer adds a single stud of reach — it only accelerates Leg Level — and at the old price the entire ladder cost more climbing time than it saved training time, which makes it a purchase no informed player should make. It also now shares a wallet with the whole cosmetic catalogue (§6), so leaving headroom is the difference between a choice and a formality. At 1.5× the ladder costs **~80 min of farming to save ~250 min of pit time**, which is the right side of the line by a wide enough margin to survive tuning.

| Tier | Island | Cost | Grind runs | ≈ Time |
|---|---|---|---|---|
| 2 Cardboard Braces | 2 | **75** | 1.5 | 1.4 min |
| 3 Duct-Tape Wraps | 3 | **225** | 1.5 | 3.0 min |
| 4 Wooden Stilts | 4 | **360** | 1.5 | 4.7 min |
| 5 Coil Springs | 5 | **600** | 1.5 | 6.5 min |
| 6 Shock Absorbers | 6 | **900** | 1.5 | 8.4 min |
| 7 Hydraulic Pistons | 7 | **1,275** | 1.5 | 10.5 min |
| 8 Turbine Calves | 8 | **1,800** | 1.5 | 12.8 min |
| 9 Rocket Boosters | 9 | **2,475** | 1.5 | 15.2 min |
| 10 Antigrav Struts | 10 | **3,375** | 1.5 | 17.7 min |

**~13.5 grind runs, ~1 h 20 m of medal farming** for the full ladder — down from 28 runs and 2 h 47 m, with the difference deliberately handed back to the pit, which is now where progress actually lives.

> ⚠️ **The marginal tier is a weak purchase; the ladder is a strong one.** Buying tier 5 the moment island 5 unlocks saves ~2.5 min over waiting until island 6 unlocks tier 6 — so a player who delays every purchase by one island pays only ~24 extra minutes across the whole arc. Skipping them *permanently* costs ~250. The economy therefore has a soft, cumulative pull rather than a sharp per-purchase one, and the thing to watch in testing is whether players notice the pull at all. If they don't, steepen `TierGain` at the top end (tiers 8–10) rather than raising prices — that widens the gap without closing the barefoot route at the bottom.

Medals top out in the low thousands. This is still a deliberate rejection of the Ninja Legends pattern (rank 45 costs 5.6 septendecillion): a currency you can't grind in place doesn't need inflation to stay interesting, and keeping "is this tier worth another climb, or is that aura?" a legible question matters more than a big number. **With one wallet, the exact-number rule in §9 earns its keep — every price on the shelf is now directly comparable to every other one.**

### 5.5 Why the checkpoint reset is non-negotiable

Per-cloud checkpoints (settled) plus a cash-out teleport is an open exploit: bank the cache, then deliberately fall, and the respawn returns you to your top checkpoint with the medals already paid. Free infinite medals, no climb.

So cashing out must clear the chain back to Island 1. Server-side, in one transaction: award → wipe checkpoint list → teleport. Never trust a client-reported checkpoint after a cash-out.

### 5.6 **[answers OQ: how do you retrain without fast travel?]**

The cash-out *is* the ride down, and it's opt-in — which resolves the open question without adding a recall. You bank at island 6, land at island 1, and the re-climb passes every pit on the way up. Training is no longer a dedicated trip; it's something you top up in passing on a journey you were making anyway.

The trade is that **the backtracking chokepoint question gets sharper, not softer**: every player is funnelled through Island 1 constantly, so its spawn, sand pit entrance, and first ledges are now the highest-traffic geometry in the game. Size them for it (§7.4) and treat them as the first thing to stress-test with 16 players.

---

## 6. The one shelf — what Medals buy

Coins are gone. There is a single currency and a single shopfront (island 1), and everything on it is priced against everything else, which is the whole reason the numbers stay four digits.

### 6.1 What a jump pays

Nothing, in currency. **A jump pays Leg Level and only Leg Level** — §4.1 for a pit jump, §4.5 for a climbing one, §4.2.1 and §2.3 for the quality weights, §3.2 for combo. Medals come from one place and one place only: a cache you walked into on purpose (§5).

That's a stronger statement than the old design could make. Under two currencies, "every jump pays" meant every jump minted a vanity token; now it means every jump makes you permanently better at the game, which is what the phrase was always trying to say.

| Jump | What it grants (on schedule, island 5) |
|---|---|
| Pit, Perfect | **25** Leg Level |
| Pit, Good | 15 |
| Pit, tap or Okay | 5 |
| Climb, Perfect | **6.3** |
| Climb, Good | 4.4 |
| Climb, tap or Okay | 2.5 |
| Any jump, any quality | **0 Medals** |

### 6.2 Why gain is flat per jump and not scaled by height

Coins were `0.3 × peak²`, i.e. `∝ S⁴` — the higher you jumped, the more you earned. That was safe precisely *because* coins bought nothing that made you jump higher.

**Reusing that formula for Leg Level would close the loop into a runaway**: more power → more gain per jump → more power, compounding at the fourth power of S. A player two islands ahead would train four to six times faster than one behind, and the barefoot route would collapse from slow to hopeless. So the gain rate deliberately **does not** see your jump height, your altitude, or your island — only the quality of your timing, your gear tier, and your combo, all of which are bounded and all of which you chose.

The one place height-scaling survives is where it can't compound: the *medal* cache values (§5.2) still rise with altitude, so climbing higher still pays better — in the currency that can't buy power.

### 6.3 Medal sinks — eggs and auras

Priced on one rule: **an island's egg costs the same as its Leg Tier** (§5.4). That makes the game's central tension a single legible sentence at the shopfront — *this run bought the Coil Springs or the Storm Egg, not both* — and it is the honest consequence of putting power-rate and flair in the same wallet.

As with tiers, the island column is the **unlock gate**: every egg and aura is bought on island 1.

| Island | Egg | Cost | Aura | Cost |
|---|---|---|---|---|
| 1 | Sand Egg | **40** | Sand Aura | **20** |
| 2 | Meadow Egg | **75** | Meadow Aura | **40** |
| 3 | Pebble Egg | **225** | Pebble Aura | **75** |
| 4 | Breeze Egg | **360** | Breeze Aura | **225** |
| 5 | Storm Egg | **600** | Storm Aura | **360** |
| 6 | Aurora Egg | **900** | Aurora Aura | **600** |
| 7 | Thunder Egg | **1,275** | Thunder Aura | **900** |
| 8 | Nebula Egg | **1,800** | Nebula Aura | **1,275** |
| 9 | Void Egg | **2,475** | Void Aura | **1,800** |
| 10 | Summit Egg | **3,375** | Summit Aura | **2,475** |

Auras keep the old rule — **one band below eggs** (island N aura = island N−1 egg price). **Aura reroll: 150 flat, unlimited** — roughly five rerolls per island-10 run, and the repeatable sink that absorbs late-game overflow once the ladder is bought out.

The curve is ≈ ×1.45 per band, the same shape as the old coin prices; only the units changed. Still deliberately flatter than the Bubble Gum Simulator shape (×4–10 per tier) — BGS can afford runaway pricing because its pets multiply income, and ours multiply nothing.

**Total catalogue: ~11,100 in tiers, ~11,100 in eggs, ~7,800 in auras ≈ 30,000 Medals to own everything**, against ~7,400 in one-time arrival bonuses. A power-only player finishes the ladder in ~13.5 runs; a completionist is looking at roughly **30 further island-10 runs (~6 h)** on top of the arc. That's the endgame, and it's a genuine one — the old design had no medal sink at all past tier 10.

> ⚠️ **Risk worth flagging, now sharper than before.** Every comparable game's pets carry a stat multiplier, which is what makes players buy the 40th one. Cosmetic-only pets are a weaker sink to begin with, and under one wallet they're also *competing with power*, so an unappealing catalogue doesn't just fail to absorb medals — it makes buying anything but tiers feel like a mistake. The design doc's warning ("don't let this catalog ship shallow") is doing more load-bearing work than ever. If the catalogue can't be deep at launch, the uncapped 150-medal reroll is the pressure valve.

> ⚠️ **The other half of that risk: a completed ladder makes medals worthless to a power-focused player.** Once tier 10 is bought, medals buy nothing they want, and the cash-out loop — the game's whole risk/reward decision — loses its stake. Under two currencies the coin loop covered this. Watch for it; the likeliest fix is a repeatable medal sink that isn't cosmetic (a consumable gain booster, an arc-2 tier ladder), not more auras.

---

## 7. World geometry

### 7.1 Why reach is 0.835× the flat number

Platforms rise as you climb, so you land above your takeoff and lose airtime. With rise/hop set to **0.55 × your peak height**, the maths falls out cleanly:

```
t_landing = 1.671 · Vy / g   vs.   2.000 · Vy / g flat     →    83.5% of flat reach
```

**Rising 55% of your peak height costs you exactly 16.5% of your horizontal reach.** All gaps below are computed from the rising figure, not the flat one.

### 7.2 Per-level values

Level N = the cloud field between island N and island N+1. Main-line gap is **80% of a Perfect jump** at expected strength, so Perfect clears with margin and Good just makes it.

Rebuilt for Starter Legs: horizontal quantities (reach, gap, cloud Ø) are **×3.16**, vertical ones (peak, rise/hop) are **×10**. The sky is now much taller than it is wide, which is the shape a moon jump asks for.

> ⚠️ **Every horizontal number in this table is now 1.32× low, and the table has not been rebuilt.** The light fall (§1) stretches airtime by `Config.airtimeStretch()` = 1.32, and horizontal speed doesn't change mid-flight, so reach, main gap and cloud Ø all scale with it — level 1's gap is really 33 studs, level 10's is 251. The **Ø as % of gap** column is unaffected, so the difficulty curve is still the one that was tuned. Left un-rewritten deliberately: the two gravities are still being felt out and this factor moves with them, and nothing past island 1 is built, so there is no cost to waiting until they settle. Multiply before building anything.

| Level | Rising reach | **Main gap** | Peak height | Hops | Rise/hop | Cloud Ø | Ø as % of gap |
|---|---|---|---|---|---|---|---|
| 1 | 33 | **25** | 75 | 10 | 41 | 38 | 150% |
| 2 | 39 | **32** | 88 | 11 | 48 | 38 | 120% |
| 3 | 46 | **35** | 104 | 12 | 57 | 38 | 109% |
| 4 | 55 | **44** | 125 | 13 | 69 | 38 | 86% |
| 5 | 67 | **54** | 152 | 14 | 84 | 38 | 71% |
| 6 | 82 | **67** | 187 | 15 | 103 | 38 | 57% |
| 7 | 104 | **82** | 237 | 16 | 130 | 38 | 46% |
| 8 | 132 | **105** | 302 | 17 | 166 | 41 | 38% |
| 9 | 174 | **139** | 398 | 18 | 219 | 54 | 38% |
| 10 | 238 | **190** | 544 | 20 | 299 | 73 | 38% |

All in studs. **146 hops total** — deliberately short, because the cash-out loop makes you climb this ~14 times for the tier ladder alone, and dozens more if you want the catalogue (§6.3). The Ø-as-%-of-gap column is untouched, so the difficulty curve is exactly the one that was tuned; only the units moved.

> ⚠️ **Two things this rescale breaks that aren't fixed yet.**
> 1. **Island diameters (§7.4) did not scale.** A level-10 cloud is now 73 studs across against island 10's 100 — a landing pad nearly the size of the island it leads to. Either islands go ×3.16 (island 1 becomes 885, and it's already built at 280) or the late cloud sizes come down. Needs a call.
> 2. **Hop time is now airtime-dominated** and varies with altitude: 1.7 s of flight at level 1 against 4.7 s at level 10, where before it was 0.54–1.49 s. §5.2's flat 5.6 s hop and everything derived from it (medals/min, the 167-minute grind figure in §8) are estimates until that's re-derived from a real run.

Two things to read off this table:
- **Level 1's gap is 8 studs, below the 11.9 a vanilla character clears.** A brand-new player makes the first hop with untrained legs and zero upgrades, no tutorial required.
- **The last column is the difficulty curve.** The landing target starts *wider than the gap itself* (level 1 is nearly impossible to miss) and settles at 38%. Difficulty comes from target size and spacing, never from the bar.

### 7.3 Forgiveness — the catch ratio

"Good/Okay still reaches nearby platforms" is implemented as filler-cloud density. **Catch ratio** = distance to the nearest off-route cloud ÷ your Perfect reach:

| Level | Catch ratio | Lowest quality that lands *somewhere* |
|---|---|---|
| 1–2 | 0.35 | Weak (0.42) |
| 3–4 | 0.50 | Weak (0.42 — marginal) |
| 5–6 | 0.65 | Okay (0.67) |
| 7–8 | 0.78 | Good (0.85) |
| 9–10 | 0.88 | Good only |

Low altitude is a forgiving cloud soup; the summit is a tightrope. Same bar throughout ✓.

### 7.4 Altitudes and sizes

Altitudes are ×10, since every hop now rises ten times as far.

| Island | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | Summit |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Altitude (studs) | 0 | 400 | 950 | 1600 | 2500 | 3700 | 5250 | 7350 | 10150 | 14100 | **20100** |

| Thing | Size (studs) |
|---|---|
| **Island 1 diameter** | **280** — spawn, sand pit, **every shop in the game**, *and the arrival pad every cash-out lands on*. Oversized on purpose (§5.6): this is the busiest geometry in the game, and now more so, since islands 2–10 carry no vendor at all. |
| Islands 2–3 | 160 |
| Islands 4–6 | 140 |
| Islands 7–9 | 120 |
| Island 10 | 100 |
| Cash-out arrival pad (Island 1) | 40 diameter, **non-collidable between players**, ≥30 studs clear of the first jump ledge |
| Training area diameter | 45 (island 1) → 30 (island 10) |
| Pit rim height / depth | 3 / 2 |
| Medal cache footprint | 6 × 6, with a 3-stud no-walk buffer so nobody strays into the prompt |
| Leg-tier vendor footprint | 8 × 8, **island 1 only** — all ten tiers sold from one shopfront. Islands 2–10 have a cache and a pit, no vendor. |
| Egg / aura shop footprint | **island 1 only**, same as above. An island's number gates what its shelf will sell you, not where you buy it. |
| Cloud thickness | 4 (levels 1–6), 6 (levels 7–10) |
| Leg bulk scale | 1.00 → 1.35 across a tier band, resets to 1.00 on tier purchase |

Islands shrink with altitude to sell "sparser the higher you go" ✓ — Island 1 is the deliberate exception.

**Streaming:** `StreamingEnabled = true`, `StreamingTargetRadius = 1024`, `StreamingMinRadius = 256`.

### 7.5 Fall cost

No fall damage; respawn at the last cloud landed.

| Phase | Time |
|---|---|
| Freefall to below-checkpoint trigger | ~2.5 s |
| Fade + reposition | ~1.5 s |
| Walk to edge + re-attempt | ~4 s |
| **Total per miss** | **~8 s** |

Much gentler than Steep Steps (checkpoint every 100 m), because the run loop already supplies the long-form risk: the real punishment for weak legs is that your cash-out ceiling is low, not that a single miss hurts.

### 7.6 Cloud reposition

| Parameter | Value |
|---|---|
| Reposition interval | 30 min |
| Forced-reposition timer (occupied level) | 60 min, resets instantly when level empties |
| Warning before forced swap | 15 s |
| Fade overlap (both sets solid) | **6 s** |
| Reposition scope | X/Z only; islands never move |

Worth noting the loop makes repositioning bite harder than it did: a player who has memorised a route re-runs it ~14 times for the ladder and far more than that chasing cosmetics, and the reshuffle invalidates that muscle memory. That's the hook working as intended, but it means the 30-minute interval should be watched in testing — it's the difference between "the world changed" and "my grind route got deleted."

---

## 8. Time to finish arc 1

**On schedule** — buys every tier on the island that unlocks it:

| Activity | Amount | Total |
|---|---|---|
| Medal grind runs | ~13.5 runs (§5.4) | 80 min |
| Training | ~1,200 Perfect pit jumps, the other 300 covered by the trickle | 82 min |
| Discovery pushes to new personal bests | — | ~25 min |
| Falls, menus, walking | — | ~25 min |
| | | **≈ 3 h 32 m** |

**Barefoot** — buys nothing, ever, and still summits:

| Activity | Amount | Total |
|---|---|---|
| Medal grind runs | 0 required | 0 min |
| Training | ~4,600 pit jumps at ×1.0, the rest from the trickle | 314 min |
| Discovery, falls, menus, walking | — | ~50 min |
| | | **≈ 6 h 04 m** |

**The balance of the clock inverted, and that's the change working.** The old arc was 167 minutes of farming against 33 of training — you climbed to buy power. It's now 80 against 82: you climb to buy *speed*, and the jumping itself is what makes you strong. The arc also got ~35 minutes shorter overall, entirely out of the medal grind; that time is available back in §4.1's band widths if the pacing wants it.

Training is ~99 minutes played well and ~96 spammed at these widths — the two routes still come out level across the arc, for the reason in §4.2.1's warning, and fixing that is now more urgent than it was because the prize is power rather than flair.

Those are efficient runs. Realistically **4–8 hours** for an average player on the schedule path, since miss rate climbs steeply past level 7 and a failed high push costs a whole run's time. Comparable to Steep Steps, where only ~1% of players reach 1,000 m — expect a similar completion cliff, and expect islands 8–10 to be seen by a small minority.

**Past the ladder, the game is the catalogue**: ~30 further island-10 runs (~6 h) to own every egg and aura (§6.3), plus an unbounded reroll sink after that.

---

## 9. HUD display

| Element | Format |
|---|---|
| Altitude | **display studs with an "m" suffix** — "1,410 m". Genre convention; the true conversion (1 stud = 0.28 m) would read 395 m and undersell the climb. Cosmetic call, flag if you'd rather be honest. |
| Medals | never abbreviated — always exact ("1,200"). One currency, four digits, and every price on the shelf directly comparable to it. |
| Leg Level | `L 12,750 / 17,250` — current / next island's threshold. **This is the power bar now**, so it gets the prominent slot the coin counter used to have. |
| **Gain rate** | `+25 / jump` on the Leg Level bar, and it must visibly change the instant a tier is bought — that number *is* what the shop sells, so if the player can't see it move, they can't see what they paid for. |
| Leg Tier | icon + name + `7/10` + its `×3.0`, so the tier track reads as a collection *and* as a multiplier |
| **Cache preview** | when in range of a cache, show **"+200 Medals — returns you to the bottom"** and the value of the *next* island's cache alongside it. The decision only works if both sides of it are visible. |
| Combo | only visible above ×1.5, so it reads as a reward not a nag. Label it as a gain multiplier, not a score — it must never look like it is making the jump bigger. |

**One counter, not two**, so the top-left stack loses a row: Medals, Leg Level (+ rate), altitude. The right-side action rail is unchanged.

---

## 10. Extending past island 10

| Quantity | Per additional island |
|---|---|
| Leg Level band | × 1.33 |
| Pit gravity multiplier | × 1.28 |
| Medal cache value | × 1.38 |
| Medal first-arrival bonus | × 1.4 |
| **Leg Tier** | **one per island**, cost = 1.5× that island's arrival bonus |
| **TierGain** | **× 1.33 — must match the Leg Level band exactly**, or the ~150-jumps-a-band pacing drifts |
| Main gap | × 1.25 |
| Hops in level | + 2 |
| Egg price | **× 1.4** (= the tier cost of the same island, per §6.3's rule) |
| Aura price | one band below the egg, unchanged rule |

**Nothing scales itself any more.** The coin formula used to extend for free because it was a pure function of jump height; Leg Level gain is deliberately *not* a function of anything that grows (§6.2), so the `TierGain` row above is now mandatory rather than optional. Add an island without it and that island's band takes 1.33× the jumps of every band before it.

`LevelMult`'s denominator also has to move with the cap: `LevelMult = 1 + (S_max − 1) × (LegLevel / L_cap)`, with `S_max` and `L_cap` both re-read from the extended §2.4 table. It is a linear map with two ends — don't let anyone reintroduce a curve here, because §7's geometry is derived from it.

---

## Reference data (what these are calibrated against)

- **Roblox engine constants** — gravity 196.2 studs/s², default `JumpPower` 50 / `JumpHeight` 7.2, `WalkSpeed` 16, 1 stud = 0.28 m: [Jumping — Roblox Wiki](https://roblox.fandom.com/wiki/Jumping), [Stud (unit) — Roblox Wiki](https://roblox.fandom.com/wiki/Stud_(unit)), [WalkSpeed — Roblox Wiki](https://roblox.fandom.com/wiki/Class:Humanoid/WalkSpeed)
- **Vanilla jump reach (~10–12 studs horizontal, 11-stud block height)** — [Calculating maximum player jump distance](https://devforum.roblox.com/t/calculating-maximum-player-jump-distance/455105), [How to calculate studs jumped from JumpPower](https://devforum.roblox.com/t/how-to-calculate-how-many-studs-a-player-can-jump-based-on-jumppower/100480). Our S=1 model reproduces 11.9 studs, so the whole curve is anchored to measured vanilla behaviour.
- **Ninja Legends** — 62 ranks with costs escalating into quadrillions/septillions and multipliers to ×9.6 billion: [Ranks — Ninja Legends Wiki](https://roblox-ninja-legends.fandom.com/wiki/Ranks), [highest rank breakdown](https://www.sportskeeda.com/roblox-news/what-highest-rank-roblox-ninja-legends). Its *many small ranks* structure is the model for a tier per island; its number inflation is the anti-pattern we avoid for Medals.
- **Muscle Legends** — rebirth cost `10,000 + 5,000x`, first rebirth at 10,000 strength, stacking permanent multipliers: [How Rebirthing Works](https://muscle-legends.fandom.com/wiki/How_Rebirthing_Works). The reset-and-re-run structure is the closest shipped analogue to our cash-out loop, and the source of the "flat cost curve, multiplicative reward" shape behind our tier pricing.
- **Bubble Gum Simulator** — egg costs 10 → 110 → 450 → 5,000 → 15,000 across worlds (~×4–10 per tier): [Eggs — BGS Infinity Wiki](https://bgs-infinity.fandom.com/wiki/Eggs). Our egg curve is deliberately flatter (×1.45) because our pets carry no income multiplier.
- **Steep Steps** — 1,000–2,000 m mountains, bonfire checkpoints every 100 m, only ~1% of players reach 1,000 m: [Rolimon's](https://www.rolimons.com/game/11606818992), [Steep Steps — Roblox Wiki](https://roblox.fandom.com/wiki/Steep_steps/STEEP_STEPS). Anchor for climb length, checkpoint density, and realistic completion rates.
- **Exponential cost-curve practice** — pure `level^3` / `base × 2^level` curves are widely reported as unbalanced (front-loaded then trivial); adding a linear term or fitting the curve empirically is the standard fix: [Balancing exponential upgrade progression](https://devforum.roblox.com/t/balancing-exponential-upgrade-progression/2434950), [Simulator Formulas](https://devforum.roblox.com/t/simulator-formulas/853976). Why our bands scale ×1.25–1.5 rather than ×2+.
