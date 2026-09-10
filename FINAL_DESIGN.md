# FlySmith — researched neural design

This document freezes the V1 experimental design. It replaces the unresolved choices in `PLAN.md`.

## Decision summary

FlySmith will **not** map savings features to arbitrary sensory neurons and will **not** use turning descending neurons as the primary value readout.

V1 is a synthetic conditioning assay built around the fly's mushroom body:

```text
Bondsmith option
  -> normalized feature vector
  -> deterministic sparse code in real MaleCNS Kenyon cells
  -> full MaleCNS recurrent LIF simulation
  -> MBON11 approach activity and MBON01 avoidance activity
  -> scalar neural valence for that option

all currently valid options
  -> score independently with the same bilateral cue
  -> highest neural valence wins
  -> fixed tranche is allocated to that product

resolved financial outcome
  -> replay the chosen KC cue
  -> positive outcome: stimulate PAM15
     negative outcome: stimulate PPL101
  -> dopamine-gated depression of existing KC->MBON synapses
  -> next presentation can score differently
```

The point is not to pretend that interest rates are smells. Financial observations are encoded directly as artificial conditioned stimuli in the associative-learning layer. The biological claim is therefore limited to the part we actually use: real MaleCNS KC/MBON/DAN anatomy constrains representation, memory and value readout.

---

## 1. Dataset and simulator

Freeze V1 to **MaleCNS v1.0**.

The official v1.0 release contains 166,700 annotated neurons and the complete brain + VNC. The public static connectivity table is `connectome-weights-male-cns-v1.0-minconf-0.5.feather`; neuron neurotransmitter predictions and other tables are also available from the official download page.

The reference full-graph runtime is the Neural Canvas-style MaleCNS LIF model: 166,700 retained entries and 25,582,938 directed connection rows. It is an engineered transfer of LIF dynamics to MaleCNS, not a validated physiological reconstruction. FlySmith must preserve that distinction in logs and UI.

Pin and hash every imported data artifact. Never silently update the dataset.

Sources:

- MaleCNS: https://male-cns.janelia.org/
- MaleCNS downloads: https://male-cns.janelia.org/download/
- MaleCNS publication/data description: https://pmc.ncbi.nlm.nih.gov/articles/PMC12636603/
- Neural Canvas reference implementation: https://huggingface.co/spaces/Xenova/fruit-fly-simulation

---

## 2. Why the original pairwise left/right plan is rejected

The previous plan proposed putting financial features into bilateral sensory populations and reading left/right turning DNs. That is attractive as a demo but weak as a learning experiment:

1. There is no biological reason for `rate`, `term`, `liquidity`, and `exposure` to correspond to four particular peripheral sensory cell types.
2. A left/right code risks learning side rather than option value.
3. DNa02/DNg13 and similar DN mappings are useful motor interfaces, but a financial preference decoder would still be an engineered bridge from motor activity to product selection.
4. The learning mechanism we actually care about lives in the mushroom body, so forcing the signal through an arbitrary sensory code makes validation harder.

V1 therefore evaluates each option as a **bilateral conditioned stimulus** and reads mushroom-body valence directly. A later embodied V2 can ask whether that learned valence propagates to native descending/motor choices.

---

## 3. Input neurons: Kenyon cells

### 3.1 Population

Use all MaleCNS neurons annotated as **Kenyon cells (KCs)** that satisfy both conditions:

- they are part of the mushroom-body intrinsic KC population;
- they have at least one retained structural edge to either MBON01 or MBON11.

Do not hard-code a guessed list. Resolve this population from the pinned MaleCNS annotations and connectivity at build time, save the resulting body IDs in `data/registry-v1.json`, and fail startup if the resolved registry hash changes.

MaleCNS contains thousands of KCs; a recent independent full-MaleCNS implementation reports 4,064 annotated KCs. That number is a useful cross-check, not a constant to force.

### 3.2 Why direct KC input

Direct KC stimulation is deliberate.

In normal olfactory learning, projection-neuron input is expanded into sparse KC representations and dopamine modifies KC->MBON transmission. For FlySmith there is no natural receptor mapping for an interest rate or deposit term. Inventing one would add an unnecessary sensory hypothesis. Directly creating a sparse KC conditioned stimulus starts at the associative representation layer and leaves the full downstream recurrent connectome intact.

This is also computationally standard in mushroom-body models, which represent stimuli as KC activity patterns and make KC->MBON weights the learned variables.

