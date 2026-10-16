# Project Scope: Local Laptop AI Agent

## 1. Product Definition

The product is an open-source, local-first AI operating layer for a personal laptop. It continuously observes the host, builds a longitudinal machine state, detects abnormal or degrading behavior, investigates likely causes, explains findings with evidence, recommends concrete action items, and performs approved remediation through a constrained tool system. The product is intentionally closer to a personal AIOps/SRE agent for an endpoint than to a conventional desktop assistant or system-monitoring dashboard.

The core interaction is a closed loop: Observe -> Understand -> Detect -> Investigate -> Explain -> Recommend -> Request approval -> Execute -> Verify -> Learn. The model is never the source of truth for system facts and never receives unrestricted operating-system control. Deterministic collectors provide facts, analytical components derive features and anomalies, the local model plans and reasons over a bounded context, a policy engine controls authority, and an executor performs only typed capabilities that have passed validation and approval.

The project must remain useful with no cloud dependency. Cloud inference, remote telemetry, external APIs, and hosted models may be optional adapters, but the local execution path must be a complete first-class mode. The design should preserve the possibility of running the agent on Windows, Linux, and macOS with OS-specific collectors and executors under a common domain model.

## 2. Product Thesis and Differentiation

The product should not compete as another Task Manager clone with an LLM chat box. Its differentiating abstraction is the Laptop Digital Twin: a continuously updated representation of hardware, operating-system state, processes, applications, network behavior, storage behavior, power state, thermal state, workload mode, historical baselines, incidents, user preferences, and previous remediations. The AI reasons over this machine state rather than repeatedly inspecting raw metrics.

The second differentiator is evidence-backed root-cause investigation. A response such as "Chrome is using 8 GB" is an observation, not a diagnosis. The agent must compare absolute consumption against historical baseline, correlate resource changes with preceding events, understand workload context, consider competing causes, assign a confidence level, and explain the evidence used for the diagnosis.

The third differentiator is safe closed-loop remediation. A recommendation is not considered complete until an approved action executes and post-action telemetry verifies whether the intended effect actually occurred. Failed remediation must return the system to investigation rather than falsely claiming success.

The fourth differentiator is explicit human authority. Read-only capabilities can execute autonomously. State-changing capabilities must be classified by risk and routed through a policy engine. Human approval must be represented as durable run state, not as a UI-only button. Rejection, expiration, cancellation, and partial approval are first-class states.

## 3. Primary User Experience

The primary interface is a natural-language desktop agent accompanied by a lightweight dashboard. The user can ask questions such as "Why is my laptop slow?", "What is consuming memory?", "What changed since yesterday?", "Why did my battery drain so quickly?", "What happened while I was away?", and "Optimize my laptop for an ML workload." The agent can independently gather information required to answer these questions by calling observation and analysis tools.

When the agent identifies an actionable issue, the UI should present the diagnosis, severity, supporting evidence, estimated impact, candidate actions, expected outcomes, risks, and an explicit approve or reject decision. After approval, the UI should show the exact typed action being executed, the result, post-action verification, and the final incident state.

The interface must expose operational transparency without exposing private chain-of-thought. It should show user-safe reasoning artifacts such as evidence, hypotheses, confidence, tool names, sanitized arguments, action policy, approval status, execution result, and verification metrics.

## 4. Functional Scope

### 4.1 Observation Plane

The observation plane collects CPU utilization and frequency, memory usage and pressure, swap/page activity, per-process CPU and memory, disk capacity and I/O, filesystem usage, network I/O and connection counts, GPU utilization and memory where available, temperatures and throttling indicators where available, battery and power state, uptime and boot time, running services, application metadata, startup items where supported, system and application logs, container runtime state where installed, and selected operating-system events.

Collection must use the lowest-overhead native interface appropriate to the operating system. The agent must not rely on scraping terminal UI output when a supported system API exists. OS-specific collectors must normalize into a common event and metric model while retaining source-specific attributes where necessary.

The telemetry vocabulary should align with OpenTelemetry system and process semantic conventions where those conventions fit the information. The current conventions include system CPU, memory, filesystem, network, process, and uptime concepts and process-level CPU, memory, disk I/O, network I/O, thread, handle, paging, and uptime metrics. Their status should be treated carefully because several system semantic conventions remain under development; the project should pin its own stable internal schema and map to OTel rather than blindly depending on changing semantic-convention stability.

### 4.2 Machine State and Digital Twin

