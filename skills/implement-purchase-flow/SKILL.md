---
name: implement-purchase-flow
description: "Use when: adding an in-app license purchase flow, buy button, Stripe checkout session, checkout result handling, credential persistence, or post-purchase SDK refresh."
agent: monaiq
auto-invoke:
  - "User wants to add in-app purchases or checkout to their application"
  - "User wants to integrate Stripe checkout for license purchasing"
  - "User asks how to embed a buy button or purchase flow"
tags: [sdk, checkout, purchase, licensing, stripe]
category: integration
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, register_or_login, profile, offering, feature_offering, implement_purchase_flow, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__implement_purchase_flow, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "platform (dotnet|dotnet/blazor-server|react|react/vite|react/nextjs)"
tier: 3
invoked-by: [implement-licensing, manage-catalog]
---

<input-context>
Receives from implement-licensing or manage-catalog:
- sdkConfig (optional): { platform, credentialSource } — confirms SDK is ready
- catalogSpec (optional): { offerings: [{ code, classification, baseRate }] } — available offerings to sell

If invoked without upstream context, verifies SDK integration and discovers offerings from catalog.
</input-context>

<output-context>
Provides completion context:
- checkoutImpl: { architecture: "server-initiated" | "client-only", offeringCodes: [], checkoutRoute: string, successHandler: string }

This skill is a leaf node — no downstream skills depend on it.
</output-context>

<monaiq-agent-handoff>
This skill is intended to run under the `monaiq` custom agent. If invoked directly and the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.

If host activation is unavailable, warn exactly: "Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."

The compatibility fallback still uses `monaiq_journal` for startup, checkpoints, `record_file_changes`, and `skill_completed`.
</monaiq-agent-handoff>

<state-detection>
Before implementing checkout:
1. Verify SDK is integrated — look for licensing package references
2. Call `offering` (list) — check for published offerings to sell
3. Check for existing checkout implementation — look for checkout session creation code

Based on detected state:
- SDK not integrated → Route to implement-licensing first
- No offerings exist → Route to manage-catalog to create offerings
- Offerings exist but all Draft → Warn that offerings need to be published (Status = Public) for checkout
- Existing checkout code found → Show current implementation, offer modifications
</state-detection>

<objective>
Guide an agent through adding an embedded purchase flow to a .NET or React application so users can buy licenses without leaving the app. Covers four areas: offering discovery and architecture decisions, backend checkout session creation, success handling with credential provisioning, and context provider wiring.
</objective>

<process>

## Journal Hook

Fetch `monaiq://protocols/implementation-journal`, call `monaiq_journal get_state`, handle resume through `CHECKPOINT-RESUME`, then call `skill_started` for `implement-purchase-flow`. Honor `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`, and `CHECKPOINT-SKILL-COMPLETE`; record build/test failures with `record_error` when known and changed paths only with `record_file_changes`. Do not journal ApiKey or EncodedCredential values.

`CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` is mandatory before checkout route, success-handler, provider behavior, post-purchase refresh, or other application behavior changes. Preserve `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE` and `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`; route any ApiKey/frontend exposure ambiguity through host-native ask and record the checkpoint result before proceeding.

## Evidence-Backed Implementation Pattern

Before code/config/business-logic changes, present a business-readable evidence summary and business-readable impact before compact technical backing. Technical backing includes codebase evidence, journal decisions, backend/profile/catalog/offering facts, route-packet evidence, labeled assumptions, confidence, and missing evidence.

Purchase-flow recommendations cite SDK state, offering availability, checkout architecture, credential persistence policy, profile/session state, and journal decisions before checkout behavior changes. If missing SDK/catalog/offering/profile/code evidence prevents a confident implementation recommendation, route to the appropriate prerequisite step before code/config/business-logic changes.

## Purchase Experience Contract

Design purchase UI around the host app's job, not a universal Monaiq screen. A full license page or purchase page is appropriate when the user is comparing plans, managing billing, or intentionally reviewing account state; a compact feature-level upsell is appropriate beside a locked action when the user already understands the task and needs the shortest trustworthy upgrade route. In both cases, keep the experience host-native, calm, task-focused, and persuasive without turning the application into a marketing page.

Every purchase surface must explain the current license state, the target offering, included features, limits, price, and fit in business-readable language. Offer comparison surfaces should help the user choose confidently by showing plan differences, included features, quota or usage limits, price, and who each plan is for without leading with technical licensing vocabulary. Upgrade copy must explain the current-to-target upgrade path and what value is gained.

Before direct checkout, require reliable email, customer identity, and offering context. Customer email is the purchase identity used for Stripe pre-fill, license delivery, and payment follow-up; if the app cannot provide a reliable email, pause for a host-native question or route to identity/profile work before checkout. Direct checkout from a compact upsell is allowed only when the customer, reliable email, target offering, success path, and recovery path are known.

