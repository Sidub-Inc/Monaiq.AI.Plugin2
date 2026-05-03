---
name: implement-licensing
description: "Use when: integrating the Monaiq licensing SDK into a .NET or React app, installing packages, configuring credentials/endpoints, registering services/providers, or verifying runtime license validation."
agent: monaiq
auto-invoke:
  - "User wants to integrate licensing SDK into their application — has a product catalog already set up"
  - "User wants to add license validation to their project and needs package installation and configuration"
  - "User asks how to set up Monaiq licensing in code — not asking about pricing or catalog design"
tags: [sdk, integration, licensing, setup, dotnet, react]
category: integration
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, register_or_login, profile, product, product_feature, implement_base, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__implement_base, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "platform (dotnet|dotnet/blazor-server|react|react/vite|react/nextjs)"
tier: 2
invoked-by: [getting-started]
---

<input-context>
Receives from manage-catalog (or direct invocation):
- catalogSpec (optional): { productCode, features: [{ key, type }], offerings: [{ code, classification }] } — if provided, skip catalog verification
- platform (optional): "dotnet" | "dotnet/blazor-server" | "react" | "react/vite" | "react/nextjs" — if provided from user intent, skip platform detection

If invoked without upstream context, the skill detects current state and asks the user for missing information.
</input-context>

<output-context>
Provides to downstream skills (implement-feature, implement-purchase-flow):
- sdkConfig: { platform: "dotnet" | "dotnet/blazor-server" | "react" | "react/vite" | "react/nextjs", credentialSource: "configuration" | "user-managed", serviceOptionsConfigured: boolean, diRegistered: boolean }
- targetFeatures: [{ featureKey, featureType }] — features available for gating (from catalog or user input)

Chain: manage-catalog [catalogSpec] → implement-licensing [sdkConfig] → implement-feature
</output-context>

<monaiq-agent-handoff>
This skill is intended to run under the `monaiq` custom agent. If invoked directly and the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.

If host activation is unavailable, warn exactly: "Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."

The compatibility fallback still uses `monaiq_journal` for startup, checkpoints, `record_file_changes`, and `skill_completed`.
</monaiq-agent-handoff>

<state-detection>
Before executing, check what's already set up. For the exact package names, configuration keys, DI extensions, and provider components to grep for, resolve:

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Probe order:
1. Are SDK packages installed? Look for the platform's SDK package name in the project manifest (`*.csproj` for .NET, `package.json` for React).
2. Is the licensing configuration present? Look for the platform's licensing configuration section (settings file for .NET, provider component props for React) in source.
3. Is DI registration wired? Look for the platform's registration extension / provider component in the application composition root.

Based on detected state:
- Nothing found → Full flow from Step 1
- Packages installed → Skip Step 2, start at credential configuration
- Packages + config found → Skip to DI registration
- Everything configured → Show summary, offer credential recovery utility
</state-detection>

<objective>
Guide an agent through integrating the Monaiq licensing SDK into a .NET or React application — from credential discovery through runtime license validation. Covers six areas: credential source determination, package installation, namespace reference, service options configuration, dependency injection registration, and runtime usage.
</objective>

<process>

## Journal Hook

Fetch `monaiq://protocols/implementation-journal`, call `monaiq_journal get_state`, handle resume through `CHECKPOINT-RESUME`, then call `skill_started` for `implement-licensing`. Honor `CHECKPOINT-FRAMEWORK-CHOICE`, `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, and `CHECKPOINT-PRE-CREDENTIAL-WRITE`; present each through host-native ask and call `save_checkpoint` again with the result. At completion, record changed paths only with `record_file_changes`, save `CHECKPOINT-SKILL-COMPLETE` when useful, then call `skill_completed`.

`CHECKPOINT-PRE-CREDENTIAL-WRITE` is mandatory before writing ApiKeys, IssuerClientId, endpoint URLs, SDK provider configuration, appsettings, user-secrets, `.env`, or persisted Monaiq configuration. Record the user's result before proceeding. Do not store raw ApiKeys, EncodedCredential values, `.env` contents, or user-secrets content in `.monaiq`.

## Evidence-Backed Implementation Pattern

Before code/config/business-logic changes, present a business-readable evidence summary and business-readable impact before compact technical backing. Technical backing includes codebase evidence, journal decisions, backend/profile/catalog/offering facts, route-packet evidence, labeled assumptions, confidence, and missing evidence.

SDK setup recommendations cite detected platform/app structure, profile/session state, catalog or target feature context, journal decisions, and route-packet evidence. If missing SDK/catalog/offering/profile/code evidence prevents a confident implementation recommendation, route to the appropriate prerequisite step before code/config/business-logic changes.

## Prerequisites

- Establish a session using the `register_or_login` tool.
- Determine the credential source. Reseller profile credentials (`ApiKey`, `IssuerClientId`) are used for checkout setup; application runtime `EncodedCredential` values come from completed purchases or an existing application credential store.
- If any required canonical resource is unavailable, stop before catalog mutations, code edits, credential/config writes, or validation remediation. Do not guess endpoint URLs, SDK signatures, setup snippets, pitfalls, or post-purchase behavior.
- Resolve the platform-specific SDK setup narrative, API surface, and endpoint config:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://config/endpoints` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://docs/anti-patterns/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Interactive Step-by-Step Flow

