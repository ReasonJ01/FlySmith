# FlySmith — V1 build specification

This document is the implementation contract for FlySmith V1. It supersedes unresolved choices in `PLAN.md`.

## 1. Objective

FlySmith connects a local Bondsmith-compatible savings environment to a whole-MaleCNS neural simulation. Each savings option is encoded as an artificial conditioned stimulus in Kenyon cells (KCs), propagated through the complete retained MaleCNS graph, valued from mushroom-body output neuron (MBON) activity, converted into one savings action, and later reinforced through dopamine-gated KC→MBON plasticity when the option's outcome resolves.

The V1 hypothesis is:

> A sparse artificial option representation imposed on MaleCNS KCs can acquire outcome-dependent value through compartment-specific dopamine-gated depression of existing KC→MBON synapses, producing reproducible changes in an MBON approach-minus-avoidance score that alter subsequent option selection.

No real money is involved. Time advances only when the local environment is explicitly advanced.

---

# 2. Pinned upstreams

## 2.1 MaleCNS dataset

Use the public neuPrint dataset:

```text
male-cns:v1.0
```

The exact exported source hashes, dataset UUID/version metadata and graph artifact hash must be persisted in every experiment manifest.

## 2.2 Numerical simulator baseline

Use the DoomFly whole-connectome implementation as the numerical reference and provenance source:

```text
repository: https://github.com/nftechie/doomfly
commit:     71ecf53d78eaffaf1a57ed7b0ccf5d458abc9f33
model:      adaptive-centered-v6
license:    MIT
```

Relevant upstream files at that commit:

```text
doom/prepare.py
doom/transmitters.py
doom/engine.py
doom/native.py
doom_learning_v6/brain.py
doom_learning_v6/kernel.cpp
doom_learning_v6/rule.py
doom_learning/circuit.py
```

FlySmith must vendor or port the required implementation at this exact commit. It must not import a moving upstream `main` at runtime.

If upstream code is copied or substantially adapted, retain the MIT copyright/license notice as required by the upstream license.

## 2.3 Graph construction

Follow `doom/prepare.py` at the pinned commit:

- retain every released directed edge, including weak and self edges;
- preserve MaleCNS source IDs as the canonical neuron IDs;
- represent the graph as CSR arrays;
- derive synaptic sign from the presynaptic neurotransmitter assumption used by the pinned upstream;
- initial edge weight is:

```text
weight = synapse_count × transmitter_sign × 0.275
```

Do not threshold or crop the graph for FlySmith.

A generated graph manifest must include SHA-256 hashes of `ids`, `ptr`, `post`, `weight`, annotation input and normalized source files. A run must refuse a graph whose hashes do not match its configured manifest.

---

# 3. Neural dynamics

V1 inherits the pinned `adaptive-centered-v6` numerical assumptions unless explicitly overridden here:

```text
dt                         0.1 ms
spike threshold            -45 mV
rest, non-KC               -52 mV
rest, KC                   -60 mV
membrane time constant      20 ms
synaptic time constant       5 ms
transmission delay           1.8 ms
refractory period            2.2 ms
KC adaptation jump           8 mV
KC adaptation tau          200 ms
```

The model is deterministic given graph, initial state and external currents. FlySmith therefore does not invent a neural random seed. Seeds are used for synthetic cue generation, random projections, bootstrap statistics and environment generation only.

### Important input correction

The pinned simulator accepts **external current**, not Poisson firing-rate input. FlySmith's artificial cue is therefore delivered by adding a constant current to selected KC indices for a defined interval.

Retinal and lamina drives are disabled for V1 option trials:

```text
luminance = all zeros
lamina_bias = 0
```

No financial information is injected anywhere except the selected KC cue. Other tonic/model currents must be fixed by configuration and hashed into the experiment manifest.

---

# 4. Repository/runtime split

Recommended implementation split:

```text
flysmith/
  fly/                 Python 3.12 + C++17 kernel
  experiment/          Python 3.12
  schemas/             JSON Schema 2020-12
  bondsmith-adapter/   implementation chosen by Bondsmith engineer
  config/
  outputs/
```

