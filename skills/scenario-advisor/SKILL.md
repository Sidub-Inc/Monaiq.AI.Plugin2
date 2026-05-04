---
name: scenario-advisor
description: "Use when: recommending a licensing model for an app type, comparing subscription, trial, perpetual, freemium, tiered, or usage-based scenarios before pricing design."
agent: monaiq
auto-invoke:
  - "User wants to know which licensing scenario fits their application"
  - "User asks what licensing model to use for their product"
  - "User wants licensing recommendations for their app type"
tags: [discovery, scenarios, recommendations, strategy]
category: discovery
allowed-tools: [Read, Grep, Glob, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "appType (saas|desktop|plugin|package|cli|api)"
tier: 3
invoked-by: [analyze-codebase, getting-started]
---

<objective>
Recommend licensing scenarios for the user's app type using capability evidence, route context, and Monaiq scenario patterns. The skill narrows strategic direction before pricing design; it does not design exact tier prices or mutate catalog data.
</objective>

<input-context>
Receives from analyze-codebase:
- capabilities (optional): [{ name, type, suggestedFeatureKey }] — identified licensable features
- techStack (optional): detected technology stack — used to infer app type

If invoked without upstream context, asks the user about their application type directly.
</input-context>

<output-context>
Provides to design-monetization:
- selectedScenario: { name, model, featureTypes: ["access" | "ratelimit"], offeringModel, enforcement }
- appType: "saas" | "desktop" | "plugin" | "package" | "cli" | "api"

Discovery chain: analyze-codebase [capabilities] → scenario-advisor [selectedScenario] → design-monetization [pricingPlan]
</output-context>

<execution_context>
Before scenario recommendation steps, follow `_shared/workflows/startup.md`, `_shared/workflows/checkpoint.md`, `_shared/workflows/completion.md`, `_shared/response-patterns.md`, `_shared/handoff-schemas.md`, and `_shared/protocols.md`. This skill contributes only licensing-scenario recommendation and handoff context; exact prices and catalog mutation belong to `design-monetization` and `manage-catalog`.
</execution_context>

<monaiq-agent-handoff>
This skill is intended to run under the `monaiq` custom agent. If invoked directly and the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.

If host activation is unavailable, warn exactly: "Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."

The compatibility fallback still fetches `monaiq://protocols/implementation-journal`, calls `monaiq_journal get_state` or `monaiq_journal init`, calls `skill_started`, and enforces relevant checkpoints before consequential follow-up actions.
</monaiq-agent-handoff>

<workflow>
1. Fetch `monaiq://protocols/implementation-journal`, call `monaiq_journal get_state`, initialize/apply returned `.monaiq/*` operations if needed, then call `skill_started` for `scenario-advisor`.
2. Resolve `monaiq://patterns/scenarios`, `monaiq://domain/model`, and platform resources such as `monaiq://platforms/api-surface/{platform}` or `monaiq://platforms/pitfalls/{platform}` when the stack is known before presenting options. Stop before recommendation if scenario building blocks or domain context are unavailable.
3. Infer application type from `analyze-codebase` capabilities, tech stack, route packet, and conversation evidence before asking the user to choose an app type.
4. Present a business-readable evidence summary first, then compact technical backing. Technical backing includes codebase evidence, journal decisions, backend/profile/catalog facts, route-packet evidence, labeled assumptions, confidence, missing evidence, and app-type inference rationale.
5. Present 2-3 scenario options with trade-offs. Keep the recommendation narrow: one preferred option plus alternatives only when they resolve real ambiguity.
6. Use `CHECKPOINT-SCENARIO-CHOICE` before locking the recommended scenario for pricing design.
7. Output `selectedScenario` and `appType` for `design-monetization`, save `CHECKPOINT-SKILL-COMPLETE` when useful, apply returned operations, then call `skill_completed`.
</workflow>

<reference>
## State Routing Reference

- Capabilities provided -> infer app type and present scenario options.
- No upstream context -> ask the narrowest app-type question needed.

## Determine Application Type

Infer the application type from available context:

- **SaaS web application** — React/Angular/Next.js frontend, API controllers, multi-tenant patterns
- **Desktop application** — WPF/WinForms/Electron, local file I/O, installer references
- **Plugin/extension** — Plugin manifest, extension API, host app integration
- **Package/library** — NuGet/npm publish config, public API surface, no entry point
- **CLI tool** — Console app, command parsers, argument handling
- **API service** — REST/GraphQL endpoints, auth middleware, rate limiting

If `analyze-codebase` was run previously, use its output (tech stack, identified capabilities) to infer the type. If insufficient context to infer, present the application type list and ask the user to select.

## Present Licensing Model Options

Using the composition examples from `monaiq://patterns/scenarios`, present 2–3 licensing model options tailored to the inferred application type:

| Aspect | Option A: [Name] | Option B: [Name] | Option C: [Name] |
|--------|------------------|------------------|------------------|
| **Model** | [e.g., Freemium + Rate Limits] | [e.g., Tiered Subscriptions] | [e.g., Usage-Based] |
| **Feature Types** | [Feature flags + Usage limits] | [Feature flags only] | [Usage limits only] |
| **Pricing Tiers** | [Tier] | [Tier + Add-on] | [Metered] |
| **How Licenses Are Checked** | [Access Gate + Rate Limit] | [Feature Flag] | [Rate Limit] |
| **Best For** | [trade-off summary] | [trade-off summary] | [trade-off summary] |

Reference specific building blocks from `monaiq://patterns/scenarios` by name — do not duplicate resource content.

## Capture User Selection

Let the user review the options, ask questions, and select a licensing model. Record the chosen scenario.

## Route to Design

After the user selects a licensing model, suggest: "To design your pricing tiers and feature assignments based on this scenario, run the `design-monetization` skill." Include the selected scenario context (application type, chosen model, feature types) for continuity.

</reference>

<success_criteria>
- Application type was inferred from context (not immediately asked)
- 2–3 scenario options presented with trade-offs
- Options reference building blocks from `monaiq://patterns/scenarios`
- User selected a scenario
- `design-monetization` suggested as next step with scenario context
</success_criteria>