Use `implement_base` with `startStep=all` when platform, app context, required resources, and credential/config safety posture are already known and no unresolved checkpoint/resource/validation/config blocker exists. Use numeric `startStep` mode (`startStep=1` through `startStep=6`) when step-by-step review improves safety or clarity. Read `journalReadyUpdates` from tool envelopes and apply them by calling `monaiq_journal`; never treat intents as already-applied state.

If validation fails, call `monaiq_journal record_validation_failure` with the failed command, observed result, likely cause, blocked next action, retry choices, and checkpoint requirement before continuing. `provision_api_key_config` returns a local plan with non-secret token markers and `CHECKPOINT-PRE-CREDENTIAL-WRITE`; require approval before local writes.

## Step 1: Determine Credential Source

Before writing any code, determine how end-users will provide their license credentials.

**Core question:** Do individual users or tenants each provide their own separate license credentials?

| Answer | Credential Source | Typical Use Case |
|--------|------------------|------------------|
| No | Configuration-Based | Single organization license, one credential in config |
| Yes | User-Managed | Multi-tenant SaaS, marketplace apps, per-user subscriptions |

**Configuration-Based** — one license credential stored in application settings covers the entire application. The SDK's built-in configuration-based credential resolver handles this automatically. For the exact configuration shape and resolver type names, resolve `monaiq://platforms/api-surface/{platform}`.

**User-Managed** — each user/tenant provides their own credential at runtime. You must implement a custom credential-provider type that resolves credentials from your storage (user profile, tenant settings, etc.). For the authoritative provider interface / hook signature, resolve `monaiq://platforms/api-surface/{platform}`. Consider the `implement_purchase_flow` tool for embedded in-app purchases that provision credentials automatically.

## Step 2: Install Packages

Install the SDK package for the target stack. For the installation command, package name, and transitive-dependency notes, resolve:

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

> **Note (.NET):** Requires Sidub.Platform 1.10.38+. Consumers using this version or later do not need any `<ExcludeAssets>` Metalama workaround — the Metalama build tool is no longer a transitive dependency (DEV-08 / RT-6 resolved in Sidub.Platform.Core).

## Step 3: Namespace Reference

For the authoritative type-to-namespace mappings (which imports expose credential types, service interfaces, feature types, and platform core), resolve:

Fetch `monaiq://domain/namespaces` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Use ONLY the namespaces from the reference. Do not guess or assume type locations.

## Step 4: Configure Service Options

Configure the licensing service with endpoint URIs and (for Configuration-Based credential source) the encoded credential.

- For the platform-specific configuration shape (settings file keys, provider component props, environment variable wiring), resolve:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the endpoint URL values to populate into that configuration, resolve:

  Fetch `monaiq://config/endpoints` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the authoritative property names and their types on the configuration surface, resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For **User-Managed** credential source, omit the encoded credential from configuration — it is resolved at runtime by your custom credential resolver (see Step 5).

## Step 5: Register Dependency Injection

Register the licensing services with your application's DI / provider graph.

- For the platform-specific registration pattern (service collection extensions in .NET, provider component in React), resolve:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the exact extension-method / component signatures (including the generic overload used for user-managed credential resolvers), resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For **User-Managed** credential source, register your custom credential-resolver type via the generic registration overload documented in the platform's api-surface resource.

## Step 6: Runtime Usage

Integrate the licensing service into your application's runtime.

- For the platform-specific type signatures — the service interface (.NET) or hook (React) used to request authorization — resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the runtime wiring narrative — how to inject the service, when to call it, error-state handling — resolve:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For known platform-specific pitfalls (null-semantics differences, provider remount behavior), resolve:

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 7: Displaying License State

Your application often needs to render the user's current license, entitlements, or consumption
— entitled features, rate-limit usage, expiry. The Monaiq SDK exposes this snapshot in-process; it
does **not** ship UI chrome. Render with your existing component library.

