# FlySmith v1 — Normative Experiment Specification

**Status:** implementation specification. The v1 design choices in this document are frozen. There are no open biological I/O or learning-design questions required to start implementation.

FlySmith is a local experiment that places the `male-cns:v1.0` Drosophila connectome inside a short-horizon savings environment modeled on Bondsmith. No real money is used. Time advances only when the operator advances a simulated day.

The scientific question is narrow:

> Can a fixed MaleCNS connectome, augmented only with localized dopamine-gated mushroom-body plasticity, learn useful preferences over synthetic financial-state cues from realized consequences?

The experiment does **not** claim that a fly understands money, that connectome synapse counts are physiological weights, or that the resulting agent is a financial optimiser.

See [REFERENCES.md](REFERENCES.md) for the evidence behind the choices below.

---

## 1. Frozen v1 choices

| Component | v1 decision |
|---|---|
| Connectome | Janelia `male-cns:v1.0` |
| Neural dynamics | Whole-graph LIF, Shiu-style / Xenova-compatible external Poisson drive |
| Synthetic cue neurons | `KCg-m` + `KCg-d` gamma Kenyon cells |
| Cue construction | Deterministic sparse distributed 4-D code; nearest 5% of eligible gamma KCs |
| Cue drive | 100 Hz external Poisson drive for 500 ms |
| Readout | `MBON11` approach channel minus `MBON01` avoidance channel |
| Reward DAN | `PAM01` / PAM-gamma5 |
| Punishment DAN | `PPL101` / PPL1-gamma1pedc |
| Plastic edges | Existing gamma-KC -> `MBON01` and gamma-KC -> `MBON11` edges only |
| Plasticity | Compartment-specific dopamine-gated depression; topology unchanged |
| Delayed credit | Replay the original allocation cue when its consequence resolves |
| Choice | Evaluate each feasible option independently; choose highest normalized MB valence |
| Allocation size | Fixed by environment, not neurons: 10% of initial assets per tranche, capped by liquid cash/capacity |
| Clock | Manual simulated days only |
| Real money | Never |

The previous proposal to encode two options as left/right sensory populations and read left/right descending turn neurons is removed from v1. It introduced a motor-side convention unrelated to associative value and made learning harder to localize.

---

## 2. System boundary

```text
Local Bondsmith-compatible world
          |
          | WorldState
          v
Financial feature encoder
          |
          | x = [yield, lock, postLiquidity, postConcentration]
          v
Deterministic synthetic gamma-KC cue
          |
          | external Poisson rates[neuronCount]
          v
Whole MaleCNS LIF simulation
          |
          | per-neuron spike counts
          v
MBON valence decoder
          |
          | scalar candidate score
          v
Candidate selector
          |
          | HOLD or ALLOCATE(productId, tranche)
          v
Local savings world
          |
          | realized outcomes on later simulated days
          v
Conditioning replay + DAN teaching signal
          |
          v
Localized KC->MBON efficacy update
```

The fly never receives JSON, currency symbols, provider names, product IDs, or a hand-computed utility score. The financial layer never receives arbitrary spikes; it receives one decoded action.

---

## 3. MaleCNS identity and neuron registry

Pin the biological data source to **Janelia MaleCNS `male-cns:v1.0`**. The peer-reviewed 2026 Cell resource contains about 166.7k neurons spanning brain and VNC. Do not use neuron counts copied from third-party game demos as dataset truth.

All source data must retain the canonical neuPrint `bodyId`. Build a one-time registry mapping:

```ts
interface NeuronRef {
  bodyId: number;
  type: string;
  simulationIndex: number;
}
```

The simulator may use dense array indices internally, but experiment configuration and logs must always identify biological neurons by `bodyId` and `type` as well.

### 3.1 Cue population

Resolve all MaleCNS neurons of type:

```text
KCg-m
KCg-d
```

The MaleCNS Cell Type Explorer currently reports approximately 1,340 `KCg-m` and 206 `KCg-d` neurons. These gamma Kenyon cells are the synthetic conditioned-stimulus substrate.

Construct the **eligible cue pool** as gamma KCs that have at least one retained structural edge to **both** the MBON01 pair and the MBON11 pair. This guarantees that every cue neuron has an anatomical route into both opponent valence channels.

Initialization MUST fail if the resulting eligible pool contains fewer than 512 cells. Do not silently substitute another KC class.

### 3.2 Readout neurons

