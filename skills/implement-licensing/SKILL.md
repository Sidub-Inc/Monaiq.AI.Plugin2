> [!WARNING]
> **Requires monaiq MCP server attached.** This skill references MCP resources
> via <resource-ref> blocks. Without the monaiq MCP server wired to your
> agent, these references cannot be resolved. Connect the MCP server at
> https://api.monaiq.com/runtime/webhooks/mcp before invoking this skill.
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
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, register_or_login, profile, product, product_feature, implement_base, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__implement_base]
argument-hint: "language (dotnet|react)"
tier: 2
invoked-by: [getting-started]
---

<input-context>
Receives from manage-catalog (or direct invocation):
- catalogSpec (optional): { productCode, features: [{ key, type }], offerings: [{ code, classification }] } — if provided, skip catalog verification
- language (optional): "dotnet" | "react" — if provided from user intent, skip language detection

If invoked without upstream context, the skill detects current state and asks the user for missing information.
</input-context>

<output-context>
Provides to downstream skills (implement-feature, implement-purchase-flow):
- sdkConfig: { language: "dotnet" | "react", credentialSource: "configuration" | "user-managed", serviceOptionsConfigured: boolean, diRegistered: boolean }
- targetFeatures: [{ featureKey, featureType }] — features available for gating (from catalog or user input)

Chain: manage-catalog [catalogSpec] → implement-licensing [sdkConfig] → implement-feature
</output-context>

<state-detection>
Before executing, check what's already set up. For the exact package names, configuration keys, DI extensions, and provider components to grep for, resolve:

<resource-ref uri="monaiq://sdk/{language}/setup"/>

<resource-ref uri="monaiq://platforms/api-surface/{platform}"/>

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

## Prerequisites

- Establish a session using the `register_or_login` tool.
- Retrieve credentials via the `profile` tool — you need your license credential (the encoded license key returned by `profile`).
- Resolve the platform-specific SDK setup narrative, API surface, and endpoint config:

  <resource-ref uri="monaiq://sdk/{language}/setup"/>

  <resource-ref uri="monaiq://platforms/api-surface/{platform}"/>

  <resource-ref uri="monaiq://config/endpoints"/>

  <resource-ref uri="monaiq://docs/anti-patterns/{platform}"/>

## Interactive Step-by-Step Flow

You MUST invoke the `implement_base` tool with `startStep=1` and iterate through each step consecutively — `startStep=2`, `startStep=3`, `startStep=4`, `startStep=5`, `startStep=6`. Each step returns content later steps depend on (credential source → packages → namespaces → service options → DI → runtime); skipping causes wrong namespaces, missing registrations, and broken builds. After each call, apply the step's directives before requesting the next. The Step 1–6 overview below mirrors what `implement_base` returns and serves as a map.

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

Install the SDK package for the target language. For the installation command, package name, and transitive-dependency notes, resolve:

<resource-ref uri="monaiq://sdk/{language}/setup"/>

> **Note (.NET):** Requires Sidub.Platform 1.10.38+. Consumers using this version or later do not need any `<ExcludeAssets>` Metalama workaround — the Metalama build tool is no longer a transitive dependency (DEV-08 / RT-6 resolved in Sidub.Platform.Core).

## Step 3: Namespace Reference

For the authoritative type-to-namespace mappings (which imports expose credential types, service interfaces, feature types, and platform core), resolve:

<resource-ref uri="monaiq://domain/namespaces"/>

Use ONLY the namespaces from the reference. Do not guess or assume type locations.

## Step 4: Configure Service Options

Configure the licensing service with endpoint URIs and (for Configuration-Based credential source) the encoded credential.

- For the platform-specific configuration shape (settings file keys, provider component props, environment variable wiring), resolve:

  <resource-ref uri="monaiq://sdk/{language}/setup"/>

- For the endpoint URL values to populate into that configuration, resolve:

  <resource-ref uri="monaiq://config/endpoints"/>

- For the authoritative property names and their types on the configuration surface, resolve:

  <resource-ref uri="monaiq://platforms/api-surface/{platform}"/>

For **User-Managed** credential source, omit the encoded credential from configuration — it is resolved at runtime by your custom credential resolver (see Step 5).

## Step 5: Register Dependency Injection

Register the licensing services with your application's DI / provider graph.

- For the platform-specific registration pattern (service collection extensions in .NET, provider component in React), resolve:

  <resource-ref uri="monaiq://sdk/{language}/setup"/>

- For the exact extension-method / component signatures (including the generic overload used for user-managed credential resolvers), resolve:

  <resource-ref uri="monaiq://platforms/api-surface/{platform}"/>

For **User-Managed** credential source, register your custom credential-resolver type via the generic registration overload documented in the platform's api-surface resource.

## Step 6: Runtime Usage

