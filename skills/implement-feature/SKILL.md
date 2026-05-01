---
name: implement-feature
description: "Use when: adding Monaiq feature gates, access checks, premium feature enforcement, rate-limit assertions, consumption recording, or license feature checks to an SDK-integrated app."
agent: monaiq
auto-invoke:
  - "User wants to add feature gating to their application"
  - "User wants to implement license feature checks"
  - "User asks how to gate functionality by license features"
tags: [sdk, features, licensing, entitlements, access, ratelimit]
category: integration
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, product, product_feature, feature_offering, implement_product_feature, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__implement_product_feature, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "featureKey, featureType (access|ratelimit)"
tier: 3
invoked-by: [implement-licensing, manage-catalog]
---

<input-context>
Receives from implement-licensing:
- sdkConfig: { platform, credentialSource, serviceOptionsConfigured, diRegistered } — confirms SDK is ready
- targetFeatures (optional): [{ featureKey, featureType }] — pre-selected features to implement

If invoked without upstream context, verifies SDK integration exists and discovers features from catalog.
</input-context>

<output-context>
Provides to downstream skills:
- featureImpl: { implementedFeatures: [{ featureKey, featureType, codeFile }], buildVerified: boolean }

This is typically the last step in the implementation chain:
getting-started → manage-catalog → implement-licensing → implement-feature
</output-context>

<state-detection>
Before implementing feature checks:
1. Verify the SDK is integrated — look for the platform's licensing package in project manifests and for licensing DI / provider registration in the composition root.
2. Call `product_feature` (list) — get available features and their types.
3. Check existing source files for feature assertions — look for the platform's assertion attribute / hook usage as documented in `monaiq://platforms/api-surface/{platform}`.

Based on detected state:
- SDK not integrated → Route to implement-licensing first
- No features in catalog → Route to manage-catalog to create features
- Features exist, none implemented → Full flow from Step 1
- Some features already implemented → Show which are done, offer to implement remaining
</state-detection>

<objective>
Guide an agent through adding license feature checks to a .NET or React application. Features are polymorphic — **feature flags** (Access — binary gate: allowed or denied) and **usage limits** (RateLimit — metered consumption tracking). Covers feature discovery, implementation of the correct assertion pattern, and build verification.
</objective>

<process>

## Journal Hook

Fetch `monaiq://protocols/implementation-journal`, call `monaiq_journal get_state`, handle resume through `CHECKPOINT-RESUME`, then call `skill_started` for `implement-feature`. Honor `CHECKPOINT-FEATURE-SELECTION`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, and `CHECKPOINT-SKILL-COMPLETE`; record build/test failures with `record_error` when known and changed paths only with `record_file_changes`. Do not add a tool-call audit log or separate deferred-ideas event.

## Prerequisites

- Licensing SDK already integrated (complete the `implement-licensing` skill first).
- Product and features exist in the catalog — use the `product` and `product_feature` tools to verify.
- Resolve the domain model, namespace table, and platform API surface:

  Fetch `monaiq://domain/model` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://domain/namespaces` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://docs/anti-patterns/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Interactive Step-by-Step Flow

You MUST invoke the `implement_product_feature` tool with `startStep=1` then `startStep=2` consecutively. Each step returns authoritative type information and code snippets later steps depend on; skipping causes wrong assertion types and broken builds. After each call, apply the step's directives before requesting the next. Build-and-verify guidance is included in step 2's hints. The Step 1–2 overview below mirrors what `implement_product_feature` returns and serves as a map.

## Step 1: Discover Feature

Identify the feature to gate and determine its type.

**Use the `product_feature` tool** to list features for a product. Each feature exposes:

| Field | Purpose |
|-------|---------|
| Feature key | The unique identifier you defined in your catalog; used in code to reference this feature. |
| Kind | `ServiceAccess` or `RateLimit` — determines the assertion pattern. |
| Display name | Human-readable name. |
| Service type | (Access only) The service-type classification. |

For the authoritative field / property names on the platform-specific feature record, resolve:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

**Feature-Type Decision Table (domain guidance — platform-neutral):**

| Aspect | Feature Flag (Access) | Usage Limit (RateLimit) |
|--------|----------------------|-------------------------|
| Pattern | Assert → allowed/denied | Get feature → record consumption → assert |
| Use case | Premium content, feature flags, capability toggles | API rate limits, usage quotas, metered operations |
| Cardinality | One assertion per check | Multi-step: retrieve → record → assert |

**Critical:** Each feature type has its own specific assertion type. Mixing types causes runtime failures. Resolve `monaiq://platforms/api-surface/{platform}` for the authoritative assertion types — use the Access assertion for Access features, the RateLimit assertion for RateLimit features.

For the enum/type values of the feature kind and how to inspect them programmatically, resolve:

Fetch `monaiq://domain/model` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 2: Implement Feature Check