- For the platform-specific signature (`LicensingClient.getState()` for React,
  `ILicenseStateProvider.GetState()` for .NET) and the `LicenseStateView` shape, resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the canonical post-purchase refresh + snapshot pattern, resolve:

  Fetch `monaiq://platforms/manifest/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Typical render patterns (implementer's choice, no SDK opinion):
- **Badge** — show "Licensed · {planName}" or a compact status chip in the app header.
- **Settings card** — dedicated panel listing features + consumption bars on an account/settings page.
- **Dashboard widget** — surface near-limit features on the main landing view.

> The SDK ships no UI. Build the view with your existing component library.

## Verification

After completing the integration:

1. Build the project — no compilation errors.
2. Run the application and verify the SDK's authorization call returns a non-null result.
3. Check logs for any licensing configuration exception — this indicates incomplete credential fields.

## Related Tools

- `implement_base` — Interactive step-by-step SDK integration (follows the same 6-area structure; call `startStep=1` through `startStep=6` consecutively).
- `register_or_login` — Establish a session before integration.
- `profile` — Retrieve credentials and onboarding status.

## Related Resources

- `monaiq://sdk/dotnet/setup` — .NET SDK setup narrative.
- `monaiq://sdk/react/setup` — React SDK setup narrative.
- `monaiq://platforms/api-surface/dotnet` — .NET method signatures and null semantics.
- `monaiq://platforms/api-surface/react` — React method signatures and null semantics.
- `monaiq://platforms/manifest/dotnet` — Full machine-readable .NET platform manifest.
- `monaiq://platforms/manifest/react` — Full machine-readable React platform manifest.
- `monaiq://platforms/pitfalls/dotnet` — .NET known issues.
- `monaiq://platforms/pitfalls/react` — React known issues (including provider-remount behavior).
- `monaiq://docs/anti-patterns/dotnet` — .NET licensing anti-patterns to avoid.
- `monaiq://docs/anti-patterns/react` — React licensing anti-patterns to avoid.
- `monaiq://domain/namespaces` — Authoritative namespace-to-type mappings.
- `monaiq://domain/model` — Cross-platform entity relationships (offerings, licenses, features).
- `monaiq://config/endpoints` — Canonical endpoint URLs.

## Utility Workflows

When state detection shows SDK is already integrated, offer these options:

- **Credential recovery** — Re-read the completed checkout result or the application's credential store to recover the purchased EncodedCredential. The profile tool only returns reseller checkout credentials.
- **Update service URIs** — Modify licensing configuration to point to different environments (staging, production). Resolve `monaiq://config/endpoints` for the current authoritative URIs.
- **Switch credential source** — Migrate from configuration-based to user-managed credentials (or vice versa). Involves implementing or removing a custom credential resolver; resolve `monaiq://sdk/{stack}/setup` for the migration narrative.
- **Verify integration** — Run a quick health check: confirm packages are installed, configuration is present, DI is registered, and a test authorization call succeeds.

</process>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Package install fails | NuGet/npm error during SDK package installation | Check network connectivity and package source configuration. Retry with an explicit registry/source (`--source nuget.org` for .NET, clear npm cache for React). Resolve `monaiq://sdk/{stack}/setup` for the exact package name and commands. |
| Credential retrieval fails | `profile` tool returns an error | Verify the session is active via `register_or_login`. Re-authenticate if the session expired. |
| Config binding fails | Licensing configuration exception at startup | Verify the configuration section name and key casing match the platform-specific configuration shape. Check that the credential field is not empty or malformed. Resolve `monaiq://sdk/{stack}/setup` for the authoritative configuration keys. |
| DI registration fails | Build error on the SDK's registration extension or provider component | Verify the package is installed and the correct imports/usings are present. Resolve `monaiq://platforms/api-surface/{platform}` for the authoritative extension-method or component signatures. |
| Authorization call returns null | License validation fails at runtime | Verify the purchased EncodedCredential is present in configuration or the application's credential store. Resolve `monaiq://config/endpoints` to confirm service URIs are correct for the target environment. Resolve `monaiq://platforms/pitfalls/{platform}` for platform-specific null-semantics differences. |

SDK integration steps are non-destructive — each step modifies source files that can be edited again safely.
</error-recovery>

<brownfield>
## Brownfield Credential Contract

Persist the EncodedCredential (brownfield) only after an ownership scope decision. Before storage advice or schema changes, stop at `CHECKPOINT-PRE-BROWNFIELD-MIGRATION` and choose one scope: app-wide configuration credential, tenant-level credential, or user-level credential. The selected scope determines where the application reads the credential, who can update it, and which privacy boundary applies.

For existing apps, prefer additive nullable changes that preserve current reads and allow a null/unlicensed rollout state. Do not make existing users fail because a credential column, tenant setting, or config value is absent during rollout. Plan a separate backfill for existing customers, define rollback and recovery steps, and keep old code paths readable until the migration is verified.

Use approved local secret or configuration stores for app-wide credentials, tenant storage for tenant-level credential scope, and user/profile storage for user-level credential scope. Never place raw ApiKey, EncodedCredential, `.env`, user-secrets, or secret-bearing values in prompts, journals, docs, generated plugin output, or client-visible code/config. Use placeholders and resource-backed instructions instead.

For checkout or license delivery, reliable email remains a prerequisite. If the existing user/customer/tenant table lacks a trustworthy email, pause before purchase-flow code and decide how the host app will collect or verify it.

For rate-limit defaults during migration, preserve unlimited and capped semantics. Do not accidentally introduce low capped limits while backfilling or assigning features; make quota, reset cadence, and recovery behavior explicit.

For platform-specific persistence pitfalls (web storage semantics, lifecycle on desktop shells), resolve:

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.
</brownfield>

<success_criteria>
- Project builds without compilation errors after SDK integration.
- The SDK's authorization call returns a non-null result at runtime.
- No licensing configuration exception in application logs.
- Correct credential source pattern used (Configuration-Based or User-Managed).
- All namespaces match the `monaiq://domain/namespaces` reference.
</success_criteria>
