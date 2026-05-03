---
name: manage-catalog
description: "Use when: creating or managing a Monaiq product catalog, products, features, offerings, pricing tiers, and feature assignments after onboarding or monetization design."
agent: monaiq
auto-invoke:
  - "User wants to create or manage their product catalog"
  - "User wants to set up products, features, and offerings"
  - "User asks how to define pricing tiers or feature assignments"
tags: [catalog, products, features, offerings, orchestration]
category: catalog
allowed-tools: [register_or_login, profile, product, product_feature, offering, feature_offering, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
tier: 2
invoked-by: [getting-started]
---

<objective>
Create or update a Monaiq product catalog through evidence-backed recommendation, checkpointed mutation, and read-back verification. The skill owns product, feature, offering, and feature-assignment tool calls; it does not own SDK setup or app code changes.
</objective>

<monaiq-agent-handoff>
This skill is intended to run under the `monaiq` custom agent. If invoked directly and the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.

If host activation is unavailable, warn exactly: "Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."

The compatibility fallback still uses `monaiq_journal` for startup, checkpoints, `record_file_changes`, and `skill_completed`.
</monaiq-agent-handoff>

<input-output-contract>
Input from `getting-started`: `userScenario`, `detectedState`, and optional `quickStartSpec`. Direct invocation must rebuild equivalent context through session/profile/catalog state detection.

Output: `catalogSpec: { productCode, features: [{ key, type, displayName }], offerings: [{ code, classification, baseRate, currency, interval }], assignments: [{ featureKey, offeringCode, accessLevel | rateLimit }] }` plus `catalogComplete: true` after read-back verification. Downstream chain: `manage-catalog` -> `implement-licensing` -> `implement-feature`.
</input-output-contract>

<workflow>
1. Fetch `monaiq://protocols/implementation-journal`, call `monaiq_journal get_state`, initialize and apply returned `.monaiq/*` operations if needed, then call `skill_started` for `manage-catalog`.
2. Establish prerequisites: active session, `ResellerStatus = Enabled`, and `monaiq://domain/model` fetched. Stop before catalog mutation when session/profile/domain evidence is missing.
3. Detect backend catalog state in order: `product` list, `product_feature` list for relevant products, `offering` list, and `feature_offering` list for offerings. Backend catalog state wins; journal records decisions and outcomes only.
4. Use evidence before asking a new question. Infer catalog recommendation next steps from codebase evidence, route packet context, existing product/offering facts, and journal decisions. You must confirm inferred decisions in the next existing checkpoint, especially `CHECKPOINT-PRE-CATALOG-MUTATION`, with labeled assumptions for product structure, feature path, offering shape, credential handling impact, checkout architecture impact, and next steps. Do not add a new checkpoint name solely for evidence inference.
5. Present a business-readable evidence summary and business-readable impact before compact technical backing. Technical backing includes codebase evidence, journal decisions, backend/profile/catalog/offering facts, route-packet evidence, labeled assumptions, confidence, missing evidence, and the exact `product`, `product_feature`, `offering`, or `feature_offering` operations proposed.
  Every product, feature, offering, pricing, or assignment proposal is an evidence-backed catalog recommendation.
6. Stop at `CHECKPOINT-PRE-CATALOG-MUTATION` before any `product`, `product_feature`, `offering`, or `feature_offering` create/update/delete call. Record the user's result before tool calls run.
7. Execute catalog mutations in dependency order: product, features, offerings, feature assignments. Keep offerings in Draft unless the user explicitly approves publication through `CHECKPOINT-PRE-PUBLISH-OFFERING`.
8. Perform mandatory read-back verification after create/update of paid-tier assignments: list `feature_offering` rows for each paid offering, verify intended `ServiceAccessLevel` or rate-limit values, and stop on mismatches before claiming catalog completion.
9. Record changed catalog entities with `monaiq_journal`, save `CHECKPOINT-SKILL-COMPLETE` when useful, apply returned operations, call `skill_completed`, and hand off `catalogSpec` to `implement-licensing`.
</workflow>

<reference>
## State Routing Reference

- No products -> full creation flow.
- Products exist, no features -> create features for existing product.
- Products and features exist, no offerings -> create offerings.
- Products, features, offerings exist but assignments are missing -> create assignments.
- Everything exists -> show catalog summary and use utility workflows.

## New Catalog Flow

### Interaction 1: "What features does your product have?"

Ask the user to describe their product and its capabilities in plain language.

If `quickStartSpec.appDescription` was provided from getting-started, use it instead of asking.

From the user's description, infer:
- **Product**: name and code (derive code from name, e.g., "My Cool App" → `my-cool-app`)
- **Features**: map described capabilities to feature types:
  - Binary on/off capabilities → feature flags (called `ProductAccessFeature` in Monaiq)
  - Usage-counted capabilities → usage limits (called `ProductRateLimitFeature` in Monaiq)
  - Generate a feature key for each: lowercase-hyphenated from capability name (e.g., "Premium Dashboard" → `premium-dashboard`)

Present the inferred structure to the user for confirmation before creating.

[CONFIRM] Execute tool calls: `product` (create) → `product_feature` (create for each feature).

Smart defaults: Product Status = `Active`, feature key derived from name.

### Interaction 2: "How should they be priced?"

Ask how the user wants to charge, or use `quickStartSpec.pricingChoice` if provided.

Present pricing options in plain language:
- **"Free trial + paid subscription"** → Create a Trial offering (14 days, $0) + Subscription offering ($X/month)
- **"Just a subscription"** → Create a Subscription offering ($X/month)
- **"One-time purchase"** → Create a Perpetual offering ($X one-time)
- **"Free tier + paid tier"** → Create a free Subscription ($0/month) + paid Subscription ($X/month)
- **"Usage-based"** → Create a Subscription with usage limit values scaled per tier

For each pricing tier (called an Offering in Monaiq), apply smart defaults:
- Currency: `USD`
- Interval: 1 Month for Subscription, none for Perpetual, 14 days for Trial
- Status: `Draft` (safe default — user publishes when ready)

[CONFIRM] Execute tool calls: `offering` (create for each tier) → `feature_offering` (assign features to each offering).

Default assignment values:
- Paid tiers: All feature flags (`ProductAccessFeature`) set to `Allowed`, usage limits (`ProductRateLimitFeature`) at user-specified or suggested quotas
- Free/Trial tiers: Feature flags selectively `Allowed`/`Denied` based on user description, usage limits at reduced quotas

- Catalog identifiers (offering ids, product ids, feature keys) referenced in code MUST live in a constants module. Inline string literals are forbidden (D-30).

**Required payload shape — `feature_offering` create for an access feature.** `ServiceAccessLevel` is required; omitting it is rejected by the API. Always send it explicitly:

```json
{
  "__sidub_entityType": "ProductFeatureOffering.ServiceAccess",
  "OfferingId": "<offering-id>",
  "FeatureKey": "<feature-key>",
  "ServiceAccessLevel": "Allowed"
}
```

For rate-limit features, both `SampleSeconds` and `RateLimit` are required (use matching `"unlimited"` strings or matching positive integers — asymmetric combinations are rejected).

**Read-back verification (mandatory after any create/update of paid-tier assignments):**
1. Call `feature_offering` (list) for each paid offering.
2. Assert every `ProductFeatureOffering.ServiceAccess` row has `ServiceAccessLevel: "Allowed"` (unless the user explicitly intended `Denied` for that tier).
3. If any assignment shows `ServiceAccessLevel: "Denied"` that the user did not intend, stop and surface it to the user — this is the silent-failure mode where customers receive credentials but `AssertLicense` always returns `false`. Re-run `feature_offering` (update) with the correct level.

## Utility Workflows

When state detection shows everything already exists, offer these options:

- **Update features**: Add new features to existing product, modify feature key, change feature type
- **Modify offerings**: Change pricing, update billing interval, change license type (e.g., promote Trial to Subscription — called `LicenseClassification` in Monaiq)
- **Modify assignments**: Change feature values for a pricing tier — toggle access, adjust rate limits
- **Add tier**: Create a new pricing tier (Offering) and assign features (common when adding a free or enterprise tier)
- **View catalog**: Show full product → features → pricing tiers → feature assignments summary

</reference>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Product creation fails | Tool returns error on `product` create | Retry once. If persistent, check session is authenticated via `register_or_login`. |
| Feature creation fails after product created | `product_feature` returns validation error | Product is safe — retry the feature creation with corrected input. List existing features first to avoid duplicates. |
| Offering creation fails after features created | `offering` returns error | Product and features are safe. Retry offering creation. Check that LicenseClassification is valid (Trial, Subscription, or Perpetual). |
| Feature assignment fails after offering created | `feature_offering` returns error | Offering exists but is incomplete. Retry assignment. Verify FeatureKey matches exactly and feature type matches assignment type (Access → ServiceAccessFeatureOffering, RateLimit → RateLimitFeatureOffering). |
| Partial catalog state | Some entities created, process interrupted | Re-run state detection — the skill will resume from the last incomplete step. Already-created entities are preserved. |

All catalog operations are idempotent at the entity level — creating a product that already exists will return the existing entity. Feature assignments can be updated in place.
</error-recovery>

<success_criteria>
- Product exists with features, offerings configured with pricing, all features assigned to offerings with correct values
- Read-back verification passes
- catalogSpec output-context populated for downstream skills
</success_criteria>