Use these exact MaleCNS cell types and body IDs:

```text
APPROACH / positive-action channel
MBON11 = MBON-gamma1pedc>alpha/beta
bodyIds: 10704, 11402

AVOIDANCE / negative-action channel
MBON01 = MBON-gamma5beta'2a
bodyIds: 520151, 10013
```

The decoder operates on both hemispheres together; there is no artificial left/right financial meaning.

### 3.3 Teaching neurons

Use:

```text
POSITIVE REINFORCEMENT
PAM01 = PAM-gamma5
resolve all PAM01 cells dynamically from male-cns:v1.0
(current MaleCNS explorer: 44 cells total)

NEGATIVE REINFORCEMENT
PPL101 = PPL1-gamma1pedc
bodyIds: 11327, 11900
```

Do not hard-code the PAM01 body-ID list. Resolve type membership from the pinned dataset and save the resolved registry/hash with each experiment.

Biological rationale: PAM-gamma5 innervates the gamma5 compartment containing MBON01; rewarding reinforcement can reduce conditioned KC drive to avoidance-output channels. PPL1-gamma1pedc provides aversive teaching in the gamma1 compartment and aversive reinforcement depresses KC->MBON11 drive. MBON11 is an approach-favoring output; MBON01 is an avoidance-favoring output.

---

## 4. Financial observation

Every feasible candidate is reduced to exactly four observable state variables in `[0,1]`:

```ts
interface OptionFeatures {
  yield: number;
  lock: number;
  postLiquidity: number;
  postConcentration: number;
}
```

Definitions:

```text
yield = effectiveDailyRate / configuredMaxDailyRate

lock = daysUntilFundsAccessible / configuredMaxLockDays

postLiquidity = liquidCashAfterThisAllocation / totalAssets

postConcentration = balanceInThisDestinationAfterAllocation / totalAssets
```

Clamp each value to `[0,1]`. All normalization bounds are fixed in experiment configuration before Day 0 and never recomputed from that day's available products.

If the local Bondsmith-compatible world exposes AER rather than a daily rate, the adapter converts it to an effective daily rate:

```text
effectiveDailyRate = (1 + AER)^(1/365) - 1
```

The short simulated term changes access/maturity timing; it does not require inventing a different interest formula unless the local world deliberately defines one.

### Cash / hold cue

`CASH` is a candidate state:

```text
yield             = hubDailyRate / configuredMaxDailyRate
lock              = 0
postLiquidity      = currentLiquidCash / totalAssets
postConcentration  = currentLiquidCash / totalAssets
```

For the initial synthetic world, `hubDailyRate = 0`.

### What is deliberately absent

V1 does not encode provider name, bank identity, FSCS status, marketing labels, previous choice, product rank, future product changes, or an externally calculated expected utility. If a variable is not in the four numbers above, the fly does not know it.

---

## 5. Financial vector -> synthetic neural cue

Finance has no natural fly sensory mapping. V1 therefore uses the mushroom body's known ability to associate arbitrary sparse KC ensembles with reinforcement, rather than pretending that rates are odors or that term length is taste.

### 5.1 Deterministic 4-D receptive fields

For every eligible gamma KC `i`, derive a deterministic pseudo-random preferred financial vector:

```text
mu_i = [u1, u2, u3, u4], each u in [0,1]
```

using a reproducible cryptographic hash of:

```text
"FlySmith:v1" + bodyId
```

The mapping MUST be independent of product IDs and experiment outcomes.

For candidate vector `x`, compute Euclidean distance:

```text
d_i = ||x - mu_i||_2
```

Select exactly the closest **5%** of eligible gamma KCs, rounding to the nearest whole neuron with a minimum of 1.

This creates a sparse distributed cue in which nearby financial states share more KCs than distant states. That gives the model a defined route for stimulus generalization without a learned encoder outside the fly.

### 5.2 External drive

During a candidate presentation:

```text
selected cue KCs: 100 Hz external Poisson drive
all other external inputs: 0 Hz
```

V1 does not encode feature magnitude in firing-rate amplitude. Feature values determine **which** KCs participate in the sparse pattern.

100 Hz is chosen because whole-brain Drosophila LIF work commonly probes sensory drive over roughly 10-200 Hz and explicitly treats absolute rates as model quantities rather than physiological reconstructions.

---

## 6. Candidate trial protocol

Evaluate candidates **one at a time**, never simultaneously.

For each candidate:

