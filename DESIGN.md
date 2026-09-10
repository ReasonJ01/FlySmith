# FlySmith — V1 build specification

This is the implementation source of truth for FlySmith V1. `FINAL_DESIGN.md` records the research rationale; where wording or provisional parameters differ, this file governs implementation.

## 1. Objective

FlySmith connects a local Bondsmith-compatible short-term savings environment to a complete MaleCNS neural simulation.

```text
Bondsmith-compatible state
        ↓
financial option features
        ↓
deterministic sparse KC conditioned stimulus
        ↓
complete MaleCNS recurrent simulation
        ↓
MBON11 approach / MBON01 avoidance readout
        ↓
option score
        ↓
HOLD or fixed-tranche ALLOCATE
        ↓
realized short-term outcome
        ↓
exact cue replay + DAN teaching
        ↓
restricted KC→MBON efficacy change
```

V1 tests this falsifiable hypothesis:

> A sparse artificial option representation imposed on MaleCNS Kenyon cells can acquire outcome-dependent value through compartment-specific dopamine-gated depression of existing KC→MBON synapses, producing specific and reversible changes in an MBON approach-minus-avoidance score that alter later option selection.

No real money is involved. Simulated time moves only after an explicit one-day advance command.

---

## 2. Pinned neural provenance

### 2.1 Connectome

Use:

```text
MaleCNS dataset: male-cns:v1.0
```

Persist the MaleCNS dataset UUID/version metadata and hashes of every normalized source artifact used to build the graph.

### 2.2 Numerical reference

Use this exact DoomFly revision as the numerical reference:

```text
repo:    nftechie/doomfly
commit:  71ecf53d78eaffaf1a57ed7b0ccf5d458abc9f33
model:   adaptive-centered-v6 numerical dynamics
license: MIT
```

Reference files:

```text
doom/prepare.py
doom/transmitters.py
doom/engine.py
doom/native.py
doom_learning_v6/brain.py
doom_learning_v6/kernel.cpp
doom_learning/circuit.py
```

Implementation rule: vendor the required upstream source at that commit under `vendor/doomfly/` and preserve its MIT notice. Do not import or clone a moving upstream branch at runtime.

FlySmith uses the upstream graph construction and numerical integration assumptions, but implements its own explicit two-compartment teaching rule. The DoomFly learning rule is a reference, not FlySmith's policy.

### 2.3 Graph construction

Match pinned `doom/prepare.py`:

- keep every released directed connection row, including weak and self edges;
- use MaleCNS source IDs as canonical neuron IDs;
- store the graph as CSR arrays;
- infer sign from the pinned presynaptic neurotransmitter mapping;
- calculate initial weight as

```text
weight = synapse_count × transmitter_sign × 0.275
```

Do not prune the graph to the mushroom body.

The generated graph manifest must hash at least `ids`, `ptr`, `post`, `weight`, annotations and normalized source inputs. FlySmith must refuse to run if the current artifacts do not match the manifest.

---

## 3. Numerical model

Use the pinned baseline constants:

```text
dt                         0.1 ms
spike threshold            -45 mV
rest, ordinary neuron      -52 mV
rest, KC                   -60 mV
membrane tau                20 ms
synaptic tau                  5 ms
transmission delay           1.8 ms
refractory period            2.2 ms
KC adaptation jump             8 mV
KC adaptation tau            200 ms
```

The chosen engine accepts **constant external current**, not Poisson-Hz stimulation. Financial cues and teaching pulses are therefore current injections into explicitly registered neurons.

For V1 option and teaching assays:

```text
retinal luminance = 0
lamina_bias       = 0
background tonic  = 0 for every neuron
```

The model is deterministic for a fixed graph, transient state, learned state and external-current schedule. V1 has no neural random seed. Seeds exist only for the synthetic KC projection, calibration cue generation, bootstrapping and optional environment fixtures.

---

## 4. State categories

Implement these as separate state classes/files. They must never be silently conflated.

### Structural — immutable

```text
MaleCNS graph topology
structural graph weights
neuron registry
projection algorithm + seed
simulator source/build hashes
```

### Learned — persistent between simulated days

```text
efficacy multipliers on registered KC→MBON11 edges
efficacy multipliers on registered KC→MBON01 edges
```

### Transient neural — cleared/restored by protocol

```text
membrane voltages
synaptic conductances
refractory counters
delay queues
KC adaptation
eligibility/modulator traces
spike counters
external drive
```

Across financial days, **only learned efficacy persists**. This makes persistent behavioral change attributable to the declared memory state rather than residual electrical activity.

