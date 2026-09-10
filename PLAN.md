# FlySmith plan

## Goal

FlySmith will connect a local Bondsmith-compatible savings environment to a simulated MaleCNS fly connectome and use the fly's neural activity to choose between savings actions.

The experiment is deliberately closed-loop and local:

1. The local Bondsmith state defines the options available on a simulated day.
2. FlySmith converts those options into stimulation of selected MaleCNS input neurons.
3. The fly simulation propagates that activity through the connectome.
4. FlySmith reads a small, predefined set of descending-neuron outputs.
5. A deterministic decoder converts those outputs into one local savings action.
6. The user manually advances the simulated day.
7. When outcomes resolve, a teaching signal can modify a restricted set of synaptic efficacies inside the simulated mushroom-body circuit.

No real money is involved.

---

## Design principles

- **Explicit neural I/O.** Every financial feature must resolve to a documented set of neuron IDs and an exact stimulation rule.
- **Minimal decoder.** The fly should choose; ordinary code should only translate neural output into a valid environment action.
- **Fixed environment semantics.** Bondsmith provides the account/product vocabulary and local API surface. FlySmith controls the experimental clock.
- **Reproducible trials.** Identical environment state + random seed + neural state must reproduce a trial.
- **Separate choice from learning.** A choice trial should not silently mutate weights. Learning occurs only during an explicit teaching phase.
- **Keep most of MaleCNS fixed.** V1 learning changes only a biologically motivated subset of existing synapses, not the topology of the connectome.
- **Treat biological mappings as hypotheses.** Input populations, output populations, and plasticity rules must be versioned and testable.

---

# 1. System boundary

```text
local Bondsmith-compatible API
          |
          v
     EnvironmentState
          |
          v
        encoder
          |
          v
  external neural drive
          |
          v
   MaleCNS simulation
          |
          v
    neural readouts
          |
          v
        decoder
          |
          v
       FlyAction
          |
          v
 local Bondsmith action
```

The fly model never receives JSON, pounds, product names, or API objects. It receives only stimulation of selected neurons.

The Bondsmith layer never receives spikes. It receives only a small action object emitted by the decoder.

---

# 2. Environment state

A simulated day should expose only the information available to the fly at decision time.

Initial state shape:

```ts
interface EnvironmentState {
  day: number;
  cash: number;
  totalAssets: number;
  products: ProductState[];
}

interface ProductState {
  id: string;
  rate: number;
  termDays: number;
  balance: number;
  available: boolean;
}
```

The first version should use short products measured in days, not realistic month/year durations.

The environment owns:

- cash balances
- product balances
- term countdowns
- maturities
- interest/reward accrual
- product availability
- scheduled rate changes
- optional liquidity requirements
- manual `advance day`

The neural model owns none of these rules.

---

# 3. Decision format: pairwise choice

V1 should avoid asking the network to choose directly among an arbitrary number of products.

Instead, every neural trial compares exactly two alternatives:

```text
LEFT OPTION  vs  RIGHT OPTION
```

Cash is itself a valid option.

For N available alternatives, FlySmith can run pairwise trials and aggregate the results into a daily winner. The aggregation rule must be deterministic and logged.

Example action space:

```ts
type FlyAction =
  | { type: "hold" }
  | { type: "allocate"; productId: string; amount: number };
```

Allocation amount should initially be external and fixed, for example a configurable tranche or fixed percentage of available cash. Neural activity should choose the destination before it is allowed to control position size.

---

# 4. Financial observation -> neural input

## 4.1 Initial observable features

Each option is represented by a small numerical feature vector.

V1:

```text
rate
term
post-action liquidity
post-action exposure
```

These are observations, not precomputed utility scores.

For example, `term = 1.0` means "at the high end of the configured term range". It does not mean good or bad.

Each feature is normalized using experiment-wide bounds that are fixed before a run:

```ts
interface OptionFeatures {
  rate: number;       // 0..1
  term: number;       // 0..1
  liquidity: number;  // 0..1
  exposure: number;   // 0..1
}
```

## 4.2 Neural codebook

Each feature gets a fixed bilateral input codebook:

