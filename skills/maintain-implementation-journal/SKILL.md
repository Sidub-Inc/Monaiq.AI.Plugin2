---
name: maintain-implementation-journal
agent: monaiq
tier: 2
invoked-by:
  - getting-started
  - troubleshoot-integration
  - analyze-codebase
  - design-monetization
  - manage-catalog
  - profile-onboarding
  - scenario-advisor
  - implement-licensing
  - implement-feature
  - implement-purchase-flow
allowed-tools:
  - monaiq_journal
  - fetch_step_resources
  - mcp__plugin_monaiq_monaiq__monaiq_journal
  - mcp__plugin_monaiq_monaiq__fetch_step_resources
---

# Maintain Implementation Journal

<journal-contract>
Maintain a `.monaiq/` folder at the consumer project root while Monaiq skills guide implementation. The canonical layout is exactly `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/CHECKPOINTS/*.md`, and optional non-secret `.monaiq/config.json`. This is the optional non-secret .monaiq/config.json slot for local, non-credential configuration hints only.

Read `ROUTING-MAP.md` and `maintain-implementation-journal.md` as the first shared contract sources before a skill invents local route, checkpoint, or resume vocabulary. `ROUTING-MAP.md` owns shared journey vocabulary and route packet phase boundaries; `maintain-implementation-journal.md` owns checkpoint anatomy, approval/result recording, and journal state transitions.

Use `monaiq_journal` as the canonical write path. It returns `fileOperations` for the calling agent to apply locally under the consumer project root. Direct agent writes are not a replacement fallback when the hosted tool is unavailable or stale.

Record skill lifecycle, consequential decisions, todos, blockers, questions, validation failures, checkpoint prompts/results, errors, and code-change summaries. Decisions, todos, blockers, questions, validation failures, and code-change summaries use durable append-only IDs in `JOURNAL.md`. File-change summaries contain paths and intent only, not file contents.

Tool envelopes may include `journalReadyUpdates`. Treat them as proposed journal intents, not applied state. Apply supported intents by calling `monaiq_journal` with the specified action and safe summary data, then apply the returned file operations locally after path validation.
</journal-contract>

<operation-application-protocol>
Calling `monaiq_journal` is incomplete until the caller applies and verifies the returned operations. You must apply returned file operations locally under the consumer project root before claiming journal progress.

