# PROPOSAL: Notion-Based Reasoning System (NBRS)

**Internal Research Roadmap**

**Status:** Track 1 design mature and approved for implementation. Tracks 2 and 3 are exploratory and will be refined based on Track 1 outcomes.

---

## Preamble

This document describes a multi-stage investigation into neurosymbolic reasoning with explicit dependency graphs.

- **Track 1** is a self-contained, falsifiable experiment testing whether explicit dependency tracking and recurrent module reuse improve compositional depth generalisation when a perfect reasoning graph is provided. It will be published as a standalone paper at a top ML venue, framed purely as a learning-to-execute study. The broader NBRS vision will appear only in a brief "Future Work" paragraph in that paper.
- **Track 2** extends the executor with a neural policy that learns to construct reasoning graphs autonomously from a fixed set of primitive operations.
- **Track 3** is a long-term research agenda exploring whether the system can invent its own primitive operations through experience.

This separation ensures that the rigorous Track 1 result is not diluted by speculative promises, while internally we maintain a coherent long-term direction.

---

## 1. Vision and Motivation

Current large language models simulate multi-step reasoning through autoregressive token prediction. This approach fails on tasks requiring precise compositional depth—counting, nested arithmetic, transitive inference—because reasoning is simulated rather than structurally implemented.

We hypothesize that robust reasoning requires:

1. **Explicit representation of dependency structure** between sub-problems.
2. **Guaranteed argument availability** before any operation executes.
3. **Recurrent reuse of a capacity-limited processor**, decoupling problem depth from the model's internal context length.

The **Notion-Based Reasoning System (NBRS)** embodies these principles. A *notion* is a typed operation node in a directed acyclic graph (DAG). Reasoning proceeds by traversing the graph, executing each node only when all its parent values are resolved, using a small shared neural module applied repeatedly.

The NBRS is intended to serve as a reasoning co-processor beside a language model—the language model handles perception and generation, while the NBRS performs precise multi-step reasoning.

---

## 2. Architectural Overview

| Component | Track 1 | Track 2 | Track 3 |
|-----------|---------|---------|---------|
| Expansion grammar | Oracle, hand-crafted | Fixed schemas; policy selects and binds | Schemas augmented with learned primitives |
| Traversal engine | Dependency-driven scheduler | Same | Same |
| Node Processor (shared Transformer) | Trained per-primitive + end-to-end on depth-3 traces | Pre-trained, fine-tuned jointly with policy | Continually updated as primitives are added |
| Graph Policy Network (GPN) | Not present | Neural policy: selects primitive type, binds arguments; trained via RL + limited bootstrapping | GPN can also propose new primitive schemas |
| Search and backtracking | None | MCTS with learned value function; limited to linear chains initially | Extended search with meta-reasoning |
| Primitive Library | Fixed set of ~15 primitives | Same fixed set | Grows via compositional compression and induction |

---

## 3. Track 1: Compositional Execution Study

**Goal:** Determine whether explicit dependency tracking and recurrent module reuse improve out-of-distribution depth generalisation over implicit processing of the same decomposition.

**Status:** Design complete; implementation-ready.

### 3.1 Research Question

Given an oracle-supplied DAG specifying all operations, types, and dependencies, does a Graph-Structured Recurrent Executor (GSRE) that enforces argument availability and reuses a shared Transformer outperform (a) a weight-tied graph transformer with causal ancestor masking, and (b) a sequence transformer receiving the serialised DAG, on generalisation to graphs deeper than those seen during training?

### 3.2 GSRE Architecture

**Oracle Grammar:** A hand-written, deterministic module that expands a task specification into a complete DAG. Used only for data generation and input preparation; no model learns it.

**Traversal Engine:** Maintains node states (`PENDING`, `READY`, `EXECUTING`, `RESOLVED`, `FAILED`). A node transitions to `READY` when all parents are `RESOLVED`. All ready nodes are batched and processed by the shared Node Processor. The engine guarantees strict dependency order.

**Node Processor:** A 2-layer Transformer (~2M parameters). Input per node: type embedding, spec encoding, and resolved values of direct parents (formatted with delimiters). No history vector, no global graph embedding, no execution-order signal. Output: a resolution value via a type-appropriate head (regression, classification, or copy).

**Input to all models:** The full DAG is provided in identical form. Text-to-graph translation is purely symbolic and transparent.

### 3.3 Task Design

**Primary dataset (Tier 5):** Sequential heterogeneous operation chains. Operations include `ADD_MOD`, `SUB_MOD`, `SHIFT_CHAR`, `SWAP_CASE`, `REVERSE`, `COMPARE`, `SELECT`.

- Graph depth (chain length): 1–10.
- Primitive input complexity held constant: numbers 0–100 (modulo prime 101), strings ≤5 characters.
- **Training:** 100K instances of depth 1–3.
- **In-distribution test:** 10K instances depth 1–3 (held-out).
- **OOD test:** 10K instances each at depths 5, 7, 9, 10. Depth 4 held out for interpolation.

### 3.4 Baselines

- **B4-tied (primary):** Weight-tied graph transformer with causal ancestor masking. Same per-node inputs and output heads as GSRE. Nodes sorted by depth-first preorder; attention restricted to ancestors appearing earlier.
- **B4 (unshared):** Same architecture, distinct per-layer parameters. Auxiliary ablation for weight tying.
- **B5:** Sequence transformer receiving the serialised DAG as flat text. Two variants: decoder-only autoregressive trace generation, and encoder-decoder.
- **GSRE-unshared:** GSRE variant with distinct parameters per recurrence step. Isolates weight sharing.
- **B2/B3 (auxiliary):** Scratchpad models receiving only natural language. Demonstrate difficulty of learning decomposition; not primary comparisons.

