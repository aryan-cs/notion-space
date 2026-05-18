# notion-space

> **Notion-Based Reasoning System (NBRS)** — a research programme on neurosymbolic reasoning with explicit dependency graphs.

This repository hosts the research proposal, theoretical foundations, and (in due course) the implementation of the NBRS programme. It is a planning repository at present: the design has been formalised, the experimental track has been approved, and code will land here as Track 1 is implemented.

---

## What is NBRS, in one paragraph?

Modern large language models simulate multi-step reasoning by predicting tokens one at a time. This works for shallow problems and fails on deep ones: nested arithmetic, transitive inference, structured games. The NBRS hypothesis is that reasoning is not a sequence of tokens but a partial-order of operations — a typed directed acyclic graph (DAG) of *notions*, each a small operation that consumes typed inputs and produces a typed output. A tiny shared neural module (the *node processor*) is applied recurrently to each node once its arguments are ready. Because the architecture enforces dependency order by construction, the model never has to learn that constraint from data, and compositional depth is decoupled from neural capacity.

NBRS is intended to live alongside a language model — the LM handles perception and language, NBRS handles precise multi-step reasoning.

---

## Why this might matter

If the hypothesis holds, it has three immediate consequences:

1. **Depth generalisation by construction.** A small node processor trained on shallow graphs is theoretically expected to execute deeper graphs with only geometric degradation in correctness, with no growth in parameter count. The companion proof develops this argument formally as a *capacity–depth decoupling theorem*.
2. **Interpretable execution traces.** Every reasoning step is a typed operation on named values. The trace of an NBRS execution is, by construction, a transparent computation graph rather than a vector of activations.
3. **A path to learned reasoning beyond fixed primitives.** Tracks 2 and 3 of the programme propose, respectively, learning to construct the reasoning graph autonomously, and learning to extend the primitive library itself.

If the hypothesis does *not* hold — and Track 1 is designed to be a falsifiable test — that is also a meaningful contribution. It would tell us that the architectural complexity of explicit scheduling is not justified for compositional reasoning at the depths we tested, and that simpler weight-tied attention-masked graph transformers already capture the relevant inductive bias.

---

## The three tracks

| Track | Status | Question |
|-------|--------|----------|
| **1. Compositional execution study** | *Approved, implementation-ready* | Given a perfect reasoning DAG, does explicit dependency scheduling + recurrent neural reuse improve out-of-distribution depth generalisation vs. weight-tied graph transformers and sequence transformers? |
| **2. Learned decomposition** | *Exploratory* | Can a neural policy learn to construct reasoning DAGs autonomously from a fixed primitive library, trained by RL + MCTS? |
| **3. Primitive invention** | *Open research agenda* | Can the system extend its own primitive library — abstracting reusable subgraphs and inventing new operations through experience? |

The separation matters: Track 1 is a self-contained, falsifiable experiment that will be published as a standalone learning-to-execute study. The broader NBRS vision will appear there only as a brief "Future Work" paragraph. Tracks 2 and 3 will be refined based on Track 1 outcomes.

For the long form, read [PROPOSAL.md](PROPOSAL.md).

---

## Architecture in 30 seconds

```
       ┌─────────────────────────────────────────────┐
       │              Task specification              │
       └─────────────────────────────────────────────┘
                          │
                          ▼
          ┌──────────────────────────────┐
          │   Oracle Grammar (Track 1)   │   Track 2: a learned
          │      Hand-crafted DAG        │   Graph Policy Network
          │        expansion             │   builds the DAG.
          └──────────────────────────────┘
                          │  (typed DAG)
                          ▼
          ┌──────────────────────────────┐
          │     Traversal Engine          │
          │  PENDING → READY → EXEC →     │
          │  RESOLVED   (hard-gated by    │
          │   parent state)               │
          └──────────────────────────────┘
                          │   per node:
                          │   (type, spec, parent values)
                          ▼
          ┌──────────────────────────────┐
          │   Node Processor              │
          │   shared 2-layer Transformer  │
          │   (≈ 2M parameters)           │
          │   reused across all nodes     │
          └──────────────────────────────┘
                          │  resolved value
                          ▼
                       answer
```

The key architectural commitments:

