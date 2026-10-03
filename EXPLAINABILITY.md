# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **AgentKit** (`agentkit`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** AgentKit (`agentkit`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Agent Developer Stack & Serverless Agent Deployment  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

AgentKit executes agent workflows, verifies configuration schemas, and provisions serverless deployments through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Manifest Parsing & Configuration Validation]                           |
|  - Ingest lamatic.config.ts, parse flow schemas, verify environment secrets       |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Flow DAG Compilation & Dependency Resolution]                          |
|  - Build execution graph, resolve node dependencies, validate data payloads       |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Readiness Scoring & Constraint Evaluation]                             |
|  - Compute deployment readiness metric S_deploy against SLA and latency bounds    |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Verification & Guardrail Check]                              |
|  - Verify tau >= 0.70; inspect schema adherence, sandbox quotas, and auth scopes  |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Serverless Packaging, Edge Dispatch & Telemetry Stream]                |
|  - Deploy edge container, trigger flow execution, emit structured telemetry logs  |
+-----------------------------------------------------------------------------------+
```

### 2. Decision Logic & Routing Formulations

For a candidate agent kit $K_i$ evaluated against target deployment environment $E$, the deployment readiness score $S_{\text{deploy}}(K_i, E)$ is formulated as:

$$S_{\text{deploy}}(K_i, E) = w_{\text{conf}} C(K_i) + w_{\text{test}} T(K_i) + w_{\text{lat}} L(K_i, E) + w_{\text{sec}} S(K_i)$$

Where:
- $C(K_i) \in \{0, 1\}$ is a binary indicator of configuration schema correctness (`lamatic.config.ts` passes structural linting).
- $T(K_i) \in [0, 1]$ represents the ratio of passed unit tests and mock integration evaluations.
- $L(K_i, E) = \max\left(0, 1 - \frac{t_{\text{coldstart}}(K_i, E)}{t_{\max}}\right)$ measures cold-start and execution latency compliance relative to budget $t_{\max} = 2000\,\text{ms}$.
- $S(K_i) \in [0, 1]$ measures security compliance, including secret encryption, minimum permission scoping, and network egress restrictions.
- Weights: $w_{\text{conf}} = 0.30$, $w_{\text{test}} = 0.30$, $w_{\text{lat}} = 0.20$, $w_{\text{sec}} = 0.20$ with $\sum w = 1.0$.

Deployment and live dispatch require:

$$S_{\text{deploy}}(K_i, E) \ge \tau \quad (\tau = 0.70) \quad \land \quad C(K_i) = 1$$

### 3. Thresholding & Refusal Decision Criteria

AgentKit enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_CONFIG_INVALID**: $C(K_i) = 0$ (missing or invalid `lamatic.config.ts`) halts execution with code `ERR_CONFIG_INVALID`.
- **Refusal on ERR_SECRET_KEY_MISSING**: Required integration API key or token absent halts execution with code `ERR_SECRET_KEY_MISSING`.
- **Refusal on ERR_FLOW_EXECUTION_TIMEOUT**: Node execution duration $> 30,000\,\text{ms}$ halts execution with code `ERR_FLOW_EXECUTION_TIMEOUT`.
- **Refusal on ERR_SCHEMA_MISMATCH**: Flow step output violates downstream input contract halts execution with code `ERR_SCHEMA_MISMATCH`.
- **Refusal on ERR_DEPLOY_TARGET_UNREACHABLE**: Edge deployment endpoint health check fails halts execution with code `ERR_DEPLOY_TARGET_UNREACHABLE`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Automated Node Retry & Backoff):** On transient network or upstream provider timeout, retry the failed node up to 3 times with exponential backoff and jitter.
- **Tier 2 (Cached Response & Graceful Degradation):** If upstream inference or data providers remain unavailable, serve cached semantic responses or execute degraded rulebased handlers.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Tier 3 (HumanintheLoop Triage):** When unresolvable schema deviations or billing quota limits occur, pause the active workflow and alert the administrator via webhook with full execution trace.
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

AgentKit operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Configuration Files**: `lamatic.config.ts`, TypeScript typings, and workflow node specifications.
- **Workflow Payloads**: JSON input payloads, webhook triggers, and event bus messages.
- **Client Metadata**: Request headers, tenant identifiers, and trace IDs.

### 2. Configuration & Reference Data

- **Template Registry**: `registry.json` and `highlights.json` catalogs indexing certified kits, bundles, and flow templates.
- **Integration Schemas**: OpenAPI manifests for supported vector stores (Pinecone, Qdrant), LLMs (OpenAI, Anthropic), and SaaS tools.
- **Execution History**: Persisted step-level execution records and latency benchmarks.

### 3. Base Model & Inference Lineage

- **Model Agnostic**: Interfaces with upstream LLMs via standardized API adapters (OpenAI, Anthropic Claude, Google Gemini, Mistral).
- **Weight Integrity**: Operates strictly on API-based provider endpoints or verified containerized open-weights deployments with pinned versions.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of AgentKit is essential for effective deployment.

### 1. Complex multi-flow kits that execute numerous
- **Limitation**: Complex multi-flow kits that execute numerous sequential LLM calls can incur latency buildup.
- **Mitigation**: The compiler identifies independent flow branches and automatically executes them concurrently using Promise.all DAG scheduling.

### 2. Serverless edge runtimes impose maximum execution
- **Limitation**: Serverless edge runtimes impose maximum execution memory caps (typically 512 MB to 2 GB).
- **Mitigation**: Heavy compute tasks (e.g., local embedding generation or PDF parsing) are offloaded to dedicated asynchronous worker queues.

### 3. Third-party API rate limits can throttle
- **Limitation**: Third-party API rate limits can throttle agent workflows during burst traffic events.
- **Mitigation**: AgentKit implements client-side token bucket rate limiting and distributed caching to minimize redundant API calls.

### 4. Asynchronous webhook callbacks may arrive out-of-order
- **Limitation**: Asynchronous webhook callbacks may arrive out-of-order in distributed serverless environments.
- **Mitigation**: State transitions are guarded by monotonic timestamp sequence counters and distributed lock primitives.

### 5. Schema drifts in external third-party APIs
- **Limitation**: Schema drifts in external third-party APIs can cause unexpected runtime deserialization errors.
- **Mitigation**: AgentKit validates all node ingress and egress against strict Zod/JSON schemas before passing payloads downstream.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Complex multi-flow kits that execute numerous | Section 1 | Verified |
| - Serverless edge runtimes impose maximum execution | Section 2 | Verified |
| - Third-party API rate limits can throttle | Section 3 | Verified |
| - Asynchronous webhook callbacks may arrive out-of-order | Section 4 | Verified |
| - Schema drifts in external third-party APIs | Section 5 | Verified |