After checkout completes, refresh license state, read the current entitlement snapshot, show newly unlocked features or the current entitlement state, and provide the next useful action. Cover loading, available, locked, near-limit, over-limit, expired, success, misconfigured, and recovery states so the user never lands in a vague purchased-but-still-blocked experience.

Placement is a product decision. Prefer account/settings/billing pages for broad license management, feature-adjacent surfaces for contextual locked-feature upsells, and admin/tenant surfaces for buyer-managed purchases, but verify against the host app's navigation, component system, accessibility, responsive behavior, and existing visual language. Do not impose a Monaiq visual standard.

Before UI code changes, present `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` with a summary naming affected pages, components, flows, and user-visible impact. Preserve `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE` before writing purchased credentials and record the approval result without ApiKey or EncodedCredential values.

## Prerequisites

- Licensing SDK already integrated (complete the `implement-licensing` skill first).
- Establish a session using the `register_or_login` tool.
- Retrieve credentials via the `profile` tool (step 2) — you need your reseller account identifier (`IssuerClientId`) and `ApiKey`.
- Published offerings exist — use the `offering` and `feature_offering` tools to browse the catalog.
- If any required canonical resource is unavailable, stop before catalog mutations, code edits, credential/config writes, or validation remediation. Do not guess checkout request fields, endpoint URLs, credential persistence semantics, or post-purchase refresh behavior.
- Resolve the SDK setup narrative and checkout endpoint configuration:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://config/endpoints` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

  Fetch `monaiq://docs/anti-patterns/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Interactive Step-by-Step Flow

Use `implement_purchase_flow` with `startStep=all` when platform, offering availability, required resources, credential handling, and checkout architecture are already known and no unresolved checkpoint/resource/validation/config blocker exists. Use numeric `startStep` mode (`startStep=1` through `startStep=4`) when step-by-step review improves safety or clarity. Read `journalReadyUpdates` from tool envelopes and apply them through `monaiq_journal`; never treat intents as already-applied state.

If checkout, build, or validation fails, call `monaiq_journal record_validation_failure` with command, observed result, likely cause, blocked next action, retry choices, and checkpoint requirement. For credential setup, `provision_api_key_config` returns a local plan with non-secret token markers and `CHECKPOINT-PRE-CREDENTIAL-WRITE`; require approval before local writes.

## Step 1: Discovery

Determine your application's checkout architecture and identify the offerings to sell.

**Architecture decision (platform-neutral):**

| Architecture | Flow | Recommendation |
|-------------|------|----------------|
| Has backend (API, BFF, SSR) | Server-Initiated Checkout | Recommended — secure; correlation ID stays server-side. |
| Frontend only (SPA) | Client-Only Checkout | Viable — ensure HTTPS; correlation ID managed client-side. |

**CorrelationId** is a tracking identifier (called `CorrelationId`) that links the purchase back to the buyer. It is an opaque string your application provides that round-trips through the entire checkout flow:

1. Your app sends `CorrelationId` with the checkout request.
2. Stripe Checkout completes, the webhook fires, the license is created.
3. The checkout-result call returns `CorrelationId` plus the license key.
4. Your app matches `CorrelationId` back to the user and stores the credential.

Use a stable, unique identifier — user ID, tenant ID, or a composite key.

**Browse offerings** using the `offering` tool to see what is available for purchase. Free, trial, and zero-dollar offerings are fully supported — the API handles them without a Stripe redirect.

For the authoritative offering entity shape and checkout-request field names:

Fetch `monaiq://domain/model` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 2: Backend Integration

Create a checkout session by calling the SDK's checkout service. The session carries the offering ID, the reseller identifier (`IssuerClientId`), the correlation ID, the customer email, and success / cancel return URLs. The response is either a redirect URL (paid offerings) or `null` (free / trial offerings, which auto-complete).

For the platform-specific checkout-service type, request/response shapes, and invocation pattern:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the runtime wiring narrative (where to inject the service, how to configure transport):

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the checkout endpoint base URL (do not hardcode — resolve at configuration time):

Fetch `monaiq://config/endpoints` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

**Security notes (platform-neutral):**
- Never expose the `ApiKey` to frontend code in production — proxy through your backend.
- `IssuerClientId` identifies your reseller account — obtained from the `profile` tool.
- Use HTTPS for every success / cancel URL.

## Step 3: Success Handling

After the user completes Stripe Checkout, retrieve the result using the session ID. The result exposes the checkout status, the `CorrelationId` you originally supplied, the license ID, and the encoded credential string.

Persist the credential against the user identified by `CorrelationId` — this credential is what the licensing SDK uses at runtime.

Purchased `EncodedCredential` values come from checkout-result retrieval or application storage, not the reseller profile.