- A node only executes when **all** of its parents are resolved. This is enforced by the state machine, not by attention.
- The node processor's input is **only**: the primitive type, the spec, and the resolved values of the direct parents. No global graph embedding. No history vector. No depth signal.
- The same parameters are used at every node, every step. Depth is unrolled in time, not in parameters.

---

## Repository layout

```
notion-space/
├── README.md               ← you are here
├── PROPOSAL.md             ← the research roadmap (the canonical proposal)
└── docs/
    ├── proof.tex           ← formal theory: definitions, theorems, proofs
    └── proof.pdf           ← compiled PDF (run tectonic to regenerate)
```

When code lands, the expected structure will be:

```
notion-space/
├── nbrs/                   ← Python package
│   ├── primitives/         ← primitive library and oracle semantics
│   ├── traversal/          ← the state-machine executor (Algorithm 1)
│   ├── node_processor/     ← the shared neural module
│   ├── data/               ← Tier 5 task generator
│   └── baselines/          ← B4-tied, B4 unshared, B5
├── experiments/            ← training + eval scripts for Track 1
├── tests/
└── pyproject.toml
```

---

## How to read the documents

You probably want, in order:

1. **[README.md](README.md)** *(this file)* — five-minute orientation.
2. **[PROPOSAL.md](PROPOSAL.md)** — the human-readable research roadmap. The three tracks, their goals, scopes, baselines, and expected outcomes. Approximately 20-minute read.
3. **[docs/proof.pdf](docs/proof.pdf)** — the theoretical foundations. Formal definitions of the typed DAG, the executor's operational semantics, the four core theorems (determinism, order independence, termination, soundness), the capacity–depth decoupling argument, and the OOD generalisation theorem. Approximately 60–90-minute read. Read with pencil.

If you only have time for one section of the proof, read **§3 (The Graph-Structured Recurrent Executor)** and **§5 (Information-Theoretic Analysis)**. Those two sections contain the load-bearing claims.

---

## Building the proof PDF

The proof is written in standard LaTeX and compiles cleanly with [Tectonic](https://tectonic-typesetting.github.io/), which downloads the packages it needs on first use.

```bash
# install once
brew install tectonic            # macOS
# or follow instructions for your platform

# compile
cd docs
tectonic proof.tex
```

This produces `docs/proof.pdf`. The pre-compiled PDF is committed to the repository so casual readers do not need a LaTeX toolchain.

A traditional `pdflatex` or `latexmk` toolchain works equivalently if you have one installed:

```bash
cd docs && latexmk -pdf proof.tex
```

---

## Status

| Milestone | State |
|-----------|-------|
| Proposal written and reviewed | ✅ done |
| Theoretical foundations formalised | ✅ done |
| Track 1 design approved for implementation | ✅ done |
| Track 1 data generator (Tier 5) | ⏳ pending |
| Track 1 GSRE implementation | ⏳ pending |
| Track 1 baselines ($B_4$-tied, $B_4$ unshared, $B_5$) | ⏳ pending |
| Track 1 training runs (5 seeds) | ⏳ pending |
| Track 1 paper draft | ⏳ pending |
| Track 2 design refinement | ⏳ blocked on Track 1 outcomes |
| Track 3 problem decomposition | ⏳ blocked on Track 2 progress |

---

## A note on framing

NBRS makes a *structural* claim: that explicit, hard, architectural enforcement of dependency order is qualitatively different from soft, learned, attention-based enforcement, and that the difference matters for compositional depth generalisation. This is not a claim about scale, and it is not a claim about emergent capabilities. The Track 1 experiment is designed to be small, controlled, and reproducible, with all confounders held constant across architectures.

If you are a reviewer or collaborator, the place to push back is on the theoretical framing in [docs/proof.pdf](docs/proof.pdf) — particularly the local-context invariance argument in §6 and the expressivity-containment proposition in §4. Empirical disagreement is welcome but, given that the experiment has not yet been run, premature.

---

## Citation

A formal preprint will be released when Track 1 results are available. For now, please cite the repository:

```
@misc{nbrs2026,
  title  = {Notion-Based Reasoning System (NBRS): A Research Roadmap},
  author = {NBRS Working Group},
  year   = {2026},
  note   = {\url{https://github.com/aryan-cs/notion-space}}
}
```

---

## License

To be determined. Until a license file is added to this repository, treat the contents as "all rights reserved" with permission granted only for reading and academic discussion. A permissive open-source license will be added before any code is published.