- Place all licensing-related string literals (feature keys, offering ids, redirect URLs) in a single constants module (e.g. `LicensingConstants.cs` for .NET, `licensingConstants.ts` for React). Reference them by symbol — do NOT inline the literal in business logic. This is mandatory (Phase 14 D-30).

### Feature Flag Check (Access — Binary Gate)

Use when: a binary "allowed / not allowed" decision is sufficient (premium gate, capability toggle, feature-flag rollout).

For the platform-specific assertion type and the call sequence used to evaluate it:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For runtime wiring narrative (where to place the check, how to render the denied state):

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

### Usage Limit Check (RateLimit — Metered Consumption)

Use when: usage must be metered and bounded (API rate limits, per-period quotas, consumption-based features). Pattern is always multi-step — retrieve the feature record, record consumption, then assert whether the limit is still respected.

<!-- SEM-01-stopgap -->
> **Unlimited assignments.** In the domain model, `RateLimit = 0` represents an unlimited entitlement. When creating or updating a rate-limit assignment through the `feature_offering` MCP tool, send both `RateLimit` and `SampleSeconds` as the string `"unlimited"`; the tool maps that to the internal zero representation. Non-zero capped values must be positive integers for both fields.
<!-- /SEM-01-stopgap -->

For the platform-specific feature retrieval, consumption recording, and assertion types:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the runtime wiring and error-handling narrative:

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For known platform-specific pitfalls (e.g., null semantics, provider remount, async hook reuse):

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

**Error-boundary pattern for rate-limit violations** (consumer-built — Monaiq ships
the typed `RateLimitError`, you build the UI):

```tsx
class RateLimitBoundary extends React.Component<{children: React.ReactNode}, {hit: boolean}> {
  state = { hit: false };
  static getDerivedStateFromError(e: unknown) { return { hit: e instanceof RateLimitError }; }
  render() { return this.state.hit ? <p>Rate limited — try again soon.</p> : this.props.children; }
}
```

### Build and Verify

Build-and-verify guidance — compile the project, resolve missing namespaces against `monaiq://domain/namespaces`, test the allow path, and exercise the deny path — is delivered by `implement_product_feature` step 2 via its `hints` array. Follow those hints after implementing the feature check.

## Related Tools

- `implement_product_feature` — Interactive step-by-step feature integration (call `startStep=1` then `startStep=2` consecutively).
- `product` — List products in the catalog.
- `product_feature` — List features for a product.
- `implement_base` — SDK integration (prerequisite).

## Related Resources

- `monaiq://domain/model` — Entity relationships and feature-type hierarchy.
- `monaiq://domain/namespaces` — Authoritative namespace-to-type mappings.
- `monaiq://platforms/api-surface/dotnet` — .NET assertion types, method signatures, semantics.
- `monaiq://platforms/api-surface/react` — React assertion types, hook signatures, semantics.
- `monaiq://platforms/pitfalls/dotnet` — .NET known issues.
- `monaiq://platforms/pitfalls/react` — React known issues (provider remount, async hook reuse).
- `monaiq://docs/anti-patterns/dotnet` — .NET anti-patterns for license checks and rate-limit enforcement.
- `monaiq://docs/anti-patterns/react` — React anti-patterns for license checks and rate-limit enforcement.
- `monaiq://sdk/dotnet/setup` — .NET SDK runtime wiring narrative.
- `monaiq://sdk/react/setup` — React SDK runtime wiring narrative.

</process>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Feature not found in catalog | `product_feature` list returns empty or missing feature | Verify the product code is correct. Route to `manage-catalog` to create missing features. |
| Wrong assertion type used | Runtime type-mismatch error | Check the feature's kind — resolve `monaiq://platforms/api-surface/{platform}` for the correct assertion type per kind, and `monaiq://domain/model` for the kind enum. |
| Namespace import errors | Build fails with missing type references | Resolve `monaiq://domain/namespaces` and verify all imports match the authoritative reference. |
| Assertion returns unexpected result | Feature check returns denied when it should be allowed | Verify the feature is assigned to the user's offering with the correct value (Allowed, not Denied). Check via the `feature_offering` tool. |
| Consumption recording fails | Recording operation throws for RateLimit features | Verify the feature type is RateLimit (not Access). Check that the consumption amount is a positive number. Resolve `monaiq://platforms/pitfalls/{platform}` for platform-specific error modes. |

Feature-gate code is additive — failed implementations can be corrected by editing the source file. No rollback needed.
</error-recovery>

<success_criteria>
- Correct assertion type used for each feature kind (Access vs. RateLimit).
- Feature checks return allowed for valid licenses with the feature granted.
- Application handles the denied case gracefully.
- No namespace errors — all imports match `monaiq://domain/namespaces`.
- Project builds without errors.
</success_criteria>