The machine-state layer maintains a current snapshot and longitudinal history. The state model contains host identity, hardware inventory, operating-system information, active workload mode, top processes, active applications, resource pressure, thermal and power state, network state, storage state, active incidents, known baselines, user policy preferences, and recently executed actions.

State changes are represented as events rather than only as sampled values. Examples include process started, process exited, service changed state, application opened, GPU workload began, battery charging started, temperature crossed a threshold, memory pressure increased, new startup item appeared, and a remediation completed.

The digital twin must support temporal queries. Examples include comparing the current state with an hourly baseline, asking what changed over the previous day, reconstructing the sequence leading to an incident, and determining whether a pattern is recurrent or novel.

### 4.3 Baseline and Workload Modeling

The baseline engine must learn normal behavior without requiring manual threshold configuration for every metric. Baselines should be conditioned on context such as time-of-day, power state, workload mode, and application set. A machine used for model training has a different normal profile from the same machine in an idle state.

The initial baseline implementation should use rolling robust statistics and exponentially weighted aggregates. The design should reserve interfaces for future seasonal models or learned anomaly models. Baseline calculations must produce both the expected range and the provenance needed for explanation.

Workload classification should initially be deterministic and feature-based, using active processes, GPU use, CPU characteristics, foreground applications, network state, and configured user hints. Later iterations may add an ML classifier. Initial workload labels should include IDLE, BROWSING, DEVELOPMENT, ML_TRAINING, GAMING, VIDEO_WORKLOAD, MEETING, BATTERY_SAVER, and HEAVY_COMPUTE, while allowing extensions.

### 4.4 Anomaly and Incident Engine

The anomaly engine operates independently of the LLM. It combines deterministic rules, baseline deviation, trend analysis, change-point detection, and cross-metric correlation. It must distinguish high absolute utilization from abnormal behavior. A 95 percent GPU load during a known training workload is normal; the same utilization during IDLE may be anomalous.

An incident is a structured object with incident_id, timestamp range, severity, category, affected resources, hypotheses, evidence references, baseline comparison, confidence, current status, recommended actions, approval state, execution state, verification state, and resolution summary.

Incident categories should include memory pressure, CPU saturation, thermal throttling, disk pressure, storage exhaustion, abnormal network activity, battery degradation pattern, process runaway, service failure, application crash pattern, startup anomaly, and stability degradation. The taxonomy must remain extensible.

### 4.5 Investigation and Root-Cause Analysis

The investigation engine builds a compact evidence graph from telemetry, events, process relationships, application ownership, historical baselines, and recent changes. It should answer "why" questions by ranking hypotheses rather than delegating raw diagnosis to the model.

The LLM receives a bounded incident context containing relevant facts, evidence references, uncertainty, historical comparison, user policies, tool availability, and the investigation state. The LLM may request additional observation tools when the evidence is insufficient. Every diagnosis must be traceable to concrete evidence IDs.

The initial root-cause ranking algorithm should prefer direct causal proximity, temporal alignment, deviation from baseline, resource magnitude, and repeated historical correlation. The score is a heuristic and must be exposed to evaluation, not presented as a mathematically proven causal inference system.

### 4.6 Agent and Tool Plane

The agent plane uses the OpenAI Agents SDK as the orchestration layer. The SDK provides agents, function tools, sessions, guardrails, human-in-the-loop approval interruptions, and tracing. Because the application is designed around a local model, the model adapter must point the SDK at a local OpenAI-compatible endpoint rather than assuming an OpenAI-hosted model.

Gemma 4 is the preferred primary local model family. Gemma 4 provides native function calling, reasoning modes, system-prompt support, and long-context variants. Gemma 4 E2B and E4B are the default laptop-oriented candidates, with selection based on a local capability probe and resource budget. A larger local Gemma variant may be supported on high-memory workstations but must never be required for the base product.

FunctionGemma should be considered a future specialized tool-routing or fine-tuned function-calling component rather than the primary conversational model. Google's documentation explicitly positions FunctionGemma as a lightweight model intended for specialized function calling and recommends task-specific fine-tuning; it is not intended to serve as the direct dialogue model.

The agent should support tools as typed functions or local MCP-compatible interfaces. The MVP should use typed local function tools because they provide the strongest control over schema validation and host permissions. MCP should be added behind the same tool registry so new capabilities can be plugged in without changing the planner.

### 4.7 Tool Catalog