1. Reset fast neural state: membrane voltage, synaptic current, refractory state, delay buffers and spike counters.
2. Preserve all learned plastic multipliers.
3. Reset external drive to zero.
4. Run 100 ms with zero external cue drive.
5. Apply the candidate's 5% gamma-KC cue at 100 Hz for 500 ms.
6. Record per-neuron spikes.
7. Decode the MBON output over the final 400 ms of the cue window, excluding the first 100 ms as an onset/transmission transient.
8. Repeat for **5 independent Poisson seeds**.
9. Candidate output is the median of the 5 replicate valence scores.

Random seeds are deterministic:

```text
seed = hash(experimentSeed, day, candidateId, replicateIndex, phase)
```

If the underlying simulator cannot be seeded, deterministic RNG is an implementation prerequisite.

Resetting fast state between candidates prevents candidate ordering from becoming an unintended input. Learned synaptic efficacy is the only memory that persists between candidate trials.

---

## 7. MBON valence decoder

For each replicate, compute mean firing rate of each bilateral pair during the 400 ms read window:

```text
A = mean Hz of MBON11 [10704, 11402]
V = mean Hz of MBON01 [520151, 10013]
```

Do not subtract raw Hz directly. Different cell types can have different native gains.

### 7.1 Naive-brain calibration

Before any learning:

1. Generate 256 deterministic feature vectors uniformly over `[0,1]^4`.
2. Present them using the normal trial protocol with plastic multipliers fixed at `1.0`.
3. Measure the MBON11 and MBON01 distributions.
4. Freeze robust baseline statistics:

```text
mA = median(A)
sA = 1.4826 * MAD(A) + epsilon
mV = median(V)
sV = 1.4826 * MAD(V) + epsilon
```

Then:

```text
zA = (A - mA) / sA
zV = (V - mV) / sV
valenceScore = zA - zV
```

The calibration remains frozen after learning. Otherwise normalization would erase the memory-induced shift we are trying to measure.

### 7.2 Noise threshold

Use 64 deterministic cue states. For each state, independently compute two 5-replicate median scores. Define:

```text
theta = 95th percentile of abs(scoreSet1 - scoreSet2)
```

This is the empirically measured neural-choice noise band.

### 7.3 Daily selection

Evaluate `CASH` and every feasible product. Sort by candidate median valence score.

```text
if topScore - secondScore >= theta:
    choose top candidate
else:
    HOLD
```

If `CASH` is the confident winner, `HOLD`.

No pairwise tournament is used in v1.

---

## 8. Neural choice -> savings action

The neurons choose **destination only**.

At experiment start:

```text
baseTranche = 10% of initial total assets
```

For a product winner:

```text
amount = min(baseTranche, currentLiquidCash, productRemainingCapacity)
```

Exclude a product candidate before neural evaluation if `amount` cannot satisfy its minimum deposit or any other local-world feasibility rule.

Action type:

```ts
type FlyAction =
  | { type: "hold" }
  | { type: "allocate"; productId: string; amount: number };
```

V1 has no neural withdrawal action. Fixed/notice positions resolve according to the environment's own rules and maturities return to hub cash.

---

## 9. Teaching protocol

Learning occurs only from **realized consequences**. Product appearance, advertised rate changes and counterfactual missed opportunities are observations, not reinforcement.

### 9.1 Outcome ledger

Every allocation stores:

```ts
interface AllocationMemory {
  allocationId: string;
  day: number;
  productId: string;
  amount: number;
  originalFeatures: OptionFeatures;
  originalCueBodyIds: number[];
}
```

This exact cue can be replayed later.

### 9.2 Positive outcome

When interest is actually credited for an allocation, principal return contributes zero reward.

```text
positiveCashOutcome = interestCredited
```

Normalize the event relative to the experiment's expected maximum per-tranche return:

```text
positiveScale = baseTranche * configuredMaxDailyRate * configuredMaxLockDays
r = tanh(positiveCashOutcome / positiveScale)
```

`r` is in `(0,1]`.

### 9.3 Negative liquidity outcome

If the environment generates a liquidity requirement and available cash cannot meet it:

```text
shortfall = requiredCash - availableCash
```

Distribute the negative event over currently inaccessible allocations in proportion to their locked principal:

```text
blame_i = lockedPrincipal_i / totalLockedPrincipal
negativeScale = baseTranche
r_i = -tanh((shortfall * blame_i) / negativeScale)
```

