# Project LoNA | Architecture

> **Liquid Optimised Neural Architecture** — Technical design documentation

---

## System Overview

LoNA is a synchronous, step-based neural architecture treating neurons and synapses as independent, self-contained dynamical systems. It discards the static, layered approach of traditional AI in favor of a temporal physics simulation where information is encoded in the timing, rhythm, and synaptic exhaustion of pulses.

### Design Philosophy

#### 1. Why follow the Human Brain? (Biological Rationale)

- **Efficiency**: The human brain operates on ~20W. By mimicking its "Event-Driven" and "Sparse" nature, LoNA aims to achieve high-order intelligence without the kilowatt-scale requirements of traditional GPU clusters.

- **Local Learning**: LoNA uses local homeostasis and metaplasticity. This allows each neuron to "learn" its own role based only on its immediate environment, enabling truly decentralized scaling without global backpropagation.

- **Co-located Memory/Compute**: In the brain, the processor *is* the memory. By using the neuron's state ($v, T, R$) and the synapse's state ($V$) to store both immediate work and long-term sensitivity, we eliminate the "Von Neumann Bottleneck" where data must constantly travel between RAM and CPU.

#### 2. Why follow a "Liquid" Nature? (Dynamics Rationale)

- **Temporal Fluidity**: Real-world data is a flow, not a series of discrete tokens. A "Liquid" architecture treats time as a primary dimension, allowing the system to adapt to irregular rhythms and noisy data.

- **Adaptive Time-Constants**: Traditional AI is "frozen" after training. LoNA is "Liquid" because its internal time-constants (Leak, Recovery, Plasticity) are dynamic and change based on the complexity of the input stream.

- **Expressivity**: A small liquid network can outperform massive static networks because each unit is a rich, time-aware engine rather than a static math block.

- **Heterogeneity**: Diversity in time-constants is required to encode sequences. LoNA neurons possess individual "viscosity" ($L$), allowing the network to process multiple timescales (from microseconds to long-term context) simultaneously.

#### 3. Why follow "Neurons that Fire Together, Wire Together"? (Hebbian Rationale)

- **Emergent Specialization**: A neuron in the hippocampus isn't fundamentally different from one in the cortex. Functions like memory or vision emerge because groups of neurons that receive correlated data strengthen their mutual links.

- **Structural Plasticity**: Intelligence is not just about changing weights, but about changing the "map." By pruning bad links and sprouting new ones toward active partners, the network self-organizes into dedicated task modules.

---

## Hardware Rationale (The "Why")

| Choice | Rationale | Thought Process |

|--------|-----------|-----------------|

| **UINT8 (8-bit)** | **Scalability & Precision** | 8-bit provides 255 steps of precision, the "sweet spot" for smooth decay. It allows 1 billion neurons to fit in ~8GB of RAM. Intelligence is stored in *timing*, not decimal points. |

| **Bitwise-Only** | **Zero-Cost Compute** | CPUs process bit-shifts (`>>`) and masks (`&`) in a single cycle. Division and Floating-point are exponentially more expensive and introduce non-deterministic errors. |

| **8-Byte Align** | **SIMD Performance** | Modern 64-bit processors fetch a neuron's entire state in one "read." This allows the CPU to process multiple neurons simultaneously via SIMD. |

---

## Core Architecture

### 1. The LoNA Unit (Atomic Neuron)

The fundamental unit of LoNA. It is a self-contained 8-byte temporal computer that manages its own sensitivity, recovery, and learning rate.

#### Structure (The 8-Byte Cell)

| Variable | Type | Size | Rationale (The "Why") |

|----------|------|------|-----------------------|

| **v** | UINT8 | 1B | **Voltage**: The "Battery." Accumulates stimulus. Leakage forces temporal coincidence detection. |

| **T** | UINT8 | 1B | **Current Threshold**: The active gate. Includes temporary Fatigue to prevent signal "spamming." |