```text
RATE_L        RATE_R
TERM_L        TERM_R
LIQUIDITY_L   LIQUIDITY_R
EXPOSURE_L    EXPOSURE_R
```

Each entry is a set of resolved MaleCNS simulation indices.

```ts
interface NeuralPopulation {
  bodyIds: number[];
  simulationIndices: number[];
}

interface InputCodebook {
  rate:      { left: NeuralPopulation; right: NeuralPopulation };
  term:      { left: NeuralPopulation; right: NeuralPopulation };
  liquidity: { left: NeuralPopulation; right: NeuralPopulation };
  exposure:  { left: NeuralPopulation; right: NeuralPopulation };
}
```

The exact biological cell types are **not yet fixed**. Selecting and validating these populations is a prerequisite for claiming that the experiment uses meaningful sensory pathways rather than arbitrary neuron buckets.

Requirements for candidate input populations:

- clear left/right homologues
- upstream of the chosen decision/output pathway
- large enough for robust stimulation
- not themselves part of the motor output decoder
- stable annotation/body-ID resolution in MaleCNS
- no overlap between feature channels unless intentionally designed

A likely direction is to use projection/sensory populations capable of driving mushroom-body and downstream decision circuitry, but this must be verified from connectivity rather than assumed.

## 4.3 Numeric value -> stimulation

The encoder produces the full external-drive vector expected by the simulator.

Conceptually:

```ts
externalHz = new Float32Array(NEURON_COUNT);
```

All neurons default to zero external drive.

For a simple rate code:

```ts
hz = baselineHz + featureValue * featureRangeHz;
```

Every simulation index belonging to that feature/side receives that external drive for the trial.

Initial calibration values such as `0..200 Hz` should be configuration, not constants baked into the experiment. The range must be calibrated so that inputs propagate without simply saturating the network.

Population coding can replace scalar rate coding later if generalisation across unseen rates/terms is poor.

---

# 5. Passing the input through MaleCNS

A choice trial is:

```text
1. reset or restore the agreed pre-trial neural state
2. encode LEFT features into LEFT neural populations
3. encode RIGHT features into RIGHT neural populations
4. run the LIF/connectome simulation for a fixed duration
5. collect spike counts for every neuron
6. extract only the registered output populations
```

The simulation duration is configurable. A starting value such as 500-1000 ms can be tested, but it must be chosen from calibration data rather than treated as biologically meaningful by default.

Every trial log should include:

```ts
interface TrialRecord {
  experimentId: string;
  day: number;
  seed: number;
  leftOptionId: string;
  rightOptionId: string;
  leftFeatures: OptionFeatures;
  rightFeatures: OptionFeatures;
  stimulusConfigVersion: string;
  neuralConfigVersion: string;
  plasticityStateVersion?: string;
  durationMs: number;
  output: NeuralDecisionOutput;
}
```

---

# 6. Neural output -> choice

V1 should use a native bilateral motor readout rather than assigning financial semantics to an unrelated neuron such as P1.

Initial candidate output channels:

```text
TURN_LEFT
TURN_RIGHT
```

Candidate descending-neuron types include the left/right turning populations used by existing MaleCNS demonstration simulators, but FlySmith must resolve and verify the exact body IDs and simulation indices before freezing the codebook.

For each population:

```ts
populationHz = totalSpikes / neuronCount / trialSeconds;
```

Then:

```ts
preference = leftHz - rightHz;
```

Decoder:

```ts
if (preference > threshold) return LEFT;
if (preference < -threshold) return RIGHT;
return NO_PREFERENCE;
```

The threshold is a calibrated experimental parameter.

```ts
interface NeuralDecisionOutput {
  leftHz: number;
  rightHz: number;
  preferenceHz: number;
  winner: "left" | "right" | "none";
}
```

The raw population rates and preference must always be retained; never store only the decoded winner.

---

# 7. Pairwise choice -> local Bondsmith action

Given alternatives:

```text
cash
product A
product B
product C
```

run all configured pairwise trials. Each produces `left`, `right`, or `none`.

V1 aggregation can be a simple tournament/Copeland score:

```text
win  = +1
loss =  0
tie  = +0.5
```

Highest score wins. Ties use a deterministic rule defined in configuration.

Translation:

```text
cash wins       -> HOLD
product X wins  -> ALLOCATE fixed tranche to X
```

The decoder does not decide whether an API call is financially clever. It only translates the fly's winner into an action supported by the local environment.

---

# 8. Manual day loop

```text
DAY N
  |
  +-- query local Bondsmith state
  |
  +-- derive currently valid alternatives
  |
  +-- build feature vectors
  |
  +-- run neural choice trials
  |
  +-- decode daily winner
  |
  +-- apply one local action
  |
  +-- log everything
  |
  `-- stop

USER: advance day
  |
  +-- accrue daily reward / interest
  +-- decrement terms
  +-- process maturities
  +-- apply scheduled environment changes
  +-- resolve any outcomes eligible for teaching
  `-- DAY N+1
```

Nothing should advance on wall-clock time.

---

# 9. Teaching the fly

## 9.1 Scope

Do **not** train all MaleCNS weights.

V1 learning keeps:

- neuron topology fixed
- most synaptic efficacies fixed
- decoder fixed
- financial encoder fixed during a run

Only a restricted, biologically motivated set of mushroom-body synapses is plastic.

Primary candidate:

```text
Kenyon cell (KC) -> mushroom body output neuron (MBON)
```

The exact set must be derived from MaleCNS annotations/connectivity and versioned.

## 9.2 Weight representation

Preserve the structural connectome weight separately from learned efficacy:

```ts
interface PlasticSynapse {
  structuralWeight: number;
  multiplier: number;
}

effectiveWeight = structuralWeight * multiplier;
```

Learning changes `multiplier`, not topology or structural synapse count.

This distinction is required so that a learned fly can always be reset to the original connectome.

## 9.3 Teaching signal

When the consequence of a previous action resolves, the environment emits a scalar outcome:

```ts
reward: number // normalized e.g. -1..+1
```

That outcome is translated into stimulation of registered dopamine-neuron populations.

Conceptual codebook:

```text
positive reinforcement -> reward-associated DAN population
negative reinforcement -> punishment-associated DAN population
```

PAM/PPL1 systems are candidates, but the exact compartments and direction of plasticity must be selected from a specific published learning model before implementation.

## 9.4 V1 credit assignment

Short terms make delayed credit assignment manageable.

Use **cue replay** first rather than inventing a multi-day eligibility trace:

```text
choice on day N
    |
    v
outcome resolves on day N+k
    |
    +-- reconstruct/replay the chosen option's neural cue
    +-- stimulate the appropriate DAN teaching population
    `-- apply the plasticity update to eligible KC->MBON synapses
```

This creates an explicit conditioning event and keeps the first plasticity implementation inspectable.

Later, replace or compare cue replay with a decaying eligibility trace:

```text
eligibility(t) = eligibility(0) * exp(-t / tau)
```

and dopamine-gated weight updates.

## 9.5 Plasticity rule

Do not invent the final sign/timing rule ad hoc.

Implementation sequence:

1. pick one published compartment-level Drosophila KC->MBON dopamine-gated learning rule
2. map its required KC, MBON and DAN populations onto MaleCNS IDs
3. reproduce a simple conditioning sanity test independent of finance
4. only then expose the rule to Bondsmith outcomes

The code should support a generic interface:

```ts
interface PlasticityRule {
  onDecisionActivity(activity: NeuralActivity): void;
  onTeachingSignal(signal: TeachingSignal): WeightUpdate[];
  reset(): void;
  snapshot(): PlasticitySnapshot;
}
```

---

# 10. Bondsmith adapter

Keep Bondsmith-specific transport outside the neural code.

```text
src/
  bondsmith/
    client.*
    mapper.*

  environment/
    state.*
    clock.*
    outcomes.*

  fly/
    simulator.*
    neuron-registry.*
    encoder.*
    decoder.*
    plasticity.*

  experiment/
    pairwise.*
    day-loop.*
    logging.*
