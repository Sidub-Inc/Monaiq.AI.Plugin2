# Startup Workflow

Use this workflow before catalog mutation, SDK/config edits, feature implementation, purchase-flow work, troubleshooting fixes, or validation remediation.

1. Resolve readiness for `fetch_step_resources`, `monaiq_journal`, and any required MCP implementation tool for the selected skill.
2. Fetch `monaiq://protocols/implementation-journal` and read `ROUTING-MAP.md`, `_shared/handoff-schemas.md`, and `_shared/response-patterns.md` when available.
3. Call `monaiq_journal get_state` for the consumer project root.
4. If state is missing or stale, call `monaiq_journal init` and apply only returned `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, and `.monaiq/CHECKPOINTS/*` file operations after path validation.
5. Verify `.monaiq/STATE.md` contains the master journey checklist. Restore it through the journal protocol before substantive work if absent.
6. Call `monaiq_journal skill_started` with the selected skill and safe route context.
7. For new or materially changed journeys, save and present `CHECKPOINT-WORKFLOW-START` before substantive work.
8. Route to the narrowest missing prerequisite when profile/session, catalog/offering, SDK, code evidence, journal state, or MCP readiness is missing.

Direct skill invocation is first-class. The custom `monaiq` agent is an optional orchestrator, not a correctness dependency.

## Readiness Blocker Report

When required MCP capabilities are unavailable, render a user-visible blocker report instead of continuing into half-execution:

- **Missing:** unavailable tool, resource, or runtime capability.
- **Affected gate:** checklist gate or skill blocked.
- **What can continue:** read-only analysis, domain explanation, or recommendation-only work.
- **What is blocked:** catalog mutation, credential/config writes, application behavior edits, checkpoint recording, or validation remediation.
- **Recovery:** refresh/reload the Monaiq plugin or rerun with a working MCP server.
- **Journal:** record a no-secret diagnostic summary when `monaiq_journal` is available.