For every `monaiq_journal` response:
1. Inspect `fileOperations` before continuing.
2. You must validate each fileOperations path against the `.monaiq` allowlist: `.monaiq/`, `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/config.json`, `.monaiq/CHECKPOINTS`, and `.monaiq/CHECKPOINTS/*.md`.
3. Apply only supported operation kinds: `ensureDirectory`, `writeFile`, and `appendFile`.
4. Resolve paths under the consumer project root and never outside it.
5. You must verify .monaiq/STATE.md exists when state is initialized or updated.
6. You must verify .monaiq/JOURNAL.md exists when journal entries are initialized or appended.
7. You must verify .monaiq/CHECKPOINTS exists when init or checkpoint operations are returned.
8. You must verify .monaiq/CHECKPOINTS/*.md exists after a `save_checkpoint` call with a `result` payload.

Canonical checkpoint prompt call:

```json
{"checkpoint":"CHECKPOINT-PRE-CREDENTIAL-WRITE","summary":"Credential target selected.","question":"Approve the credential write plan?"}
```

Canonical approval recording call:

```json
{"checkpoint":"CHECKPOINT-PRE-CREDENTIAL-WRITE","summary":"Credential target selected.","question":"Approve the credential write plan?","result":{"answer":"approved"}}
```

Do not use checkpointName; the hosted parser reads checkpoint.
</operation-application-protocol>

<state-detection>
At Step 1 of every invoking skill, call `monaiq_journal get_state`. If no journal exists, call `monaiq_journal init`, apply the returned file operations, then call `skill_started`.

If state exists, read the bounded resume packet source. `STATE.md` includes current skill/step, scenario, route packet summary, last checkpoint, last activity, open todo/question/blocker/validation status, recent changed areas, and explicit next action. When `currentSkill` is set, summarize the current skill, current step, scenario, route packet summary, last checkpoint, open todos, open questions, validation status, blockers, recent changed areas, next action, and recent activity before continuing.
</state-detection>

<checkpoint-protocol>
When a step declares a `CHECKPOINT-*` checkpoint, call `monaiq_journal save_checkpoint` with the checkpoint name, summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options. The default sign-off choices are exactly: `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, `stop`. Present the returned envelope through the host-native ask mechanism, such as `vscode_askQuestions`, Claude `ask`, or Codex `AskUserQuestion`.

Checkpoint prompts must be complete before the user is asked to approve consequential work. Present the business-readable summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options; keep technical identifiers available in compact backing fields rather than making them the primary user-facing explanation.

Do not treat an envelope as approval. Approval exists only after the user responds and a follow-up `save_checkpoint` call records the `result` payload. Required checkpoint examples include `CHECKPOINT-WORKFLOW-START`, `CHECKPOINT-STATE-CONFLICT`, `CHECKPOINT-FRAMEWORK-CHOICE`, `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, `CHECKPOINT-PRE-CATALOG-MUTATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, `CHECKPOINT-FEATURE-SELECTION`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`, and `CHECKPOINT-SKILL-COMPLETE`.

Validation failures use a failure-specific checkpoint recording failure context, likely cause, remediation options, and selected retry/change/stop path before further catalog, code, config, or remediation work.

When validation fails after code/config work, call `monaiq_journal` action `record_validation_failure` with `validationCommand`, `observedResult`, `likelyCause`, `blockedNextAction`, `retryChoices`, and `checkpointRequired`. Pause before more edits until the remediation checkpoint is resolved.
</checkpoint-protocol>

<degraded-orchestration>
If the custom agent cannot be activated, the compatibility fallback still uses this protocol. If `monaiq_journal` is missing, stale, or unavailable, skills may continue read-only guidance but must halt before catalog mutation, code edits, credential/config writes, or validation remediation rather than writing `.monaiq` directly.
</degraded-orchestration>

<resume-and-reconciliation>
Backend wins for catalog, profile, license, billing, product, offering, and credential facts. Local wins for journey progress, implementation decisions, todos, questions, checkpoints, user preferences, and handoff notes.

When backend state and local journal state diverge, update the journal to match backend-canonical facts and raise `CHECKPOINT-STATE-CONFLICT` before proceeding. Never let `.monaiq` override backend truth.

backend/catalog changes after completed implementation can reopen journey work as a coherent next action while preserving completed discovery and approval history.
</resume-and-reconciliation>

<credential-safety>
`.monaiq/` is committed by default so teams and future agent sessions can understand the implementation journey. Opt-out via `.gitignore` is possible but is not the default.

Credential safety is prose-only in this phase. Do not journal ApiKeys, EncodedCredential strings, JWT values, Stripe keys, PEM blocks, private keys, license credentials, user secrets, `.env` contents, or secret-bearing file contents. Mention paths and decisions, never raw credential values. This phase does not implement a server-side credential leak filter.

Do not place raw ApiKeys, EncodedCredential values, JWTs, Stripe keys, or secret-bearing file contents in prompts, journals, checkpoint result payloads, plugin output, or generated guidance.
</credential-safety>

<file-operation-contract>
`monaiq_journal` is deterministic C# MCP/file-operation logic. It does not write arbitrary server filesystem paths, does not add a local transport, and does not operate outside `.monaiq/`. The calling agent applies returned operations locally after validating they target `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/CHECKPOINTS/*.md`, or `.monaiq/config.json`.

Do not add Semantic Kernel, Agent Framework, OpenAI, model calls, model-generated summarization, or hidden LLM orchestration inside `monaiq_journal`. The outer host LLM interprets tool responses.
</file-operation-contract>

<handoff-contract>
Before a skill exits, call `record_file_changes` with changed paths only, record any unresolved todos or questions, call `save_checkpoint` for `CHECKPOINT-SKILL-COMPLETE` when meaningful, then call `skill_completed`. If routing to another skill, the next skill calls `get_state` and `skill_started`; do not duplicate that specialist skill's protocol or implementation instructions.
</handoff-contract>