Replay and punish each implicated allocation cue separately.

This is the fixed v1 delayed-credit rule. It is intentionally explicit and auditable; there is no hidden portfolio optimiser assigning counterfactual value.

### 9.4 Cue replay

When an outcome resolves, reconstruct the **original** 5% KC cue used at allocation time. Do not recalculate the cue from the product's current terms.

Conditioning trial:

```text
100 ms reset/zero-drive
500 ms original cue drive
500 ms teacher drive concurrently with the cue
```

Positive event:

```text
cue + PAM01
```

Negative event:

```text
cue + PPL101
```

Teacher external drive:

```text
teacherDriveHz = 100 * abs(r)
```

with a maximum of 100 Hz.

The teaching population must actually spike. Plasticity is gated by measured DAN activity, not merely by the environment's reward number.

---

## 10. Plasticity rule

Do not train the whole 166k-neuron graph.

### 10.1 Plastic edge set

Only existing structural edges in these two sets can change efficacy:

```text
eligible gamma KC -> MBON01 [520151, 10013]
eligible gamma KC -> MBON11 [10704, 11402]
```

No new edge is created. No edge is deleted. Structural connectome weight remains immutable.

For each plastic edge:

```text
w_effective = w_structural * p
p_initial = 1.0
p_min = 0.2
p_max = 1.0
```

### 10.2 Teacher-specific direction

During **positive** conditioning, PAM01 gates depression of active cue KC -> MBON01 (avoidance-channel) synapses.

During **negative** conditioning, PPL101 gates depression of active cue KC -> MBON11 (approach-channel) synapses.

There is no potentiation and no passive decay in v1. Reversal is achieved by learning in the opponent channel.

### 10.3 Local update

For each selected cue KC `i`, let:

```text
a_i = min(1, KC_spikes_i / median_spikes_of_selected_cue_KCs)
```

For the appropriate teacher population, calibrate `teacherReferenceHz` as the median measured DAN firing produced by a 100 Hz teacher-drive-only trial. During conditioning:

```text
d = min(1, measuredTeacherHz / teacherReferenceHz)
```

Then for every eligible structural edge from KC `i` to the teacher's target MBON pair:

```text
p_new = clamp(p_old - eta * a_i * d, 0.2, 1.0)
eta = 0.02
```

If the teacher population does not spike, `d = 0` and memory does not change.

### 10.4 Critical isolation rule

**Plasticity updates are disabled during all ordinary candidate/choice trials.**

Endogenous or accidentally evoked PAM/PPL activity is logged, but it cannot alter weights outside an explicit conditioning phase. This is required because existing MaleCNS game-learning experiments have observed teacher contamination: broad input can recruit DANs without providing a clean causal teaching event.

The DAN stimulus therefore has two roles during conditioning:

1. it participates in the simulated neural state as a real identified neuron population;
2. its measured spikes gate the explicit localized plasticity rule.

The model does not pretend that the base LIF implementation natively simulates dopamine-dependent biochemical plasticity.

---

## 11. Manual day loop

```text
DAY N
  |
  +-- snapshot local savings state
  +-- generate feasible CASH/product candidates
  +-- calculate each candidate's four features
  +-- generate deterministic 5% gamma-KC cue
  +-- run five neural replicates per candidate
  +-- decode MBON valence
  +-- choose winner or HOLD via theta
  +-- apply at most one allocation tranche
  +-- persist action + neural evidence
  `-- stop

OPERATOR ADVANCES DAY
  |
  +-- accrue interest
  +-- decrement access/maturity timers
  +-- process maturities and interest credits
  +-- process optional liquidity requirement
  +-- emit resolved positive/negative outcomes
  +-- replay each responsible historical cue
  +-- apply explicit PAM01/PPL101 conditioning
  +-- persist plasticity diff
  `-- expose DAY N+1
```

Nothing advances because wall-clock time passed.

---

## 12. Local Bondsmith boundary

Bondsmith's public developer site describes a REST Savings API, while Bondsmith's product model centers a hub account plus easy-access, notice and fixed-term deposits. FlySmith should copy those **domain semantics**, while the local experimental world is free to compress access periods to days.

Do not invent undocumented Bondsmith endpoint paths in neural code. Implement a narrow adapter:

```ts
interface SavingsWorld {
  snapshot(): Promise<WorldState>;
  allocate(productId: string, amount: number): Promise<ActionResult>;
  advanceDay(): Promise<DayResult>;
}
```