The baseline tool surface includes system.get_time, system.get_timezone, system.get_uptime, system.get_os_info, system.get_hardware_inventory, performance.get_cpu_metrics, performance.get_memory_metrics, performance.get_gpu_metrics, performance.get_disk_metrics, performance.get_network_metrics, thermal.get_temperature, thermal.get_throttling, power.get_battery_status, process.list_processes, process.inspect_process, process.get_process_history, services.list_services, services.inspect_service, logs.query_system_logs, logs.query_application_logs, storage.get_usage, storage.find_large_items, network.get_connections, history.query_metrics, history.query_events, history.get_baseline, incident.get_incident, and workload.get_current_mode.

State-changing tools should include process.terminate, service.restart, application.restart where safely supported, container.restart where a supported runtime exists, power.set_profile, and cleanup.execute for explicitly defined cleanup operations. Destructive file deletion must not be exposed through a generic shell tool in the MVP.

A capability must always declare a stable name, description, argument schema, effect class, risk class, required authority, timeout, idempotency properties, supported platforms, dry-run support, and verification strategy.

### 4.8 Human-in-the-Loop

Human approval is mandatory for all medium, high, and destructive actions. The approval decision must be represented inside agent run state and survive process restarts when the run is configured for durable execution. The approval request must include a human-readable action description, exact structured parameters, reason, evidence, expected benefit, foreseeable risk, and rollback or recovery behavior where applicable.

The agent must never infer approval from vague conversational intent such as "do whatever is needed." Approval must be explicit for the concrete action or a previously established bounded policy. A policy such as "allow restarting Docker containers" may authorize a class of future actions, but only within a defined scope.

The OpenAI Agents SDK's current HITL model pauses execution when an approved tool is requested, surfaces interruptions, serializes state through RunState, and resumes after approval or rejection. This behavior should be used where the SDK is the runtime authority, but the product policy engine must remain the final security boundary for local OS capabilities.

### 4.9 Execution and Verification

All actions must pass through a policy engine and executor. The executor must accept only structured commands produced by registered tools. No LLM-generated shell string may be executed directly.

Every action creates an execution record containing action_id, tool name, validated parameters, requesting agent, policy decision, user decision, start time, end time, exit status, sanitized stdout/stderr, affected resources, and verification result.

Verification must be tool-specific. Restarting a service should verify service health. Terminating a process should verify process absence and measure system impact. Changing a power profile should verify the effective profile. Cleanup should verify the claimed disk-space improvement. A remediation is successful only when its verification predicate passes.

### 4.10 Notifications and Proactive Intelligence

The agent may proactively notify the user when an incident crosses a configured threshold, a trend indicates a meaningful future problem, a previously approved automation policy is about to execute a consequential operation, or a recurring issue merits attention. Notifications must be rate-limited and deduplicated so the product does not create alert fatigue.

The agent should provide daily and weekly summaries covering significant performance events, abnormal trends, major resource consumers, resolved incidents, recurring issues, and suggested preventive actions.

### 4.11 Machine Memory

Machine memory is distinct from conversational memory. It stores stable facts about the host, user-approved preferences, learned workload patterns, recurring incidents, prior diagnoses, remediation history, and post-action outcomes.

Memory retrieval must be relevance- and time-aware. Historical information must be clearly marked as historical; the agent must not treat an old process, service, or configuration as currently active without fresh observation.

## 5. Agent Harness Architecture

Codex is a development and repository execution harness for building the project, not the production authority over the user's laptop. The repository will contain AGENTS.md and instructions.md that encode architecture rules, test gates, safe development procedures, and task boundaries. Codex sandboxing and approval settings should remain restrictive during implementation, especially for operations touching the host, credentials, network, or generated artifacts.

The production agent runtime is the OpenAI Agents SDK. Its Agent and Runner abstractions should manage model turns, tools, guardrails, sessions, interruptions, and execution state. The application must use an explicit model adapter so local Gemma can be swapped for another OpenAI-compatible local model without changing agent business logic.

The recommended runtime split is a Python agent-service process for orchestration and analytics plus a Rust local daemon for host-level collection and privileged capabilities. Communication between them should use a local authenticated IPC channel. For Windows, Linux, and macOS, the Rust daemon should expose a platform-independent RPC contract with OS-specific implementations beneath it.

## 6. System Architecture

