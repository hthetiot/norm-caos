# ⬡ CAOS Neural Demo

A browser-based artificial life simulation inspired by **Creatures 1** (CyberLife, 1996) — the first commercial application of machine learning in an interactive simulation. A small creature called a **Norn** lives in a 2D world, has a full biochemical system, and learns behaviour through a neural network trained by experience.

No build step. No server. No dependencies except TensorFlow.js from CDN.

---

## What it is

The original Creatures game simulated living organisms through three interacting systems: **biochemistry**, **neural networks**, and **genetics**. This demo faithfully reimplements the first two in the browser.

Every Norn has a bloodstream with real metabolic chemicals. Hunger doesn't rise on a timer — it rises because glycogen breaks down into glucose, and glucose gets burned by movement. Eating injects starch (delayed nutrition) and saccharin (immediate hunger satisfaction). Poison injects glycotoxin that destroys glycogen — the life force chemical. When glycogen hits zero, the Norn dies.

The neural network learns which actions reduce drives through reward and punishment chemicals, produced automatically by the biochemistry when drives fall or rise — exactly the mechanism Steve Grand described in his 1997 academic paper on Creatures.

---

## The biochemistry

Creatures 1 uses **256 chemical slots**, each holding a concentration from 0–255. Every effect in the game — hunger, sleep, learning, death — emerges from genetically defined chemical reactions between these slots. This demo implements the core subset:

### Metabolic chain

```
Food eaten → Starch + Saccharin injected
Starch → Glucose  (digestion, ~64 ticks)
Glucose → Glycogen  (energy storage when glucose is high)
Glycogen → Glucose  (energy release when glucose is low → raises Hunger)
Glucose burned by movement (Hexokinase / CO₂)
Glycogen = 0 → Death
```

### Drive chemicals

| # | Drive | Rises when | Falls when |
|---|-------|-----------|-----------|
| 0 | Hunger | Glycogen/glucose low | Eating (Saccharin) |
| 1 | Tiredness | Moving | Sleeping |
| 2 | Sleepiness | Time × Tiredness | Sleeping |
| 3 | Fear | Near poison | Time (decay) |
| 4 | Boredom | Idle | Moving, eating |
| 5 | Need Pleasure | Time | Interaction, Say Need |

### Reward / Punishment triad (C1 CHEM 49 / 50 / 51)

This is the heart of learning in Creatures. Every tick:

- **Drive falls** → `DriveDecrease + Drive → Reward (CHEM 49)` produced
- **Drive rises** → `DriveIncrease → Drive + Punishment (CHEM 50)` produced
- **Both decay** → `Reinforcement (CHEM 51)` produced

Reward strengthens the synaptic weights of whatever action the Norn last took. Punishment weakens them. Reinforcement signals "this moment matters" regardless of valence. The neural network learns from these signals, not from an externally computed score.

### Poison (Glycotoxin, CHEM 67)

Contact injects:
- **Glycotoxin** — destroys glycogen (life force)
- **Fear Increase (CHEM 26)** — raises Fear drive
- **Pain Increase (CHEM 17) → Punishment (CHEM 50)** — directly weakens the action that brought the Norn close

### Teaching (tickle / slap)

- **Tickle (Reward)** — injects CHEM 49 (Reward) + CHEM 34 (NFP Decrease) + CHEM 33 (Endorphin / Fear Decrease)
- **Slap (Punish)** — injects CHEM 50 (Punishment) + CHEM 26 (Fear Increase)

---

## The neural network

Each Norn has a **TensorFlow.js DQN** (Deep Q-Network) with:

- **Input** — 31-dimensional state vector: 9 sensor inputs + 16 concept lobe activations + 6 drive levels
- **Hidden layers** — Dense(32, relu) → Dense(24, relu)
- **Output** — 8 Q-values, one per action

Training uses **experience replay** — a buffer of past (state, action, reward, nextState) tuples, sampled randomly each batch to break temporal correlation.

### Actions

| # | Action | Effect |
|---|--------|--------|
| 0 | Move Left | vx = −SPEED |
| 1 | Move Right | vx = +SPEED |
| 2 | Move Up | vy = −SPEED |
| 3 | Move Down | vy = +SPEED |
| 4 | Eat | Steers toward nearest food |
| 5 | Sleep | Triggers sleep (reduces Tiredness + Sleepiness) |
| 6 | Wander | Random velocity nudge |
| 7 | Say Need | Reduces Need for Pleasure |

Sleep is also **involuntary** — it overrides all actions when Sleepiness ≥ 85, matching C1's involuntary action system.

### Recommended learning settings

| Setting | Value | Reason |
|---------|-------|--------|
| Learning rate | 0.002 | Low enough not to overwrite good memories |
| Memory size | 180 | Large buffer = diverse batch samples |
| Discount γ | 0.92 | Values near-future rewards appropriately |
| Auto-train | every 25t | Frequent enough without overfitting consecutive states |

---

## Controls

### Toolbar

