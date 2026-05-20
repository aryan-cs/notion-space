# PROPOSAL: Notion-Based Reasoning System (NBRS)

**Research Roadmap**

**Status:** Architectural pivot complete. Empirical programme defined.

---

## Preamble

The Notion-Based Reasoning System (NBRS) is a research programme on cross-domain procedural reasoning. Its central claim is that contemporary large language models fail at transfer — the ability to apply an abstract procedure learned in one domain to a structurally analogous problem in a different domain — because they do not factorise the *structure* of a problem from its *content*. NBRS proposes a neurosymbolic architecture that maintains an explicit, content-independent library of abstract procedures (*notions*), induced from few examples, stored as parameterised templates, and applied across domains via structure-mapping.

The architecture is informed by two convergent lines of evidence:

1. **Cognitive-map neuroscience.** The hippocampal–entorhinal system factorises structural codes (relational scaffolding) from sensory codes (content). The same grid-cell apparatus that supports spatial navigation supports navigation through abstract conceptual spaces. The Tolman–Eichenbaum Machine formalises this; the Constantinescu et al. "bird-space" study demonstrates it empirically in humans.
2. **Structure-mapping theory.** Gentner's account of analogy as mappings of *relational structure* (not surface features), and its computational implementation in the Structure-Mapping Engine, predict exactly the kind of transfer that humans display and that LLMs lack.

What is missing in current AI is an integration: an architecture that combines the abstraction power of cognitive-map-style factorisation, the procedural depth of program-induction-style library learning, and the perceptual breadth of LLM-style content processing. NBRS is that architecture.

