<p align="center">
  <img src="images/CAF-AotW-banner.svg" width="100%" alt="CAF AotW banner">
</p>

# 07/08/2026 &mdash; AotW#10: Alchemy — Agentic 3D Preliminary Design and Structural Analysis for Nuclear Facilities

---

## Science Story

The global expansion of nuclear power is driving demand for faster preliminary design of reactor systems — primary coolant circuits, secondary heat exchange loops, cooling/ventilation networks, and the buildings that house them. Today this is a labor-intensive, multi-stage process: engineers define system topology in 2D piping and instrumentation diagrams (P&IDs), manually recreate that topology in 3D BIM tools, then manually translate those models into finite element models (FEMs) for structural and seismic analysis. Each handoff between tools introduces interpretation errors, geometric inconsistencies, and loss of engineering intent, and any design change means repeating the whole chain. **Alchemy**, developed at Idaho National Laboratory, closes this gap by taking natural-language system descriptions or structured P&ID exports and autonomously producing analysis-ready 3D facility models — turning a process that once took hours of manual modeling into one that takes minutes.

---

## Agentic Motivation

Traditional BIM/FEM workflows rely entirely on manual re-authoring at every stage, with no shared data model connecting system intent to 3D geometry to structural analysis. Alchemy's agentic architecture removes this bottleneck in several ways:

- **Autonomous multi-step orchestration:** A LangGraph-based state graph carries a design end-to-end — parsing input, extracting and validating system intent, generating IFC geometry, converting to FEM, and running structural analysis — without a human re-entering data at each handoff.
- **Two-step reasoning to separate concerns:** Rather than asking one LLM call to simultaneously interpret engineering intent *and* conform to a strict schema, Alchemy splits this into (1) free-form intent extraction and (2) schema mapping guided by few-shot examples — making the pipeline far more reliable than one-shot generation.
- **Self-correction via deterministic validation:** Before any geometry is generated, the agent runs deterministic topological checks (duplicate IDs, dangling edges, edge symmetry). Failures are fed back to the LLM as targeted repair prompts (up to 3 iterations) rather than surfaced as hard errors — combining rule-based rigor with LLM flexibility.
- **Tool/backend delegation:** Computationally intensive tasks — layout optimization, IFC authoring, A* pipe routing, BIM-to-FEM conversion, Ansys structural analysis — are delegated to specialized backend components, keeping the LLM focused on interpretation and orchestration rather than geometry math.
- **Composable, agent-native deployment:** Built as a first-class Agent-to-Agent (A2A) participant rather than a wrapped standalone tool, Alchemy can be invoked interactively by an engineer or orchestrated automatically by other agents in a larger multi-agent nuclear engineering workflow — with no interface changes needed either way.

---

## Implementation

Alchemy is a two-layer system: an **AI agent layer** handling natural language understanding, schema generation, and orchestration, and a **computational backend layer** executing geometry, routing, and structural analysis.

**Agent orchestration:** Built on **LangGraph**, the agent is implemented as a subclass of the `PrometheusAgent` base class and deployed on Idaho National Laboratory's **Prometheus** multi-agent platform via the **A2A protocol**. At startup it registers its agent card and advertises its `generate_optimized_ifc` skill to a platform registry; incoming task requests are routed to it by the Prometheus orchestrator or any A2A-compliant client, with progress streamed back via an `on_progress` callback. LLM calls are routed through **LiteLLM** for provider-agnostic access.

**Core workflow (7-node LangGraph `StateGraph`):** `parse` → `transform` (two-step intent extraction + schema mapping, with the topological validation/self-correction loop running inside this node) → `validate` → `generate` (layout optimization, IFC generation, pipe routing) → optionally `convert_apdl` → `run_analysis` → `finalize` (artifact registration). A shared conditional routing function evaluates state after each node and can short-circuit to `END` on unrecoverable errors or when clarification is needed.

**Backend pipeline:**
- *Layout optimization* — grid search over candidate equipment positions/orientations minimizing total pipe length, with collision checks and building-size determination.
- *BIM authoring* — native IFC 4.0 geometry (no proprietary intermediate formats) via **IfcOpenShell**, using constructive solid geometry for equipment and semantic IFC classes/PredefinedTypes (e.g., `IfcTank` / `REACTOR_PRESSURE_VESSEL`) to preserve engineering identity.
- *Pipe routing* — **A\*** search with a Manhattan-distance heuristic and elbow-minimizing cost penalties, routing pipes in priority order by nuclear safety classification.
- *BIM-to-FEM conversion* — IFC pipe/elbow geometry mapped to **Ansys APDL** PIPE288/ELBOW290 elements, with equipment ports converted to boundary conditions, for modal and seismic time-history analysis.

**Data management:** All generated artifacts (IFC models, FE models, analysis results) are registered as typed records in **DeepLynx**, INL's linked engineering data catalog, with references persisted in Redis — providing traceability across the full design lifecycle.

**Demonstrated results:** Across three test cases (a two-loop PWR system from natural language, a multi-system PWR+HVAC configuration, and an imported AVEVA P&ID export), Alchemy produced validated, zero-error IFC models and full structural analysis in roughly 90 seconds to 4 minutes — work that previously took hours manually.

---

## To Know More

### Source Code
- **Repository:** *[link not provided in source material — add if available]*
- **Documentation:** *[add if available]*
- **License:** Copyright 2025, Battelle Energy Alliance, LLC

### Additional Resources
- **Paper/Publication:** ICAPP 2026 submission — *Alchemy: An Agentic AI Framework for Automated Nuclear Facility Preliminary Design and Structural Analysis* (add DOI/link once available)
- **Blog Post:** *[add if available]*
- **Video/Presentation:** *[add if available]*
- **Website:** *[add if available]*
- **Contact:** Harleen Kaur Sandhu — HARLEEN.SANDHU@inl.gov · Drew Rizk — DREW.RIZK@inl.gov

---

*Last Updated: [Date]*
*Contributed by: [Author/Team Name — e.g., Idaho National Laboratory]*
