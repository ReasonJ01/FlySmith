# FlySmith — resolved experimental design

This document supersedes the unresolved neural-I/O choices in `PLAN.md`. It is the implementation target for V1.

## 1. Decision

FlySmith will **not** encode finance into arbitrary sensory neurons and will **not** decode savings choices from left/right turning neurons.

The V1 assay is a mushroom-body value-learning assay:

```text
Bondsmith option
  -> normalized feature vector
  -> fixed sparse Kenyon-cell (KC) code
  -> retained MaleCNS network
  -> approach/avoidance MBON activity
  -> scalar option value
  -> choose highest-value valid option
  -> local Bondsmith action
  -> outcome
  -> DAN-gated KC->MBON plasticity
```

This is closer to the biological circuit we are trying to exploit. KCs are the sparse stimulus representation used for associative learning; DANs provide reinforcement; KC->MBON synapses are a major plastic locus; MBON balance contributes to learned approach/avoidance.

The financial world is deliberately abstract. We therefore inject an artificial conditioned stimulus at the KC representation layer instead of pretending that a percentage rate is a smell, colour, pheromone, or motor command.

## 2. Dataset and simulator

Pin the public neuPrint dataset:

```text
male-cns:v1.0
```

The public `malecns` tooling now defaults to this release. Store the neuPrint dataset UUID/version metadata alongside every experiment.

Retain the complete selected MaleCNS graph used by the simulator. Do not crop to the mushroom body. Plasticity is restricted; propagation is not.

The LIF implementation is an engineering model, not measured MaleCNS physiology. All neuronal parameters, transmitter-sign assumptions, background drive, edge filtering, timestep and random seeds must be logged.

## 3. Financial observation

For every valid destination, including cash, compute:

```ts
interface OptionFeatures {
  yield: number;      // normalized 0..1 payoff/rate
  lock: number;       // normalized 0..1 days inaccessible
  liquidity: number;  // normalized 0..1 liquid assets after action
  exposure: number;   // normalized 0..1 concentration after action
}
```

These are observations, not utility terms. No sign is attached to them by the encoder.

Bounds are fixed before an experiment. Values are clipped to `[0,1]`.

Cash is represented by the same schema, e.g. lock=0 and its configured yield.

## 4. Input representation: sparse KC code

### Why KCs

The mushroom body is an associative-learning centre. Sensory cues are represented sparsely across Kenyon cells and learned value is imposed downstream through dopamine-gated plasticity at KC->MBON synapses. Direct KC stimulation is an explicit experimental abstraction: FlySmith supplies the conditioned-stimulus representation while leaving the downstream connectome intact.

This is preferable to choosing four antennal-lobe or visual projection-neuron types whose innate tuning and lateral-horn pathways would inject undocumented biological priors into financial features.

### KC pool

Resolve all annotated olfactory/main-calyx KCs in MaleCNS V1 and include the canonical olfactory KC classes where present in the MaleCNS annotation vocabulary:

```text
KCg-m / gamma-main
KCab / alpha-beta classes
KCa'b' / alpha-prime-beta-prime classes
```

Exclude visual/accessory-calyx KC classes from V1 (`KCg-d`, `KCab-p`) so the artificial code is not mixed with a special visual stream.

The registry builder must query the actual MaleCNS annotations and save the exact body IDs. Never hard-code body IDs copied from another connectome.

### Fixed random expansion

Do not assign one group of neurons to `yield`, another to `lock`, etc. That would make feature semantics depend on hand-selected cells.

Instead use a fixed seeded random projection from the four normalized features plus a constant bias into the resolved KC pool:

```text
x = [yield, lock, liquidity, exposure, 1]
z = W x
active KCs = top-k(z)
```

`W` is generated once from a recorded seed and then frozen for the entire experiment, including all controls.

Initial sparsity target:

```text
k = 5% of eligible KCs
```

This creates a distributed code in which similar option vectors tend to share some active KCs while no single KC is declared to mean “interest rate” or “term”. The 5% value is a calibration starting point, not a biological measurement.