---

## 5. Neuron registry

Generate `config/neuron-registry.json`. It must validate against `schemas/neuron-registry.schema.json`.

### 5.1 Required typed populations

Resolve exact MaleCNS `cell_type` values:

```text
MBON11   primary approach-side readout; MBON-γ1pedc>α/β
MBON01   primary avoidance-side readout; MBON-γ5β′2a
MBON02   secondary validation readout
MBON03   secondary validation readout
PPL101   negative teacher; PPL1-γ1pedc
PAM15    positive teacher; PAM-γ5β′2a
```

PAM15 is deliberately chosen over PAM02 because PAM15 spans the γ5/β′2a compartment associated with MBON01, while PAM02 is the β′2a DAN type only.

Do not hard-code body IDs from papers, other connectomes or other projects. Resolve them from the pinned MaleCNS export and map each source ID to its simulation index.

### 5.2 Eligible KC pool

A neuron is `kcEligible` iff:

1. its normalized `cell_type` starts with `KC`; and
2. it has at least one retained outgoing structural edge to a registered MBON01 or MBON11 neuron.

This removes KCs that cannot participate directly in either V1 memory compartment and avoids an arbitrary sensory-subtype allowlist.

Record aligned `rootSide`/hemisphere metadata for KCs. Only KCs resolving unambiguously to `L` or `R` are eligible in V1. Registry generation fails if fewer than 100 eligible KCs exist on either side.

### 5.3 Plastic edge sets

```text
negative plastic set = every existing kcEligible → MBON11 edge
positive plastic set = every existing kcEligible → MBON01 edge
```

The graph's structural weight remains immutable. Each plastic edge gets an independent efficacy multiplier initialized to `1.0`.

### 5.4 Registry validation

Generation must fail if:

- any required typed population resolves to zero cells;
- any source ID is absent from the simulation graph;
- any simulation index maps back to a different source ID;
- either KC hemisphere has fewer than 100 eligible cells;
- either plastic edge set is empty;
- a negative plastic edge does not terminate in MBON11;
- a positive plastic edge does not terminate in MBON01;
- graph/annotation provenance hashes differ from the pinned manifest.

---

## 6. Financial observation contract

Every destination, including cash, becomes four observations:

```text
yield
lock
liquidity
exposure
```

At the adapter boundary money is integer minor units and rates are integer basis points. Never use floating-point currency.

For a candidate allocation of `a` minor units:

```text
yield     = clip((annualRateBps - minRateBps) / (maxRateBps - minRateBps), 0, 1)
lock      = clip(effectiveLockDays / maxLockDays, 0, 1)
liquidity = clip((cashBalanceMinor - a) / totalAssetsMinor, 0, 1)
exposure  = clip((destinationBalanceMinor + a) / totalAssetsMinor, 0, 1)
```

Cash is evaluated as an option with:

```text
a = 0
annualRateBps = configured cash rate, default 0
effectiveLockDays = 0
destinationBalanceMinor = cashBalanceMinor
```

Normalization bounds are experiment configuration, frozen before a run. They may not be recomputed from the products available on a particular day.

---

## 7. Exact feature → KC encoding

Let:

```text
x = [yield, lock, liquidity, exposure]
u = 2x - 1
```

For **each hemisphere separately**, enumerate eligible KCs in ascending simulation-index order and create a fixed projection using NumPy PCG64:

```python
rng = np.random.Generator(np.random.PCG64(projection_seed))
W_L = rng.normal(0.0, 0.5, size=(K_L, 4))
b_L = rng.normal(0.0, 0.25, size=K_L)
W_R = rng.normal(0.0, 0.5, size=(K_R, 4))
b_R = rng.normal(0.0, 0.25, size=K_R)

z_L = W_L @ u + b_L
z_R = W_R @ u + b_R
```

Select independently per hemisphere:

```text
k_L = ceil(kcSparsity × K_L)
k_R = ceil(kcSparsity × K_R)
```

using the largest `z` values; exact ties break by ascending simulation index.

V1 defaults:

```text
projectionSeed = 20260910
kcSparsity     = 0.05
```

The bilateral cue is the union of selected L and R indices. Store the exact indices and SHA-256 cue hash with every action. Delayed teaching replays stored indices; it does not recompute them.

No KC is assigned a semantic label such as “rate neuron.” The artificial interface is the distributed conditioned stimulus itself.

---

## 8. Decision-day neural protocol

### 8.1 Day-start state

