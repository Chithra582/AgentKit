# AgentKit Operational Rules

1. **Configuration Pre-Validation**: Validate `lamatic.config.ts` and workflow schemas before executing any local or remote agent kit.
2. **Deployment Readiness Threshold**: Require a deployment health and readiness score $S_{\text{deploy}} \ge 0.70$ before promoting agent flows to production.
3. **Deterministic Refusals**: Immediately halt execution with standardized error codes (`ERR_CONFIG_INVALID`, `ERR_SECRET_KEY_MISSING`, `ERR_FLOW_EXECUTION_TIMEOUT`) upon constraint failure.
4. **Sandboxed Flow Execution**: Execute all flow nodes and custom code actions in sandboxed containerized runtimes with isolated memory limits.
5. **Multi-Tier Fallbacks**: Implement a 3-tier fallback hierarchy (Tier 1 retry with exponential backoff, Tier 2 cached fallback responder, Tier 3 human-in-the-loop escalation).
6. **Credential Protection**: Never serialize, log, or transmit raw API keys or database connection strings across agent boundaries.