The fly core and Bondsmith integration communicate only through the adapter contract in section 13. The fly code must not call undocumented Bondsmith internals directly.

---

# 5. Neuron registry

## 5.1 Registry source of truth

Generate `config/neuron-registry.json` from the pinned MaleCNS export. Never hand-copy body IDs into source code.

Resolve against the normalized neuron table used to build the graph so that every `source_id` maps to exactly one simulation index.

## 5.2 Required populations

### Eligible KC pool

Include all neurons whose normalized `cell_type` begins with `KC`, except these visual/accessory-calyx classes:

```text
KCg-d
KCab-p
```

If a future pinned export introduces additional KC classes, generation must print them and fail until the allow/exclude list is explicitly reviewed.

### Primary readouts

```text
MBON11   # MBON-gamma1pedc>alpha/beta; approach-side readout
MBON01   # MBON-gamma5beta'2a; avoidance-side readout
```

### Secondary validation readouts

```text
MBON02
MBON03
```

### Teaching populations

```text
PPL101   # negative teaching; PPL1-gamma1pedc
PAM02    # positive teaching; PAM-beta'2a
PAM01    # validation/ablation only in V1
```

The current MaleCNS v1.0 public explorer reports two MBON01 neurons, bilateral MBON11, two PPL101 neurons, and 17 PAM02 neurons; registry generation must nevertheless derive the exact IDs from the pinned export and treat these counts as cross-checks rather than hard-coded identity.

## 5.3 Registry schema

The generated file must conform to `schemas/neuron-registry.schema.json` and have this logical shape:

```json
{
  "schemaVersion": 1,
  "dataset": {
    "name": "male-cns:v1.0",
    "uuid": "<resolved-at-build>",
    "sourceManifestSha256": "<sha256>"
  },
  "graph": {
    "neuronCount": 166700,
    "edgeCount": 25582938,
    "idsSha256": "<sha256>",
    "ptrSha256": "<sha256>",
    "postSha256": "<sha256>",
    "weightSha256": "<sha256>"
  },
  "populations": {
    "kcEligible": {
      "selector": "cell_type startsWith KC; exclude KCg-d,KCab-p",
      "bodyIds": [1],
      "simulationIndices": [0]
    },
    "mbon11": { "cellType": "MBON11", "bodyIds": [1], "simulationIndices": [0] },
    "mbon01": { "cellType": "MBON01", "bodyIds": [1], "simulationIndices": [0] },
    "mbon02": { "cellType": "MBON02", "bodyIds": [1], "simulationIndices": [0] },
    "mbon03": { "cellType": "MBON03", "bodyIds": [1], "simulationIndices": [0] },
    "ppl101": { "cellType": "PPL101", "bodyIds": [1], "simulationIndices": [0] },
    "pam02": { "cellType": "PAM02", "bodyIds": [1], "simulationIndices": [0] },
    "pam01": { "cellType": "PAM01", "bodyIds": [1], "simulationIndices": [0] }
  },
  "plasticEdges": {
    "negative": [{ "preIndex": 0, "postIndex": 0, "edgeIndex": 0, "structuralWeight": 1.0 }],
    "positive": [{ "preIndex": 0, "postIndex": 0, "edgeIndex": 0, "structuralWeight": 1.0 }]
  }
}
```

`plasticEdges.negative` is every existing eligible-KC → MBON11 edge.

`plasticEdges.positive` is every existing eligible-KC → MBON01 edge.

Generation must fail if:

- any required population resolves to zero neurons;
- a body ID is missing from the simulation graph;
- any listed simulation index maps back to a different body ID;
- either plastic edge set is empty;
- a plastic edge's post neuron is outside its declared MBON population;
- graph hashes differ from the pinned graph manifest.

---

# 6. Financial observation

Every destination, including cash, is represented as exactly four normalized observations:

```text
yield
lock
liquidity
exposure
```

The Bondsmith adapter exposes integer financial values. The neural encoder never receives pounds, product names or JSON objects directly.