At the beginning of each day create an immutable neural snapshot containing the current learned efficacies with all transient state reset to canonical resting values.

Every blank and option assay on that day starts from this exact snapshot.

### 8.2 Blank assay

Restore day-start state and run for calibrated `trialDurationMs` with:

```text
option current = 0
learning       = false
retina         = 0
lamina bias    = 0
tonic          = 0
```

Record MBON01/02/03/11 spike counts and rates.

### 8.3 Option assay

Restore day-start state. Inject calibrated constant `kcCurrent` into exactly the cue KCs for calibrated `trialDurationMs`; learning remains disabled. Record full spike counts plus MBON rates.

Starting duration before calibration:

```text
500 ms
```

### 8.4 Population rate

For registered population `P`:

```text
rateHz(P) = total spikes in P / number of cells in P / trial seconds
deltaHz(P) = option rateHz(P) - blank rateHz(P)
```

Option assays must not mutate persistent efficacy.

---

## 9. Neural value decoder

Gate B freezes naive calibration statistics:

```text
mu11, sigma11 for deltaHz(MBON11)
mu01, sigma01 for deltaHz(MBON01)
```

For an option:

```text
approachZ  = (deltaHz(MBON11) - mu11) / sigma11
avoidanceZ = (deltaHz(MBON01) - mu01) / sigma01
score       = approachZ - avoidanceZ
```

MBON02 and MBON03 are always logged but are not part of V1's primary decoder.

If `sigma11 < 0.5 Hz` or `sigma01 < 0.5 Hz`, Gate B fails.

### Choice

Evaluate cash plus every valid product independently from the same day-start state.

Let `bestProduct` be the product with maximum score. Exact/tolerance ties within `1e-9` break lexicographically by `productId`.

```text
if no valid product:
    HOLD
elif score(bestProduct) < score(cash) + indifferenceMarginZ:
    HOLD
else:
    ALLOCATE trancheMinor to bestProduct
```

Default:

```text
indifferenceMarginZ = 0.5
```

V1 neural activity selects destination only; position size is fixed by configuration.

---

## 10. Learning mechanism

### 10.1 Positive reinforcement

Teacher:

```text
PAM15 / PAM-γ5β′2a
```

Mutable edges:

```text
kcEligible → MBON01
```

Positive teaching depresses the active cue's contribution to the avoidance-side output.

### 10.2 Negative reinforcement

Teacher:

```text
PPL101 / PPL1-γ1pedc
```

Mutable edges:

```text
kcEligible → MBON11
```

Negative teaching depresses the active cue's contribution to the approach-side output.

### 10.3 Weight representation

For every registered plastic edge:

```text
structuralWeight = immutable original graph weight
efficacy         = 1.0 initially
effectiveWeight  = structuralWeight × efficacy
```

Bounds:

```text
0.10 <= efficacy <= 1.00
```

No potentiation, new edge or topology change exists in V1.

### 10.4 Explicit teaching episode

When a non-neutral outcome resolves, replay the exact stored cue:

```text
0–100 ms     cue current only
100–300 ms   cue current + teacher-DAN current; plasticity armed
300–400 ms   no artificial current; plasticity disarmed
```

Only explicit teacher stimulation can arm FlySmith plasticity. Endogenous modeled dopamine activity never changes FlySmith efficacy by itself.

After the episode preserve efficacy, then reset **all transient neural state** before any subsequent decision or teaching episode.

### 10.5 FlySmith plasticity update

During the 200 ms paired window calculate measured rates for every active cue KC and the teacher population:

```text
kcActivity  = min(kcRateHz / kcActivityReferenceHz, 1)
danActivity = min(danPopulationRateHz / danActivityReferenceHz, 1)

efficacy *= exp(-eta × teachingMagnitude × kcActivity × danActivity)
efficacy = max(efficacy, efficacyFloor)
```

Starting values before Gate C/D calibration:

```text
eta                      0.05
efficacyFloor            0.10
kcActivityReferenceHz   50
danActivityReferenceHz  50
```

`eta` may be changed only by the synthetic conditioning calibration procedure. It must never be selected based on financial return.

This is an explicit model assumption with a biologically motivated compartment/sign constraint; it is not claimed to reproduce all in-vivo dopamine plasticity.

---

## 11. Outcome semantics

Every allocation stores its exact cue and principal.

A resolved allocation receives:

```text
interestReward   ∈ [0, 1]
liquidityPenalty ∈ [-1, 0]
total            ∈ [-1, 1]
```

### Interest