For the platform-specific result type and retrieval call pattern:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For platform-specific persistence patterns and pitfalls (e.g., web storage semantics, secure-string handling):

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

### After purchase completes

Once the credential is persisted, **refresh the client and read current state** so the rest of the app
sees the new entitlements without a full reload:

- **React / Node:** `await client.refresh({ encodedCredential }); const state = await client.getState();`
- **.NET:** `var state = await provider.GetState(serviceReference, context);`

Wire these two calls into whatever success-callback shape your app already uses — the Stripe webhook
handler, a redux thunk, an API-route response, a Blazor event handler. The SDK exposes no
`onPurchaseComplete` hook; this is the canonical two-line idiom.

For the platform-specific signatures:

Fetch `monaiq://platforms/manifest/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 4: Context Provider Wiring

Connect the purchased credential to the licensing SDK so feature checks work at runtime.

**Decision (platform-neutral):**

| Credential scope | Wiring pattern |
|------------------|----------------|
| User-managed (multi-tenant, per-user) | Implement a custom licensing-context provider that resolves the credential from your user storage; register it in place of the default provider. |
| Configuration-based (single-organization license) | Store the encoded credential in your app configuration and restart; the default configuration-driven provider handles the rest. |

For the platform-specific provider interface, method signatures, and registration call:

Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For the SDK wiring narrative, including where the provider slots into the composition root:

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For platform-specific pitfalls (provider lifetime, re-resolution on credential change, lazy initialization):

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Verification

1. Create a checkout session with a test offering.
2. Complete the Stripe Checkout flow (or verify a free / trial offering auto-completes).
3. Retrieve the checkout result and confirm the encoded credential is returned.
4. Store the credential and verify the runtime authorization call returns a valid result.
5. Verify feature checks work with the new license.

## Related Tools

- `implement_purchase_flow` — Interactive step-by-step checkout integration (call `startStep=1` through `startStep=4` consecutively).
- `offering` — Browse available product offerings.
- `feature_offering` — View feature assignments for an offering.
- `profile` — Retrieve `IssuerClientId` and `ApiKey`.

## Related Resources

- `monaiq://platforms/api-surface/dotnet` — .NET checkout service, request/result types, provider interface.
- `monaiq://platforms/api-surface/react` — React client methods, provider component, context-hook signatures.
- `monaiq://platforms/manifest/dotnet` — Full machine-readable .NET platform manifest, including post-purchase state patterns.
- `monaiq://platforms/manifest/react` — Full machine-readable React platform manifest, including post-purchase refresh patterns.
- `monaiq://platforms/pitfalls/dotnet` — .NET pitfalls (DI lifetime, credential persistence).
- `monaiq://platforms/pitfalls/react` — React pitfalls (provider remount, storage semantics).
- `monaiq://docs/anti-patterns/dotnet` — .NET checkout and license-handling anti-patterns.
- `monaiq://docs/anti-patterns/react` — React checkout and license-handling anti-patterns.
- `monaiq://sdk/dotnet/setup` — .NET SDK wiring narrative.
- `monaiq://sdk/react/setup` — React SDK wiring narrative.
- `monaiq://config/endpoints` — Checkout/licensing endpoint base URLs.
- `monaiq://domain/model` — Offering, license, and checkout-session entity shapes.

</process>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Checkout session creation fails | Session-create call returns error | Verify `ApiKey` and `IssuerClientId` are correct (re-fetch from `profile`). Confirm the offering has Status = Public. |
| Stripe redirect fails | User sees Stripe error page | Verify success and cancel URLs use HTTPS and include the session-ID placeholder expected by the checkout service (see `monaiq://platforms/api-surface/{platform}`). Confirm the offering's `BaseRate` and `Currency` are valid. |
| Checkout-result retrieval returns no credential | Result poll returns an empty encoded-credential value | The checkout may not be complete yet — retry after a short delay. If persistent, verify the session ID is correct. |
| Credential storage fails | License purchased but credential lost | Re-call the checkout-result retrieval with the same session ID; results are persistent server-side. |
| Context provider wiring fails | Feature checks return null after purchase | Verify the custom context provider returns the stored credential. Check DI / composition-root registration against `monaiq://sdk/{stack}/setup`. |

Checkout sessions are idempotent — creating a new session for the same offering is safe. Completed purchases are recorded server-side and can be recovered.
</error-recovery>

<success_criteria>
- Checkout session creation works — returns a Stripe URL (paid) or null (free/trial).
- Checkout-result retrieval returns the encoded credential and the correlation ID after payment completes.
- Credential is stored and the runtime authorization call returns a valid result using the purchased license.
- Feature checks work at runtime with the purchased credential.
- `ApiKey` is not exposed to frontend code in production.
</success_criteria>