```text
                           Desktop UI
                      Tauri + React/TypeScript
                               |
                               | local authenticated IPC
                               v
                    +--------------------------+
                    |      Agent Service       |
                    | Python                   |
                    | OpenAI Agents SDK        |
                    | Planner / Reasoner       |
                    | Tool Registry            |
                    | Guardrails / HITL        |
                    +-----------+--------------+
                                |
                  OpenAI-compatible localhost API
                                |
                                v
                    +--------------------------+
                    |    Local Gemma Runtime   |
                    |  Gemma 4 E2B/E4B         |
                    | Ollama or llama.cpp      |
                    +--------------------------+

                                ^
                                |
                    +-----------+--------------+
                    |      Intelligence        |
                    | Baseline Engine           |
                    | Anomaly Engine            |
                    | Correlation / RCA         |
                    | Workload Classifier       |
                    | Forecasting               |
                    +-----------+--------------+
                                ^
                                |
                    +-----------+--------------+
                    |       State Store         |
                    | SQLite WAL                 |
                    | snapshots / events /      |
                    | metrics / incidents /      |
                    | policies / action logs     |
                    +-----------+--------------+
                                ^
                                |
                    +-----------+--------------+
                    |     Rust Local Daemon      |
                    | collectors / OS adapters   |
                    | executor / policy hooks    |
                    +-----------+--------------+
                                |
                 +--------------+--------------+
                 |              |              |
                CPU            GPU            RAM
                 |              |              |
                Disk         Network         Battery
                 |              |              |
              Thermal       Processes        Services
```

## 7. Data Architecture

SQLite is the default local source of truth because the workload is single-user, local, and durability-sensitive. WAL mode should be enabled. High-frequency samples should be aggregated before long-term retention. The storage model should separate raw short-retention samples, minute-level rollups, hourly/daily summaries, events, incidents, and durable action history.

Recommended logical entities are host, hardware_component, process_snapshot, application, metric_sample, metric_rollup, system_event, workload_state, baseline, anomaly, incident, hypothesis, evidence, recommendation, approval_request, action_execution, verification_result, user_policy, notification, memory_fact, and model_run. Every mutable or security-sensitive entity must have a created_at and where appropriate an updated_at plus provenance metadata.

The retention strategy should default to seconds-level samples for a short window, minute-level aggregates for a medium window, and hourly/daily aggregates plus incidents for long-term history. Retention must be configurable and deletion must be deterministic and auditable.

## 8. Observability

The product itself is an observability system and must also be observable. Internal telemetry should use OpenTelemetry APIs where practical. Trace spans should cover agent runs, model inference, tool selection, tool execution, approval waits, remediation, and verification. Metrics should cover collector overhead, inference latency, token usage where available, tool-call rate, approval latency, anomaly rate, false-positive rate, remediation success, verification success, and notification volume.

The default deployment should keep telemetry local. Export to an external collector must be opt-in and must require explicit configuration. Secrets, raw user command text, file contents, network payloads, and sensitive logs must never be exported accidentally.

## 9. Security Architecture

The security model is zero-trust between model output and host authority. Model output is untrusted data. Tool schemas, parameters, platform capability checks, policy rules, and execution allowlists are the authority.

The local agent service must authenticate its connection to the Rust daemon. On operating systems that support it, use a per-user local transport with OS-level access controls. If TCP localhost is unavoidable, require a random per-install token stored in the OS credential store or a protected local secret file, reject non-loopback access, and use replay-resistant request identifiers.

The Rust daemon should run with the minimum OS privileges required. Privileged operations should be isolated into the smallest possible code path. Secrets must be stored using platform credential facilities or protected configuration, never in source code or telemetry payloads.

The project must not expose a generic unrestricted shell execution tool in the base product. Any future shell capability must be isolated, policy-scoped, approval-gated, timeout-bound, resource-limited, and separately audited.

Prompt injection is relevant even for a local assistant because process names, file names, log lines, application content, network metadata, and user-controlled artifacts can contain malicious instructions. Tool results must therefore be treated as untrusted observations. The agent must not follow instructions embedded inside telemetry, logs, or process metadata unless the user explicitly asks it to analyze that content.

## 10. Model Strategy

The local model abstraction must support at least three providers: Gemma through llama.cpp OpenAI-compatible server, Gemma through Ollama, and an optional remote provider adapter for comparative evaluation. Google currently documents Gemma 4 function calling and local deployment paths, and its small E2B/E4B variants are designed for edge/laptop execution. Google also documents llama.cpp as a local inference path with an OpenAI-compatible server endpoint.

The default configuration should probe the host and choose a model profile based on available RAM, GPU memory, accelerator availability, and configured latency budget. The agent should prefer an E4B-class model when the host can comfortably run it while maintaining application responsiveness; E2B should be the fallback on constrained devices.