For a candidate allocation amount `a`:

```text
yield     = clip((annualRateBps - minRateBps) / (maxRateBps - minRateBps), 0, 1)
lock      = clip(effectiveLockDays / maxLockDays, 0, 1)
liquidity = clip((cashBalanceMinor - a) / totalAssetsMinor, 0, 1)
exposure  = clip((destinationBalanceMinor + a) / totalAssetsMinor, 0, 1)
```

For the cash option:

```text
a = 0
effectiveLockDays = 0
destinationBalanceMinor = cashBalanceMinor
```

All normalization bounds come from the experiment config and are frozen before a run. They must not be tuned after observing portfolio performance.

Use integer minor currency units and integer basis points at the environment boundary; do not use floating-point money.

---

# 7. Option feature → KC cue

## 7.1 Exact projection

Let:

```text
x = [yield, lock, liquidity, exposure]
u = 2x - 1
```

Generate a fixed matrix and bias once per experiment configuration using NumPy `PCG64`:

```python
rng = np.random.Generator(np.random.PCG64(projectionSeed))
W = rng.normal(0.0, 0.5, size=(K, 4))
b = rng.normal(0.0, 0.25, size=K)
z = W @ u + b
```

where `K` is the number of eligible KCs in the generated registry.

Select:

```text
k = ceil(kcSparsity × K)
```

KCs with the largest `k` values of `z`. Ties are broken by ascending simulation index.

V1 default:

```text
kcSparsity = 0.05
projectionSeed = 20260910
```

Persist the selected simulation indices and a SHA-256 cue hash with every decision. Delayed teaching replays the stored indices, not a recomputed projection.

## 7.2 Artificial cue semantics

No KC means “yield”, “lock”, or any other financial feature. The fixed random projection is the only artificial representational boundary. Similar feature vectors tend to produce overlapping sparse KC ensembles; the downstream connectome and plasticity assign value.

---

# 8. Trial protocol

## 8.1 Day checkpoint

At the start of a decision day, create one immutable `day-start` neural checkpoint containing:

- all membrane/synaptic/delay/adaptation state;
- all current learned efficacy values;
- graph/configuration hashes.

Every option trial for that day starts from the exact same checkpoint.

No option trial is allowed to mutate the day's persistent learned weights.

## 8.2 Blank baseline

From the day-start checkpoint, run one blank trial with:

```text
KC option current = 0
retina = 0
lamina_bias = 0
learning = false
```

for `trialDurationMs`.

Record MBON01/02/03/11 rates.

## 8.3 Option trial

Restore the day-start checkpoint, stimulate the option's selected KC ensemble with constant `kcCurrent` for `trialDurationMs`, disable learning, then record spike counts for all neurons and primary/secondary MBON rates.

V1 starting duration:

```text
trialDurationMs = 500
```

The calibrated value replaces this default in `config/calibration.json`.

## 8.4 Rate computation

For population `P`:

```text
rateHz(P) = sum(spikes_i for i in P) / |P| / trialSeconds
```

Baseline-subtracted response:

```text
deltaHz(P) = optionRateHz(P) - blankRateHz(P)
```

---

# 9. MBON value decoder

Calibration produces frozen naive statistics:

```text
mu11, sigma11
mu01, sigma01
```

from the synthetic cue panel in Gate B.

For an option:

```text
approachZ  = (deltaHz(MBON11) - mu11) / sigma11
avoidanceZ = (deltaHz(MBON01) - mu01) / sigma01
score       = approachZ - avoidanceZ
```

If either sigma is below `0.5 Hz`, Gate B fails and the financial loop must not run.

MBON02 and MBON03 are logged but not included in the primary V1 score.

## 9.1 Choice rule

Evaluate cash and every valid product independently.

Let `bestProduct` be the valid product with maximum score. Product ties within `1e-9` are broken lexicographically by `productId`.

V1 action rule:

```text
if no valid product:
    HOLD
else if score(bestProduct) < score(cash) + indifferenceMarginZ:
    HOLD
else:
    ALLOCATE configured tranche to bestProduct
```