```

The adapter should expose a narrow interface regardless of the exact local Bondsmith API:

```ts
interface SavingsEnvironment {
  getState(): Promise<EnvironmentState>;
  allocate(productId: string, amount: number): Promise<void>;
  advanceDay(): Promise<ResolvedOutcome[]>;
}
```

This allows a fully in-memory environment to be used in tests before the Bondsmith integration is complete.

---

# 11. Required experiment logging

Every run must preserve enough state to replay it:

```text
experiment configuration
MaleCNS dataset/version
neuron codebook version
input normalization bounds
stimulus Hz calibration
trial duration
random seed(s)
financial state for each day
all pairwise neural readouts
decoded actions
all environmental outcomes
all teaching events
plasticity snapshots / diffs
```

Outputs should support both machine-readable JSONL and a compact human-readable report.

---

# 12. Baselines and controls

Before interpreting learned behaviour, run the same environment against:

1. intact fixed MaleCNS
2. intact plastic MaleCNS before training
3. trained plastic MaleCNS
4. trained fly with plastic weights reset
5. selected neural lesion(s)
6. shuffled/randomised connectivity control where feasible
7. trivial non-neural policies such as random choice and highest-rate choice

The objective is not to prove that a fly is a good savings optimiser. The objective is to determine what behaviour emerges from the connectome, how experience changes it, and which circuits are responsible for those changes.

---

# 13. First calibration experiment

Before calling Bondsmith at all:

1. Resolve one candidate bilateral input population.
2. Resolve the left/right descending output populations.
3. Stimulate only the left input at several Hz levels.
4. Stimulate only the right input at the same levels.
5. Measure left/right output activity over multiple seeds.
6. Confirm that the system produces a stable, non-saturated bilateral bias.
7. Repeat for every candidate feature channel.
8. Test simultaneous channels for interference.

If this does not work, no financial-layer work can rescue the experiment. The neural I/O must be established first.

---

# 14. Milestones

## M0 - Pin dependencies

- choose MaleCNS dataset release
- choose/reference the LIF simulator implementation
- import neuron metadata and body-ID -> simulation-index mapping
- record versions in config

## M1 - Neural registry

- query candidate bilateral input populations
- query candidate output populations
- save codebook with body IDs and simulation indices
- validate no unintended overlap

**Exit:** one documented input population can reproducibly bias a documented output population.

## M2 - Pairwise neural assay

- implement feature normalization
- implement left/right stimulation encoder
- fixed-duration trial runner
- output population firing-rate decoder
- deterministic seeding
- trial logging

**Exit:** arbitrary two-option feature vectors produce repeatable raw neural readouts and a left/right/no-preference result.

## M3 - Local environment

- implement cash and short-term products
- fixed tranche allocation
- maturity and reward accrual
- manual day advancement
- deterministic scenarios

**Exit:** environment can run end-to-end without Bondsmith.

## M4 - Bondsmith adapter

- map local Bondsmith product/account objects into `EnvironmentState`
- map `FlyAction` into local allocation calls
- keep manual time under FlySmith control

**Exit:** `advance day -> neural decision -> local Bondsmith action` works end-to-end.

## M5 - Fixed-connectome experiments

- predefined 10-30 day scenarios
- rate/product changes
- optional liquidity requirements
- intact vs lesion/control runs

**Exit:** complete replayable experimental logs and baseline plots.

## M6 - Plastic mushroom body

- select published plasticity rule
- implement structural weight + learned multiplier
- resolve KC/MBON/DAN populations
- reproduce non-financial conditioning sanity test
- implement cue-replay teaching
- persist/reset learning state

**Exit:** identical stimulus can produce a measurably different neural choice after conditioning, and resetting plasticity removes that acquired change.

## M7 - Learned financial environment

- map resolved financial outcomes to teaching signals
- train across repeated short-term product decisions
- evaluate generalisation to unseen product combinations
- lesion trained circuitry
- compare against controls

**Exit:** quantify what is learned, where the memory is stored, and how circuit perturbations alter learned savings behaviour.

---

# 15. Immediate next task

Do not start with Bondsmith endpoints.

The first implementation task is to establish the **neural I/O codebook**:

```text
financial feature
    -> exact MaleCNS body IDs
    -> exact simulator indices
    -> stimulation strength/duration
    -> whole-connectome propagation
    -> exact output body IDs
    -> measured left/right firing-rate bias
```

Once that assay is calibrated, the Bondsmith adapter is ordinary application plumbing around a defined experimental core.