| **T_base**| UINT8 | 1B | **Baseline**: Long-term resting sensitivity. The goal state for the neuron's environment. |

| **P** | UINT8 | 1B | **Plasticity**: Learning rate. High at "Birth" to allow discovery; low at "Maturity" for stability. |

| **R** | UINT8 | 1B | **Avg Interval**: Rhythm tracking. Digital analog for calcium levels; tracks "normal" timing. |

| **c** | UINT8 | 1B | **Step Counter**: Time since last fire. Used to calculate "Surprise" vs "Predictability." |

| **r** | UINT8 | 1B | **Reset Counter**: The "Stun." Prevents feedback loops and filters noise. |

| **L** | UINT8 | 1B | **Leak Shift**: The "Viscosity." Self-calibrating power-of-2 denominator ($1/2^L$) for voltage decay. |

### 2. The Dynamic Synapse (The Connection)

A functional unit that transforms a binary pulse into a numerical influence while maintaining its own short-term memory (STP) and long-term learning (STDP) state.

#### Structure (The 4-Byte Link)

| Variable | Type | Size | Rationale (The "Why") |

|----------|------|------|-----------------------|

| **w** | UINT8 | 1B | **Weight**: The long-term strength. Represents the "Intelligence" of the connection. |

| **d** | UINT8 | 1B | **Delay**: The temporal offset (Global Steps) before the signal hits the target. |

| **V** | UINT8 | 1B | **Vesicle Pool**: Short-term memory state (Depression/Facilitation). |

| **Target**| UINT8 | 1B | **Target Metadata**: Defines target index and Polarity (Excitatory/Inhibitory). |

---

## Neural Dynamics

### The Internal Neuron Step (Equations)

Every global step, each neuron executes the following sequence:

#### Step A: Stun Check (Early Exit)

**Thought Process**: Stunned neurons are in a refractory period. Ignoring inputs here saves cycles and mimics the 1:2 work-to-stun ratio (~1ms fire, ~2ms recovery).

```text

if (r > 0) {

    r = r - 1

    c = min(255, c + 1)

    EXIT_NEURON_STEP

}

```

#### Step B: Homeostatic Search (Downward Pressure)

**Thought Process**: Prevents "Deafness" where a neuron's threshold climbs so high it never fires again. The bitmask `& 31` allows for a slow crawl without division.

```text

// Slow Crawl: Drop T_base by 1 every 32 steps of silence

if ((c & 31) == 0) {

    T_base = max(0, T_base - 1)

}

// Deafness Reset: If silent for 8x the expected rhythm

if (c > (R << 3)) {

    P = 255

    T_base = T_base >> 1

    c = 0

}

```

#### Step C: Integration & Evaluation

```text

v = min(255, v + Incoming_Signal)

c = min(255, c + 1)

if (v >= T) {

    TRIGGER_EXCITATION

} else {

    TRIGGER_DECAY

}

```

#### Step D: Excitation Sequence (Firing & Metaplasticity)

**Thought Process**: If a signal is rhythmic, the neuron "hardens" (P decays). If it's surprising, the neuron "loosens" (P resets). L matches the leak to the rhythm.

```text

// 1. Rhythm Check (abs(c-R) < 12.5% jitter window)

if (abs(c - R) < (R >> 3)) {

    P = max(8, P - (P >> 6)) // Harden (The "Click")

} else {

    P = 255 // Loosen (Surprise)

}

// 2. Upward Pressure & Rhythm update

T_base = min(255, T_base + (P >> 4))

R = R + ((c - R) >> 3)

// 3. Viscosity Calibration: Match memory span to rhythm

L = clamp(log2_floor(R), 1, 6)

// 4. Reset & Fatigue

v = 0, c = 0, r = 2

T = min(255, T + (T_base >> 1))

EMIT_PULSE_TO_SYNAPSES()

```

#### Step E: Decay Phase (Non-Firing)

