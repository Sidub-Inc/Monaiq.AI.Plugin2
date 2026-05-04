# Shared Protocols

## Skill Layout Contract

Monaiq skills should follow the same readable command shape as the GSD skills while keeping Monaiq's route, journal, checkpoint, and evidence requirements explicit.

Use this section order for user-facing skills unless a read-only reference skill has a good reason to be shorter:

1. Frontmatter with `name`, `description`, `agent: monaiq`, `auto-invoke` when useful, `tags`, `category`, `allowed-tools`, `tier`, and `invoked-by`.
2. `<objective>` with the user outcome and mutation boundary.
3. `<input-context>` and `<output-context>` for route packets, evidence packets, and downstream handoffs.
4. `<execution_context>` naming the shared workflow files and the skill's narrow responsibility.
5. `<monaiq-agent-handoff>` using the standard degraded-orchestration warning when direct invocation cannot activate the custom agent.
6. Optional authority blocks, such as `<tool-first-authority>`, only when a hosted MCP tool or canonical platform resource is authoritative.
7. `<workflow>` as numbered executable steps that start with startup/journal readiness and end with completion/handoff.
8. `<reference>` for domain facts, decision tables, examples, and platform resource URIs.
9. `<success_criteria>` for observable completion evidence.

The required shared route packet fields are exactly `scenario`, `targetApp`, `platform`, `profileState`, `catalogState`, `codeEvidenceSummary`, `journalEvidenceSummary`, `assumptions`, `recommendedSkill`, `recommendationRationale`, `checkpointName`, and `authorizedBy`. Do not rename them in handoff text.

Standard direct-invocation warning: "Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."

Business-readable output comes first. Compact technical backing follows with route packet fields, evidence summaries, checkpoint names, and no-secret implementation details.

## Tool Operation Rules

- `design-monetization`: read/list catalog facts only. Do not create, update, publish, or delete products, features, offerings, or feature assignments.
- `scenario-advisor`: read/list only plus resource reads.
- `analyze-codebase`: local read/search plus resource reads; no catalog mutation.
- `domain-reference`: resource reads only.
- `profile-onboarding`: profile/session reads and profile-directed onboarding only; no catalog or code mutation.
- `manage-catalog`: product, product_feature, offering, and feature_offering create/update/delete are allowed only after `CHECKPOINT-PRE-CATALOG-MUTATION` or publication-specific checkpoint approval.
- `implement-licensing`: code/config edits are allowed only after required SDK resources, checkpoint approval, and secret-safety checks.
- `implement-feature`: business-logic edits are allowed only after `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` and tool/resource evidence.
- `implement-purchase-flow`: checkout edits and credential persistence are allowed only after SDK/tool/resource evidence and purchase-specific checkpoints.
- `troubleshoot-integration`: read/diagnose first; fixes that mutate code/config/business logic require the validation and checkpoint workflows.

When a skill has broad tool exposure for platform compatibility, these operation rules are the binding behavioral constraint.