| Button | Action |
|--------|--------|
| ⏸ PAUSE / ▶ PLAY | Toggle simulation |
| ▶ STEP | Advance one tick (pauses first) |
| ◀ BACK | Undo last tick (up to 20 steps) |
| ✎ TEACH | Enable teach mode — opens TEACH tab |
| + FOOD | Spawn food at random position |
| ✎ (food group) | Place food mode — click world to place |
| + POISON | Spawn poison at random position |
| ✎ (poison group) | Place poison mode — click world to place |
| ⚙ TRAIN | Run one backprop batch from memory |
| ↺ RESET | Reset world and Norns (keeps neural weights) |

### Canvas interactions

- **Click on a Norn** — select it (syncs all panels to that creature)
- **Click in place mode** — plant food or poison exactly where you click

---

## Tabs (side panel)

### DRIVES
Six vertical bars showing live biochemical drive levels. Bar colors match the Norn's body color (the highest drive determines body color).

### SENSE
- **Sensor inputs** — 9 raw values fed to the neural network: food proximity/direction, poison proximity/direction, wall distances (LRTB), touch
- **Concept lobe activations** — 16 internal neurons combining sensor + drive state (cyan = positive, red = negative)

### ACTIONS
8 action cards showing the neural network's current Q-value probability for each action. The currently executing action is highlighted green.

### TEACH
Manual training interface:
1. Enable **TEACH** mode in toolbar
2. Click an action card to pin it as the target
3. Press **✓ REWARD** (tickle — injects Reward + NFP Decrease + Endorphin)
4. Press **✗ PUNISH** (slap — injects Punishment + Fear Increase)
5. Use **⚙ TRAIN** to run backprop on collected memories

If "Pause when TEACH on" is checked in Settings (default), the sim pauses when you enter teach mode so you can step through decisions manually.

### SETTINGS

**SIMULATION** — tick speed (1–60 ticks/s)

**OBJECTS**
- Auto-spawn food / poison checkboxes
- Spawn rate sliders (every N ticks)

**DRIVE RISE RATES** — tune how quickly Hunger, Tiredness, and Boredom accumulate

**LEARNING**
- Learning rate (0.001–0.010)
- Memory size (replay buffer capacity)
- Discount γ (future reward weighting)
- Auto-train every N ticks (0 = OFF; requires sim to be running)
- Pause when TEACH on

**CREATURES**
- Select active creature (drives all panels)
- Add new creature
- Import creature from `.json` file
- Remove selected creature
- Export selected creature to `.json`

### LOG
Live CAOS-style command log. Each line shows the tick number and a CAOS macro equivalent of what just happened — chemical injections, drive events, sleep/wake, training steps, selections.

---

## Status bar

```
TICK 1240  EATEN 8  POISONED 2  TRAINED 47  LOSS 0.0312  CAGE Youth  GLYCOGEN 87  GLUCOSE 62
```

| Field | Meaning |
|-------|---------|
| TICK | Total simulation ticks elapsed |
| EATEN | Food items consumed (all Norns) |
| POISONED | Poison contact events |
| TRAINED | Backprop batches completed |
| LOSS | Last training batch MSE loss |
| CAGE | Age stage of selected Norn (Baby → Ancient) |
| GLYCOGEN | Life force of selected Norn (0 = death) |
| GLUCOSE | Short-term energy of selected Norn |

---

## Multi-creature world

Multiple Norns can coexist. Each is ticked independently every simulation step — they each sense the same world (foods, poisons, walls) but have independent drives, biochemistry, and epsilon-greedy exploration. All Norns share the **same neural network and replay buffer** — they collectively train a single model, similar to how C1's instinct system allowed norns to learn from each other.

Click a Norn on the canvas to select it, or use the dropdown in SETTINGS → CREATURES. The selected Norn's data is shown in DRIVES, SENSE, ACTIONS, and TEACH tabs.

Norns can be exported as `.json` and re-imported into any session, carrying their full biochemical state.

---

## Technical notes

- **No build step** — TF.js loaded from CDN, works offline after first load
- **Step back** — up to 20 ticks of world history stored as JSON snapshots, including all Norn states, foods, and poisons
- **Epsilon decay** — each Norn starts with ε=0.3 (30% random exploration), decaying to ε=0.05 as it trains
- **Death** — when a Norn's glycogen reaches 0 it is marked dead and skipped in the tick loop; its corpse remains visible on canvas

---

## References

- Steve Grand et al., *Creatures: Entertainment Software Agents with Artificial Life* (1998) — [ResearchGate](https://www.researchgate.net/publication/226997131)
- Alan Zucconi, *The AI of Creatures* (2020) — [alanzucconi.com](https://www.alanzucconi.com/2020/07/27/the-ai-of-creatures/)
- Creatures Wiki — [creatures.wiki/Biochemistry](https://creatures.wiki/Biochemistry)
- C1 Chemical List — [creatures.fandom.com/wiki/C1_Chemical_List](https://creatures.fandom.com/wiki/C1_Chemical_List)
- Original CAOS Development Guide — CyberLife Technology (1997)

## License

(MIT License)

Copyright (c) 2026 Harold Thetiot

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