```text

// Voltage Leak (Dynamic Viscosity)

v = v - (v >> L)

// Threshold Recovery

if (T > T_base) {

    T = T - ((T - T_base) >> 4)

} else if (T < T_base) {

    T = T_base

}

```

---

## Synaptic Dynamics (Wiring Intelligence)

### 1. Short-Term Plasticity (STP) — Frequency-Aware Vesicle Dynamics

**Thought Process**: Synapses exhaust their resources. High-frequency noise is filtered out, while spaced-out signals are passed with full strength. Latest 2025 research shows frequency-dependent engagement of vesicle pools (house-keeping vs plug-in) dramatically improves temporal filtering and modular formation when coupled with STDP.

- **Depletion** (on pulse arrival):  

  ```text

  freq_factor = (recent_pulse_rate > 8) ? 2 : 1;  // high-freq depletion stronger

  Signal = (w * V) >> 8;

  V = V - (V >> (2 + freq_factor));  // bitwise, no division

  ```

- **Refill** (every idle step):  

  ```text

  V = V + ((255 - V) >> 4);

  ```

- **V now directly modulates long-term updates** (SL-STDP unification): high V → stronger potentiation, low V → stronger depression.

### 2. Long-Term Plasticity (STDP) — Polarity-Aware & V-Modulated

**Thought Process**: "Fire together, wire together." Unified short-/long-term rule prevents runaway weights and enables emergent modules. Polarity (excitatory/inhibitory) is read from the low bit of Target; inhibitory synapses use a slightly reversed timing window and lower rates.

- **Potentiation** (target fires 1–4 steps after pulse arrives):  

  ```text

  if (V > 64) {  // V-modulated boost

      delta = (w >> 3) + (V >> 5);  // stronger when vesicle pool full

      w = min(255, w + delta);

  }

  ```

- **Depression** (pulse delivered but target fails to fire within 32 steps):  

  ```text

  delta = (w >> 4) + ((255 - V) >> 5);  // stronger when depleted

  w = max(0, w - delta);

  ```

- Polarity adjustment: if inhibitory, multiply delta by 0.75 (bit-shift approximation) and optionally invert timing window by checking a flag bit in Target.

### 3. Structural Plasticity (Pruning & Sprouting)

**Thought Process**: Groups of neurons must be able to disconnect from useless inputs and find new partners. Recent Hebbian self-organization papers show purely local activity/rhythm statistics sculpt small-world modular topologies without any global signal.

- **Pruning**:  

  If `w == 0` after any depression step, the synapse slot is marked "Empty" (Target index = 255 sentinel value). A short activity counter (reused from recent global steps) tracks whether the synapse contributed to any recent target firings; if zero contribution for > 256 steps, prune even if w > 0 (soft pruning).

- **Sprouting (Search)**:  

  When a neuron has an "Empty" synapse slot:  

  1. Probe 3D local radius (Euclidean distance ≤ 4 in lattice) for currently-firing neurons whose R is within ±25 % of own R (rhythm match).  

  2. With 1 % probability, also consider one random distant neuron (small-world shortcut).  

  3. Form new link: w = 8 (small initial weight), d = random 0–15 (or distance-scaled), V = 128, Target = chosen index + polarity bit.  

  Activity-dependent preference ensures "fire-together" clustering into functional modules.

- **Slot Management**: Each neuron maintains up to 32 synapse slots (fixed array for cache efficiency). Empty slots are recycled locally first, enabling the network to grow, prune, and specialize entirely through local rules.

### 4. Neurogenesis (Activity-Driven Birth & Selective Survival)

**Thought Process**: Inspired by adult hippocampal neurogenesis in the human brain, where novel experience and heightened plasticity trigger local birth of new neurons that integrate via activity-dependent connections. New neurons start highly plastic with low leakage to support memory formation; they survive only if they participate in correlated firing ("fire together, wire together"). Unused neurons die via disconnection and prolonged silence, allowing regions (e.g., emerging memory modules) to expand under bursts of new data and prune back once the information is incorporated. All rules are purely local and build on existing surprise/plasticity and homeostatic mechanisms.