The full formal treatment is in [`proof.pdf`](https://aryan-cs.github.io/notion-space/proof.pdf). This document gives the research roadmap.

---

## 1. The Transfer Gap

Frontier LLMs underperform humans on tasks specifically designed to require abstract analogical transfer:

- **Letter-string analogies in novel alphabets** (Lewis & Mitchell 2024): humans, including children, generalise robustly; GPT-class models collapse in proportion to alphabet unfamiliarity.
- **ARC-AGI** (Chollet 2019, 2024): designed to require sample-efficient abstraction from a handful of grid examples. Even substantially-engineered LLM-based systems struggle on the harder subset.
- **Raven's Progressive Matrices** and digit-pattern analogies (Webb et al. 2023): LLM performance drops sharply on stimuli unlikely in training data, while human performance is stable.

The failure mode is consistent: LLMs match on surface features, not on relational structure. Where the abstract pattern requires a content-independent representation, the model has nothing to fall back on.

---

## 2. Architectural Hypothesis

NBRS hypothesises that the transfer gap is closed by an architecture with three structural commitments:

1. **Structure–content factorisation.** The relational scaffold of a problem is represented separately from its content. Notions are content-independent templates; bindings combine them with content at instantiation time.
2. **Explicit notion library.** Transferable procedures are stored as first-class objects in a growing library, induced from successful traces, not entangled in network weights.
3. **Structure-mapping at application time.** Selecting a notion to apply to a new problem is a structural-alignment operation, not a surface-similarity lookup.

The architecture is a five-component system (full specification in [`proof.pdf`](https://aryan-cs.github.io/notion-space/proof.pdf)):

| Component | Role |
|-----------|------|
| **Content layer** (frontier LLM) | Parse problems, supply values, generate output. No multi-step reasoning. |
| **Structure extractor** | Convert parsed problems into typed relational graphs. |
| **Notion library** | Store parameterised relational templates with type, value, and operator variables. |
| **Analogy engine** | Match problem graphs to notions via structural alignment; produce bindings. |
| **Execution substrate** (GSRE) | Deterministically run instantiated notion graphs with the standard executor guarantees (determinism, termination, soundness). |

A sixth component, **consolidation**, periodically abstracts new notions from successful traces and adds them to the library — analogous to hippocampal replay.

---

## 3. Empirical Programme

The architecture's claims are empirical. The research programme is organised around two cross-domain transfer demonstrations chosen to test the central hypothesis directly, plus diagnostics that localise failure when it occurs.

### Demonstration 1: ARC-AGI subset

**Task.** Each ARC-AGI task provides 2–5 input/output grid pairs demonstrating an abstract transformation; the system must apply the transformation to a held-out input.

**Why it matters.** Chollet designed ARC-AGI specifically to require the kind of sample-efficient abstraction that NBRS is built around. LLM-based systems systematically underperform on this benchmark. A successful demonstration would constitute a public, leaderboard-comparable result.

**Scope.** Target a subset of ARC tasks expressible as compositions of grid-level primitives (rotate, reflect, recolour, fill, count, replicate, translate, mask, overlay). The notion-induction layer mines transformations from each task's demonstration examples; the analogy engine binds the mined notion to the held-out test input.

**Baselines.** Frontier LLM with chain-of-thought, frontier LLM with retrieval, NBRS ablations (library disabled; analogy engine replaced by random selection), ARC Prize leaderboard for context.

**Success criteria.**
- *Minimum:* match frontier-LLM accuracy on the subset with substantially less compute.
- *Aspirational:* exceed frontier LLM, particularly on tasks classified as "hard".
- *Interpretive:* demonstrate that notions induced from one subset of tasks transfer to a held-out subset whose tasks were never used in induction.

### Demonstration 2: Cross-discipline scientific equilibrium analogy

**Task.** Train the notion library on mechanical-equilibrium problems (force balance, springs at rest, lever arms, fluid pressures). Test on novel problems from three unrelated disciplines that share the abstract equilibrium structure but no shared surface content.

**Why it matters.** This is the ambitious test of the architecture's central claim — that a notion learned in one field can apply, without retraining, to a structurally analogous problem in a different field. It is the capability most closely identified with general intelligence in the cognitive-science literature.

**The structural pattern.** Identify opposing influences expressed as functions of a quantity; set them equal at the equilibrium point; solve. The notion `EQUILIBRIUM(α₊, α₋, x)` instantiates differently in each domain but is the same notion in the library.

**Test problems.**
- Supply–demand market equilibrium (price-clearing).
- Chemical equilibrium under perturbation (Le Chatelier-style shift).
- Predator–prey steady state (Lotka–Volterra fixed point).
- Nash equilibrium in a 2-player normal-form game.

**Baselines.** Same as Demonstration 1, plus a frontier LLM with explicit cross-domain-analogy prompt scaffolding.

**Success criteria.**
- *Minimum:* exceed the frontier-LLM baseline on at least one of the four test domains while not regressing on within-discipline (mechanical) problems.
- *Aspirational:* exceed the frontier LLM on all four, with a library that included no economic, chemical, biological, or game-theoretic training examples.
- *Interpretive:* any negative result must localise to a specific component (parser, analogy engine, executor) via the diagnostic measurements below.

### Diagnostics

Both demonstrations are accompanied by:

- **Notion-induction efficiency.** Learning curve: traces required per notion to reach the verification threshold.
- **Analogy-engine accuracy.** Per-test breakdown into wrong-notion, wrong-binding, no-match.
- **Per-primitive error rate and correlation structure** on the executor, calibrating the depth-decoupling theorem.
- **Marginal-invariance diagnostic** (KL / MMD divergence between training and test context marginals), calibrating the depth-generalisation corollary.
- **Notion-library ablation.** Performance of NBRS with empty library vs. random notion selection vs. full library, attributing performance to the right components.

---

## 4. Path of Work

The empirical programme decomposes naturally into work phases. Phases are intentionally serial: each unlocks the next.

| Phase | Deliverable | Gating question |
|-------|-------------|-----------------|
| **0. Infrastructure** | Executor (GSRE) implementation; primitive library for grid operations; toy data generator for sanity tests. | Does the executor faithfully implement the formal semantics? |
| **1. Notion definition + instantiation** | Parameterised notion data structure; binding mechanism; instantiation pipeline. | Can a hand-written notion be cleanly instantiated and executed? |
| **2. Analogy engine v0** | Neural matcher that takes a problem graph and a notion and proposes a binding. Trained on synthetic structure-matched pairs. | Does the matcher recover correct bindings on synthetic data? |
| **3. Notion induction v0** | Compositional-compression pipeline that mines candidate notions from a trace corpus. | Does a hand-crafted toy corpus yield the expected notion (e.g. `FOLD`)? |
| **4. ARC-AGI Demonstration** | Full pipeline against the chosen ARC subset, with baselines and diagnostics. | Does NBRS match or exceed frontier-LLM performance? |
| **5. Equilibrium Demonstration** | Cross-discipline transfer pipeline. | Does NBRS transfer notions across scientific disciplines? |
| **6. Consolidation** | Replay-driven library growth from open-ended trace corpora. | Does the library improve with more usage, the way the architecture predicts? |

A successful Phase 4 is a publishable result. A successful Phase 5 is a contribution to the literature on general intelligence. Phase 6 is the long-run autonomous-growth story.

---

## 5. Open Problems and Risks

The architecture is not a guaranteed solution. The hard problems are real and identified:

- **Notion induction in non-trivial domains is open.** DreamCoder demonstrates library learning works for restricted DSLs; whether it scales to the typed-parameterised notions NBRS requires is unknown.
- **Structure-mapping in learned representations is open.** Classical SME-style matching works on symbolic representations; doing it over learned embeddings is novel research.
- **Structure-extraction quality bounds everything.** If the LLM-driven parser produces noisy problem graphs, the analogy engine has nothing to work with. This is the most likely failure mode.
- **Continual learning of the executor's primitives** (stability vs. plasticity) is unresolved. The procedural-notion mitigation pushes this into the library, but does not eliminate it entirely.
- **No empirical validation yet.** The architecture is theoretically motivated and architecturally specified. Whether it survives contact with training is what the empirical programme tests.

Each of these is an honest research risk, with a documented failure-diagnosis pathway and a clear localisation in the architecture's components.

---

## 6. Summary

NBRS targets the transfer gap that most distinguishes contemporary LLMs from human general intelligence. Its architectural commitment — structure–content factorisation with an explicit, growing notion library and a structure-mapping analogy engine — is grounded in cognitive neuroscience and classical analogical-reasoning theory. The empirical programme is direct: two cross-domain transfer demonstrations, one on a public benchmark (ARC-AGI) and one on cross-discipline scientific analogy, each accompanied by diagnostics that localise failure.

If the programme succeeds, NBRS will have demonstrated a computational realisation of cross-domain procedural transfer, which is the capability cognitive science has long identified as central to general intelligence and which contemporary LLMs do not exhibit. If it fails, the diagnostics localise the failure to a specific architectural component, and the research direction is correspondingly refined.

The formal foundations of the architecture are developed in [`proof.pdf`](https://aryan-cs.github.io/notion-space/proof.pdf). The empirical work is the next phase.