Default:

```text
indifferenceMarginZ = 0.5
```

Position size is not neural in V1.

---

# 10. Plasticity

## 10.1 Structural vs learned weight

Every plastic edge stores:

```text
structuralWeight = original graph weight
efficacy         = 1.0 initially
effectiveWeight  = structuralWeight × efficacy
```

Only efficacy is mutable.

V1 bounds:

```text
0.10 <= efficacy <= 1.00
```

There is no potentiation and no topology change in V1.

## 10.2 Negative teaching

A negative outcome uses `PPL101` and may modify only eligible-KC → MBON11 edges.

## 10.3 Positive teaching

A positive outcome uses `PAM02` and may modify only eligible-KC → MBON01 edges.

`PAM01` is a validation/ablation population only and is not part of the primary learning rule.

## 10.4 Teaching episode

For a resolved allocation, replay the exact stored KC cue and apply the appropriate DAN current using this schedule:

```text
0–100 ms     cue only
100–300 ms   cue + DAN, learning enabled
300–400 ms   neither, learning disabled
```

Start the episode from the persistent neural checkpoint at the point teaching is applied. After the episode, keep learned efficacies but clear transient membrane, delay-queue, eligibility, modulation and adaptation state before the next financial decision. This makes memory reside only in the declared plastic weights across simulated days.

## 10.5 Learning update

For each eligible plastic edge whose presynaptic KC fired during the teaching window:

```text
kcActivity = min(kcRateHz / kcActivityReferenceHz, 1)
danActivity = min(danPopulationRateHz / danActivityReferenceHz, 1)

efficacy *= exp(-eta × teachingMagnitude × kcActivity × danActivity)
efficacy = max(0.10, efficacy)
```

Defaults before calibration:

```text
kcActivityReferenceHz  = 50
danActivityReferenceHz = 50
eta                     = 0.05
```

`eta` is calibrated only against synthetic conditioning Gates C/D, never against financial return.

This is a FlySmith-specific, deliberately conservative compartment rule. It uses the experimentally supported depression direction but does not claim to reproduce the complete biological plasticity rule.

---

# 11. Outcome and reward semantics

Each allocation stores the cue active when it was chosen.

When an allocation resolves, the environment creates:

```text
interestReward   in [0, 1]
liquidityPenalty in [-1, 0]
total            in [-1, 1]
```

## 11.1 Positive component

```text
interestReward = clip(
  interestCreditedMinor / interestReferenceMinor,
  0,
  1
)
```

Default:

```text
interestReferenceMinor = ceil(trancheMinor × 0.005)
```

That is a 50-basis-point-of-principal reference for the deliberately compressed short-term experiment. It is an experimental scale, not an AER interpretation.

## 11.2 Liquidity component

Liquidity shocks are FlySmith environment events. If a shock produces `shortfallMinor > 0`, distribute a negative penalty across currently locked allocations pro rata by locked principal:

```text
allocationShare = lockedPrincipalMinor / totalLockedPrincipalMinor
allocationShortfall = shortfallMinor × allocationShare
liquidityPenalty = -clip(allocationShortfall / trancheMinor, 0, 1)
```

## 11.3 Total and teaching magnitude

```text
total = clip(interestReward + liquidityPenalty, -1, 1)
teachingMagnitude = abs(total)
```

```text
total > 0  -> PAM02 teaching on stored cue
total < 0  -> PPL101 teaching on stored cue
total = 0  -> no teaching
```

A HOLD/cash decision creates no plastic teaching event in V1 unless a later explicit extension defines one.

---

# 12. Calibration and mandatory gates

Calibration is a build/run prerequisite. It writes `config/calibration.json`; production experiment commands must fail if that file is absent or its graph/config hashes are stale.

Use NumPy PCG64 seed `20260910` to generate a fixed panel of 64 feature vectors uniformly on `[0,1]^4` for Gates A/B. Generate 32 conditioning cue pairs from the same RNG stream, rejecting pairs whose KC-set Jaccard overlap exceeds `0.25`.

