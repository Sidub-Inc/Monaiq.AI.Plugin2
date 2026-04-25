> [!WARNING]
> **Requires monaiq MCP server attached.** This skill references MCP resources
> via <resource-ref> blocks. Without the monaiq MCP server wired to your
> agent, these references cannot be resolved. Connect the MCP server at
> https://api.monaiq.com/runtime/webhooks/mcp before invoking this skill.
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
allowed-tools: [Read, Grep, Glob]
argument-hint: "appType (saas|desktop|plugin|package|cli|api)"
tier: 3
invoked-by: [analyze-codebase, getting-started]
---

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

<state-detection>
Before presenting scenarios:
1. Check if capabilities were provided from analyze-codebase upstream
2. If so, infer app type from techStack and skip Step 1

Based on state:
- Capabilities provided → Auto-infer app type, jump to scenario presentation
- No upstream context → Ask user about their app type (Step 1)
</state-detection>

<objective>
Map a user's application type to recommended licensing models from `monaiq://patterns/scenarios`. Infer the application type from codebase context (if `analyze-codebase` was run) or from conversation context, present 2–3 licensing model options with trade-offs, and route to `design-monetization` after the user selects.
</objective>

<process>

## Prerequisites

- Resolve the scenario building blocks and domain model before presenting options:

  <resource-ref uri="monaiq://patterns/scenarios"/>

  <resource-ref uri="monaiq://domain/model"/>

## Step 1: Determine Application Type

Infer the application type from available context:

- **SaaS web application** — React/Angular/Next.js frontend, API controllers, multi-tenant patterns
- **Desktop application** — WPF/WinForms/Electron, local file I/O, installer references
- **Plugin/extension** — Plugin manifest, extension API, host app integration
- **Package/library** — NuGet/npm publish config, public API surface, no entry point
- **CLI tool** — Console app, command parsers, argument handling
- **API service** — REST/GraphQL endpoints, auth middleware, rate limiting

If `analyze-codebase` was run previously, use its output (tech stack, identified capabilities) to infer the type. If insufficient context to infer, present the application type list and ask the user to select.

## Step 2: Present Licensing Model Options

Using the composition examples from `monaiq://patterns/scenarios`, present 2–3 licensing model options tailored to the inferred application type:

| Aspect | Option A: [Name] | Option B: [Name] | Option C: [Name] |
|--------|------------------|------------------|------------------|
| **Model** | [e.g., Freemium + Rate Limits] | [e.g., Tiered Subscriptions] | [e.g., Usage-Based] |
| **Feature Types** | [Feature flags + Usage limits] | [Feature flags only] | [Usage limits only] |
| **Pricing Tiers** | [Tier] | [Tier + Add-on] | [Metered] |
| **How Licenses Are Checked** | [Access Gate + Rate Limit] | [Feature Flag] | [Rate Limit] |
| **Best For** | [trade-off summary] | [trade-off summary] | [trade-off summary] |

Reference specific building blocks from `monaiq://patterns/scenarios` by name — do not duplicate resource content.

## Step 3: Capture User Selection

Let the user review the options, ask questions, and select a licensing model. Record the chosen scenario.

## Step 4: Route to Design

After the user selects a licensing model, suggest: "To design your pricing tiers and feature assignments based on this scenario, run the `design-monetization` skill." Include the selected scenario context (application type, chosen model, feature types) for continuity.

</process>

<success_criteria>
- Application type was inferred from context (not immediately asked)
- 2–3 scenario options presented with trade-offs
- Options reference building blocks from `monaiq://patterns/scenarios`
- User selected a scenario
- `design-monetization` suggested as next step with scenario context
</success_criteria>
