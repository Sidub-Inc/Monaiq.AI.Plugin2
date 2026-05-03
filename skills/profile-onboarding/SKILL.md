---
name: profile-onboarding
description: "Use when: viewing Monaiq reseller profile status, retrieving reseller ApiKey and IssuerClientId, or reviewing terms and privacy documents."
agent: monaiq
auto-invoke:
  - "User wants to view their reseller profile or retrieve SDK credentials"
  - "User wants to review terms of service before accepting"
  - "User asks about their onboarding status or account details"
tags: [profile, onboarding, credentials, terms]
category: onboarding
allowed-tools: [register_or_login, profile, monaiq_journal, fetch_step_resources, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources]
tier: 3
invoked-by: [getting-started]
---

<input-context>
Receives from getting-started:
- detectedState (optional): { profileComplete, resellerEnabled } — skip viewing if state is known

If invoked without upstream context, checks profile state directly.
</input-context>

<output-context>
Provides to downstream skills:
- profileData: { issuerClientId, profileStatus, resellerStatus, apiKeyAvailable }

Used by implement-purchase-flow for checkout configuration. Purchased EncodedCredential values come from checkout results, not the reseller profile.
</output-context>

<monaiq-agent-handoff>
This skill is intended to run under the `monaiq` custom agent. If invoked directly and the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.

If host activation is unavailable, warn exactly: "Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."

The compatibility fallback still fetches `monaiq://protocols/implementation-journal`, calls `monaiq_journal get_state` or `monaiq_journal init`, calls `skill_started`, and enforces relevant checkpoints before consequential follow-up actions.
</monaiq-agent-handoff>

<state-detection>
Before showing profile:
1. Call `profile` tool to get current status
2. Check ProfileStatus and ResellerStatus

Based on state:
- ProfileStatus = NotStarted → Guide through full profile setup
- ProfileStatus = Incomplete → Show what's missing, guide completion
- ProfileStatus = Completed, ResellerStatus = Pending → Show status, explain approval process
- Everything complete → Show summary, offer credential retrieval
</state-detection>

<objective>
Guide an agent through reviewing a reseller profile, retrieving reseller checkout credentials, and reading the Terms of Service and Privacy Policy. This covers three read-only steps. Terms acceptance (step 4) is a separate tool action — not part of this skill workflow.
</objective>

<process>

## Journal Hook

Fetch `monaiq://protocols/implementation-journal`, call `monaiq_journal get_state`, then call `skill_started` for `profile-onboarding`. Use `CHECKPOINT-PRE-TERMS-ACCEPTANCE` before terms/profile acceptance; save `CHECKPOINT-SKILL-COMPLETE` when useful, then call `skill_completed`.

## Prerequisites

- Establish a session using the `register_or_login` tool
- If any required canonical resource is unavailable, stop before catalog mutations, code edits, credential/config writes, or validation remediation. Do not infer credential ownership or write secret-bearing values to `.monaiq`.

## Step 1: View Profile

Call the `profile` tool with `startStep=1` to view the reseller profile.

**Key fields:**

| Field | Description |
|-------|-------------|
| `IssuerClientId` | Unique reseller identifier — used in SDK configuration and checkout requests |
| `ContactEmail` | Primary contact email for the account |
| `LegalName` | Legal business name on file |
| `ProfileStatus` | Account completeness: `NotStarted` → `Incomplete` → `Completed` |
| `ResellerStatus` | Onboarding state: `Disabled` → `Pending` → `Enabled` |

**Status meanings:**

- **ProfileStatus = NotStarted** — Account created but no profile information submitted
- **ProfileStatus = Incomplete** — Some profile fields are filled but required information is missing
- **ProfileStatus = Completed** — All required profile information is on file
- **ResellerStatus = Disabled** — Reseller features not activated; cannot create products or offerings
- **ResellerStatus = Pending** — Reseller application submitted, awaiting approval
- **ResellerStatus = Enabled** — Full reseller access; can create products, offerings, and issue licenses

The `IssuerClientId` is the primary identifier for the reseller account. It is used as the `IssuerClientId` parameter in checkout requests and appears in license metadata.

## Step 2: Retrieve Credentials

Call the `profile` tool with `startStep=2` to retrieve reseller credentials used for catalog and checkout operations.

**Credential fields:**

| Credential | Purpose | Where Used |
|-----------|---------|------------|
| `ApiKey` | Authenticates checkout API calls | `CreateCheckoutSession` and `GetCheckoutResult` API key parameter |
| `IssuerClientId` | Reseller identity for checkout requests | `CheckoutRequest.IssuerClientId` |

EncodedCredential is not a reseller profile credential. It is produced after an offering is purchased and returned by checkout-result retrieval.

**Security:** Do not persist the `ApiKey` to disk or commit it to source control. Use environment variables or a secrets manager for production deployments.
Do not write raw `ApiKey`, `IssuerClientId` plus secret context, `EncodedCredential`, `.env`, or user-secret values into prompts, `.monaiq`, summaries, or generated plugin output.

**How credentials connect to checkout setup:**

- `ApiKey` → Used when calling `ICheckoutService` methods (see the `implement-purchase-flow` skill)
- `IssuerClientId` → Used in `CheckoutRequest` for embedded purchases

## Step 3: Read Terms

Call the `profile` tool with `startStep=3` to read the Terms of Service and Privacy Policy.

Both documents must be presented to the user before acceptance. The terms cover:

- **Terms of Service** — Account registration, use of services, intellectual property, data processing, limitation of liability
- **Privacy Policy** — Data collection, usage, storage practices, user rights under applicable privacy laws

Review both documents carefully before proceeding to accept terms.

## Accepting Terms (Step 4)

**Important:** Accepting terms is NOT part of this skill workflow. It is a **tool action** that modifies state.

To accept terms, call the `profile` tool directly with `startStep=4` and `data={"customerTerms": true, "resellerTerms": true}`. Both `customerTerms` and `resellerTerms` must be set to `true` to complete terms acceptance. This action records the user's agreement and updates the profile status.

## Related Tools

- `register_or_login` — Establish a session (prerequisite for all profile operations)
- `profile` — The tool that executes each step of this workflow
- `implement_base` — SDK integration (configures where purchased credentials are supplied at runtime)
- `implement_purchase_flow` — Checkout integration (uses ApiKey and IssuerClientId from step 2)

## Utility Workflows

- **Credential recovery**: Re-retrieve reseller checkout credentials from your profile. Use purchase-flow checkout results or your application storage for purchased `EncodedCredential` values.
- **View terms**: Review Terms of Service and Privacy Policy before accepting.
- **Check approval status**: See current ResellerStatus and what to expect next.

</process>

<success_criteria>
- Profile information is visible including `IssuerClientId`, `ProfileStatus`, and `ResellerStatus`
- Reseller credentials (`ApiKey`, `IssuerClientId`) are retrieved successfully
- Terms of Service and Privacy Policy are presented for review
- Step 4 (terms acceptance) is understood as a separate tool action with `startStep=4`
</success_criteria>