Everything downstream of `SavingsWorld` must run against both:

- an in-memory deterministic test world; and
- the local Bondsmith-compatible stack.

The neural experiment should not care which transport is underneath.

---

## 13. Validation gates — mandatory before closed-loop savings runs

The biggest lesson from existing whole-connectome game experiments is that changing weights is not evidence of learning. FlySmith MUST pass the following isolated tests first.

### Gate 1 — registry integrity

- all pinned body IDs exist in `male-cns:v1.0`;
- simulation index mapping is one-to-one;
- PAM01/PPL101/MBON01/MBON11 type membership matches the registry;
- eligible KC pool >= 512;
- every eligible KC has retained edges to both readout types.

### Gate 2 — cue integrity

For 100 random feature vectors:

- exactly 5% of eligible KCs receive external cue drive;
- same vector + same config yields identical body-ID set;
- similar feature vectors have greater cue overlap than distant vectors;
- no non-cue neuron gets external cue drive.

### Gate 3 — cue propagation

Across 20 Poisson seeds:

- candidate cues evoke measurable MBON01 and/or MBON11 activity above no-drive baseline;
- the full graph does not enter runaway global firing;
- output is not saturated at the firing ceiling.

### Gate 4 — teacher integrity

100 Hz teacher-only trials must produce reliable spikes in the intended DAN population and no numerical instability.

Cue-only PAM01/PPL101 activity is measured as a contamination diagnostic. It does not update weights because ordinary-trial plasticity is disabled.

### Gate 5 — positive conditioning unit test

Pick one fixed cue A. Measure its naive valence. Repeatedly pair cue A with PAM01 using the exact conditioning protocol. Re-test with learning disabled during test.

Expected result:

```text
valence_after > valence_before
```

### Gate 6 — negative conditioning unit test

Pick cue B. Pair it with PPL101.

Expected result:

```text
valence_after < valence_before
```

### Gate 7 — discrimination

Reward A and punish B with otherwise identical exposure counts.

Expected:

```text
score(A) - score(B) > theta
```

### Gate 8 — generalization

After conditioning A, create `nearA` and `farA` in feature space without conditioning them.

The absolute learned score transfer to `nearA` must exceed transfer to `farA` in the expected direction.

### Gate 9 — reversal

After A has been positively conditioned and B negatively conditioned, reverse the reinforcement schedules. The ordering must eventually reverse without resetting weights.

### Gate 10 — controls

Repeat conditioning with:

- plasticity frozen;
- teacher population silenced;
- shuffled mapping from cue KCs to plastic-edge multipliers while preserving edge-count/weight distributions.

The learned discrimination must disappear or materially weaken.

### Statistical acceptance rule

For Gates 5-9, use 20 independent experiment seeds. A gate passes only if:

- at least **18/20** seeds change in the predicted direction; and
- the median effect magnitude exceeds the precomputed neural noise threshold `theta` where applicable.

**Do not connect learning to the savings loop until Gates 1-7 pass.** Gates 8-10 may be developed immediately afterward but are required before making a learning claim.

---

## 14. Whole-experiment controls and benchmarks

Run identical predetermined market histories against:

1. MaleCNS + v1 plasticity;
2. same MaleCNS with plasticity frozen;
3. same MaleCNS after learned multipliers are reset to 1.0;
4. teacher-silent condition;
5. shuffled gamma-KC plastic-edge assignment control;
6. random destination policy;
7. CASH-only policy;
8. greedy highest-current-yield policy;
9. deterministic horizon-aware oracle that knows the future, used only as an upper-bound benchmark.

The oracle must never provide features or reinforcement to the fly.

Primary metrics:

```text
cumulative realized interest
liquidity shortfall / penalty
fraction of assets liquid
allocation concentration
allocation switching rate
MBON valence trajectory per cue
plastic multiplier distribution
learned cue discrimination
```

Scientific success does not require beating the greedy or oracle policies.

---

## 15. Logging and reproducibility

Every run must persist:

```text
experiment ID and git commit
MaleCNS dataset ID and source hashes
annotation/edge file hashes
bodyId <-> simulationIndex registry hash
LIF parameter set
RNG implementation and master seed
feature normalization bounds
KC receptive-field seed/version
eligible KC body IDs
cue body IDs for every candidate
all external drive vectors in sparse form
all candidate replicate spike counts for registered readouts
DAN activity during choices and conditioning
naive MBON calibration stats
noise threshold theta
decoded scores/actions
world snapshots and realized outcomes
all conditioning events
all plastic edge diffs/checkpoints
```