Inference quality must be evaluated specifically for tool selection, argument correctness, evidence-based diagnosis, refusal of unsafe actions, multi-step investigation, and recovery after tool errors. Generic language-model benchmarks are not sufficient for this product.

## 11. Model Context Contract

The LLM should never receive the entire telemetry database. Each turn receives a generated context package containing current machine summary, relevant historical baseline, active workload mode, incident summary, ranked hypotheses, evidence objects, available tools, user policies, pending approval state, prior tool results for the current run, and a strict instruction to use tools whenever fresh host state is required.

Tool results must be normalized and size-limited. Large logs and process tables must use paging, top-k, filtering, or summarized evidence. The agent must be able to request more detail when needed rather than flooding the model context.

## 12. Permission Model

The product should expose autonomy modes as policy presets: OBSERVE means no state changes; ASSIST means the agent can investigate and recommend but every state-changing action requires approval; GUARDED allows predefined low-risk actions under explicit user policy; AUTONOMOUS allows only narrowly scoped action classes the user has explicitly enabled.

Policy evaluation should consider tool name, effect class, target resource, parameters, current incident severity, user mode, time restrictions, rate limits, and whether the target falls inside an approved scope. Policy evaluation must fail closed when information is missing or ambiguous.

## 13. Performance Budgets

The monitoring agent must be a good citizen on the laptop it is monitoring. The collector target is less than 1 percent average CPU overhead under normal operation on a modern laptop, with short bursts tolerated for event processing. Baseline memory overhead excluding the model runtime should target less than 150 MB. Storage growth should be bounded by retention policies.

Inference should be event-driven rather than continuously invoked. The model should be called for user requests, material anomalies, scheduled summaries, or active investigations. The telemetry loop must continue functioning when the model is unavailable.

## 14. Failure Modes

The product must degrade gracefully. If Gemma is unavailable, the telemetry daemon and deterministic dashboard continue operating. If the agent service crashes, collection resumes and queued events remain durable. If a tool times out, the run records a structured failure and the planner may choose a safer alternative. If approval expires, the action is canceled. If execution succeeds but verification fails, the incident remains open and the agent investigates rather than declaring resolution.

The product must never fabricate a tool result, claim an action executed when the executor did not confirm execution, or claim a problem is fixed without passing verification.

## 15. Phased Delivery

### Phase 0: Architecture and Contracts

Freeze the domain model, local RPC contract, tool schema contract, event taxonomy, permission model, model adapter interface, persistence schema, and repository conventions. Produce ADRs for Rust/Python boundary, SQLite schema, local inference provider, and approval authority.

Acceptance is reached when schemas compile, contract tests pass, architecture diagrams match interfaces, and a minimal end-to-end harness can record a mock observation -> diagnosis -> approval -> action -> verification cycle without real host changes.

### Phase 1: Read-Only Laptop Intelligence

Implement the Rust collectors, normalized telemetry ingestion, SQLite persistence, machine-state snapshot, deterministic dashboard, baseline engine, and initial anomaly detector. Add the read-only tool registry and local Gemma inference through an OpenAI-compatible server.

Acceptance is reached when the agent can answer real questions about current CPU, memory, processes, disk, network, uptime, battery, and workload state using fresh tool results and can explain deviations from its own historical baseline.

### Phase 2: Agentic Investigation

Add the investigation planner, evidence graph, root-cause ranking, history retrieval, incident lifecycle, structured diagnosis, and multi-step tool calling. Implement explicit traceability from diagnosis claims to evidence IDs.

Acceptance is reached when seeded incidents can be diagnosed with measurable accuracy and the agent can recover from failed or incomplete tool calls.

### Phase 3: Human-Gated Remediation

Add typed action tools, policy engine, HITL interruptions, durable approval state, executor, and verification predicates. Start with safe actions such as restarting a user-approved development container or service and switching power profiles.

Acceptance is reached when every state-changing action is policy-checked, explicitly approved, executed through typed capabilities, and verified. No arbitrary shell path exists.

### Phase 4: Proactive Personal AIOps

Add scheduled summaries, recurring-incident detection, trend forecasting, notification policies, machine memory, user policy presets, and workload-aware recommendations.

Acceptance is reached when the product can identify recurrent issues, distinguish normal workload behavior from true anomalies, and produce useful preventive recommendations without excessive alert noise.

### Phase 5: Cross-Platform Hardening and Open-Source Release

Complete platform adapters, packaging, update strategy, signed release artifacts where practical, security review, performance profiling, benchmark suite, documentation, contributor guide, reproducible development environment, and release automation.

