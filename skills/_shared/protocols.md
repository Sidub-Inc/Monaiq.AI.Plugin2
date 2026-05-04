# Shared Protocols

## Skill Layout Contract

Monaiq skills should follow the same readable command shape as the GSD skills while keeping Monaiq's route, journal, checkpoint, and evidence requirements explicit.

Monaiq guidance is distributed to Claude, GitHub Copilot / VS Code, and Codex runtimes. Shared guidance must therefore describe behavior in target-host terms instead of assuming one runtime's tool names apply everywhere.

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

The required shared route packet fields are exactly `scenario`, `targetApp`, `platform`, `activePlatform`, `targetProject`, `outOfScopePlatforms`, `profileState`, `catalogState`, `codeEvidenceSummary`, `journalEvidenceSummary`, `assumptions`, `recommendedSkill`, `recommendationRationale`, `checkpointName`, and `authorizedBy`. Do not rename them in handoff text.

`platform` may describe the broad stack family (`dotnet`, `react`, `node`), while `activePlatform` is the concrete implementation target currently approved for changes (`dotnet/blazor-server`, `react/vite`, etc.). `targetProject` names the project or package that may be changed. `outOfScopePlatforms` records stacks that must not be installed or modified without a new checkpoint.

## Direct Invocation Contract

This is the canonical fallback every skill applies when invoked directly without the `monaiq` custom agent. Skills should reference this section by name rather than restating it.

**Mutation-capable skills** (catalog, SDK, feature, purchase, journal, troubleshoot fixes):
1. If the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.
2. If host activation is unavailable, warn exactly once: *"Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."*
3. The compatibility fallback still runs `_shared/workflows/startup.md`, uses `monaiq_journal` for `skill_started`, hard checkpoints, milestone progress, and `skill_completed`, and batches routine updates through `_shared/workflows/completion.md`.

**Read-only reference skills** (`domain-reference`, and read-only paths in `scenario-advisor` / `profile-onboarding` view modes) may answer without journal startup. They must enter the full mutation contract above the moment the user pivots to catalog, credential/config, code, or purchase work.

Business-readable output comes first. Compact technical backing follows with route packet fields, evidence summaries, checkpoint names, and no-secret implementation details. See `_shared/response-patterns.md` "Evidence Backing" for the canonical evidence shape used by specialist workflow steps.

Route, handoff, journal, and checkpoint payloads must stay secret-free. Record `session established`, credential source status, redacted placeholders, and package/version facts; never record raw ApiKeys, EncodedCredential values, JWTs, Stripe keys, `.env` contents, user-secrets contents, or encoded credential fragments.

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