JSONL is the canonical event log. Human-readable reports are derived artifacts.

A checkpoint must include world state, RNG state, all plastic multipliers and experiment configuration. Fast neural state does not need to persist between manually separated candidate trials because the protocol deliberately resets it.

---

## 16. Implementation layout

Suggested modules:

```text
src/
  bondsmith/
    adapter.*
    types.*

  environment/
    world.*
    clock.*
    outcomes.*

  fly/
    connectome.*
    registry.*
    lif.*
    cues.*
    trial.*
    mbon-decoder.*
    plasticity.*
    conditioning.*

  experiment/
    calibration.*
    validation.*
    day-loop.*
    logging.*
    checkpoint.*

config/
  v1.json

tests/
  registry.*
  cues.*
  decoder.*
  conditioning.*
  controls.*
```

---

## 17. Build order

### M0 — data and simulator

- pin `male-cns:v1.0` data sources and hashes;
- load the full retained graph;
- reproduce the chosen LIF baseline;
- provide deterministic RNG and arbitrary per-neuron external Poisson drive;
- emit per-neuron spike counts.

**Exit:** deterministic stimulation replay works.

### M1 — biological registry

- resolve `KCg-m`, `KCg-d`, MBON01, MBON11, PAM01, PPL101;
- build bodyId/index map;
- build eligible gamma-KC intersection;
- run Gate 1.

**Exit:** registry is frozen and hashable.

### M2 — cue and readout assay

- implement 4-D feature normalization;
- deterministic KC receptive fields;
- 5% cue generation;
- 500 ms presentation protocol;
- MBON calibration and `theta`;
- run Gates 2-4.

**Exit:** arbitrary financial-state vectors produce stable, reproducible valence measurements.

### M3 — isolated learning

- add plastic multipliers only to selected gamma-KC->MBON edges;
- implement PAM01/PPL101 conditioning and measured-DAN gate;
- implement plasticity snapshots;
- run Gates 5-10.

**Exit:** associative conditioning works without any financial world.

### M4 — deterministic savings world

- implement cash, 1-10 day products, rates, capacity, maturity, interest credit and liquidity requirements;
- implement 10%-initial-assets tranche rule;
- implement outcome ledger and cue replay;
- run scripted market histories.

**Exit:** complete manual day loop works in memory.

### M5 — local Bondsmith adapter

- map the local stack's actual product/account/deposit operations into `SavingsWorld`;
- keep manual time in the experimental environment;
- verify no real-money endpoint/configuration can be reached.

**Exit:** the same M4 tests pass through the local adapter.

### M6 — experiment suite

- frozen/static/shuffled/teacher-silent controls;
- deterministic market corpus;
- lesions after training;
- reports and plots.

**Exit:** results are reproducible from config + seed + checkpoint.

---

## 18. Success and failure criteria

A successful FlySmith v1 demonstrates all of the following:

1. synthetic financial vectors produce reproducible sparse gamma-KC representations;
2. those representations propagate through the retained MaleCNS graph to the selected MBONs;
3. explicit PAM01/PPL101 reinforcement changes only the permitted existing KC->MBON efficacies;
4. conditioning changes subsequent MBON valence for the reinforced cue;
5. nearby unseen financial states generalize more than distant states;
6. opponent reinforcement can reverse learned preference;
7. frozen/teacher-silent/shuffled controls do not show the same effect;
8. learned valence changes the destination selected in the local savings environment.

V1 has failed scientifically if weights change but cue valence and action selection do not change reproducibly. In that case the failure is retained and reported; the decoder or validation threshold must not be retuned against the desired financial outcome.

The defensible result statement is:

> A MaleCNS-based associative agent learned preferences in a local short-horizon savings task under an explicitly specified gamma-KC/MBON dopamine-plasticity augmentation.

It is not:

> A fruit fly understands savings or optimizes a bank account.

---

## 19. Frozen-v1 rule

Implementation discoveries may reveal bugs, dataset mismatches or failed scientific gates. Those are reasons to fail a gate and version a v2 hypothesis, not reasons to silently change v1 until it works.

For v1, the neurons, cue code, timing, decoder, reward rule, plasticity rule, tranche rule and acceptance criteria above are fixed.