Integrate the licensing service into your application's runtime.

- For the platform-specific type signatures — the service interface (.NET) or hook (React) used to request authorization — resolve:

  <resource-ref uri="monaiq://platforms/api-surface/{platform}"/>

- For the runtime wiring narrative — how to inject the service, when to call it, error-state handling — resolve:

  <resource-ref uri="monaiq://sdk/{language}/setup"/>

For known platform-specific pitfalls (null-semantics differences, provider remount behavior), resolve:

<resource-ref uri="monaiq://platforms/pitfalls/{platform}"/>

## Step 7: Displaying License State

Your application often needs to render the user's current license, entitlements, or consumption
— entitled features, rate-limit usage, expiry. The Monaiq SDK exposes this snapshot in-process; it
does **not** ship UI chrome. Render with your existing component library.

- For the platform-specific signature (`LicensingClient.getState()` for React,
  `ILicenseStateProvider.GetState()` for .NET) and the `LicenseStateView` shape, resolve:

  <resource-ref uri="monaiq://platforms/api-surface/{platform}"/>

- For the canonical post-purchase refresh + snapshot pattern, resolve:

  <resource-ref uri="monaiq://platforms/manifest/{platform}"/>

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

- **Credential recovery** — Call the `profile` tool to re-retrieve your license key. Useful when credentials are lost or need refreshing.
- **Update service URIs** — Modify licensing configuration to point to different environments (staging, production). Resolve `monaiq://config/endpoints` for the current authoritative URIs.
- **Switch credential source** — Migrate from configuration-based to user-managed credentials (or vice versa). Involves implementing or removing a custom credential resolver; resolve `monaiq://sdk/{language}/setup` for the migration narrative.
- **Verify integration** — Run a quick health check: confirm packages are installed, configuration is present, DI is registered, and a test authorization call succeeds.

</process>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Package install fails | NuGet/npm error during SDK package installation | Check network connectivity and package source configuration. Retry with an explicit registry/source (`--source nuget.org` for .NET, clear npm cache for React). Resolve `monaiq://sdk/{language}/setup` for the exact package name and commands. |
| Credential retrieval fails | `profile` tool returns an error | Verify the session is active via `register_or_login`. Re-authenticate if the session expired. |
| Config binding fails | Licensing configuration exception at startup | Verify the configuration section name and key casing match the platform-specific configuration shape. Check that the credential field is not empty or malformed. Resolve `monaiq://sdk/{language}/setup` for the authoritative configuration keys. |
| DI registration fails | Build error on the SDK's registration extension or provider component | Verify the package is installed and the correct imports/usings are present. Resolve `monaiq://platforms/api-surface/{platform}` for the authoritative extension-method or component signatures. |
| Authorization call returns null | License validation fails at runtime | Retrieve a fresh credential from the `profile` tool. Resolve `monaiq://config/endpoints` to confirm service URIs are correct for the target environment. Resolve `monaiq://platforms/pitfalls/{platform}` for platform-specific null-semantics differences. |

SDK integration steps are non-destructive — each step modifies source files that can be edited again safely.
</error-recovery>

<brownfield>
## Persist the EncodedCredential (brownfield)

On checkout completion the SDK hands your code an `EncodedCredential` string. The SDK does NOT persist
it for you — you must store it somewhere your next session can read and pass back to
`LicensingClient` / `ILicenseStateProvider` at startup.

**Branch 1 — Multi-user app (has a users / customers / identity table):**
Add a nullable text column (e.g. `encoded_credential`) to that table. Respect the existing schema and
the project's migration workflow — EF Core, Prisma, Drizzle, raw SQL, whatever the codebase uses. If
credentials are 1:1 with users, add a scoped unique index. Prefer additive migrations — do not break
existing functionality. Evaluate column-level encryption per your project's data-classification policy.

**Branch 2 — Single-user app (CLI, desktop, developer tool):**
Three options, in rough order of increasing security:

| Storage | Pro | Con |
|---------|-----|-----|
| Config file | Simple, zero deps | Version-control leakage risk; must be gitignored |
| Environment variable / `.env.local` | Matches `provision_api_key_config` pattern; gitignored by default | Plain-text on disk |
| OS keychain (DPAPI, macOS Keychain, libsecret) | Most secure | Platform-specific; adds native dependency |

For platform-specific persistence pitfalls (web storage semantics, lifecycle on desktop shells), resolve:

<resource-ref uri="monaiq://platforms/pitfalls/{platform}"/>
</brownfield>

<success_criteria>
- Project builds without compilation errors after SDK integration.
- The SDK's authorization call returns a non-null result at runtime.
- No licensing configuration exception in application logs.
- Correct credential source pattern used (Configuration-Based or User-Managed).
- All namespaces match the `monaiq://domain/namespaces` reference.
</success_criteria>