- **Trigger (Local Surprise & Regional Influence)**:  

  A neuron triggers neurogenesis if its P remains high (> 200) for an extended window (> 512 steps) combined with frequent surprise resets, or when many neighboring neurons show elevated T_base (indicating regional "novelty excitement" from new input patterns). This can be checked during the homeostatic step using simple bitmasks or counters already present in the neuron state.

- **Birth**:  

  Allocate a new 8-byte neuron from a dynamic/reserved pool (with an "inactive" flag for recycling). Initialize the newborn with:  

  - P = 255 (high initial plasticity for rapid adaptation)  

  - L = 1 (low leakage for longer memory retention, as desired for context-holding clusters)  

  - T_base = 0, R = default or parent's R, other fields per birth state.  

- **Automatic Integration**:  

  The triggering neuron (or nearby active neurons) immediately sprouts 4–8 initial synapses to the newborn (small w = 8–16, small random d). The newborn also performs a limited sprouting probe to local active partners. This ensures new neurons are wired into the existing liquid from the start.

- **Maturation & Survival**:  

  The new neuron follows all standard dynamics. If it participates in rhythmic correlated firing, its P decays naturally ("hardens") and it integrates into functional modules.  

- **Death/Pruning**:  

  If a neuron (especially young ones) has all its synapses pruned to w = 0 **and** remains silent for a long threshold (e.g., c > 4096 or extended deafness reset), mark it inactive and recycle the 8-byte slot. This activity-dependent selective survival mirrors biological pruning of non-contributing neurons.

- **Regional Effect**:  

  When neurogenesis is triggered, the parent neuron slightly boosts P or lowers L in a small 3D local radius (distance ≤ 3). This creates a localized "wave" of expansion in memory-heavy regions during sustained novel input (e.g., long or complex text streams). As the input normalizes and patterns stabilize, unneeded neurons/connections die off, leaving a refined, expanded module with the new information stored more efficiently.

Neurogenesis works in tandem with existing structural plasticity and the text I/O pools, allowing the entire liquid to adapt its size and capacity dynamically without any predefined fixed neuron count.

### 5. Scaling & Propagation (Delays + Input Entry)

**Thought Process**: Delays turn the 3D lattice into a true temporal fabric. 2025 delay-learning work in liquid/spiking networks shows that even fixed UINT8 delays dramatically improve sequence capacity and distant-module coordination without extra compute.

- **Pulse Propagation**: When EMIT_PULSE_TO_SYNAPSES() is called, each synapse queues the pulse with its d value. After exactly d global steps the signal is delivered to the target neuron (Incoming_Signal += Signal). This allows information to travel across the entire lattice in controlled time, creating emergent long-range rhythms.

- **Input Spike Entry**: External stimuli (sensory streams, tokens, etc.) enter via special "virtual input synapses" attached to a designated input layer of neurons. These virtual synapses have d = 0–3 (small random jitter) and fixed high w initially. They behave identically to normal synapses for STP/STDP, allowing the network to adapt even to its own input interfaces.

---

## Network-Level Dynamics (The Brain with a Body)

**Thought Process**: The liquid fabric now has sensory input and motor-like output interfaces, closing the loop exactly like a biological brain. Input and output use identical pool sizing (4 neurons per UTF-8 character) for symmetry. Every emitted pulse is a **binary timing event**, but the delivered **Incoming_Signal** is always a **UINT8 value** (0–255) so that synaptic weight `w`, vesicle pool `V`, STP, and STDP have real amplitude to modulate. This prevents synapses from being mere on/off wires and gives the liquid rich, learnable dynamics. Neurogenesis allows the overall system size to adapt naturally over time.