```text
interestReferenceMinor = ceil(trancheMinor × interestReferenceFractionOfTranche)
interestReward = clip(interestCreditedMinor / interestReferenceMinor, 0, 1)
```

Default reference fraction:

```text
0.005
```

This deliberately makes short simulated terms teachable; it is an experimental reward scale, not an AER conversion.

### Liquidity shock

Liquidity shocks belong to the FlySmith scenario. If a required payment has `shortfallMinor > 0`, allocate that penalty over currently locked allocations pro rata by locked principal:

```text
share = allocation.lockedPrincipalMinor / totalLockedPrincipalMinor
allocationShortfall = shortfallMinor × share
liquidityPenalty = -clip(allocationShortfall / trancheMinor, 0, 1)
```

### Teaching sign

```text
total = clip(interestReward + liquidityPenalty, -1, 1)
teachingMagnitude = abs(total)

total > 0  → PAM15 teaching on stored cue
total < 0  → PPL101 teaching on stored cue
total = 0  → no teaching
```

Do not include opportunity cost in V1. HOLD/cash receives no teaching in V1.

---

## 12. Calibration protocol and gates

Calibration writes `config/calibration.json`, validating against `schemas/calibration.schema.json`. `decide` and `advance-day` must fail if calibration is absent, any gate is false, or provenance hashes are stale.

### Fixed calibration data

Use PCG64 seed `20260910` to generate 64 feature vectors uniformly on `[0,1]^4`.

Generate 32 conditioning cue pairs from the same RNG stream. Reject a pair if the Jaccard overlap of its complete bilateral KC sets exceeds `0.25`.

Bootstrap statistics:

```text
resamples:       10,000
bootstrap seed:  9001
CI:              percentile 95%
```

### Gate A — KC propagation / network saturation

Test KC current candidates:

```text
[16, 18, 20, 24, 30]
```

in the pinned simulator's current units, using 500 ms assays for all 64 cues.

A current passes only if:

1. median stimulated-KC rate is 5–80 Hz;
2. on at least 90% of cues, at least 90% of stimulated KCs fire at least once;
3. median fraction of non-KC neurons firing at least once is < 0.25;
4. 95th percentile of that non-KC active fraction is < 0.40.

Choose the lowest passing current. No pass means Gate A fails.

### Gate B — MBON readout

At selected KC current:

1. run blank + all 64 cue assays;
2. compute deltaHz for MBON11 and MBON01;
3. require standard deviation of both primary channels >= 0.5 Hz;
4. require at least 75% of cues to cause at least one spike in each primary MBON population;
5. require score IQR >= 0.5.

Store `mu11`, `sigma11`, `mu01`, `sigma01`. Any failed criterion fails Gate B.

### DAN current calibration

For PPL101 and PAM15 separately test:

```text
[8, 12, 16, 20, 24, 30]
```

over the 200 ms paired window.

Select the lowest current producing population rate 20–80 Hz while keeping the global non-KC active fraction <= 0.40. If either teacher has no passing current, Gates C/D cannot run.

### Gate C — aversive conditioning

For each of 32 dissimilar pairs `(X,Y)`:

1. record naive scores X/Y;
2. run 5 X + PPL101 teaching episodes at magnitude 1;
3. record post scores X/Y;
4. run a matched sham from naive state with identical X presentations but no DAN current.

Define:

```text
deltaX = postX - preX
shamDeltaX = shamPostX - shamPreX
```

Pass iff:

```text
median(deltaX) <= -0.5
upper 95% bootstrap CI of median(deltaX - shamDeltaX) < 0
```

### Gate D — appetitive conditioning

Repeat Gate C with PAM15.

Pass iff:

```text
median(deltaX) >= +0.5
lower 95% bootstrap CI of median(deltaX - shamDeltaX) > 0
```

### Gate E — cue specificity

For each direction:

```text
specificity = abs(deltaX) - abs(deltaY)
```

Pass only if, for **both** aversive and appetitive conditioning:

```text
median(specificity) >= 0.5
lower 95% bootstrap CI of median(specificity) > 0
```

### Gate F — reset reversibility

After conditioning:

1. set every plastic efficacy exactly to `1.0`;
2. clear all transient state;
3. replay the original naive assays.

Pass iff:

- efficacy SHA-256 equals the original naive efficacy hash;
- each primary MBON rate matches its corresponding original value within `1e-6 Hz` on the same binary/machine;
- every score matches within `1e-6`.

A financial run may not be called a learned run unless A–F pass for exactly its graph, registry, executable and calibration hashes.