Sources:

- Aso et al., neuronal architecture of the MB: https://pubmed.ncbi.nlm.nih.gov/25535793/
- Takemura et al., MB learning/memory connectome: https://pmc.ncbi.nlm.nih.gov/articles/PMC5550281/
- Bennett et al., adult MB connectome: https://pmc.ncbi.nlm.nih.gov/articles/PMC7909955/
- Jiang & Litwin-Kumar, dopamine-gated MB models: https://pmc.ncbi.nlm.nih.gov/articles/PMC8354444/

---

## 4. Financial observation encoding

### 4.1 Features

V1 exposes exactly four product features:

```text
rate
term_days
post_action_liquidity
post_action_exposure
```

`cash` is represented as an ordinary option with `term_days = 0`, `post_action_liquidity = 1`, and experiment-defined cash return.

Each value is normalized to `[0,1]` using bounds frozen in the experiment config. Bounds may not be recomputed from the day's available products, because that would change the meaning of the same product across days.

### 4.2 Sparse distributed KC code

Do not use one KC group per feature and do not encode a scalar only as firing rate.

Generate a fixed random projection matrix once from the experiment seed:

```text
P: [number_of_selected_KCs x 4]
```

For option feature vector `x`:

```text
z = P x + b
```

Select the top **5%** of KCs by `z` independently in each hemisphere and stimulate those cells. The same feature vector must always produce the same KC ensemble.

Use a small amount of smooth feature noise only in explicit robustness experiments, never during the baseline condition.

Why 5%: the biological MB uses sparse KC representations; V1 needs a concrete starting sparsity without pretending the exact fraction is measured for this artificial cue. Sparsity is a calibration parameter and must be tested at 2%, 5%, and 10% before freezing a production experiment.

### 4.3 Bilateral presentation

The option cue is presented to matched KC ensembles in **both hemispheres simultaneously**. We are scoring an option, not asking the fly to turn toward a side.

This eliminates left/right product identity as a nuisance variable.

### 4.4 Stimulation

Use direct external drive to the selected KCs for a fixed cue window. Start calibration at:

```text
cue duration: 500 ms
KC external drive: 80 Hz
```

These are starting calibration values, not biological constants. Run a grid over 40/80/120 Hz and 250/500/1000 ms and choose the lowest condition that produces:

- reproducible MBON responses across seeds;
- nonzero MBON01 and/or MBON11 activity;
- no widespread network saturation;
- separable responses for distinct KC cues.

Freeze the selected values in the experiment manifest.

---

## 5. Output neurons: MBON11 and MBON01

Use a two-channel learned-valence readout.

### Approach channel — MBON11

**MBON11 = MBON-γ1pedc>α/β (MVP2)**.

It is GABAergic and is experimentally associated with positive valence / approach and food-odor seeking. In MaleCNS, the two MBON11 cells used by another full-graph implementation resolve to simulation IDs `10704` and `11402`; FlySmith must independently resolve body IDs from MaleCNS annotations and verify the mapping rather than copying those indices.

### Avoidance channel — MBON01

**MBON01 = MBON-γ5β′2a (M6)**.

It is glutamatergic. M4/M6 output, including MBON01, is associated with avoidance; blocking M4/M6 can convert naive odor avoidance into approach, while activating these neurons can drive avoidance. MaleCNS v1.0 body IDs for MBON01 are documented as:

```text
L: 520151
R: 10013
```

Resolve and verify them from the pinned annotation table at startup.

### Valence decoder

For a cue window plus a 250 ms post-cue readout window:

```text
approach_hz  = mean firing rate of bilateral MBON11
avoidance_hz = mean firing rate of bilateral MBON01
```

Before experiments, estimate each channel's naive baseline mean and standard deviation from blank trials. Convert to z-scores:

```text
A = z(approach_hz)
V = z(avoidance_hz)
score = A - V
```

The neural score for an option is `score`.

Run each option for `N=5` deterministic seeds in V1 and use the median score. The option with the highest median score wins. If the top two options differ by less than the pre-registered indifference margin, choose `HOLD`.

Sources:

- MBON valence/action selection: https://elifesciences.org/articles/04580
- M4/M6 avoidance and learned choice: https://pmc.ncbi.nlm.nih.gov/articles/PMC4416108/
- MBON11 approach/food seeking: https://pmc.ncbi.nlm.nih.gov/articles/PMC5910021/
- MaleCNS MBON01 IDs example: https://natverse.org/coconatfly/articles/metadata-columns.html