### Stimulus delivery

Active KCs receive identical external Poisson drive. Inactive KCs receive no option-specific drive.

Calibrate the drive before finance experiments. Search a grid such as 25, 50, 100, 150, 200 Hz and choose the lowest rate that produces reproducible MBON modulation without global saturation.

The chosen rate and duration are frozen in the experiment config. Start duration calibration at 250–1000 ms.

Every option is evaluated from the same pre-trial neural state and with controlled random seeds.

## 5. Output representation: mushroom-body valence

Do not use turning DNs as the primary V1 value readout. Those are useful for embodied game controllers, but they add a second arbitrary mapping from value to lateralized movement.

Resolve these MaleCNS MBON types by annotation:

### Approach channel

```text
MBON-gamma1pedc>alpha/beta
short name: MBON-11
alternative: MVP2 / MB-MVP2
```

MBON-11 is an approach-promoting output in the established olfactory-learning literature. PPL1-01-mediated punishment can weaken active KC input to this compartment.

### Avoidance channel

Primary V1 avoidance readout:

```text
MBON-gamma5beta'2a
short name: MBON-01
alternative: MB-M6
```

Secondary validation readouts, logged but not required for the first decoder:

```text
MBON-beta'2mp     (MBON-03 / M4)
MBON-beta2beta'2a (MBON-02 / M4)
```

M4/M6 activity has been shown to drive/track avoidance, and blocking these outputs can convert naive odor avoidance toward approach.

### Value decoder

For each option run an independent cue trial and measure baseline-subtracted population firing rates:

```text
A = deltaHz(MBON-11)
V = deltaHz(MBON-01)
score = z(A) - z(V)
```

`z()` uses calibration statistics from naive no-teaching trials, frozen before training. If the LIF implementation makes one channel systematically silent, fail calibration rather than silently changing the decoder.

Log MBON-02 and MBON-03 as secondary avoidance channels. A preregistered robustness analysis may use:

```text
avoidanceComposite = mean(z(MBON-01), z(MBON-02), z(MBON-03))
```

but V1's primary result remains MBON-11 minus MBON-01 so the decoder is simple and auditable.

## 6. Choice and action

Evaluate each currently valid option separately. There is no left/right pairwise presentation in V1.

```text
for option in [cash, productA, productB, ...]:
    restore identical pre-trial neural state
    stimulate KC code(option)
    run fixed duration
    compute option score

winner = argmax(score)
```

Use common random numbers across options where the simulator permits it, and rotate option evaluation order across replicate seeds to detect state/order leakage.

Action translation is deliberately minimal:

```text
cash wins      -> HOLD
product wins   -> ALLOCATE fixed tranche to product
```

Position size is not neural in V1. Use a configured fixed amount or fixed fraction of currently liquid cash.

If score differences are below a calibrated indifference margin, HOLD.

## 7. Teaching circuit

### Positive reinforcement

Positive outcomes teach approach by weakening active KC drive onto the avoidance side of the MB output balance.

Use the PAM compartments that overlap the primary avoidance output:

```text
PAM-gamma5      (PAM-01)
PAM-beta'2a     (PAM-02)
```

For the minimum V1 implementation, use **PAM-beta'2a / PAM-02** as the positive teaching population because activation of PAM-beta'2a has direct experimental support for depressing KC->MBON-gamma5beta'2a responses and reducing avoidance.

Keep PAM-gamma5 as a planned replication/ablation condition rather than mixing both in the first learning rule.

### Negative reinforcement

Use:

```text
PPL1-gamma1pedc
short name: PPL1-01
alternative: MB-MP1 / MP
```

PPL1-01 is a well-established punishment pathway for the gamma1pedc compartment; pairing a cue with PPL1-01 activity weakens KC input to approach-promoting MBON-11 and supports learned avoidance.

