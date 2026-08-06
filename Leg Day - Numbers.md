# Leg Day — Numbers (first tuning pass)

Concrete values for jump strength, gravity, Leg Level, Medals, Coins, and world geometry. Companion to [Leg Day - Design Doc.md](Leg%20Day%20-%20Design%20Doc.md) — that doc owns the *design*, this one owns the *numbers*. Where a number answers an open question, it's marked **[answers OQ]**.

All numbers are a first pass calibrated against real Roblox physics and shipped-game economies (see [Reference data](#reference-data-what-these-are-calibrated-against) at the end). They're meant to be tuned, not treated as final.

---

## 0. The two loops these numbers serve

**Medals — the run loop.** Each island holds a **medal cache**. Touching it banks the medals *and teleports you to the bottom*, clearing your checkpoint chain. One cache per run, so every run is a single decision: **how high do I dare climb before cashing out?** Higher caches are worth more per minute, so the ceiling of your legs sets your earn rate. A Leg Tier costs several full round trips.

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

| Quantity | Formula | S=1.00 | S=2.755 (max) |
|---|---|---|---|
| Airtime (flat) | `2·Vy / g` | 0.54 s | 1.49 s |
| **Peak height** | `Vy² / 2g` = `7.2·S²` | 7.2 studs | 54.6 studs |
| Flat reach | `Vx · airtime` = `11.9·S²` | 11.9 studs | 90.3 studs |
| **Rising reach** (see §7.1) | `0.835 × flat` = `9.94·S²` | 9.9 studs | 75.4 studs |

S=1 giving an 11.9-stud flat reach lands inside the measured 10–12 stud vanilla range — the model is anchored to reality, not invented.

Distance and height both scale with **S²**, so power feels superlinear: doubling S nearly quadruples your reach. Coins scale with height *squared* (§6), so they scale with **S⁴** — the same upgrade that doubles your reach multiplies your income sixteenfold. That's the whole "superhero legs" payoff, and it's why the power number itself can stay small.

**Full range: S = 1.00 → 2.755.** Deliberately small and readable. All number inflation lives in Coins.

### 2.1 Tier multipliers (Medals buy these)

| # | Leg Tier | TierMult | Bought around |
|---|---|---|---|
| 1 | Bare Legs | 1.00 | spawn |
| 2 | Cardboard Braces | 1.10 | Island 2–3 |
| 3 | Spring Boots | 1.21 | Island 4–5 |
| 4 | Piston Legs | 1.32 | Island 6–7 |
| 5 | Rocket Legs | 1.45 | Island 8–9 |

≈ ×1.10 per tier. Modest on purpose: gear is the *smaller* half of the product so training can't be bypassed by a wallet.

### 2.2 Level multiplier (Training earns this)

```
LevelMult = 1 + 0.90 × (LegLevel / 1700)      -- 1.00 at L0, 1.90 at L1700 (cap of arc 1)
```

Training carries **1.90×** of the total 2.755×, gear carries **1.45×**. Skill track dominates, as designed.

### 2.3 Jump-bar quality multipliers

| Result | QualityMult (S) | Distance | Coins (∝ S⁴) |
|---|---|---|---|
| Perfect | 1.00 | 100% | 100% |
| Good | 0.92 | 85% | 72% |
| Okay | 0.82 | 67% | 45% |
| Weak | 0.65 | 42% | 18% |
| Miss (no lock-in) | 0.45 | 20% | 4% |

Note the coin column falls off a cliff — that's automatic, not a separate rule. One formula makes timing matter for reach *and* wallet.

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

Coins are now derived from jump height, which is derived from S, which is derived from timing — so a client that could report its own coins could mint currency. Server computes the payout.

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
| 1 (Sand Pit) | 1.0 | 50 | 50 |
| 2 | 1.4 | 70 | 120 |
| 3 | 1.6 | 80 | 200 |
| 4 | 2.0 | 100 | 300 |
| 5 | 2.5 | 125 | 425 |
| 6 | 3.0 | 150 | 575 |
| 7 | 4.0 | 200 | 775 |
| 8 | 4.5 | 225 | 1000 |
| 9 | 6.0 | 300 | 1300 |
| 10 | 8.0 | 400 | **1700** (arc 1 cap) |

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
| 2 | Packed Earth | 1.03 | **1.7** | 334 |
| 3 | Gravel Bed | 1.06 | **1.8** | 353 |
| 4 | Clay Flat | 1.22 | **2.4** | 471 |
| 5 | Iron Sand | 1.28 | **2.6** | 510 |
| 6 | Basalt Pan | 1.48 | **3.5** | 687 |
| 7 | Slag Bed | 1.58 | **4.0** | 785 |
| 8 | Leadfield | 1.86 | **5.5** | 1079 |
| 9 | Deep Well | 2.02 | **6.5** | 1275 |
| 10 | Coreground | 2.45 | **9.6** | 1883 |

Small increases low down, aggressive escalation high up ✓ (matches the settled decision).

The constant 4.5-stud pit height is now doing double duty: it also fixes the pit's coin rate at **61 coins/jump forever** (§6), which is what makes training a deliberate coin desert.

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
| Leg Level from the climb | ~12 | ~64 |

A summit run grants 3.8% of the whole 1700-point track. Across the ~22 runs a full playthrough needs, climbing supplies **roughly 10–15% of total Leg Level** — meaningful, never sufficient. This is the number to move first if training feels like a chore.

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

The voluntary-interaction and no-teleport-on-arrival rules are load-bearing: without them, pushing to a new personal best would punish you by ejecting you the moment you landed, and the risk/reward decision would evaporate.

The checkpoint reset is equally load-bearing — see §5.5.

### 5.2 Cache values and run economics

Effective hop time is **5.6 s** (4.5 s cycle + ~25% miss overhead).

| Cash out at | Hops from bottom | Run time | Cache | **Medals/min** |
|---|---|---|---|---|
| Island 2 | 10 | 0.9 min | **2** | 2.15 |
| Island 3 | 21 | 2.0 min | **5** | 2.55 |
| Island 4 | 33 | 3.1 min | **8** | 2.60 |
| Island 5 | 46 | 4.3 min | **13** | 3.03 |
| Island 6 | 60 | 5.6 min | **20** | 3.57 |
| Island 7 | 75 | 7.0 min | **28** | 4.00 |
| Island 8 | 91 | 8.5 min | **40** | 4.71 |
| Island 9 | 108 | 10.1 min | **55** | 5.46 |
| Island 10 | 126 | 11.8 min | **75** | 6.38 |

Rate rises monotonically, ~3× from bottom to top. Climbing higher is always strictly better per minute — but never *so* much better that being capped at island 6 for a while feels pointless.

### 5.3 First-arrival bonuses

One-time, ≈ 3× that island's cache, so a new personal best is worth about three farm runs — a real burst that funds the next tier and rewards pushing over grinding.

| Island | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| Bonus | 6 | 15 | 24 | 40 | 60 | 85 | 120 | 165 | 225 |

Total from discovery: **740 medals.**

### 5.4 Tier costs

Priced in *runs*, since that's the real currency now. Target: 4–7 grind runs per tier after the arrival bonuses are spent.

| Purchase | Cost | Bought around | Grind runs needed | ≈ Time |
|---|---|---|---|---|
| Cardboard Braces | **40** | Island 2–3 | 4 | 8 min |
| Spring Boots | **110** | Island 4–5 | 5 | 21 min |
| Piston Legs | **260** | Island 6–7 | 6 | 45 min |
| Rocket Legs | **450** | Island 8–9 | 6 | 63 min |

Ratios flatten (×2.75, ×2.36, ×1.73) because the earn *rate* accelerates — flat ratios would make the last tier a wall. **~22 grind runs, ~2 h 20 m of medal farming** across the arc.

Medals stay in three digits all game. **This is a deliberate rejection of the Ninja Legends pattern** (rank 45 costs 5.6 septendecillion): a currency you can't grind in place doesn't need inflation to stay interesting, and small numbers keep "is this tier worth six more climbs?" a legible question.

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
Coins = 3 × (peakHeight in studs)² × comboMult
```

Since peak height = `7.2·S²`, coins scale with **S⁴**. Quality is already inside S, so bad timing is punished automatically (§2.3) with no separate rule.

| Jump | Peak height | Coins (no combo) |
|---|---|---|
| Vanilla legs, Perfect | 7.6 | **173** |
| Vanilla legs, Weak | 3.2 | **31** |
| Island 5 legs, Perfect | 15.8 | **749** |
| Island 10 legs, Perfect | 54.6 | **8,944** |
| Island 10 legs, Perfect, ×3 combo | 54.6 | **26,832** |
| **Any training-pit jump, any island** | 4.5 | **61** |

That last row is the point. Pit height is pinned at 4.5 studs by design (§4.3), so training pays **61 coins forever** — ~890/min, the worst rate in the game at every stage, and it never improves. Coins come from climbing tall, which is exactly what the rule says.

### 6.2 Per-run coin yield

At ×1.35 average combo:

| Cash out at | Coins/run | Coins/min |
|---|---|---|
| Island 2 | 2.3K | 2,512 |
| Island 4 | 10.8K | 3,518 |
| Island 6 | 32.2K | 5,752 |
| Island 8 | 91.8K | 10,818 |
| Island 10 | 286K | 24,355 |
| Summit (no cash-out) | 528K | 38,816 |

15× spread bottom to top, arriving almost entirely from leg strength rather than from any altitude bonus.

### 6.3 Coin sinks — egg prices

Each egg ≈ 20–30 minutes of income at the altitude where it unlocks, so the sink tracks the tap.

| Island | Egg | Cost |
|---|---|---|
| 1 | Sand Egg | 50,000 |
| 2 | Meadow Egg | 75,000 |
| 3 | Pebble Egg | 100,000 |
| 4 | Breeze Egg | 150,000 |
| 5 | Storm Egg | 200,000 |
| 6 | Aurora Egg | 300,000 |
| 7 | Thunder Egg | 450,000 |
| 8 | Nebula Egg | 650,000 |
| 9 | Void Egg | 1,000,000 |
| 10 | Summit Egg | 1,500,000 |

≈ ×1.4 per tier, slightly ahead of the income curve so later eggs cost marginally more playtime. Deliberately flatter than the Bubble Gum Simulator shape (10 → 110 → 450 → 5,000 → 15,000, ×4–10 per tier) — BGS can afford runaway pricing because its pets multiply income, and ours don't.

Auras: one band below eggs (island N aura = island N−1 egg price). **Aura reroll: 40,000 flat, unlimited** — the repeatable sink that absorbs late-game overflow.

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
| 1 | 10.5 | **8** | 7.6 | 10 | 4.2 | 12 | 150% |
| 2 | 11.3 | **9** | 8.1 | 11 | 4.5 | 12 | 133% |
| 3 | 14.7 | **12** | 10.7 | 12 | 5.9 | 12 | 100% |
| 4 | 16.2 | **13** | 11.7 | 13 | 6.4 | 12 | 92% |
| 5 | 21.8 | **17** | 15.8 | 14 | 8.7 | 12 | 71% |
| 6 | 24.8 | **20** | 17.9 | 15 | 9.8 | 12 | 60% |
| 7 | 34.4 | **28** | 24.9 | 16 | 13.7 | 12 | 43% |
| 8 | 40.5 | **32** | 29.3 | 17 | 16.1 | 12 | 38% |
| 9 | 59.6 | **48** | 43.1 | 18 | 23.7 | 18 | 38% |
| 10 | 75.4 | **60** | 54.6 | 20 | 30.0 | 23 | 38% |

All in studs. **146 hops total** — deliberately short, because the cash-out loop makes you climb this ~22 times. A run to the top is ~12 minutes; a mid-game run to island 6 is ~5.6 minutes.

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
| Altitude (studs) | 0 | 40 | 90 | 165 | 245 | 370 | 515 | 735 | 1010 | 1435 | **2035** |

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
| Medal cache footprint | 6 × 6, with a 3-stud no-walk buffer so nobody cashes out by accident |
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

Worth noting the loop makes repositioning bite harder than it did: a player who has memorised a route re-runs it ~22 times, and the reshuffle invalidates that muscle memory. That's the hook working as intended, but it means the 30-minute interval should be watched in testing — it's the difference between "the world changed" and "my grind route got deleted."

---

## 8. Time to finish arc 1

| Activity | Amount | Total |
|---|---|---|
| Medal grind runs | ~22 runs | 137 min |
| Training | 500 jumps × 4.1 s | 34 min |
| Discovery pushes to new personal bests | — | ~25 min |
| Falls, menus, walking | — | ~25 min |
| | | **≈ 3 h 40 m** |

That's an efficient run. Realistically **5–8 hours** for an average player, since miss rate climbs steeply past level 7 and a failed high push costs a whole run's time. Comparable to Steep Steps, where only ~1% of players reach 1,000 m — expect a similar completion cliff, and expect islands 8–10 to be seen by a small minority.

---

## 9. HUD display

| Element | Format |
|---|---|
| Altitude | **display studs with an "m" suffix** — "1,435 m". Genre convention; the true conversion (1 stud = 0.28 m) would read 402 m and undersell the climb. Cosmetic call, flag if you'd rather be honest. |
| Coins | abbreviated: `1.2K`, `450K`, `1.25M` |
| Medals | never abbreviated — always exact ("260"). They're scarce; the exact number is the point. |
| Leg Level | `L 425 / 575` — current / next island's threshold |
| **Cache preview** | when in range of a cache, show **"+20 Medals — returns you to the bottom"** and the value of the *next* island's cache alongside it. The decision only works if both sides of it are visible. |
| Combo | only visible above ×1.5, so it reads as a reward not a nag |

---

## 10. Extending past island 10

| Quantity | Per additional island |
|---|---|
| Leg Level band | × 1.33 |
| Pit gravity multiplier | × 1.28 |
| Medal cache value | × 1.38 |
| Medal first-arrival bonus | × 1.4 |
| Main gap | × 1.25 |
| Hops in level | + 2 |
| Egg price | × 1.4 |
| Leg Tier | one every 2 islands, cost × 1.9 |

Coins need no extension rule — they're a pure function of jump height, so they scale themselves.

---

## Reference data (what these are calibrated against)

- **Roblox engine constants** — gravity 196.2 studs/s², default `JumpPower` 50 / `JumpHeight` 7.2, `WalkSpeed` 16, 1 stud = 0.28 m: [Jumping — Roblox Wiki](https://roblox.fandom.com/wiki/Jumping), [Stud (unit) — Roblox Wiki](https://roblox.fandom.com/wiki/Stud_(unit)), [WalkSpeed — Roblox Wiki](https://roblox.fandom.com/wiki/Class:Humanoid/WalkSpeed)
- **Vanilla jump reach (~10–12 studs horizontal, 11-stud block height)** — [Calculating maximum player jump distance](https://devforum.roblox.com/t/calculating-maximum-player-jump-distance/455105), [How to calculate studs jumped from JumpPower](https://devforum.roblox.com/t/how-to-calculate-how-many-studs-a-player-can-jump-based-on-jumppower/100480). Our S=1 model reproduces 11.9 studs, so the whole curve is anchored to measured vanilla behaviour.
- **Ninja Legends** — 62 ranks with costs escalating into quadrillions/septillions and multipliers to ×9.6 billion: [Ranks — Ninja Legends Wiki](https://roblox-ninja-legends.fandom.com/wiki/Ranks), [highest rank breakdown](https://www.sportskeeda.com/roblox-news/what-highest-rank-roblox-ninja-legends). Used as the **anti-pattern** for Medals — we keep the power currency in three digits.
- **Muscle Legends** — rebirth cost `10,000 + 5,000x`, first rebirth at 10,000 strength, stacking permanent multipliers: [How Rebirthing Works](https://muscle-legends.fandom.com/wiki/How_Rebirthing_Works). The reset-and-re-run structure is the closest shipped analogue to our cash-out loop, and the source of the "flat cost curve, multiplicative reward" shape behind our tier pricing.
- **Bubble Gum Simulator** — egg costs 10 → 110 → 450 → 5,000 → 15,000 across worlds (~×4–10 per tier): [Eggs — BGS Infinity Wiki](https://bgs-infinity.fandom.com/wiki/Eggs). Our egg curve is deliberately flatter (×1.4) because our pets carry no income multiplier.
- **Steep Steps** — 1,000–2,000 m mountains, bonfire checkpoints every 100 m, only ~1% of players reach 1,000 m: [Rolimon's](https://www.rolimons.com/game/11606818992), [Steep Steps — Roblox Wiki](https://roblox.fandom.com/wiki/Steep_steps/STEEP_STEPS). Anchor for climb length, checkpoint density, and realistic completion rates.
- **Exponential cost-curve practice** — pure `level^3` / `base × 2^level` curves are widely reported as unbalanced (front-loaded then trivial); adding a linear term or fitting the curve empirically is the standard fix: [Balancing exponential upgrade progression](https://devforum.roblox.com/t/balancing-exponential-upgrade-progression/2434950), [Simulator Formulas](https://devforum.roblox.com/t/simulator-formulas/853976). Why our bands scale ×1.25–1.5 rather than ×2+.