---

## 13. Bondsmith adapter

The local Bondsmith integration is a replaceable adapter. The neural core knows only this loopback JSON-over-HTTP protocol. Wire shapes are also defined in `schemas/bondsmith-adapter.schema.json`.

### `GET /flysmith/v1/state`

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

`effectiveLockDays` means worst-case days until newly allocated money becomes usable under the local product rules.

### `POST /flysmith/v1/actions`

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

or `{"type":"hold"}` as the action.

Successful response:

```json
{
  "requestId": "uuid",
  "accepted": true,
  "allocationId": "alloc-123",
  "newStateVersion": "opaque-monotonic-version"
}
```

Rules:

- `requestId` is idempotent;
- stale `expectedStateVersion` returns HTTP 409 and no mutation;
- invalid product/amount returns HTTP 422 and no mutation;
- accepted HOLD returns `allocationId: null`.

### `POST /flysmith/v1/advance`

```json
{
  "requestId": "uuid",
  "days": 1,
  "expectedStateVersion": "opaque-monotonic-version"
}
```

Only `days=1` is permitted.

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

Allowed V1 adapter events:

```text
maturity
withdrawal
rate_change
product_opened
product_closed
```

FlySmith scenario liquidity shocks are not Bondsmith adapter events.

### Adapter conformance

The Bondsmith engineer must pass the same black-box contract tests as the in-memory adapter:

- integer money and bps values;
- action and advance idempotency;
- stale-state 409 rejection;
- invalid-action 422 rejection;
- exactly one-day advancement;
- maturity/withdrawal event reconciliation to known allocation IDs;
- deterministic fixture reset when running test scenarios.

---

## 14. Experiment configuration

Each run loads one immutable JSON document validating against `schemas/experiment-config.schema.json`.

Canonical V1 example:

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

At load time additionally validate `maxRateBps > minRateBps` and ensure the tranche can be represented safely as an integer in every target runtime.

---

## 15. Records and reproducibility

Option assays validate against `schemas/trial-record.schema.json`.

Every run must also persist:

### Decision record

```text
experimentId, runId, day, adapter stateVersion
all candidate option IDs and features
all cue hashes and option scores
cash score
winner
indifference/tie reasoning
submitted action
requestId
adapter receipt/newStateVersion
```

### Teaching record

```text
experimentId, runId, day
allocationId
stored cue hash + exact indices
interestReward, liquidityPenalty, total
teachingMagnitude
teacher = PAM15 | PPL101
teacher population rate
changed edge count
plasticity hash before/after
```

### Run manifest

Hash:

```text
experiment config
MaleCNS source manifest
graph arrays
neuron registry
calibration manifest
vendored upstream source
FlySmith kernel/binary
initial and final plasticity state
```

A run is replayable only when trial, decision, teaching and run-manifest records are present.

---

## 16. Exact day algorithm

```text
DECIDE
1. GET adapter state.
2. Form cash option plus products where canDeposit=true and configured tranche is valid.
3. Compute four normalized features per option.
4. Generate and persist each bilateral sparse KC cue.
5. Reset transient neural state; preserve current efficacies.
6. Run blank assay.
7. For each option in lexical optionId order:
     restore identical day-start state
     run cue assay with learning=false
     score MBON11 - MBON01
8. Apply choice rule.
9. POST action with requestId + expected stateVersion.
10. Persist all records and stop.

ADVANCE-DAY
11. POST /advance with days=1 and expected stateVersion.
12. Apply the configured FlySmith liquidity shock for the new day, if any.
13. Resolve per-allocation outcome components.
14. Process nonzero teaching outcomes in ascending allocationId order:
      load exact stored cue
      reset transient state
      run PAM15 or PPL101 teaching episode
      persist efficacy diff/hash
15. Reset transient state.
16. Fetch/persist the resulting adapter state and stop.
```

No wall-clock scheduler exists in V1.

---

## 17. Required CLI behavior

Expose these commands or exact functional equivalents:

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

`decide` and `advance-day` must hard-fail if current provenance does not match a calibration manifest with Gates A–F all passed.

`reset-learning` sets all efficacy values to `1.0`, clears transient state and records the reset event.

`replay` must perform no external mutations; it reconstructs recorded neural assays from persisted state and asserts recorded hashes/scores.

---

## 18. Work split

### Neural/simulation engineer

Owns:

```text
vendored pinned DoomFly source
graph preparation/provenance
FlySmith kernel wrapper
neuron registry
KC projection
trial/checkpoint state lifecycle
MBON readout
KC→MBON efficacy layer
PAM15/PPL101 teaching
Gates A–F
replay
```

