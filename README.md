# FlySmith

FlySmith is a local closed-loop experiment that connects a Bondsmith-compatible short-term savings environment to the complete MaleCNS v1.0 fruit-fly connectome.

The fly does not receive product JSON or financial labels. Each available savings destination is converted into a deterministic sparse conditioned-stimulus pattern in real mushroom-body Kenyon cells. The full 166,700-neuron recurrent model runs, and activity in approach-associated MBON11 and avoidance-associated MBON01 produces an option-valence score. The highest-scoring destination receives a fixed tranche.

When an allocation's consequence resolves, FlySmith replays the exact KC cue. Positive outcomes stimulate PAM15 and modify existing KC→MBON01 efficacy; negative outcomes stimulate PPL101 and modify existing KC→MBON11 efficacy. The topology stays fixed. Plasticity is enabled only during explicit teaching epochs.

The environment runs locally, uses no real money, and advances only when the user manually advances the simulated day.

## Design

**Implement [`FINAL_DESIGN.md`](FINAL_DESIGN.md).** It is the researched and frozen V1 architecture and supersedes unresolved choices in the earlier [`PLAN.md`](PLAN.md).

The first hard gate is not Bondsmith integration. It is a controlled mushroom-body conditioning assay: distinct artificial KC cues must show cue-specific appetitive learning with PAM15 and aversive learning with PPL101, with frozen, omitted-teaching, timing and memory-reset controls. If that assay fails, the financial loop stays disabled.

## Status

- MaleCNS v1.0 selected
- neural input population selected: Kenyon cells
- neural output selected: MBON11 approach vs MBON01 avoidance
- positive teaching channel selected: PAM15 / PAM-γ5β′2a
- negative teaching channel selected: PPL101 / PPL1-γ1pedc
- plastic synapses selected: existing KC→MBON01 and KC→MBON11 edges
- delayed credit assignment selected: exact cue replay on outcome resolution
- financial action decoder selected: highest neural valence, fixed 10% liquid-cash tranche
- implementation pending

## Research references

See `FINAL_DESIGN.md` for the rationale, validation gates, current MaleCNS project comparisons, and primary sources.