Acceptance is reached when Windows, Linux, and macOS support the common read-only path and the supported action matrix is documented per operating system.

## 16. Repository Shape

```text
laptop-ai-agent/
  AGENTS.md
  instructions.md
  project_scope.md
  README.md
  LICENSE
  SECURITY.md
  CONTRIBUTING.md
  Makefile.toml
  pyproject.toml
  rust-toolchain.toml
  .github/
  docs/
    adr/
    architecture/
    threat-model/
    evaluation/
  apps/
    desktop/
  services/
    agent/
      core/
      tools/
      policies/
      memory/
      investigation/
      model/
      telemetry/
      api/
  crates/
    hostd/
    collectors/
    executor/
    rpc/
  packages/
    schemas/
    test-fixtures/
  migrations/
  tests/
    unit/
    integration/
    contract/
    adversarial/
    evaluation/
  scripts/
```

## 17. Evaluation Strategy

The project needs a fixed incident benchmark rather than relying on anecdotal demonstrations. Each benchmark case should define a machine state, event sequence, ground-truth primary cause, plausible distractors, expected evidence, safe remediation, and verification predicate.

Core agent metrics are Tool Selection Accuracy, Tool Argument Validity, Diagnosis Accuracy, Evidence Attribution Precision, False Positive Rate, Mean Time to Detect, Mean Time to Diagnose, Approval Accuracy, Remediation Success Rate, Verification Success Rate, Recovery Success Rate, and Average Agent Cost in latency and local resource consumption.

The benchmark should include benign high-utilization workloads, memory leaks, thermal throttling, disk exhaustion, runaway processes, service failure, abnormal startup item, network anomaly, repeated application crash, tool failure, conflicting evidence, and prompt-injection-like malicious telemetry strings.

## 18. Definition of Done

A feature is complete only when its domain model, implementation, tests, observability, security controls, user-facing behavior, failure modes, and documentation are implemented together. A tool is not complete until it has a schema, typed input validation, platform capability declaration, policy classification, timeout, structured result, audit record, and verification strategy when it changes system state.

## 19. Research Basis

OpenAI Agents SDK documentation currently describes Agents, function tools, handoffs, guardrails, sessions, tracing, and human-in-the-loop interruptions, including pause/resume via RunState. The SDK also supports non-OpenAI providers through OpenAI-compatible endpoints and model adapters.

OpenAI Codex documentation and repository guidance describe AGENTS.md as the repository-scoped mechanism for instructions and document sandbox and approval controls for agent execution. Codex's app-server protocol supports explicit approval and sandbox configuration and event streaming, but those capabilities are treated here as development-harness primitives rather than as the production laptop authority.

Google's current Gemma documentation states that Gemma 4 supports native function calling and is designed for agentic workflows. Google's Gemma documentation lists Gemma 4 E2B and E4B as laptop-oriented sizes and documents local inference with Ollama and llama.cpp. Google's FunctionGemma documentation describes FunctionGemma as a specialized function-calling model based on Gemma 3 270M and notes that it is intended for task-specific fine-tuning rather than direct dialogue.

OpenTelemetry's current system and process semantic conventions provide a useful reference vocabulary for CPU, memory, disk, network, process, and uptime metrics. The project should map to that vocabulary where useful while retaining a versioned internal schema because the conventions are not all stable.

## 20. Source References

OpenAI Agents SDK: https://openai.github.io/openai-agents-python/

OpenAI Agents SDK tools: https://openai.github.io/openai-agents-python/tools/

OpenAI Agents SDK human-in-the-loop: https://openai.github.io/openai-agents-python/human_in_the_loop/

OpenAI Agents SDK models and non-OpenAI providers: https://openai.github.io/openai-agents-python/models/

OpenAI Codex repository and AGENTS.md guidance: https://github.com/openai/codex

OpenAI Codex app-server protocol: https://github.com/openai/codex/tree/main/codex-rs/app-server

Google Gemma 4 overview: https://ai.google.dev/gemma/docs/core

Google Gemma 4 function calling: https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4

Google Gemma local inference: https://ai.google.dev/gemma/docs/run

Google Gemma llama.cpp integration: https://ai.google.dev/gemma/docs/integrations/llamacpp

Google FunctionGemma: https://ai.google.dev/gemma/docs/functiongemma/formatting-and-best-practices

OpenTelemetry system metrics: https://opentelemetry.io/docs/specs/semconv/system/system-metrics/

OpenTelemetry process metrics: https://opentelemetry.io/docs/specs/semconv/system/process-metrics/
