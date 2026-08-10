<p align="center">
  <img src="../images/CAF-AotW-banner.svg" width="100%" alt="CAF AotW banner">
</p>

# 08/10/2026 &mdash; AotW#14: STEER Supervisor Agent &mdash; Central Orchestration for Autonomous Electrochemistry

---

## Science Story

Autonomous electrochemical materials discovery spanning batteries, electrocatalysts, and corrosion-resistant coatings, demands the continuous coordination of experiments across shared laboratory hardware. In a fully autonomous setting, AI planning agents propose experiments around the clock while lab execution agents, data-analysis agents, operator dashboards, and human supervisors all require a single, consistent view of what is running, what is safe, and what has already been tried. At small scale this coordination can be managed informally with file watchers and ad-hoc scripting, but at production scale the pattern breaks catastrophically: two processes claim the same potentiostat, a crash leaves a robot arm logically locked forever, and redundant experiments consume irreplaceable lab time because no durable memory survives across sessions.

STEER (Supervisory Tool for Experiment Execution and Recovery), developed at Argonne National Laboratory, is the central orchestration and safety layer for autonomous experimentation. It is not a scientific-reasoning agent; it is the central orchestration and safety layer for the autonomous lab — the single authoritative state manager for experiment queueing, risk-based approvals, crash recovery, duplicate detection, instrument coordination, and operator-visible fault handling. By making the multi-agent laboratory stack reliable, recoverable, and auditable, it enables the kind of sustained, lights-out experimental campaigns that accelerate materials discovery at scale.

---

## Agentic Motivation

Traditional laboratory automation relies on static scripts and sequential job runners that assume every run will succeed, every instrument will be available, and every process will stay alive. These assumptions fail in autonomous settings where experiments run continuously, multiple AI agents interact asynchronously, and physical hardware introduces unpredictable faults. The Supervisor Agent is motivated by the concrete coordination and reliability failures that emerge in real autonomous laboratories, with each failure mode mapped to an explicit agentic control mechanism:

- **Concurrent execution races:** Atomic execution locks, implemented through SQLite in Write-Ahead Logging (WAL) mode with transactional check-and-set behavior, guarantee that only one experiment can actively execute at a time, while startup checks and PID tracking reduce the risk of competing supervisor instances.
- **Competing daemon control loops:** A renewable supervisor leadership lease ensures that only one daemon instance owns queue polling, recovery checks, and pending-file intake at a time.
- **Duplicate file intake in daemon workflows:** Pending experiment files are atomically claimed into an inflight area before parsing so that polling loops cannot double-submit the same plan.
- **Instrument conflicts:** Dedicated instrument reservation locks prevent multiple experiments from simultaneously claiming the same potentiostat, robot, or shared lab subsystem.
- **Hung instruments from indefinite execution:** A configurable execution timeout (default 3600 seconds) enforces a hard time limit on orchestrator calls; on timeout, the supervisor calls the orchestrator's abort method and channels the failure through the standard recovery-gate and instrument-quarantine path, preventing stuck hardware from running uncontrolled.
- **Process crashes and orphaned state:** A heartbeat mechanism updates execution state every 30 seconds, and a recovery manager detects dead processes or heartbeat timeouts after 120 seconds, then marks interrupted runs as incomplete, quarantines affected hardware into an `UNKNOWN` state, and restores the supervisor to a safe recovery posture. When crash recovery identifies that manual intervention is required, startup is now blocked and a recovery gate is activated — the supervisor will not proceed to IDLE until the operator acknowledges.
- **Unsafe automatic resumption after faults:** After execution failures, the supervisor activates an operator acknowledgement gate and, after repeated failures, enforces a configurable cooldown before queued work is allowed to resume.
- **Misconfiguration at startup:** The supervisor validates critical runtime settings such as heartbeat timing, queue limits, execution timeout, authentication, and leadership lease configuration before starting work, failing fast on unsafe combinations instead of running with ambiguous behavior.
- **Unauthenticated access to safety-critical controls:** API key authentication (Bearer token or X-API-Key header) is enforced on all mutating REST endpoints — experiment approvals, rejections, quality updates, recovery acknowledgements, and instrument quarantine clears — so that network-accessible dashboards cannot be exploited by unauthorized users. Read-only state endpoints remain open. Startup validation rejects a configuration with authentication enabled but no key configured.
- **Unsafe autonomous execution:** A configurable safety gate evaluates experiments for HIGH or CRITICAL risk using factors such as temperature, pressure, and priority, and can require human approval before execution.
- **Redundant experimentation:** Persistent experiment memory blocks re-running experiments that already produced valid results, using parameter-based similarity checks with exact sample matching and temperature or pressure tolerance windows, while still allowing repeats of failed or incomplete runs.
- **Urgent work buried in queues:** A configurable priority queue supports LOW, NORMAL, HIGH, and CRITICAL scheduling so urgent runs can preempt routine work.
- **Tight coupling to any one interface:** Callback hooks, an authenticated REST API surface, and an event bus allow dashboards and external services to observe and interact with the supervisor without embedding orchestration logic into the UI, including acknowledgement of recovery gates and administrative return-to-service actions for quarantined instruments.

This is the core agentic contribution of the project: not replacing scientists with a monolithic model, but making a multi-agent laboratory stack reliable enough to be trusted over long-running, unattended experimental campaigns. The Supervisor Agent provides the autonomous decision-making (automatic recovery, risk-gated approvals, duplicate suppression, execution timeout enforcement), multi-step reasoning (state-machine lifecycle management across failures and restarts), tool integration (instrument locks, file-system coordination, database-backed state), real-time adaptation (heartbeat monitoring, dynamic queue management, fail-closed fault handling), and access control (API key authentication on safety-critical endpoints) that no static script or simple job runner can deliver.

