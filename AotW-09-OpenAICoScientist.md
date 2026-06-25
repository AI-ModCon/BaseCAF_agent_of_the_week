<p align="center">
  <img src="../images/CAF-AotW-banner.svg" width="100%" alt="CAF AotW banner">
</p>

# 06/25/2026 - AotW#9: Open AI Co-Scientist - Open Hypothesis Evolution for Scientific Discovery

---

## Science Story

Scientific discovery often bottlenecks before an experiment is ever run: at the stage where a researcher must turn a broad goal, a fast-growing literature base, and partial domain intuition into a small set of hypotheses worth testing. Search engines can retrieve papers, and analysis workflows can process data, but neither one systematically creates, critiques, ranks, and refines candidate scientific ideas.

**Open AI Co-Scientist**, developed at Lawrence Livermore National Laboratory, addresses this early-stage discovery problem. A scientist enters a research goal in natural language, such as "Develop new methods for increasing the efficiency of solar panels," and the system generates a ranked set of research hypotheses that have been grounded in related arXiv literature, reviewed for novelty and feasibility, compared against one another, and evolved across iterative cycles.

The motivation came from the author's biology-inspired view of innovation: new ideas can often be understood as transformations of feature vectors - recombining, mutating, measuring distance from existing ideas, and then testing whether the result is useful and feasible. Google's AI co-scientist report showed a strong match to this view through a generate-debate-evolve architecture for scientific hypothesis generation, but the report did not release runnable source code. Open AI Co-Scientist turns that idea into an open-source, lightweight implementation that DOE researchers and the broader community can inspect, run, modify, and extend.

The current system is a prototype, not a production-grade scientific discovery platform. It is not intended to replace scientists; it is a scientist-in-the-loop assistant for expanding the hypothesis search space, surfacing non-obvious combinations, and helping researchers decide which ideas deserve deeper expert review or experimental validation.

---

## Agentic Motivation

A single LLM prompt can produce a list of hypotheses, but it does not provide a durable discovery process. It has no explicit division between creative generation and critical review, no principled way to compare alternatives, no memory of earlier cycles, and no mechanism for evolving weaker ideas into stronger ones. Open AI Co-Scientist uses a multi-agent loop because hypothesis discovery is not one task; it is a sequence of different reasoning modes that need to challenge and improve one another.

The system coordinates six specialized agents:

- **Generation Agent:** creates diverse initial hypotheses from the scientist's research goal, using related arXiv literature as grounding context.
- **Reflection Agent:** reviews each hypothesis for novelty and feasibility, producing structured critique rather than a simple accept/reject judgment.
- **Ranking Agent:** compares hypotheses pairwise and updates an **Elo rating** for each candidate, creating a tournament-style ranking that can improve as more comparisons are made.
- **Evolution Agent:** combines and revises top-performing hypotheses, preserving strong ideas while addressing weaknesses discovered during review.
- **Proximity Agent:** analyzes similarity across the hypothesis pool, helping the system avoid premature convergence on duplicate or near-duplicate ideas.
- **Meta-Review Agent:** evaluates the whole hypothesis set, summarizes the strongest candidates, and recommends next steps for the scientist or for another iteration.

This agent design maps directly onto the innovation loop: generation introduces mutation and recombination, proximity measures distance between ideas, reflection tests plausibility, ranking creates selection pressure, and evolution uses that feedback to produce the next generation of hypotheses. The result is a self-improving workflow rather than a one-shot brainstorming tool.

---

## Implementation

Open AI Co-Scientist is implemented as a **Gradio 5** web application and is available both as a live Hugging Face Space and as a local Python application. The public deployment uses **OpenRouter** for LLM access, while local deployments can use any OpenRouter-supported model available to the user.

At runtime, a supervisor coordinates the six-agent cycle and maintains session state across iterations. Each cycle retrieves related arXiv context, generates hypotheses, critiques them, ranks them through pairwise Elo comparisons, evolves the strongest candidates, checks similarity across the hypothesis pool, and produces a meta-review. The application stores run outputs under `results/` so researchers can inspect the generated hypotheses, critiques, rankings, and iteration history.

The repository also includes an AI transparency statement: the code and documentation were developed with AI-assisted coding, then reviewed and verified by human developers. This matters for the project itself because Open AI Co-Scientist is both an agentic science application and an example of how AI-assisted software development can make frontier research ideas available as open, inspectable tools.

Open AI Co-Scientist is released as open-source software under the MIT license as **LLNL-CODE-2010270**. The author welcomes collaborations and funding opportunities to mature the prototype into a more full-featured system suitable for production scientific use.

---

## To Know More

### Source Code

- **Repository:** https://github.com/llnl/open-ai-co-scientist
- **Live Demo:** https://huggingface.co/spaces/liaoch/open-ai-co-scientist
- **License:** MIT (LLNL-CODE-2010270)

### Additional Resources

- **Related Research Report:** https://storage.googleapis.com/coscientist_paper/ai_coscientist.pdf
- **Motivation Blog Post:** https://chunhualiao.github.io/blog/a-systematic-approach-to-innovation.html
- **Contact:** Chunhua Liao - liao6@llnl.gov

---

*Last Updated: 06/25/2026*

*Contributed by: Chunhua Liao - Lawrence Livermore National Laboratory*