Use nonparametric bootstrap confidence intervals with:

```text
bootstrap resamples = 10,000
bootstrap seed      = 9001
confidence interval = percentile 95%
```

## Gate A — KC stimulus propagation and saturation

Test candidate KC currents:

```text
[16, 18, 20, 24, 30]
```

mV-equivalent external drive units inherited from the pinned simulator.

For each current and all 64 cues, run a 500 ms cue trial from a reset naive state.

A candidate current passes if all are true:

1. median stimulated-KC firing rate is `>= 5 Hz` and `<= 80 Hz`;
2. on at least 90% of cues, at least 90% of stimulated KCs fire at least once;
3. median fraction of non-KC neurons firing at least once is `< 0.25`;
4. 95th percentile of that non-KC active fraction is `< 0.40`.

Choose the **lowest** candidate current that passes. If none pass, Gate A fails.

## Gate B — MBON readout

Using the selected KC current and the same 64 cues:

1. run blank + cue trials;
2. compute `deltaHz(MBON11)` and `deltaHz(MBON01)`;
3. require standard deviation of each channel across cues `>= 0.5 Hz`;
4. require at least 75% of cues to evoke at least one spike in each primary MBON population during the cue trial;
5. require the resulting score distribution IQR `>= 0.5`.

If any criterion fails, Gate B fails. Store `mu11`, `sigma11`, `mu01`, `sigma01` from this panel.

## DAN-current calibration

Before Gates C/D, independently calibrate PPL101 and PAM02 current using candidates:

```text
[8, 12, 16, 20, 24, 30]
```

for the 200 ms teaching window. Choose the lowest current producing a DAN population rate between `20 and 80 Hz` without pushing the non-KC active fraction above `0.40`.

If either DAN population cannot meet this criterion, conditioning gates fail.

## Gate C — aversive conditioning

For each of 32 cue pairs `(X,Y)`:

1. measure pre-training scores for X and Y;
2. perform 5 X + PPL101 teaching episodes at `teachingMagnitude=1`;
3. measure post-training X and Y;
4. run a matched sham condition with the same cue schedule but no DAN current.

For each pair:

```text
deltaX = postX - preX
deltaY = postY - preY
shamDeltaX = shamPostX - shamPreX
```

Gate C passes if:

- median `deltaX <= -0.5` score units;
- upper bound of the bootstrap 95% CI for `median(deltaX - shamDeltaX)` is `< 0`.

## Gate D — appetitive conditioning

Repeat Gate C using PAM02 teaching.

Gate D passes if:

- median `deltaX >= +0.5` score units;
- lower bound of the bootstrap 95% CI for `median(deltaX - shamDeltaX)` is `> 0`.

## Gate E — cue specificity

Evaluate the same conditioning runs.

For aversive and appetitive conditioning separately define:

```text
specificity = abs(deltaX) - abs(deltaY)
```

Gate E passes only if, for both teaching directions:

- median specificity `>= 0.5`;
- lower bound of the bootstrap 95% CI of median specificity is `> 0`.

## Gate F — reset reversibility

After conditioning:

1. reset all plastic efficacies to exactly `1.0`;
2. clear all transient neural state;
3. rerun all pre-training cue trials.

Gate F passes if:

- plastic-efficacy SHA-256 equals the original naive efficacy hash;
- every primary MBON rate differs from its corresponding pre-training value by at most `1e-6 Hz` on the same machine/build;
- every decoded score differs by at most `1e-6`.

No financial experiment may be described as learned behavior unless Gates A–F all pass for the exact graph, simulator build and calibration manifest used by that experiment.

---

# 13. Bondsmith adapter contract

The Bondsmith-specific engineer owns a local adapter. The fly core depends only on the following JSON-over-HTTP contract.

Base URL is configured as `bondsmithAdapterBaseUrl`. The adapter is expected to bind to loopback/local development infrastructure only in V1.

## 13.1 `GET /flysmith/v1/state`