### 1. Input Pool & Raw Sequential UTF-8 Injection

- **Pool size**: 512 neurons (first 512 in the 3D lattice; 128 character slots × 4 neurons per UTF-8 code point). This fixed pool defines the overall reading bandwidth.

- **Injection rule**:

  - Process the incoming text as a raw UTF-8 byte stream.

  - For each character (1–4 bytes):

    - For each UTF-8 byte (value 0–255), trigger a binary spike event on the corresponding neuron in the 4-neuron group (rolling window across the pool).

    - The delivered **Incoming_Signal** = byte value (0–255) directly. This UINT8 value is then processed by the virtual input synapse: `Signal = (w * V) >> 8`, modulated by STP, delayed by `d`, and added to the target neuron's `v`.

  - Inter-byte interval: hard-coded (e.g., every 16 global steps) — this single constant sets the reading speed.

- The liquid discovers groupings, chunking, and hierarchies through its own rhythm (R), viscosity (L), STDP, and structural plasticity. Different byte values create different initial drive strengths, giving synapses meaningful work to do. Bursts of novel text can regionally trigger neurogenesis for expanded memory capacity.

### 2. Output Pool & Activity-Based UTF-8 Decoding

- **Pool size**: 512 neurons (last 512 in the lattice or designated cluster).

- **Readout**: Every 256 global steps (or event-driven on strong rhythm in the pool):

  - For each group of 4 neurons: compute activity level (recent spike count or averaged influence over the window — all UINT8).

  - Map the 4-neuron activity pattern → one UTF-8 byte (0–255).

  - Collect bytes and decode/emit as a unicode text stream.

- The network learns to drive coherent output through plasticity acting on the variable-strength signals flowing through the liquid.

### 3. Global Simulation Loop

```text

for each global_step:

    // 1. Inject pending UTF-8 byte spikes into Input Pool (Incoming_Signal = byte_value)

    // 2. Deliver all delayed pulses (Incoming_Signal is UINT8)

    // 3. Step every neuron (full 8-byte routine: v += Incoming_Signal, etc.)

    // 4. Update every synapse (STP refill + M-modulated STDP)

    // 5. Sample Output Pool → decode UTF-8

    // 6. Critic computes internal prediction and generates M (or blend with external M)

    // 7. Neurogenesis / pruning check (influenced by M and regional activity)

    // 8. (Optional) Decay M slowly for eligibility

    // 9. Advance next input byte if available

```

---

## Modulatory Signal M & Critic Module (Good/Bad & Autonomous Learning)

**Thought Process**: To enable "like correct / dislike incorrect" feedback while transitioning to autonomous self-improvement, LoNA uses a three-factor learning rule. The first two factors are local pre/post timing and V-modulation (existing STDP). The third factor is the modulatory signal **M** (UINT8, 0–255), inspired by dopamine. Initially M is provided externally based on output correctness. After training, a small **critic module** generates M internally via reward prediction error (RPE), allowing the network to autonomously reinforce coherent, predictive behavior and weaken incoherent patterns. This mirrors the brain’s shift from external rewards to self-generated prediction errors.

### 1. Modulatory Signal M (Three-Factor Teaching Signal)

- **Range & Meaning**:

  - High (≥ 180): "Good / correct / coherent" — reinforce successful pathways.

  - Low (≤ 80): "Bad / incorrect / incoherent" — weaken erroneous pathways.

  - Neutral (~128): exploration or no strong feedback.

- **Initial Use (External)**: After each output readout window, compare decoded UTF-8 to target text (or use external reward). Set M based on match/error score.