---

## Implementation

The STEER Agent is implemented in **deterministic Python**, with no LLM dependency in the core orchestration path. Its single source of truth is a persistent **SQLite database running in WAL mode**, which stores supervisor state, experiment history, queue contents, instrument locks, instrument status, heartbeat information, experiment quality annotations, recovery-gate metadata, and an event audit trail. This makes the system restart-tolerant, auditable, and suitable for single-machine laboratory deployment.

The implementation is organized into several coordinated components:

1. **SupervisorAgent** manages lifecycle control, startup checks (including blocking when crash recovery requires manual intervention), configuration validation, execution timeout enforcement, experiment submission, approval handling, and daemon-mode operation.
2. **StateDatabase** provides atomic execution locks, atomic plan-claim coordination, daemon leadership leasing, instrument reservations, instrument-status persistence, priority queue operations, heartbeat updates, persisted recovery-gate metadata, and full state tracking.
3. **HeartbeatMonitor and RecoveryManager** detect failures through dead-PID checks and heartbeat timeout logic, then mark interrupted experiments as `INCOMPLETE`, quarantine affected instruments, and hand control back through a fail-closed recovery workflow.
4. **SafetyGate** performs risk assessment with LOW, MEDIUM, HIGH, and CRITICAL classifications and supports human-in-the-loop approval workflows with configurable defaults.
5. **ExperimentMemory** tracks historical experiments, blocks duplicates when prior results are valid, distinguishes `VALID`, `INVALID`, and `INCOMPLETE` outcomes, and can provide context such as similar runs, success rates, and average durations.
6. **DashboardHooks, RESTAPIHooks, EventBus, and a built-in dashboard server** expose state changes to user interfaces and external services through callbacks, route definitions for web frameworks, durable event delivery, a lightweight localhost operator UI, operator acknowledgement of repeated-failure gates, and manual clearing of quarantined hardware after inspection. Safety-critical mutating endpoints support API key authentication (Bearer token or X-API-Key header) to prevent unauthorized approvals, acknowledgements, or quarantine clears.
7. **Real-agent runtime adapters** can bootstrap the sibling `lab_agent` orchestrator and the sibling `data_analysis_agent` entry point directly from the supervisor package, including either inline post-run analysis or an asynchronous event-driven analysis mode.

In the current implementation, failure handling is intentionally fail-closed rather than optimistic. A successful experiment can advance the queue automatically, but an execution fault — including timeout-triggered aborts — can pause follow-on work until an operator acknowledges the issue, and repeated failures can enforce a cooldown window before the queue is allowed to resume. If crash recovery identifies that manual intervention is required, startup itself is blocked until the operator acknowledges. This makes the supervisor more suitable for long-running physical-lab campaigns where silent automatic recovery can be riskier than temporary lost throughput.

The nominal experiment lifecycle is:

`SUBMITTED → AWAITING_APPROVAL or QUEUED → EXECUTING → SUCCESS / ERROR / RECOVERING → IDLE`

Experiments can enter through a file-based pending directory in daemon mode or through a programmatic submission API. In daemon mode, pending plans are first claimed into an inflight area before submission so they are processed exactly once per watcher pass, and a renewable leadership lease ensures that only one daemon control loop is active at a time. Configuration is environment-driven, and the daemonized supervisor is designed to validate critical settings before startup, watch pending work, process the queue, honor recovery-gate cooldowns, and detect crashed executions continuously. The design target is a production-grade orchestration layer for real-world autonomous wetlab deployment, where safety, recoverability, and state consistency matter.

In the current codebase, that control plane can also bootstrap the real sibling-agent stack directly. A supervisor entry point can now create the lab-agent orchestrator, register the Autolab and OT-2 tool wrappers when available, normalize execution outputs into artifact-ready events, either call the electrochemical data-analysis agent inline or publish `experiment.artifacts_ready` for asynchronous downstream analysis, and optionally launch a minimal built-in dashboard on localhost for queue monitoring and operator actions.

Architecture Diagram

<p align="center">
  <img src="STEER_figure.jpg" width="100%" alt="STEER Agent architecture diagram showing planning, control-plane, data-analysis, operator, lab execution, and physical instrument layers">
</p>

Current implementation status should be noted clearly: the core orchestration, recovery, startup validation, and interface paths are implemented and regression-tested, but the project remains **under active validation in the real lab with robotic instruements**.

---

## To Know More

### Source Code
- **Repository:** Currently private (ANL internal); public release expected soon
- **License:** Pending, Copyright © Argonne National Laboratory, UChicago Argonne LLC. All rights reserved.

### Additional Resources
- **Paper:**: "Control-Plane Supervision for Safe, Recoverable, and Auditable Multi-Agent Autonomous Laboratories: A Supervisor-Agent Framework for Operational Reliability" Manuscript in preparation.
- **Intended use:** Central orchestration for safe, sequential execution of autonomous electrochemistry experiments with human approval for elevated-risk runs.
- **Runtime model:** Pure Python orchestration with SQLite state, atomic file-based experiment intake, daemon leadership leasing, configurable execution timeout with abort-on-hang, persisted recovery gates, startup blocking on manual-intervention recovery, fail-fast startup validation, API key authentication on safety-critical endpoints, real sibling-agent bootstrap for lab execution, a built-in localhost operator dashboard, and optional event-driven downstream analysis.
- **Contact:** Zhenzhen Yang — yangzhzh@anl.gov
- **Contact:** Brian J. Ingram — ingram@anl.gov
---

*Last Updated: 07/06/2026*
*Contributed by: Zhenzhen Yang, Brian J. Ingram — Argonne National Laboratory,Center for Steel Electrification by Electrosynthesis, Chemical Sciences and Engineering Division*