Response:

```json
{
  "day": 7,
  "currency": "GBP",
  "cashBalanceMinor": 400000,
  "totalAssetsMinor": 1000000,
  "products": [
    {
      "productId": "fixed-3",
      "annualRateBps": 450,
      "effectiveLockDays": 3,
      "balanceMinor": 200000,
      "minDepositMinor": 1000,
      "maxDepositMinor": null,
      "canDeposit": true
    }
  ],
  "stateVersion": "opaque-monotonic-version"
}
```

Requirements:

- amounts are integer minor units;
- rates are integer basis points;
- `effectiveLockDays` is the worst-case days until funds are usable under the local product semantics;
- `stateVersion` changes whenever any value relevant to a decision changes.

## 13.2 `POST /flysmith/v1/actions`

Request:

```json
{
  "requestId": "uuid",
  "expectedStateVersion": "opaque-monotonic-version",
  "action": {
    "type": "allocate",
    "productId": "fixed-3",
    "amountMinor": 100000
  }
}
```

or:

```json
{
  "requestId": "uuid",
  "expectedStateVersion": "opaque-monotonic-version",
  "action": { "type": "hold" }
}
```

Response:

```json
{
  "requestId": "uuid",
  "accepted": true,
  "allocationId": "alloc-123",
  "newStateVersion": "opaque-monotonic-version"
}
```

The adapter must make `requestId` idempotent. If `expectedStateVersion` is stale, it must reject rather than silently apply against different state.

## 13.3 `POST /flysmith/v1/advance`

Request:

```json
{
  "requestId": "uuid",
  "days": 1,
  "expectedStateVersion": "opaque-monotonic-version"
}
```

V1 only permits `days = 1`.

Response:

```json
{
  "day": 8,
  "newStateVersion": "opaque-monotonic-version",
  "events": [
    {
      "type": "maturity",
      "allocationId": "alloc-123",
      "principalMinor": 100000,
      "interestCreditedMinor": 120
    }
  ]
}
```

Allowed event types in V1:

```text
maturity
withdrawal
rate_change
product_opened
product_closed
```

FlySmith liquidity-shock events are defined in the experiment scenario, not invented by the Bondsmith adapter.

## 13.4 Adapter conformance

Before neural integration, the adapter must pass contract tests for:

- integer money/rate fields;
- idempotent action/advance request IDs;
- stale-state rejection;
- monotonic day advancement by exactly one;
- maturity event reconciliation to an existing `allocationId`;
- repeatable reset to a configured initial fixture if the local Bondsmith stack supports reset.

---

# 14. Experiment configuration

Every run loads one immutable document conforming to `schemas/experiment-config.schema.json`.

Canonical example:

```json
{
  "schemaVersion": 1,
  "experimentId": "baseline-001",
  "maleCnsDataset": "male-cns:v1.0",
  "simulator": {
    "upstreamRepo": "nftechie/doomfly",
    "upstreamCommit": "71ecf53d78eaffaf1a57ed7b0ccf5d458abc9f33",
    "model": "adaptive-centered-v6",
    "dtMs": 0.1
  },
  "projection": {
    "seed": 20260910,
    "kcSparsity": 0.05
  },
  "normalization": {
    "minRateBps": 0,
    "maxRateBps": 1000,
    "maxLockDays": 10
  },
  "decision": {
    "trancheMinor": 100000,
    "indifferenceMarginZ": 0.5
  },
  "learning": {
    "enabled": true,
    "eta": 0.05,
    "efficacyFloor": 0.1,
    "kcActivityReferenceHz": 50,
    "danActivityReferenceHz": 50,
    "conditioningRepeats": 5
  },
  "reward": {
    "interestReferenceFractionOfTranche": 0.005
  },
  "scenario": {
    "liquidityShocks": []
  },
  "bondsmithAdapterBaseUrl": "http://127.0.0.1:8088"
}
```

The run manifest must add immutable hashes for the registry, graph, calibration, executable/kernel and experiment config.

---

# 15. Trial and run records

