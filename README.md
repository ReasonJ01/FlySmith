# FlySmith

FlySmith is a local closed-loop experiment that connects a Bondsmith-compatible short-term savings environment to the complete MaleCNS v1.0 fruit-fly connectome.

## Start here

**Implement [`DESIGN.md`](DESIGN.md).** It is the V1 engineering source of truth.

[`FINAL_DESIGN.md`](FINAL_DESIGN.md) preserves the neuroscience research/rationale that led to the build specification. [`PLAN.md`](PLAN.md) is historical. If provisional values or wording differ between them, `DESIGN.md` governs V1 implementation.

Machine-checkable contracts live in [`schemas/`](schemas/):

- `neuron-registry.schema.json` — generated MaleCNS neuron and plastic-edge registry
- `experiment-config.schema.json` — immutable experiment configuration
- `calibration.schema.json` — calibrated neural parameters and Gates A–F
- `trial-record.schema.json` — per-option neural assay record
- `run-event.schema.json` — decision, teaching, and learning-reset records
- `bondsmith-adapter.schema.json` — reusable JSON wire types
- `bondsmith-adapter.openapi.yaml` — executable HTTP contract for the local Bondsmith adapter

## V1 architecture

```text
Bondsmith-compatible option state
        ↓
[yield, lock, liquidity, exposure]
        ↓
deterministic bilateral sparse KC cue
        ↓
complete MaleCNS recurrent model
        ↓
z(MBON11) - z(MBON01)
        ↓
HOLD or fixed-tranche ALLOCATE
        ↓
realized outcome
        ↓
positive: PAM15 + cue replay → KC→MBON01 depression
negative: PPL101 + cue replay → KC→MBON11 depression
```

The fly does not receive product JSON or financial labels. The artificial boundary is a distributed conditioned stimulus imposed directly on real MaleCNS Kenyon cells. Across simulated days, only registered KC→MBON efficacy multipliers persist; transient electrical state is reset.

## Pinned numerical baseline

```text
MaleCNS:            male-cns:v1.0
reference runtime:  nftechie/doomfly
commit:             71ecf53d78eaffaf1a57ed7b0ccf5d458abc9f33
numerical model:    adaptive-centered-v6
```

The reference simulator uses constant external current injection, not Poisson-Hz inputs. `DESIGN.md` pins graph construction, numerical constants, the FlySmith-specific learning rule, exact calibration gates, and provenance requirements.

## Hard prerequisite

Bondsmith financial experiments stay disabled until all six neural gates pass for the exact graph/registry/binary/config hashes used by the run:

```text
A  KC propagation / saturation
B  MBON readout
C  aversive conditioning via PPL101
D  appetitive conditioning via PAM15
E  cue specificity
F  efficacy-reset reversibility
```

Portfolio performance is not evidence of learning by itself.

## Engineering split

The neural engineer owns the pinned graph/runtime, registry, KC encoding, MBON readout, plasticity and gates. The Bondsmith engineer owns the loopback `/flysmith/v1/state`, `/actions`, and `/advance` adapter with idempotency and optimistic state-version checks. The experiment engineer owns normalization, scenarios/outcomes, orchestration, controls and reproducibility records.

No real money is involved. The environment advances only through an explicit one-day command.
