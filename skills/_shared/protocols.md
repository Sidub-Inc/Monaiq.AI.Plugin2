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

## Evidence-First Recommendation Principle

Monaiq is positioned as a guided, evidence-based licensing assistant. Skills must research and recommend before they ask. This principle is binding for every strategic decision (which features to monetize, which licensing model fits the app, which pricing tiers to propose, which application type applies) and is referenced by name from individual skills.

**Rule.** Before asking the user any open-ended strategic question, a skill MUST first attempt to derive a concrete recommendation from available evidence. The skill then presents the user with a short, business-readable recommendation, 2–3 ranked options with rationale, a clearly marked recommended choice, and confirms via the existing checkpoint that owns the decision. Open-ended discovery questions are reserved for ambiguity that evidence cannot resolve (for example, the user's pricing intent, their commercial strategy, or constraints the agent cannot observe).

**Evidence order (consult in this order before any open question).**

1. Upstream route-packet inputs handed off by the previous skill (`selectedScenario`, `capabilities`, `appType`, `quickStartSpec`, `catalogSpec`, etc.).
2. Codebase evidence — invoke `analyze-codebase` (or reuse a prior analysis from the journal/route packet) when the user has a reachable codebase. Use detected tech stack, capabilities, integrations, and configuration as primary signal.
3. Journal evidence — `monaiq_journal` decisions, prior recommendations, prior checkpoints, prior validation outcomes.
4. Catalog/profile state — `product`, `product_feature`, `offering`, `feature_offering`, `profile` reads.
5. Conversation evidence captured in the route packet's `codeEvidenceSummary`, `journalEvidenceSummary`, `assumptions`, and `recommendationRationale`.
6. Resource patterns — `monaiq://patterns/*`, `monaiq://domain/*`, `monaiq://platforms/*`.

Only when all relevant sources are exhausted, ask the **narrowest** possible question, framed explicitly as an evidence gap rather than as the default discovery path.

**Required response shape for any strategic recommendation.**

- Evidence summary (one or two lines: what was inferred and from where).
- 2–3 options as a short table or bullets, each with trade-offs.
- One option marked as the recommendation, with the reasoning tied to the evidence summary.
- A confirmation prompt that maps to the owning checkpoint (e.g., `CHECKPOINT-SCENARIO-CHOICE`, `CHECKPOINT-PRICING-APPROVAL`, `CHECKPOINT-PRE-CATALOG-MUTATION`) — not an open-ended question.
- Anti-pattern phrases to avoid as defaults: *"What features does your app/product have?"*, *"What licensing model do you want?"*, *"Which monetization approach should we use?"*, *"You decide which features to monetize."* These are permitted only after the evidence path has been attempted and explicitly judged insufficient, and even then must be the narrowest possible question.

**Anti-patterns.**

- Asking the user to choose what to monetize, which model to use, or which tiers to define before running `analyze-codebase` (when a codebase exists), reading the journal, or pulling existing catalog state.
- Presenting a flat menu of options with no recommendation and no rationale.
- Treating an unanswered upstream input as a reason to ask the user instead of invoking the upstream skill that produces it.
- Re-asking decisions already captured in the journal or route packet.

**When the user explicitly defers ("you decide", "figure it out").** Treat this as authorization to apply the recommended option. Continue under the recommendation, record the deferral and rationale in the journal, and surface the final choice at the owning checkpoint for confirmation. Do not bounce the question back as a follow-up — that is the failure mode this principle exists to prevent.

See `_shared/response-patterns.md` "Evidence Backing" for the canonical compact technical-backing shape, and the `Tool Operation Rules` below for the read-first tool ordering each skill must respect.

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