# Handoff Schemas

Persist handoff packets in `.monaiq/STATE.md` summaries or `.monaiq/JOURNAL.md` entries with no secrets. Downstream skills should resume from these packets instead of re-asking questions already answered by evidence.

## routePacket

- `scenario`
- `targetApp`
- `platform`
- `profileState`
- `catalogState`
- `codeEvidenceSummary`
- `journalEvidenceSummary`
- `assumptions`
- `recommendedSkill`
- `recommendationRationale`
- `checkpointName`
- `authorizedBy`

## catalogSpec

- `productCode`
- `features[]`
- `offerings[]`
- `assignments[]`

## pricingPlan

- `tiers[]` with `name`, `classification`, `baseRate`, `currency`, `interval`, and `features[]`

## sdkConfig

- `platform`
- `credentialSource`
- `serviceOptionsConfigured`
- `diRegistered`

## featureImpl

- `implementedFeatures[]`
- `buildVerified`

## checkoutImpl

- `architecture`
- `offeringCodes[]`
- `checkoutRoute`
- `successHandler`

## validationProof

- `passed[]`
- `failed[]`
- `skipped[]`
- `unverified[]`