A recent MaleCNS Doom experiment independently identified two PPL101 cells as body IDs `11327` and `11900`. FlySmith must still resolve PPL1-01 from its own pinned MaleCNS metadata at build time and assert those IDs only if the dataset query agrees; the external IDs are a cross-check, not the registry source of truth.

## 8. Plastic synapses

Only existing KC->MBON edges in the teaching compartments are plastic.

V1 positive plastic set:

```text
active eligible KC -> MBON-gamma5beta'2a
```

V1 negative plastic set:

```text
active eligible KC -> MBON-gamma1pedc>alpha/beta
```

No new edges are created. Structural synapse count is immutable.

Store:

```ts
interface PlasticEdge {
  preBodyId: number;
  postBodyId: number;
  structuralWeight: number;
  efficacy: number; // starts 1.0
}

effectiveWeight = structuralWeight * efficacy;
```

## 9. Plasticity rule

Use a deliberately conservative dopamine-gated depression rule for V1, matching the experimentally supported direction at these compartments:

```text
for each active KC -> target MBON edge:
    efficacy *= exp(-eta * teachingMagnitude * normalizedKCActivity)
```

Clamp:

```text
0.1 <= efficacy <= 1.0
```

V1 therefore learns by depression only; it does not invent potentiation. This is easier to interpret and matches strong evidence that coincident KC/DAN activation can produce depression of KC->MBON transmission.

`eta` is calibrated in a conditioning assay, not tuned against financial return.

A later implementation can reproduce a published timing-dependent heterogeneous dopamine rule, but V1 should first establish that the complete MaleCNS simulation can express a conditioned MBON shift at all.

## 10. Credit assignment

Do not use a multi-day biological eligibility trace in V1.

When an outcome resolves, replay the exact KC cue that was present when the allocation was chosen and deliver the corresponding teaching event in the same conditioning window:

```text
positive outcome:
    replay chosen KC cue
    stimulate PAM-beta'2a
    depress active KC -> MBON-01 edges

negative outcome:
    replay chosen KC cue
    stimulate PPL1-01
    depress active KC -> MBON-11 edges
```

This is explicit cue-reinforcer pairing. It is appropriate for the short artificial terms in FlySmith and avoids pretending that a days-long eligibility trace has biological support.

## 11. Reward definition

The environment, not the neural network, computes the scalar teaching outcome.

V1 should separate reward sources in the log:

```ts
interface Outcome {
  interestReward: number;
  liquidityPenalty: number;
  total: number;
}
```

Use a fixed normalization scale chosen before a run:

```text
teachingMagnitude = clamp(abs(total) / OUTCOME_SCALE, 0, 1)
```

`total > 0` -> PAM teaching.

`total < 0` -> PPL1 teaching.

`total == 0` -> no teaching.

Do not use relative portfolio performance, future information, or an optimiser-generated target as the teaching signal in V1.

## 12. Day loop

```text
DAY N
  query local Bondsmith-compatible state
  build valid options including cash
  encode each option -> sparse KC pattern
  run independent option trials
  score MBON-11 - MBON-01
  choose max score or HOLD inside indifference margin
  allocate fixed tranche if applicable
  persist decision + cue + neural state hashes
  stop

ADVANCE DAY
  accrue environment reward
  decrement short terms
  mature products
  resolve liquidity events
  identify resolved allocations
  for each resolved outcome:
      replay stored cue
      apply PAM or PPL1 teaching
      persist plastic-weight diff
  expose next day
```

No wall-clock scheduling.

## 13. Mandatory calibration gates

Do not connect the learning loop to Bondsmith until all gates pass.

### Gate A — KC stimulus

A 5% sparse KC cue at the chosen drive must cause a reproducible, non-global response. Reject stimulus settings that saturate a large fraction of the CNS.

### Gate B — MBON readout

At least one of MBON-11 and MBON-01 must show reproducible cue-dependent modulation across seeds, and the score distribution must have finite variance.

### Gate C — aversive conditioning

For a fixed cue X:

```text
pre: score(X)
pair X + PPL1-01 teaching
post: score(X)
```