### Bondsmith engineer

Owns:

```text
GET  /flysmith/v1/state
POST /flysmith/v1/actions
POST /flysmith/v1/advance
stateVersion semantics
idempotency
allocation/maturity reconciliation
fixture reset
adapter conformance tests
```

### Experiment engineer

Owns:

```text
feature normalization
scenario/liquidity shocks
outcome attribution
CLI orchestration
JSONL/manifests
control policies
analysis/report generation
```

These teams can work in parallel once the schemas and in-memory adapter fixtures are in place.

---

## 19. Required controls

Any claimed learning result must run the identical financial history against:

```text
fixed-weight MaleCNS
plastic MaleCNS + sham DAN
trained plastic MaleCNS
trained then efficacy-reset MaleCNS
cue-unpaired DAN control
PPL101-disabled negative-teaching control
PAM15-disabled positive-teaching control
MBON11 readout ablation/control
MBON01 readout ablation/control
alternate projection-seed control
random-choice policy
highest-yield policy
cash-only policy
```

Portfolio return is secondary. V1's primary scientific evidence is specific, signed and reversible conditioned change in neural value that changes a later action.

---

## 20. V1 acceptance criteria

The implementation is complete only when all are true:

1. Whole MaleCNS graph preparation matches pinned source/provenance and is hash-verified.
2. Registry generation resolves every required population and both plastic edge sets without hand-copied IDs.
3. The same financial feature vector always produces the same balanced bilateral KC cue and hash.
4. Gates A–F pass and produce a calibration manifest tied to exact graph/registry/binary/config hashes.
5. In-memory adapter passes the Bondsmith adapter contract tests.
6. Local Bondsmith adapter passes those exact same black-box tests.
7. A reset 30-day fixture replays with identical actions, MBON rates/scores and plasticity hashes on the same build/machine.
8. Every required neural and non-neural control can run against the identical scenario history.
9. Runtime assertions prevent modification of graph topology or structural weights.
10. Every learned action change can be traced end-to-end:

```text
realized outcome
→ stored cue
→ explicit PAM15/PPL101 teaching
→ registered efficacy changes
→ changed later MBON score
→ changed action
```

---

## 21. Closed decisions

No engineer should reopen these during V1 implementation without a design-version change:

```text
connectome           MaleCNS male-cns:v1.0
numerical baseline   DoomFly adaptive-centered-v6 @ 71ecf53d...
graph scope          full retained graph
external input       constant-current injection
financial encoding   deterministic 5% bilateral sparse KC code
eligible KCs         KCs with retained edge to MBON01 or MBON11
primary value        z(MBON11) - z(MBON01)
negative teacher     PPL101
positive teacher     PAM15 / PAM-γ5β′2a
negative plasticity  KC → MBON11 depression
positive plasticity  KC → MBON01 depression
credit assignment    exact cue replay at outcome resolution
position size        configured fixed tranche
clock                 explicit one-day advance
persistent memory    registered efficacy multipliers only
```

Selected KC current, teacher currents and MBON calibration statistics are outputs of the mandatory calibration protocol, not free implementation choices.

---

## 22. Sources

- MaleCNS v1.0 downloads/data: https://male-cns.janelia.org/download/
- DoomFly pinned reference: https://github.com/nftechie/doomfly/tree/71ecf53d78eaffaf1a57ed7b0ccf5d458abc9f33
- Aso et al. 2014, mushroom-body architecture: https://doi.org/10.7554/eLife.04577
- Aso et al. 2014, MBON valence/action selection: https://doi.org/10.7554/eLife.04580
- Owald et al. 2015, defined MBONs and learned behavior: https://doi.org/10.1016/j.neuron.2015.03.025
- Takemura et al. 2017, learning/memory connectome: https://doi.org/10.7554/eLife.26975
- Li et al. 2020, adult mushroom-body connectome: https://doi.org/10.7554/eLife.62576
- PAM15 ontology (`PAM-γ5β′2a`): FlyBase FBbt:00049841
- PPL101 / PPL1-γ1pedc literature and MaleCNS annotation vocabulary
- Bondsmith developer portal: https://developers.bondsmith.com/

MaleCNS supplies measured anatomy/connectivity. The LIF dynamics, transmitter-sign conversion, artificial KC code, compressed financial outcomes and FlySmith plasticity rule are explicit modeling assumptions, not a validated physiological reconstruction of a living fly.
