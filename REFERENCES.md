# FlySmith research basis

This file records the sources used to freeze the FlySmith v1 design in [PLAN.md](PLAN.md). The implementation spec is intentionally stricter than the claims made by the references: where the literature does not provide a physiological parameter for MaleCNS, FlySmith labels the value as an engineering choice and validates it experimentally.

## Primary MaleCNS resource

### Berg et al., Cell (2026)

**Sexual dimorphism in the complete Drosophila male central nervous system connectome**  
Cell 189(18):5504-5526.e15, 3 September 2026.  
DOI: https://doi.org/10.1016/j.cell.2026.08.015  
Project: https://male-cns.janelia.org/  
Download/programmatic access: https://male-cns.janelia.org/download/

Supports:

- using `male-cns:v1.0` as the canonical graph;
- a complete male brain + VNC resource for sensory-to-motor circuit analysis;
- canonical `bodyId`/annotation-based neuron identity.

The peer-reviewed paper reports about 166.7k neurons. FlySmith must use the pinned v1.0 files and their hashes as data truth rather than counts copied from third-party demos.

### MaleCNS Cell Type Explorer

https://reiserlab.github.io/celltype-explorer-drosophila-male-cns/

Source repository:  
https://github.com/reiserlab/celltype-explorer-drosophila-male-cns

Used to resolve the v1 learning circuit in the released male connectome:

- `KCg-m`: gamma Kenyon-cell population, approximately 1,340 cells;
- `KCg-d`: gamma Kenyon-cell population, approximately 206 cells;
- `MBON01` / MBON-gamma5beta'2a: two cells, MaleCNS body IDs `520151`, `10013`;
- `MBON11` / MBON-gamma1pedc>alpha/beta: two cells, body IDs `10704`, `11402`;
- `PPL101` / PPL1-gamma1pedc: two dopaminergic cells, body IDs `11327`, `11900`;
- `PAM01` / PAM-gamma5: reward-associated dopaminergic type, currently 44 cells in MaleCNS; FlySmith resolves these dynamically by type rather than hard-coding the list.

The registry generated from the actual pinned dataset remains authoritative if a generated explorer page and the local files ever disagree.

---

## Mushroom-body learning and valence

### Aso et al., eLife (2014): architecture

**The neuronal architecture of the mushroom body provides a logic for associative learning**  
https://doi.org/10.7554/eLife.04577  
https://elifesciences.org/articles/04577

Key support:

- sparse Kenyon-cell representations;
- compartmentalized KC -> MBON synapses;
- compartment-specific dopaminergic teaching inputs;
- the mushroom body as a natural substrate for assigning learned value to otherwise arbitrary sensory representations.

### Aso et al., eLife (2014): MBON valence

**Mushroom body output neurons encode valence and guide memory-based action selection in Drosophila**  
https://doi.org/10.7554/eLife.04580  
https://elifesciences.org/articles/04580

Key support:

- the MBON population represents learned valence;
- different MBON types can bias attraction or repulsion;
- local dopamine-dependent changes can alter the balance of MBON output and thereby memory-based action selection.

This is the conceptual basis for reading mushroom-body valence directly rather than assigning savings semantics to left/right turning neurons.

### Vasmer et al., Frontiers in Behavioral Neuroscience (2014)

**Induction of aversive learning through thermogenetic activation of Kenyon cell ensembles in Drosophila**  
https://doi.org/10.3389/fnbeh.2014.00174  
https://www.frontiersin.org/journals/behavioral-neuroscience/articles/10.3389/fnbeh.2014.00174/full

Key support:

- sparse, random artificial KC ensembles can serve as conditioned stimuli;
- direct KC activation paired with salient reinforcement is sufficient to create a learned behavioral response.

This is the strongest precedent for FlySmith's decision to make a synthetic financial vector a deterministic sparse KC pattern instead of inventing a fake odor/taste mapping.

### Hige et al., Nature (2015)

**Plasticity-driven individualization of olfactory coding in mushroom body output neurons**  
https://doi.org/10.1038/nature15396

Key support:

- MBON tuning is experience/plasticity dependent;
- sparse sensory codes are reformatted into downstream output representations.

### Hige et al., Neuron (2015)

**Heterosynaptic Plasticity Underlies Aversive Olfactory Learning in Drosophila**  
https://doi.org/10.1016/j.neuron.2015.11.003  
https://pubmed.ncbi.nlm.nih.gov/26637800/

Key support:

- dopamine-gated plasticity at KC -> MBON synapses during aversive learning;
- mechanistic basis for localized, activity-dependent efficacy changes rather than training the complete graph.

### Felsenberg et al., Nature (2017)

**Re-evaluation of learned information in Drosophila**  
https://doi.org/10.1038/nature21716  
https://www.nature.com/articles/nature21716

Key support:

- opponent/recurrent DAN-MBON systems can update learned value after experience;
- memory can be re-evaluated rather than behaving as a permanently one-way association.

### Li et al., eLife (2021)

**The connectome of the adult Drosophila mushroom body provides insights into function**  
https://doi.org/10.7554/eLife.62576  
https://elifesciences.org/articles/62576

This is especially important for the exact FlySmith v1 circuit.

Relevant findings include:

- `PPL101` is PPL1-gamma1pedc and participates in aversive reinforcement;
- aversive reinforcement through PPL101 depresses conditioned KC -> MBON11 connections;
- `PAM01` is PAM-gamma5 and is part of the positive-valence PAM system;
- PAM01 and MBON01 occupy the gamma5 learning compartment;
- MBON11 participates in an approach-favoring network and interacts with PPL101;
- MBON/DAN circuitry contains recurrent structure, so the rest of the retained graph can influence the chosen learning subsystem.

The v1 plasticity rule is a deliberately simplified engineering implementation of these compartment-specific learning motifs. It is not claimed to reproduce the complete biochemistry or timing rule of a living fly.

---

## Whole-connectome spiking simulation

### Shiu et al., Nature (2024)

**A Drosophila computational brain model reveals sensorimotor processing**  
https://doi.org/10.1038/s41586-024-07763-9  
https://www.nature.com/articles/s41586-024-07763-9

Key support:

- a whole-connectome leaky-integrate-and-fire model built primarily from connectivity and neurotransmitter identity can make useful circuit predictions;
- sensory neurons can be represented by external drive and activity propagated across the graph;
- network structure matters: shuffled-connectivity controls impair predictive performance;
- absolute firing rates in this kind of model should not be treated as exact physiological predictions.

FlySmith therefore uses relative/calibrated MBON scores, repeated Poisson trials and shuffled controls rather than interpreting a raw number of Hertz as literal fly physiology.

### Xenova / Neural Canvas

https://huggingface.co/spaces/Xenova/fruit-fly-simulation  
Source tree: https://huggingface.co/spaces/Xenova/fruit-fly-simulation/tree/main

A current MaleCNS whole-graph LIF implementation and useful engineering reference for:

- retaining ~166.7k MaleCNS entries;
- external per-neuron stimulation;
- per-neuron spiking simulation;
- CPU/WebGPU implementation patterns.

Its own README explicitly describes the animated movements and stimulation presets as crafted/illustrative rather than validated behavior. FlySmith therefore reuses the neural-compute pattern but not its turn/walk animation semantics.

---

## Other people putting connectomes into artificial worlds

These projects are implementation references, not validation of FlySmith's scientific claims.

### DOOMFLY

https://github.com/nftechie/doomfly

Most relevant review/protocol:

- https://github.com/nftechie/doomfly/blob/main/docs/doom-learning-review.md
- https://github.com/nftechie/doomfly/blob/main/docs/doom-live-training.md

DOOMFLY runs MaleCNS in a Doom environment and has experimentally added PPL101-gated KC -> MBON11 plasticity. Its published negative results are directly useful to FlySmith.

Important lessons adopted in v1:

- changed synaptic weights are not proof of learned behavior;
- a sensory cue must first be shown to reach the intended learning/readout circuit;
- broad neural stimulation can accidentally recruit dopamine neurons ('teacher contamination');
- conditioning needs isolated validation before closed-loop gameplay/economics;
- frozen and shuffled controls are mandatory;
- teacher delivery, memory change and action change should be logged separately.

FlySmith therefore disables plastic updates outside explicit conditioning phases and requires positive/negative conditioning unit tests before the savings loop is enabled.

### Fly Brain Minecraft

https://github.com/blendi-remade/fly-brain-minecraft

Useful engineering precedents:

- resolving stimulation/readout populations by MaleCNS type/body ID;
- keeping biological neuron identities visible in telemetry;
- clearly separating emergent circuit behavior from hand-built reflexes/decoder mappings.

The project also demonstrates why FlySmith should not use a third-party demo's reported neuron/edge counts as canonical MaleCNS metadata; the pinned Janelia release is the source of truth.

### Mario / MaleCNS demo

https://github.com/ornata/fly

Useful as an example of a game receiving controls from fixed neural population readouts. It has no learning/reward objective and explicitly describes its controller mappings as authored rules. This reinforced the decision not to treat motor decoder choices as evidence of financial valuation.

---

## Bondsmith

Developer documentation:  
https://developers.bondsmith.com/

Product semantics:  
https://www.bondsmith.com/personal

Bondsmith publicly describes:

- a REST Savings API;
- a hub account;
- easy-access, notice and fixed-term savings products;
- maturities/withdrawals returning through the hub account.

FlySmith uses those concepts as the environment vocabulary. V1 deliberately compresses terms to 1-10 simulated days and uses a local stack only. The neural code depends on the `SavingsWorld` domain interface in PLAN.md, not undocumented REST endpoint guesses.

---

## Why the final v1 circuit is the one in PLAN.md

The chosen design combines the strongest parts of the evidence above:

1. **Arbitrary cue:** direct sparse KC ensembles are experimentally conditionable.
2. **Memory substrate:** KC -> MBON efficacy is a canonical locus of associative plasticity.
3. **Positive channel:** PAM01/gamma5 is paired with the MBON01 gamma5 avoidance-side compartment.
4. **Negative channel:** PPL101/gamma1pedc drives aversive learning by depressing KC -> MBON11.
5. **Readout:** MBON ensemble balance is a biologically supported valence/action-selection representation.
6. **Whole graph:** MaleCNS remains present around the learning subsystem; only a small set of existing synaptic efficacies is allowed to change.
7. **Engineering discipline:** lessons from Shiu, DOOMFLY, Minecraft and Neural Canvas are used to separate connectome-derived behavior from authored interface choices.

The remaining uncertainty is empirical performance, not an unspecified design decision. If the frozen v1 circuit fails its preregistered validation gates, the correct result is a failed v1 and a separately versioned v2 hypothesis.