---

## 6. Action decoder

The fly controls **destination only** in V1.

```ts
type FlyAction =
  | { type: "hold" }
  | { type: "allocate"; productId: string; amount: number };
```

The environment sets a fixed tranche, defaulting to **10% of currently liquid cash**, capped by product constraints.

Algorithm:

```text
for each currently valid destination, including cash:
    create OptionFeatures
    create deterministic bilateral KC cue
    run 5 seeded trials
    compute median MBON valence score

if winner does not clear indifference margin:
    HOLD
else if winner == cash:
    HOLD
else:
    ALLOCATE fixed tranche -> winner
```

Do not let spike magnitude determine allocation size in V1.

---

## 7. Teaching neurons and plastic synapses

V1 implements two compartment-specific learning channels.

### Positive reinforcement

Use **PAM15**, the dopaminergic `PAM-γ5β′2a` population.

PAM15 projects to γ5 and β′2a, overlapping the MBON01 compartment. Reward learning can depress KC drive onto avoidance-associated MBON output, shifting subsequent behavior toward approach.

Plastic edges for this channel:

```text
KC -> MBON01
```

only where that structural edge exists in MaleCNS.

### Negative reinforcement

Use **PPL101**, also known as `PPL1-γ1pedc` / `MB-MP1`.

PPL101 is the canonical aversive teaching neuron for the γ1pedc compartment. Pairing its activation with odor/KC activity depresses KC input to MBON11, an approach-associated MBON, shifting learned valence toward avoidance.

Plastic edges for this channel:

```text
KC -> MBON11
```

only where that structural edge exists in MaleCNS.

Another current MaleCNS project independently uses the two PPL101 cells as an aversive teaching signal and restricts plasticity to 4,184 KC->MBON11 edges. Its first full-network learning candidate failed because sensory activity did not produce usable KC patterns and endogenous DAN activity contaminated cue selectivity. FlySmith avoids both failure modes by (a) injecting the artificial conditioned stimulus directly at KCs and (b) enabling weight updates only inside explicit teaching epochs.

Sources:

- PPL101 / γ1pedc aversive plasticity: https://pmc.ncbi.nlm.nih.gov/articles/PMC4674068/
- PPL101 anatomy/synonyms: https://flybase.org/reports/FBbt%3A00100243
- PAM15 = PAM-γ5β′2a: https://flybase.org/cgi-bin/cvreport.pl?childdepth=2&cvterm=FBbt%3A00049841
- Reward/aversive MB compartment logic: https://pmc.ncbi.nlm.nih.gov/articles/PMC7909955/
- DOOMFLY negative learning result and controls: https://github.com/nftechie/doomfly/blob/main/docs/doom-learning-review.md

---

## 8. Plasticity rule

Do not allow ordinary endogenous dopamine spikes to modify weights in V1. Neural DAN activity still propagates through the recurrent model, but **plasticity is armed only during an explicit teaching epoch**. This is an experimental intervention analogous to controlled conditioning and prevents spontaneous modeled DAN activity from silently rewriting memory.

For each plastic structural edge, store:

```text
base_weight
plastic_multiplier
```

with:

```text
effective_weight = base_weight * plastic_multiplier
```

Initial multiplier = `1.0`.

During teaching:

1. replay the exact KC cue associated with the resolved action;
2. after 200 ms of cue activity, stimulate the appropriate DAN population for 200 ms;
3. depress active KC->target-MBON multipliers using a presynaptic eligibility trace;
4. clamp multipliers to `[0.1, 1.0]` in V1;
5. stop plasticity immediately when the teaching epoch ends.

Use:

```text
e_KC(t+dt) = e_KC(t) * exp(-dt/tau_e) + spike_KC

delta multiplier = -eta * e_KC * teaching_strength
```

Initial calibration:

```text
tau_e = 1 s
eta = 1e-3
```

These match the order of magnitude used in a current MaleCNS learning implementation but are not claimed as measured MaleCNS physiology. They must pass the conditioning gates below before being used in the financial environment.

The crucial biological constraint is the **sign and compartment**: positive reinforcement depresses active KC->MBON01 avoidance drive; negative reinforcement depresses active KC->MBON11 approach drive.

---

## 9. Reward supplied by the financial environment

The fly is not taught that a high rate is good. It is reinforced from **realized local-environment consequences**.