Require a reproducible downward shift in `score(X)` relative to an unpaired cue Y and sham-teaching control.

### Gate D — appetitive conditioning

For cue X:

```text
pre: score(X)
pair X + PAM-beta'2a teaching
post: score(X)
```

Require a reproducible upward shift in `score(X)` relative to cue Y and sham.

### Gate E — specificity

Conditioning X must change X more than a sufficiently dissimilar unpaired KC cue Y. If all cues shift equally, the model is showing global excitability change rather than associative memory.

### Gate F — reversibility/reset

Resetting efficacy multipliers to 1.0 must restore the naive response distribution within seed variance.

If any gate fails, stop. Do not tune the financial environment until the neural assay works.

## 14. Experimental controls

Every claimed learning result must include:

- fixed-weight MaleCNS
- plastic MaleCNS with sham DAN stimulation
- trained plastic MaleCNS
- trained model with plastic weights reset
- cue-unpaired DAN control
- PPL1-01 silenced during negative teaching
- PAM-beta'2a silenced during positive teaching
- MBON-11 readout lesion/control
- MBON-01 readout lesion/control
- random-choice policy
- highest-yield policy
- cash-only policy

Also run a shuffled-KC-code control: regenerate the fixed random projection with another seed while holding the financial history constant. Learned behavior that depends entirely on one lucky codebook is not robust.

## 15. What other whole-connectome projects teach us

### Mario / `ornata/fly`

The Mario project uses the complete MaleCNS network but explicitly engineers the interface: visual input is mapped onto modeled eye cells; DNg100 is read for forward movement; DNa02/DNg13 differential activity controls steering; DNp01/DNp10 bursts trigger jump. It logs and replays pictures, spikes, controls and seeds. It explicitly states there is no training or reward.

FlySmith adopts the reproducibility discipline but avoids motor DNs because the task is value selection rather than embodied locomotion.

### Doom / `nftechie/doomfly`

DoomFly is the most relevant contemporary reference. It retains 166,700 neurons and reports 25,582,938 directed edges / 124,177,617 contacts. Its current experimental learning path uses dopamine-gated memory and delivers damage-contingent stimulation to two PPL101 cells. Importantly, the project publishes failed validation gates and states that changing weights does not by itself demonstrate learning.

FlySmith adopts that standard: learning is not claimed until conditioning, specificity, sham, and reset controls pass.

### Other FlyWire demos

Several projects expose whole-brain LIF simulations with stimulus-to-neuron atlases and KC->MBON dopamine plasticity. These are useful implementation references, but many stimulus mappings and behavioral labels are engineered. FlySmith therefore treats the connectome as measured structure and the dynamics/encoder/decoder as explicit model assumptions.

## 16. Why the previous pairwise-turning plan is rejected

The old plan used:

```text
financial feature -> arbitrary bilateral sensory population
                   -> whole CNS
                   -> left/right descending neurons
                   -> product choice
```

This introduces two unnecessary arbitrary semantics:

1. why a particular sensory channel means “yield” or “lock”; and
2. why left/right turning means product A/product B.

The resolved design has one artificial boundary only: the financial option is converted into a sparse conditioned-stimulus representation in KCs. Downstream value learning and valence readout then use the mushroom-body circuit for the function it is experimentally associated with.

## 17. Registry build requirements

At setup time query `male-cns:v1.0` and materialize `config/neuron-registry.json` containing exact body IDs and simulator indices for:

```text
eligible olfactory KCs
MBON-gamma1pedc>alpha/beta (MBON-11)
MBON-gamma5beta'2a        (MBON-01)
MBON-beta'2mp             (MBON-03)
MBON-beta2beta'2a         (MBON-02)
PPL1-gamma1pedc           (PPL1-01)
PAM-beta'2a               (PAM-02)
PAM-gamma5                (PAM-01; validation only)
```

The build must fail if a required type resolves to zero neurons or if the expected left/right/cell-count structure changes materially. Save the returned type/instance names and dataset UUID with the IDs.

## 18. Implementation order