Every option evaluation writes one JSONL record conforming to `schemas/trial-record.schema.json` containing at minimum:

```text
experimentId
runId
day
stateVersion
optionId
features
cueHash
cueIndices
blank MBON rates
option MBON rates
baseline-subtracted MBON rates
score
registry hash
graph hash
calibration hash
plasticity hash before trial
plasticity hash after trial
```

Decision records must additionally contain all option scores, selected action, action receipt and tie/indifference reasoning.

Teaching records must contain allocation ID, stored cue hash, resolved outcome components, selected DAN population, DAN rate, teaching magnitude, changed-edge count and before/after plasticity hashes.

A run is not considered reproducible unless all three record classes are present.

---

# 16. State lifecycle

There are three distinct state categories and implementations must keep them separate.

### Structural state — immutable

```text
graph topology
structural synaptic weights
neuron registry
projection matrix seed/config
```

### Learned state — persistent across days

```text
KC→MBON01 efficacy multipliers
KC→MBON11 efficacy multipliers
```

### Transient neural state — never carried across financial days in V1

```text
membrane voltage
synaptic conductance
delay queues
refractory counters
KC adaptation
eligibility traces
modulator traces
spike counters
```

After each option assay and after each teaching episode, transient state is restored/cleared according to the protocol. Only learned efficacy is persistent.

This rule is mandatory because it makes a multi-day preference change attributable to the declared memory state rather than residual electrical activity.

---

# 17. Daily execution algorithm

```text
1. GET adapter state.
2. Build cash option plus every product with canDeposit=true and a valid tranche.
3. Compute four features for each option.
4. Convert each feature vector into a stored sparse KC cue.
5. Snapshot persistent learned efficacy + reset transient neural state.
6. Run blank baseline.
7. For every option:
     restore identical day-start neural state
     run cue trial with learning disabled
     compute MBON score
8. Select action using section 9.1.
9. POST action with expected stateVersion.
10. Persist decision and receipt.
11. Stop. Do not advance time automatically.

On explicit ADVANCE DAY:

12. POST /advance with days=1.
13. Apply configured FlySmith liquidity shock for the new day, if any.
14. Convert maturity/liquidity consequences into per-allocation outcomes.
15. For each nonzero resolved outcome:
      load stored cue
      execute PAM02 or PPL101 teaching episode
      persist plasticity diff/hash
16. Clear transient neural state.
17. Persist new day state and stop.
```

Teaching events resolving on the same day are processed in ascending `allocationId` order to make results deterministic.

---

# 18. Required commands

The first implementation should expose these commands, names may be CLI aliases but behavior is fixed:

```text
flysmith prepare-graph
flysmith build-registry
flysmith calibrate
flysmith verify-gates
flysmith adapter-check
flysmith decide
flysmith advance-day
flysmith reset-learning
flysmith replay <runId>
```

`decide` and `advance-day` must refuse execution when `verify-gates` has not passed for the exact current provenance hashes.

---

# 19. Acceptance tests

A build is V1-complete when all are true:

1. graph preparation reproduces and hashes the pinned whole-connectome artifact;
2. registry generation resolves all required populations and plastic edge sets;
3. cue projection is deterministic and stored cues replay exactly;
4. Gates A–F pass and produce a signed calibration manifest;
5. the in-memory test adapter passes the exact Bondsmith-port contract;
6. the real local Bondsmith adapter passes the same contract tests unchanged;
7. a 30-day fixture can be reset and replayed with identical actions, MBON scores and plasticity hashes;
8. fixed-weight, sham-learning, trained, reset-learning, random-choice, highest-yield and cash-only controls can run against the same financial history;
9. no source code path can alter structural connectivity during training;
10. every claimed learned result can be traced from environment outcome → stored cue → DAN teaching → changed KC→MBON efficacy → changed later MBON score → changed action.

---

# 20. Work split

## Neural/simulation engineer

Owns:

```text
graph preparation
upstream kernel port/vendor
neuron registry
KC projection
trial runner
MBON decoder
plasticity
calibration gates
checkpoint/replay
```