For each allocation create an outcome ledger entry. When it resolves, compute:

```text
reward = realized_return
         - liquidity_failure_penalty
         - early_exit_or_lock_penalty
```

Then normalize with fixed experiment-wide bounds to `[-1, 1]`.

Teaching rule:

```text
reward >= +epsilon -> PAM15 teaching, strength = abs(reward)
reward <= -epsilon -> PPL101 teaching, strength = abs(reward)
otherwise          -> no teaching
```

Do not include opportunity cost in V1. It requires a counterfactual and would leak an externally computed optimizer into the teaching signal. V1 teaches only from consequences the chosen action actually experienced.

For short products, teach on maturity or on a realized liquidity failure. If a product both earns return and causes a liquidity failure, aggregate those realized consequences into one signed outcome before teaching.

---

## 10. Why cue replay is chosen over multi-day eligibility

Use **cue replay** when a financial outcome resolves.

A 3-day product may mature long after the original neural trial. There is no justification for assuming a KC synaptic eligibility trace lasts simulated days. Re-presenting the original conditioned stimulus with a DAN teaching pulse is much closer to a conventional controlled conditioning trial and avoids inventing a multi-day intracellular mechanism.

The ledger therefore stores the exact feature vector, projection seed/version and resulting KC indices used for the choice. Teaching reconstructs exactly that cue.

A future experiment may compare replay against long eligibility traces, but it is not part of V1.

---

## 11. Day loop

```text
DAY N

1. Query local Bondsmith-compatible state.
2. Enumerate valid destinations: cash + available products.
3. Convert each destination to four normalized features.
4. Score each destination independently through bilateral KC -> MaleCNS -> MBON valence.
5. Choose highest score subject to indifference margin.
6. Allocate fixed tranche or hold.
7. Save action + cue + neural traces in the outcome ledger.
8. Stop.

MANUAL ADVANCE

9. Environment advances exactly one simulated day.
10. Accrue returns, decrement terms, process maturities and liquidity events.
11. Resolve ledger outcomes that became final.
12. For each non-neutral resolved outcome, run an explicit cue-replay teaching epoch.
13. Save weight diffs and a plasticity checkpoint.
14. Present DAY N+1.
```

No wall-clock time participates.

---

## 12. Bondsmith boundary

Bondsmith remains an environment/API contract, not part of neural computation.

```ts
interface SavingsEnvironment {
  getState(): Promise<EnvironmentState>;
  allocate(productId: string, amount: number): Promise<void>;
  advanceDay(): Promise<ResolvedOutcome[]>;
}
```

The neural service exposes:

```ts
interface FlyBrain {
  score(option: OptionFeatures, seeds: number[]): Promise<OptionScore>;
  teach(cue: CueRecord, reward: number): Promise<TeachingResult>;
  snapshot(): Promise<BrainCheckpoint>;
  resetPlasticity(): Promise<void>;
}
```

No Bondsmith object, product name or currency value enters the simulator directly.

---

## 13. Required pre-finance validation gates

Do not connect the local Bondsmith stack until all five gates pass.

### Gate A — cue separability

Generate 20 random feature vectors. Their KC ensembles must be sparse, reproducible and not identical. Similar feature vectors should overlap more than distant vectors on average.

### Gate B — MBON responsiveness

At least 80% of cues must produce a measurable response in MBON01 or MBON11 above blank baseline without network saturation.

### Gate C — appetitive conditioning

Take two fixed cues A and B with matched naive scores. Pair A with PAM15 teaching 10 times; leave B unpaired. A's `approach - avoidance` score must increase relative to B. Freeze weights during evaluation.

### Gate D — aversive conditioning

Reset. Pair A with PPL101 teaching 10 times. A's score must decrease relative to B.

### Gate E — specificity and timing controls

The learned shift must be substantially smaller or absent when:

- plasticity is frozen;
- DAN teaching is omitted;
- the wrong cue is replayed;
- teaching is temporally separated beyond the eligibility window;
- learned multipliers are reset to 1.0.

If C-E fail, FlySmith has not demonstrated learning and the financial experiment remains disabled.

---

## 14. Financial experiment V1

Use a 30-day deterministic local market.

Products may have 1-7 day terms. Keep 3-5 destinations available on a given day. Initial cash is arbitrary because allocation is fractional.

Primary outcome is **change in learned option valence**, not portfolio return.

Report:

```text
neural score by option/day
chosen destination
realized reward
PAM15/PPL101 teaching events
KC->MBON01 multiplier distribution
KC->MBON11 multiplier distribution
choice changes after teaching
portfolio balance as a secondary metric
```

Run matched conditions:

1. naive fixed-weight MaleCNS
2. plastic MaleCNS
3. plasticity-frozen control
4. shuffled reward/cue pairing control
5. memory-reset control
6. simple random policy
7. simple highest-rate policy

Do not claim optimization unless the plastic fly beats controls on held-out market sequences.

---

## 15. What current projects teach us

### Neural Canvas

Useful for the whole-graph WebGPU/JS runtime and for demonstrating arbitrary external stimulation. Its movement mappings are explicitly illustrative. FlySmith should borrow runtime engineering, not behavioral claims.

https://huggingface.co/spaces/Xenova/fruit-fly-simulation

### DOOMFLY

The closest current reference for MaleCNS reinforcement learning. It uses retinal input, fixed game readouts, PPL101 aversive stimulation and KC->MBON11 plasticity. Importantly, its own review reports a negative mechanism result: the visual model failed to drive usable KCs, and direct KC conditioning suffered cue-selectivity/endogenous-DAN problems. FlySmith should treat that negative result as design evidence, not something to hide.

https://github.com/nftechie/doomfly

### Mario 64 MaleCNS project

Uses visual input -> full MaleCNS -> DNg100 forward, DNa02/DNg13 steering and DNp01/DNp10 jump. It explicitly has no training. It is a useful example of keeping sensory and output adapters fixed and logging replayable neural traces.

https://github.com/ornata/fly

### FlyBrain / other FlyWire whole-brain simulators

Several projects use the older female FlyWire connectome with LIF dynamics, embodied sensory adapters and motor decoders. They reinforce the same lesson: the connectome supplies anatomy; stimulus encoding, dynamics and behavior decoding remain modeling choices that require validation.

https://github.com/snedea/flybrain

### FlyGM

A 2026 connectome-constrained graph model uses the fly connectome architecture as a learnable controller and compares it with rewired/random graph baselines. This is useful as a control philosophy, but it trains a graph model rather than claiming that synaptic learning in a LIF MaleCNS simulation has been reconstructed.

https://arxiv.org/abs/2602.17997

---

## 16. Files to implement first

```text
src/
  data/
    malecns.ts              # pinned loader + hashes
    registry.ts             # resolve KC, MBON01, MBON11, PAM15, PPL101

  fly/
    simulator.ts            # full LIF wrapper
    cue-encoder.ts          # 4D -> deterministic sparse bilateral KC code
    valence.ts              # MBON z-score readout
    plasticity.ts           # armed teaching-only KC->MBON depression
    teaching.ts             # PAM15/PPL101 cue replay
    checkpoint.ts

  environment/
    types.ts
    local.ts
    bondsmith.ts
    outcomes.ts

  experiment/
    calibration.ts
    conditioning.ts
    day-loop.ts
    controls.ts
    log.ts

config/
  malecns-v1.json
  neural-v1.json
  market-v1.json

data/
  registry-v1.json
  source-lock.json
```

---

## 17. Frozen V1 choices

| Question | V1 decision |
|---|---|
| Where do financial observations enter? | Direct sparse bilateral KC stimulation |
| Which KCs? | MaleCNS KCs with structural edges to MBON01 or MBON11, resolved from annotations |
| How are four features encoded? | Fixed seeded random projection, top 5% sparse KC code |
| Simultaneous pairwise choices? | No; score each option independently |
| Primary output | Bilateral MBON11 approach minus MBON01 avoidance |
| Product selection | Highest median neural valence across 5 seeds |
| Position size | External fixed 10% liquid-cash tranche |
| Positive teaching neuron | PAM15 / PAM-γ5β′2a |
| Negative teaching neuron | PPL101 / PPL1-γ1pedc |
| Positive plastic edges | Existing KC->MBON01 |
| Negative plastic edges | Existing KC->MBON11 |
| Plasticity outside teaching? | Disabled |
| Delayed credit assignment | Exact cue replay when outcome resolves |
| Topology changes? | Never |
| Weight state | Structural weight × plastic multiplier |
| Primary scientific gate | Cue-specific appetitive and aversive conditioning before finance |
| Manual clock | One explicit `advanceDay()` per simulated day |

This is the V1 to implement. Changes to these choices are new experimental conditions and must be versioned rather than silently tuned.