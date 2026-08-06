# Leg Day — Numbers (first tuning pass)

Concrete values for jump strength, gravity, Leg Level, Medals, Coins, and world geometry. Companion to [Leg Day - Design Doc.md](Leg%20Day%20-%20Design%20Doc.md) — that doc owns the *design*, this one owns the *numbers*. Where a number answers an open question, it's marked **[answers OQ]**.

All numbers are a first pass calibrated against real Roblox physics and shipped-game economies (see [Reference data](#reference-data-what-these-are-calibrated-against) at the end). They're meant to be tuned, not treated as final.

---

## 0. The two loops these numbers serve

**Medals — the run loop.** Each island holds a **medal cache**. Touching it banks the medals *and teleports you to the bottom*, clearing your checkpoint chain. One cache per run, so every run is a single decision: **how high do I dare climb before cashing out?** Higher caches are worth more per minute, so the ceiling of your legs sets your earn rate. Each Leg Tier costs about three full round trips.

**Coins — the height loop.** Every jump pays, and the payout scales with **how high that jump peaked**. Not with altitude, not with the island you're on — with the arc itself. Stronger legs and better timing both raise the arc, so both raise income. Training jumps are deliberately tiny (§4.3), so the pit is the *worst* coin rate in the game.

The two loops interlock: the re-climbs the medal loop forces on you are exactly when the coin loop pays out. Repetition is never unpaid.

---

## 1. Physics baseline (real Roblox constants)

| Constant | Value | Note |
|---|---|---|
| Gravity | 196.2 studs/s² | `Workspace.Gravity` default |
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
S = TierMult × LevelMult × QualityMult
```

Applied on takeoff, server-side, as a single velocity write:

```lua
local Vy = 53.2 * S
local Vx = 22.0 * S
hrp.AssemblyLinearVelocity = hrp.CFrame.LookVector * Vx + Vector3.new(0, Vy, 0)
```

Derived arc (gravity 196.2):

| Quantity | Formula | S=1.00 | S=2.75 (max) |
|---|---|---|---|
| Airtime (flat) | `2·Vy / g` | 0.54 s | 1.49 s |
| **Peak height** | `Vy² / 2g` = `7.2·S²` | 7.2 studs | 54.3 studs |
| Flat reach | `Vx · airtime` = `11.9·S²` | 11.9 studs | 89.8 studs |
| **Rising reach** (see §7.1) | `0.835 × flat` = `9.94·S²` | 9.9 studs | 75.0 studs |

S=1 giving an 11.9-stud flat reach lands inside the measured 10–12 stud vanilla range — the model is anchored to reality, not invented.

Distance and height both scale with **S²**, so power feels superlinear: doubling S nearly quadruples your reach. Coins scale with height *squared* (§6), so they scale with **S⁴** — the same upgrade that doubles your reach multiplies your income sixteenfold. That's the whole "superhero legs" payoff, and it's why the power number itself can stay small.

**Full range: S = 1.00 → 2.75.** Deliberately small and readable. All number inflation lives in Coins.

### 2.1 Tier multipliers — ten tiers, one per island

Every island unlocks its own tier, so gear is a milestone you hit on arrival rather than a rare event. ×1.053 per tier, spanning **1.00 → 1.60**.

| # | Leg Tier | TierMult | Unlocks at |
|---|---|---|---|
| 1 | Bare Legs | 1.000 | spawn |
| 2 | Cardboard Braces | 1.053 | Island 2 |
| 3 | Duct-Tape Wraps | 1.110 | Island 3 |
| 4 | Wooden Stilts | 1.169 | Island 4 |
| 5 | Coil Springs | 1.231 | Island 5 |
| 6 | Shock Absorbers | 1.297 | Island 6 |
| 7 | Hydraulic Pistons | 1.366 | Island 7 |
| 8 | Turbine Calves | 1.439 | Island 8 |
| 9 | Rocket Boosters | 1.516 | Island 9 |
| 10 | Antigrav Struts | 1.597 | Island 10 |

Junk → mechanical → powered → sci-fi, so the silhouette escalates visibly across the climb.

### 2.2 Level multiplier (Training earns this)

```
LevelMult = 1 + 0.72 × (LegLevel / 17000)     -- 1.00 at L0, 1.72 at L17000 (cap of arc 1)
```

Training carries **1.72×** of the total 2.75×, gear carries **1.60×**. Training still edges ahead, so neither track alone maxes you out ✓ — but with a tier every island the two are now much closer to equal partners than they were at five tiers.

### 2.3 Jump-bar quality multipliers

| Result | QualityMult (S) | Distance | Coins (∝ S⁴) |
|---|---|---|---|
| Perfect | 1.00 | 100% | 100% |
| Good | 0.92 | 85% | 72% |
| Okay | 0.82 | 67% | 45% |
| Weak | 0.65 | 42% | 18% |
| Miss (no lock-in) | 0.45 | 20% | 4% |

Note the coin column falls off a cliff — that's automatic, not a separate rule. One formula makes timing matter for reach *and* wallet.

### 2.4 The resulting power curve

| Level | Tier held | Leg Level | **S** | Peak height | Rising reach |
|---|---|---|---|---|---|
| 1 → 2 | 1 | 500 | 1.02 | 7.5 | 10.4 |
| 2 → 3 | 2 | 1,200 | 1.11 | 8.8 | 12.2 |
| 3 → 4 | 3 | 2,000 | 1.20 | 10.4 | 14.4 |
| 4 → 5 | 4 | 3,000 | 1.32 | 12.5 | 17.3 |
| 5 → 6 | 5 | 4,250 | 1.45 | 15.2 | 21.0 |
| 6 → 7 | 6 | 5,750 | 1.61 | 18.7 | 25.9 |
| 7 → 8 | 7 | 7,750 | 1.81 | 23.7 | 32.7 |
| 8 → 9 | 8 | 10,000 | 2.05 | 30.2 | 41.7 |
| 9 → 10 | 9 | 13,000 | 2.35 | 39.8 | 54.9 |
| 10 → summit | 10 | 17,000 | 2.75 | 54.3 | 75.0 |

**This curve is smooth, and that's the real win of ten tiers.** At five tiers, gear arrived every other island and the S curve had visible steps at levels 3, 5, 7 and 9 — an island where a purchase carried you, then an island where only grinding did. A tier per island removes the sawtooth entirely: every island contributes one gear step *and* one training band, so progress reads as continuous.

---

## 3. The jump bar

Sweep speed is **identical at every altitude** — per the settled decision, bad timing is punished by platform spacing, never by a faster bar.

| Parameter | Value |
|---|---|
| Sweep period (one-way pass) | **1.40 s** |
| Sweep pattern | left → right → left, ping-pong; centre = Perfect |
| Attempt times out after | 2 full passes (2.80 s) → auto-Weak |

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
| Client never sends | accuracy, zone name, resulting S, or **coin amount** — only "I pressed, now" |

Coins are derived from jump height, which is derived from S, which is derived from timing — so a client that could report its own coins could mint currency. Server computes the payout.

### 3.2 Combo

Consecutive Perfects. **Coins only — never power.**

```
CoinMult = 1 + 0.10 × comboCount     -- capped at ×3.0 (20 consecutive perfects)
```

Breaks on any non-Perfect, on a fall, and on cashing out.

---

## 4. Training: Leg Level and gravity

### 4.1 Leg Level bands

Design target is **~50 training jumps ≈ 3–5 min per island**. Held constant by scaling *gain per jump* with pit gravity while the band widens in step — so higher pits train faster, and old bands become trivial.

| Island | Pit gain / jump | Band width | Leg Level to leave |
|---|---|---|---|
| 1 (Sand Pit) | 10 | 500 | 500 |
| 2 | 14 | 700 | 1,200 |
| 3 | 16 | 800 | 2,000 |
| 4 | 20 | 1,000 | 3,000 |
| 5 | 25 | 1,250 | 4,250 |
| 6 | 30 | 1,500 | 5,750 |
| 7 | 40 | 2,000 | 7,750 |
| 8 | 45 | 2,250 | 10,000 |
| 9 | 60 | 3,000 | 13,000 |
| 10 | 80 | 4,000 | **17,000** (arc 1 cap) |

**500 training jumps total** across the arc, 50 per island exactly. Because every medal run passes every pit below your cash-out point, training and medal-farming happen on the same trip — you top up each band in passing rather than making dedicated training journeys.

### 4.2 Training-jump cadence **[answers OQ: exact jump count / pacing]**

Under heavy gravity airtime collapses to ~0.35 s, so raw jump spam would finish a band in one minute. The pacing knob is the **crouch wind-up** already in the design doc as a cosmetic — in a training area it becomes a real duration:

| Phase | Outside a pit | Inside a pit |
|---|---|---|
| Crouch wind-up | 0.20 s | **2.50 s** |
| Jump bar | 1.40 s avg | 1.40 s avg |
| Airtime | 0.54–1.49 s | ~0.35 s |
| **Cycle** | ~4.5 s | **~4.1 s** |

50 jumps × 4.1 s = **3.4 minutes per island** ✓ (target 3–5 min). Crouch duration is the single number to tune if pacing feels wrong; it's the same at every altitude, which is what keeps the pit feeling identical everywhere.

### 4.3 Pit gravity

Tuned so a training jump peaks at a constant **4.5 studs** (vs. 7.2 vanilla) at the strength you're expected to have on arrival:

```
g_pit = 196.2 × (7.2 · S_expected²) / 4.5    →   gravityMult = 1.60 × S_expected²
```

| Island | Pit | S on arrival | Gravity × | Effective g |
|---|---|---|---|---|
| 1 | Sand Pit | 1.00 | **1.6** | 314 |
| 2 | Packed Earth | 1.02 | **1.7** | 334 |
| 3 | Gravel Bed | 1.11 | **2.0** | 392 |
| 4 | Clay Flat | 1.20 | **2.3** | 451 |
| 5 | Iron Sand | 1.32 | **2.8** | 549 |
| 6 | Basalt Pan | 1.45 | **3.4** | 667 |
| 7 | Slag Bed | 1.61 | **4.2** | 824 |
| 8 | Leadfield | 1.81 | **5.3** | 1040 |
| 9 | Deep Well | 2.05 | **6.7** | 1315 |
| 10 | Coreground | 2.35 | **8.8** | 1726 |

Small increases low down, aggressive escalation high up ✓ (matches the settled decision) — and now a smooth ramp rather than the stepped one the five-tier curve produced.

The constant 4.5-stud pit height is doing double duty: it also fixes the pit's coin rate at **608 coins/jump forever** (§6), which is what makes training a deliberate coin desert.

**[answers OQ: does gravity run ahead of expected strength?]** — Ship it running **+8% ahead** of the table for a player's **first 15 jumps** in a newly unlocked pit, then relax to the table value. New areas feel heavy for about a minute, then you settle in. One boolean and one float to revert.

**[answers OQ: under-levelled player in a high pit]** — a hard air-time floor rather than a strength gate:

```lua
-- never let a pit jump peak below 2.2 studs, whoever you are
local gEff = math.min(gNominal, (53.2 * S_actual)^2 / (2 * 2.2))
```

At 2.2 studs the pit is miserable and clearly signals "go train lower," but is never literally unplayable.

### 4.4 Implementation (per-character gravity)

`Workspace.Gravity` is global, so fake it per character, server-side:

```lua
-- Attachment on HumanoidRootPart, VectorForce with RelativeTo = World
force.Force = Vector3.new(0, -hrp.AssemblyMass * (gravityMult - 1) * 196.2, 0)
force.ApplyAtCenterOfMass = true
```

Clear on zone exit, on death, **on cash-out teleport**, and on `CharacterRemoving`. `JumpPower` is left alone — the launch feels the same, the fall is what's heavy.

### 4.5 Climbing trickle **[answers OQ: trickle balance]**

```
LegLevel gain = 0.15 × (pit gain of the island below you) × qualityWeight
qualityWeight: Perfect 1.0 | Good 0.7 | Okay 0.4 | Weak 0.15 | Miss 0
```

**The coefficient is 0.15, not the 0.30 you'd pick for a one-way climb** — because the cash-out loop means players re-climb the same levels dozens of times. At 0.30 the trickle alone would outpace the pits and make training optional.

| Full climb to | Island 6 | Summit |
|---|---|---|
| Leg Level from the climb | ~120 | ~640 |

A summit run grants 3.8% of the whole 17,000-point track. Across the ~28 runs a full playthrough needs, climbing supplies **roughly 10–15% of total Leg Level** — meaningful, never sufficient. This is the number to move first if training feels like a chore.

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

One clean rule: **each tier costs exactly 2× its island's arrival bonus**, which lands at ~3 cache runs every time.

| Tier | Island | Cost | Grind runs | ≈ Time |
|---|---|---|---|---|
| 2 Cardboard Braces | 2 | **100** | 3 | 3 min |
| 3 Duct-Tape Wraps | 3 | **300** | 3 | 6 min |
| 4 Wooden Stilts | 4 | **480** | 3 | 9 min |
| 5 Coil Springs | 5 | **800** | 3 | 13 min |
| 6 Shock Absorbers | 6 | **1,200** | 3 | 17 min |
| 7 Hydraulic Pistons | 7 | **1,700** | 4 | 28 min |
| 8 Turbine Calves | 8 | **2,400** | 3 | 26 min |
| 9 Rocket Boosters | 9 | **3,300** | 3 | 30 min |
| 10 Antigrav Struts | 10 | **4,500** | 3 | 35 min |

**~28 grind runs, ~2 h 47 m of medal farming** across the arc — up from 5 tiers' 22 runs, so ten tiers costs about 30 extra minutes of play in exchange for a milestone on every island instead of every other one.

Medals top out in the low thousands. This is still a deliberate rejection of the Ninja Legends pattern (rank 45 costs 5.6 septendecillion): a currency you can't grind in place doesn't need inflation to stay interesting, and keeping "is this tier worth three more climbs?" a legible question matters more than a big number.

### 5.5 Why the checkpoint reset is non-negotiable

Per-cloud checkpoints (settled) plus a cash-out teleport is an open exploit: bank the cache, then deliberately fall, and the respawn returns you to your top checkpoint with the medals already paid. Free infinite medals, no climb.

So cashing out must clear the chain back to Island 1. Server-side, in one transaction: award → wipe checkpoint list → teleport. Never trust a client-reported checkpoint after a cash-out.

### 5.6 **[answers OQ: how do you retrain without fast travel?]**

The cash-out *is* the ride down, and it's opt-in — which resolves the open question without adding a recall. You bank at island 6, land at island 1, and the re-climb passes every pit on the way up. Training is no longer a dedicated trip; it's something you top up in passing on a journey you were making anyway.

The trade is that **the backtracking chokepoint question gets sharper, not softer**: every player is funnelled through Island 1 constantly, so its spawn, sand pit entrance, and first ledges are now the highest-traffic geometry in the game. Size them for it (§7.4) and treat them as the first thing to stress-test with 16 players.

---

## 6. Coins — the height loop

### 6.1 Income

**Coins are paid per jump, scaled by how high that jump peaked.** One formula, no altitude term, no island multiplier:

```
Coins = 30 × (peakHeight in studs)² × comboMult
```

Since peak height = `7.2·S²`, coins scale with **S⁴**. Quality is already inside S, so bad timing is punished automatically (§2.3) with no separate rule.

| Jump | Peak height | Coins (no combo) |
|---|---|---|
| Vanilla legs, Perfect | 7.5 | **1,688** |
| Vanilla legs, Weak | 3.2 | **307** |
| Island 5 legs, Perfect | 15.2 | **6,931** |
| Island 10 legs, Perfect | 54.3 | **88,450** |
| Island 10 legs, Perfect, ×3 combo | 54.3 | **265,350** |
| **Any training-pit jump, any island** | 4.5 | **608** |

That last row is the point. Pit height is pinned at 4.5 studs by design (§4.3), so training pays **608 coins forever** — ~8.9K/min, the worst rate in the game at every stage, and it never improves. Coins come from climbing tall, which is exactly what the rule says.

### 6.2 Per-run coin yield

At ×1.35 average combo:

| Cash out at | Coins/run | Coins/min |
|---|---|---|
| Island 2 | 23K | 25,300 |
| Island 4 | 110K | 35,400 |
| Island 6 | 323K | 57,700 |
| Island 8 | 900K | 105,800 |
| Island 10 | 2.68M | 227,300 |
| Summit (no cash-out) | 5.07M | 372,800 |

15× spread bottom to top, arriving almost entirely from leg strength rather than from any altitude bonus.

### 6.3 Coin sinks — egg prices

Each egg ≈ 25–30 minutes of income at the altitude where it unlocks, so the sink tracks the tap.

| Island | Egg | Cost |
|---|---|---|
| 1 | Sand Egg | 500,000 |
| 2 | Meadow Egg | 750,000 |
| 3 | Pebble Egg | 1,000,000 |
| 4 | Breeze Egg | 1,500,000 |
| 5 | Storm Egg | 2,000,000 |
| 6 | Aurora Egg | 3,000,000 |
| 7 | Thunder Egg | 4,500,000 |
| 8 | Nebula Egg | 6,500,000 |
| 9 | Void Egg | 10,000,000 |
| 10 | Summit Egg | 15,000,000 |

≈ ×1.45 per tier, slightly ahead of the income curve so later eggs cost marginally more playtime. Deliberately flatter than the Bubble Gum Simulator shape (10 → 110 → 450 → 5,000 → 15,000, ×4–10 per tier) — BGS can afford runaway pricing because its pets multiply income, and ours don't.

Auras: one band below eggs (island N aura = island N−1 egg price). **Aura reroll: 400,000 flat, unlimited** — the repeatable sink that absorbs late-game overflow.

> ⚠️ **Risk worth flagging, not a recommendation to change the design.** Every comparable game's pets carry a stat multiplier, which is what makes players buy the 40th one. Cosmetic-only pets are a much weaker sink, and the design doc's own warning ("don't let this catalog ship shallow") is doing a lot of load-bearing work. If the catalog can't be deep at launch, the uncapped aura reroll is the pressure valve.

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

| Level | Rising reach | **Main gap** | Peak height | Hops | Rise/hop | Cloud Ø | Ø as % of gap |
|---|---|---|---|---|---|---|---|
| 1 | 10.4 | **8** | 7.5 | 10 | 4.1 | 12 | 150% |
| 2 | 12.2 | **10** | 8.8 | 11 | 4.8 | 12 | 120% |
| 3 | 14.4 | **11** | 10.4 | 12 | 5.7 | 12 | 109% |
| 4 | 17.3 | **14** | 12.5 | 13 | 6.9 | 12 | 86% |
| 5 | 21.0 | **17** | 15.2 | 14 | 8.4 | 12 | 71% |
| 6 | 25.9 | **21** | 18.7 | 15 | 10.3 | 12 | 57% |
| 7 | 32.7 | **26** | 23.7 | 16 | 13.0 | 12 | 46% |
| 8 | 41.7 | **33** | 30.2 | 17 | 16.6 | 13 | 38% |
| 9 | 54.9 | **44** | 39.8 | 18 | 21.9 | 17 | 38% |
| 10 | 75.0 | **60** | 54.3 | 20 | 29.9 | 23 | 38% |

All in studs. **146 hops total** — deliberately short, because the cash-out loop makes you climb this ~28 times. A run to the top is ~12 minutes; a mid-game run to island 6 is ~5.6 minutes.

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

| Island | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | Summit |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Altitude (studs) | 0 | 40 | 95 | 160 | 250 | 370 | 525 | 735 | 1015 | 1410 | **2010** |

| Thing | Size (studs) |
|---|---|
| **Island 1 diameter** | **280** — spawn, sand pit, shops, *and the arrival pad every cash-out lands on*. Oversized on purpose (§5.6): this is the busiest geometry in the game. |
| Islands 2–3 | 160 |
| Islands 4–6 | 140 |
| Islands 7–9 | 120 |
| Island 10 | 100 |
| Cash-out arrival pad (Island 1) | 40 diameter, **non-collidable between players**, ≥30 studs clear of the first jump ledge |
| Training area diameter | 45 (island 1) → 30 (island 10) |
| Pit rim height / depth | 3 / 2 |
| Medal cache footprint | 6 × 6, with a 3-stud no-walk buffer so nobody strays into the prompt |
| Leg-tier vendor footprint | 8 × 8, on every island, ≥15 studs from the cache so the two prompts never overlap |
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

Worth noting the loop makes repositioning bite harder than it did: a player who has memorised a route re-runs it ~28 times, and the reshuffle invalidates that muscle memory. That's the hook working as intended, but it means the 30-minute interval should be watched in testing — it's the difference between "the world changed" and "my grind route got deleted."

---

## 8. Time to finish arc 1

| Activity | Amount | Total |
|---|---|---|
| Medal grind runs | ~28 runs | 167 min |
| Training | 500 jumps × 4.1 s | 34 min |
| Discovery pushes to new personal bests | — | ~25 min |
| Falls, menus, walking | — | ~25 min |
| | | **≈ 4 h 10 m** |

That's an efficient run. Realistically **5–9 hours** for an average player, since miss rate climbs steeply past level 7 and a failed high push costs a whole run's time. Comparable to Steep Steps, where only ~1% of players reach 1,000 m — expect a similar completion cliff, and expect islands 8–10 to be seen by a small minority.

---

## 9. HUD display

| Element | Format |
|---|---|
| Altitude | **display studs with an "m" suffix** — "1,410 m". Genre convention; the true conversion (1 stud = 0.28 m) would read 395 m and undersell the climb. Cosmetic call, flag if you'd rather be honest. |
| Coins | abbreviated: `23K`, `2.68M`, `15M` |
| Medals | never abbreviated — always exact ("1,200"). They're scarce; the exact number is the point. |
| Leg Level | `L 4,250 / 5,750` — current / next island's threshold |
| Leg Tier | icon + name + `7/10`, so the tier track reads as a collection |
| **Cache preview** | when in range of a cache, show **"+200 Medals — returns you to the bottom"** and the value of the *next* island's cache alongside it. The decision only works if both sides of it are visible. |
| Combo | only visible above ×1.5, so it reads as a reward not a nag |

---

## 10. Extending past island 10

| Quantity | Per additional island |
|---|---|
| Leg Level band | × 1.33 |
| Pit gravity multiplier | × 1.28 |
| Medal cache value | × 1.38 |
| Medal first-arrival bonus | × 1.4 |
| **Leg Tier** | **one per island**, cost = 2× that island's arrival bonus |
| Tier multiplier | × 1.053 |
| Main gap | × 1.25 |
| Hops in level | + 2 |
| Egg price | × 1.45 |

Coins need no extension rule — they're a pure function of jump height, so they scale themselves.

---

## Reference data (what these are calibrated against)

- **Roblox engine constants** — gravity 196.2 studs/s², default `JumpPower` 50 / `JumpHeight` 7.2, `WalkSpeed` 16, 1 stud = 0.28 m: [Jumping — Roblox Wiki](https://roblox.fandom.com/wiki/Jumping), [Stud (unit) — Roblox Wiki](https://roblox.fandom.com/wiki/Stud_(unit)), [WalkSpeed — Roblox Wiki](https://roblox.fandom.com/wiki/Class:Humanoid/WalkSpeed)
- **Vanilla jump reach (~10–12 studs horizontal, 11-stud block height)** — [Calculating maximum player jump distance](https://devforum.roblox.com/t/calculating-maximum-player-jump-distance/455105), [How to calculate studs jumped from JumpPower](https://devforum.roblox.com/t/how-to-calculate-how-many-studs-a-player-can-jump-based-on-jumppower/100480). Our S=1 model reproduces 11.9 studs, so the whole curve is anchored to measured vanilla behaviour.
- **Ninja Legends** — 62 ranks with costs escalating into quadrillions/septillions and multipliers to ×9.6 billion: [Ranks — Ninja Legends Wiki](https://roblox-ninja-legends.fandom.com/wiki/Ranks), [highest rank breakdown](https://www.sportskeeda.com/roblox-news/what-highest-rank-roblox-ninja-legends). Its *many small ranks* structure is the model for a tier per island; its number inflation is the anti-pattern we avoid for Medals.
- **Muscle Legends** — rebirth cost `10,000 + 5,000x`, first rebirth at 10,000 strength, stacking permanent multipliers: [How Rebirthing Works](https://muscle-legends.fandom.com/wiki/How_Rebirthing_Works). The reset-and-re-run structure is the closest shipped analogue to our cash-out loop, and the source of the "flat cost curve, multiplicative reward" shape behind our tier pricing.
- **Bubble Gum Simulator** — egg costs 10 → 110 → 450 → 5,000 → 15,000 across worlds (~×4–10 per tier): [Eggs — BGS Infinity Wiki](https://bgs-infinity.fandom.com/wiki/Eggs). Our egg curve is deliberately flatter (×1.45) because our pets carry no income multiplier.
- **Steep Steps** — 1,000–2,000 m mountains, bonfire checkpoints every 100 m, only ~1% of players reach 1,000 m: [Rolimon's](https://www.rolimons.com/game/11606818992), [Steep Steps — Roblox Wiki](https://roblox.fandom.com/wiki/Steep_steps/STEEP_STEPS). Anchor for climb length, checkpoint density, and realistic completion rates.
- **Exponential cost-curve practice** — pure `level^3` / `base × 2^level` curves are widely reported as unbalanced (front-loaded then trivial); adding a linear term or fitting the curve empirically is the standard fix: [Balancing exponential upgrade progression](https://devforum.roblox.com/t/balancing-exponential-upgrade-progression/2434950), [Simulator Formulas](https://devforum.roblox.com/t/simulator-formulas/853976). Why our bands scale ×1.25–1.5 rather than ×2+.