## Bondsmith engineer

Owns:

```text
/flysmith/v1/state
/flysmith/v1/actions
/flysmith/v1/advance
fixture/reset support
allocation/maturity reconciliation
adapter contract tests
```

## Experiment layer

Owns:

```text
normalization
option construction
liquidity-shock scenarios
outcome attribution
run manifests
controls
reports
```

These workstreams can proceed in parallel after the JSON schemas are committed.

---

# 21. Controls required for any result

Every reported training result must include, against the same environment history:

- fixed-weight MaleCNS;
- plastic MaleCNS + sham DAN;
- trained plastic MaleCNS;
- trained then efficacy-reset MaleCNS;
- cue-unpaired DAN control;
- PPL101-disabled negative-teaching control;
- PAM02-disabled positive-teaching control;
- MBON11 readout ablation/control;
- MBON01 readout ablation/control;
- alternate projection seed control;
- random-choice policy;
- highest-yield policy;
- cash-only policy.

Portfolio performance is secondary. The primary V1 result is successful, specific, reversible conditioned change in MBON value and resulting action selection.

---

# 22. Implementation decisions that are closed

The following are no longer open questions for V1:

```text
connectome          MaleCNS male-cns:v1.0
simulator baseline  DoomFly adaptive-centered-v6 @ 71ecf53d...
graph scope         complete retained graph
financial encoding  fixed sparse random KC code
primary readout     MBON11 - MBON01 standardized valence
negative teacher    PPL101
positive teacher    PAM02
plastic locus -     eligible KC -> MBON11
plastic locus +     eligible KC -> MBON01
plasticity          depression-only efficacy multiplier
credit assignment   exact cue replay at outcome resolution
position size       fixed external tranche
clock                manual one-day advancement
persistent memory    declared efficacy multipliers only
```

Remaining values such as selected KC/DAN current and final decoder normalization are **calibration outputs generated by the mandatory protocol**, not implementation choices left to individual engineers.

---

# 23. References and provenance

1. MaleCNS public neuPrint dataset `male-cns:v1.0`; natverse access tooling: https://natverse.org/malecns/
2. DoomFly repository, pinned simulator/reference implementation: https://github.com/nftechie/doomfly/tree/71ecf53d78eaffaf1a57ed7b0ccf5d458abc9f33
3. DoomFly `doom/prepare.py` at pinned commit: complete-edge CSR build and synaptic-weight/sign assumptions.
4. DoomFly `doom_learning_v6/brain.py` and `kernel.cpp` at pinned commit: adaptive-centered-v6 dynamics and explicit current stimulation.
5. Aso et al. (2014), *The neuronal architecture of the mushroom body provides a logic for associative learning*, eLife 3:e04577. https://doi.org/10.7554/eLife.04577
6. Aso et al. (2014), *Mushroom body output neurons encode valence and guide memory-based action selection in Drosophila*, eLife 3:e04580. https://doi.org/10.7554/eLife.04580
7. Owald et al. (2015), *Activity of Defined Mushroom Body Output Neurons Underlies Learned Olfactory Behavior in Drosophila*, Neuron 86:417–427. https://doi.org/10.1016/j.neuron.2015.03.025
8. Takemura et al. (2017), *A connectome of a learning and memory center in the adult Drosophila brain*, eLife 6:e26975. https://doi.org/10.7554/eLife.26975
9. Li et al. (2020), *The connectome of the adult Drosophila mushroom body provides insights into function*, eLife 9:e62576. https://doi.org/10.7554/eLife.62576
10. Springer et al. (2021), dopamine/reward-encoding mechanisms in Drosophila mushroom-body compartments, Nature Communications 12:1115. https://doi.org/10.1038/s41467-021-21388-w
11. Bondsmith developer portal: https://developers.bondsmith.com/

The numerical simulator, artificial financial encoding, current amplitudes and compressed short-term economy are model assumptions. MaleCNS supplies measured wiring; FlySmith does not represent the resulting LIF system as a validated living fly.