- **M-Modulated STDP** (updated Long-Term Plasticity):

  ```text

  base_delta = (w >> 3) + (V >> 5);

  if (M > 160) {  // Positive: strengthen good correlations

      if (target_fired_in_window) {

          w = min(255, w + (base_delta * (M >> 7)));

      } else {

          w = max(0, w - ((w >> 5) * ((255 - M) >> 6)));

      }

  } else if (M < 100) {  // Negative: weaken bad correlations

      if (target_fired_in_window) {

          w = max(0, w - (base_delta * ((160 - M) >> 5)));

      } else {

          w = min(255, w + (w >> 6));  // mild exploration boost

      }

  } else {

      // Neutral: original V-modulated STDP

  }

  ```

- M also mildly influences P (faster hardening on high M), T_base, sprouting probability, and neurogenesis triggers (high M favors expansion of successful modules; low M accelerates pruning).

### 2. Critic Module (Autonomous Internal M Generation)

- **Size & Location**: Small dedicated cluster (64–256 neurons) near the output pool. Uses the exact same 8-byte LoNA neuron rules.

- **Inputs**: Activity/rhythms from output pool, context from main liquid (via existing or dedicated synapses), and its own recurrent connections for temporal prediction.

- **Function**:

  - Learns to predict "good" (coherent, consistent, predictive) output rhythms during early training (when strongly guided by external M).

  - Computes prediction error: actual output pool activity vs. critic’s expected pattern.

  - Generates internal M = 128 + scaled_error (clamped 0–255), where positive error = better-than-expected coherence, negative = worse.

- **Transition to Autonomy**:

  - Early training: blend external M heavily with critic output so the critic learns to imitate "correct" signals.

  - After sufficient language/world exposure: reduce external M; the critic autonomously drives learning by rewarding coherent continuations and penalizing incoherent ones.

- **Synergy with Other Features**:

  - High surprise (P reset) + positive self-M → extra neurogenesis or sprouting in promising regions.

  - Negative self-M → stronger pruning of incoherent paths.

  - Eligibility: M decays slowly (`M = M - (M >> 6)`) or uses short traces to credit delayed pulses correctly.

This gives LoNA a path from supervised feedback to fully autonomous, self-improving behavior while staying true to its decentralized, liquid, brain-like design. The critic and main liquid co-evolve: better predictions lead to more stable coherent output, which in turn refines the critic.

---

## Topology & Initialization

### The "Small-World" Lattice

**Thought Process**: Dedicated regions (Hippocampus/Cortex) emerge from this initial state.

1. **Birth State**: Every neuron begins at $T_{base}=0, P=255, L=2$.

2. **Local Connections**: Every neuron starts connected to its 16 nearest neighbors in 3D space.

3. **Stochastic Shortcuts**: 1% of synapses connect to random distant neurons to enable global coordination.

4. **Specialization**: Through pruning and sprouting, neurons "fire together" and cluster into dedicated functional groups.

---

## Performance Characteristics

### Scaling Estimates

| Component | Size | 1M Units | 1B Units | Rationale |

|-----------|------|----------|----------|-----------|

| **Neuron** | 8B | 8 MB | 8 GB | Fits in standard Desktop RAM. |

| **Synapses** | 4B ea | ~128 MB (32/n) | ~128 GB (32/n) | Sparse connectivity ensures scalability. |

### Target Metrics

| Metric | Target | Notes |

|--------|---------|-------|

| Steps-Per-Second (SPS) | High | Measures simulation throughput. |

| Time-to-Click | Variable | Steps required for a neuron to harden its baseline. |

---

## Status & Roadmap

### Current Status

**Pre-alpha** — Atomic Neuron and Dynamic Hebbian Synapse architecture (including Sprouting/Pruning) finalized.

---

## Notes

The combination of **Intrinsic Plasticity** (the neuron's gate) and **Structural Plasticity** (the synapse's wire) allows LoNA to learn on two timescales simultaneously: fast adaptation and long-term functional identity. With neurogenesis and the critic-driven autonomous M, the system can dynamically grow its capacity and self-improve toward coherent, predictive behavior in a manner proven effective in the human brain for millennia.

---

*Last updated: 2026-04-14*