1. Pin and import MaleCNS v1.0 metadata/connectivity.
2. Build the neuron registry from annotations; no hand-copied IDs.
3. Implement deterministic KC random projection and cue persistence.
4. Implement option-trial runner and MBON readout.
5. Pass Gates A and B.
6. Implement restricted KC->MBON efficacy multipliers.
7. Implement explicit PPL1-01 and PAM-beta'2a teaching events.
8. Pass Gates C–F on synthetic cues.
9. Implement in-memory short-term savings environment.
10. Run fixed financial histories against naive/trained/control flies.
11. Only then connect the same environment interface to the local Bondsmith stack.

## 19. Primary V1 hypothesis

> A sparse artificial option representation imposed on MaleCNS Kenyon cells can acquire outcome-dependent value through compartment-specific dopamine-gated KC->MBON depression, producing reproducible changes in an MBON approach-minus-avoidance score that alter subsequent savings-option selection.

This is falsifiable. If the conditioning gates fail, FlySmith has not taught the fly, regardless of portfolio performance.

## 20. Sources

1. Berg et al., MaleCNS public release and associated dataset; public neuPrint dataset `male-cns:v1.0`.
2. natverse `malecns` documentation, public MaleCNS v1.0 access and annotation/connectivity tooling: https://natverse.org/malecns/
3. Aso et al. (2014), *The neuronal architecture of the mushroom body provides a logic for associative learning*, eLife 3:e04577. https://doi.org/10.7554/eLife.04577
4. Aso et al. (2014), *Mushroom body output neurons encode valence and guide memory-based action selection in Drosophila*, eLife 3:e04580. https://doi.org/10.7554/eLife.04580
5. Owald et al. (2015), *Activity of Defined Mushroom Body Output Neurons Underlies Learned Olfactory Behavior in Drosophila*, Neuron 86:417–427. https://doi.org/10.1016/j.neuron.2015.03.025
6. Takemura et al. (2017), *A connectome of a learning and memory center in the adult Drosophila brain*, eLife 6:e26975. https://doi.org/10.7554/eLife.26975
7. Li et al. (2020/2021), *The connectome of the adult Drosophila mushroom body provides insights into function*, eLife 9:e62576. https://doi.org/10.7554/eLife.62576
8. McCurdy et al. (2021), *Input Connectivity Reveals Additional Heterogeneity of Dopaminergic Reinforcement in Drosophila*, Current Biology 31. https://doi.org/10.1016/j.cub.2020.05.077
9. Springer et al. (2021), *Dopaminergic mechanism underlying reward-encoding of punishment omission during reversal learning in Drosophila*, Nature Communications 12:1115. https://doi.org/10.1038/s41467-021-21388-w
10. Gkanias et al. (2022), *An incentive circuit for memory dynamics in the mushroom body of Drosophila melanogaster*, eLife 11:e75611. https://doi.org/10.7554/eLife.75611
11. Handler et al./related dopamine-plasticity literature summarized in Springer et al. and Gkanias et al.; V1 uses only the well-supported depression direction and treats its numerical learning rule as a model assumption.
12. `nftechie/doomfly`, contemporary MaleCNS whole-connectome game/learning experiment: https://github.com/nftechie/doomfly
13. `ornata/fly`, MaleCNS-to-Super-Mario controller and replay architecture: https://github.com/ornata/fly
14. Xenova Neural Canvas, browser MaleCNS LIF demonstration: https://huggingface.co/spaces/Xenova/fruit-fly-simulation

## 21. Status of uncertainty

The architecture choices above are closed for V1. The remaining numbers are **calibration parameters**, not conceptual open questions:

- KC sparsity (start 5%)
- external KC drive (grid-calibrate)
- trial duration (grid-calibrate)
- MBON indifference margin (estimate from naive variance)
- learning rate `eta` (set from synthetic conditioning, never financial return)
- outcome normalization scale
- fixed allocation tranche

Those values should be selected by preregistered calibration procedures and then frozen before the first financial-learning run.