### 3.5 Training and Evaluation

- **Phase 0:** Primitive pre-training on isolated samples within bounded ranges (numbers ≤50, strings ≤5).
- **Phase 1:** Full-system training on depth-3 graphs with node-level supervision. Auxiliary losses at intermediate nodes.
- **Metrics:** Exact-match accuracy (≤1% relative error for numerics); depth-stratified accuracy; OOD generalisation slope (average per-depth accuracy decline beyond depth 3). 95% confidence intervals over 5 seeds.

### 3.6 Expected Outcomes and Interpretation

- **GSRE > B4-tied:** Explicit dependency scheduling aids depth generalisation beyond weight tying alone.
- **GSRE ≈ B4-tied:** Null hypothesis; implicit propagation suffices.
- **Ablation patterns:** GSRE > GSRE-unshared indicates weight sharing matters; B4-tied > B4 indicates same for graph transformer.

Either outcome is a valuable contribution.

### 3.7 Publication Strategy

Track 1 will be submitted as a standalone paper framed purely as a learning-to-execute investigation. The NBRS branding and longer vision will appear only in a short "Future Work" paragraph.

---

## 4. Track 2: Learned Decomposition with a Fixed Primitive Set

**Goal:** Remove the oracle grammar; enable the system to autonomously construct the reasoning graph given a problem goal and a known set of primitive operations.

### 4.1 Scope

Track 2 addresses neural program synthesis with a known DSL (the primitive library). It is deliberately scoped:

- Primitive library is fixed (same ~15 operations from Track 1).
- Tasks are initially linear chains (Tier 5 style).
- No invention of new primitives.

This is a proof-of-concept for learned composition.

### 4.2 Graph Policy Network (GPN)

A neural module (lightweight Transformer) that observes the current partial graph and outputs an action.

**State representation:**
- Goal embedding from problem specification.
- Per-node: type, state, and resolved value if available.
- Value pool: all currently resolved values available for binding.

**Action space (precisely defined):**
1. Select a `PENDING` node with unsatisfied arguments.
2. Choose a primitive type τ from the library whose output type matches.
3. Instantiate τ's fixed expansion schema (template specifying required child slots).
4. Bind each child slot to either an existing resolved value or a new sub-goal node (becomes `PENDING`).
5. Optionally issue `FAIL` to trigger backtracking.

**Training:**
- **Bootstrapping (optional):** Brief behavior cloning on problems where oracle traces exist (depth 1–3), to accelerate early learning.
- **Primary training:** Reinforcement learning with sparse reward (+1 for correct final answer). Policy gradient (PPO) with a learned value function trained on the same sparse signal.
- **Curriculum:** Increase chain length progressively.

**Search:** MCTS using the GPN as policy prior and value function. When a dead end is reached, the search backtracks to the most recent untried alternative. Branching factor is limited by library size.

### 4.3 Baselines

- Direct sequence-to-sequence model (no intermediate reasoning).
- Scratchpad Transformer generating text reasoning steps.
- Random expansion policy (ablates learned policy).

### 4.4 Risks Acknowledged

- **Sparse reward:** Training may be unstable. Intermediate milestone signals will be added only if necessary and fully documented.
- **Scalability:** Action space size limits extension to deep branching DAGs. Linear chains are a proof-of-concept; general DAGs are future work.
- **Dependence on Node Processor:** GPN assumes correct Node Processor execution. Track 1's characterisation of processor limits will inform design.

Track 2 will be pursued only after Track 1 validates the execution architecture.

---

## 5. Track 3: Toward Autonomous Primitive Invention

**Vision:** Enable the NBRS to expand its own primitive library, composing known operations into new abstractions and ultimately inventing novel operations.

### 5.1 Core Research Challenges

Track 3 is a long-term research agenda, not a scheduled project phase. Fundamental unsolved problems include:

1. **Subgraph abstraction and parameterisation.** Identifying reusable subgraphs in successful traces requires scalable graph-isomorphism detection and generalisation (replacing constants with typed parameters).
2. **Neural execution of novel primitives.** Adding a new primitive requires the shared Node Processor to learn it without forgetting prior ones. Continual learning for multi-task Transformers remains open.
3. **Verification of invented primitives.** A newly proposed operation must be validated. Without an external oracle, the system must generate reliable test cases automatically.
4. **Search vs. abstraction trade-off.** When to search with existing primitives vs. invent a new one involves meta-reasoning.
5. **Symbolic-neural interface stability.** When the system modifies its own operation schemas, the clean separation between symbolic scheduling and neural execution becomes recursive.

### 5.2 Possible Starting Points

- **Hybrid library learning:** Restrict to simple arithmetic or string functions with tractable formal semantics.
- **Neural module networks with introspection:** A meta-controller that monitors Node Processor failures and proposes new sub-networks trained on failure examples.
- **Benchmarks for creative reasoning:** Datasets requiring non-obvious primitive compositions, measuring whether the system can ever succeed without human-provided primitives.

Track 3 will be refined as Track 2 results and external advances (program synthesis, continual learning, library learning) mature. No timeline is attached.

---

## 6. Summary

The NBRS roadmap charts a path from a concrete, falsifiable hypothesis about explicit dependency tracking to an ambitious vision of autonomous compositional reasoning.

- **Track 1** is a rigorous, implementation-ready experiment that will yield a valuable publication and empirical foundation.
- **Track 2** extends the architecture toward autonomous decomposition, with scoped goals and acknowledged challenges.
- **Track 3** outlines fundamental research questions for long-term exploration.

The programme aims to shift the paradigm of machine reasoning from token-level simulation to explicit, structural execution—one carefully validated step at a time.

**Approved for implementation: Track 1.**  
**Tracks 2 and 3 remain exploratory and will be refined based on Track 1 outcomes.**
