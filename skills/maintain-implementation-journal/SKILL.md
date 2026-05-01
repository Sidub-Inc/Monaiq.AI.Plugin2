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
Maintain a `.monaiq/` folder at the consumer project root while Monaiq skills guide implementation. The canonical files are `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/CHECKPOINTS/*.md`, and optional `.monaiq/config.json`.

Use `monaiq_journal` as the canonical write path. It returns `fileOperations` for the calling agent to apply locally under the consumer project root. Direct agent writes are discouraged but tolerated for narrative gaps the action enum does not yet cover.

Record skill lifecycle, consequential decisions, todos, questions, checkpoint prompts/results, errors, and file-change summaries. File-change summaries contain paths and intent only, not file contents.
</journal-contract>

<state-detection>
At Step 1 of every invoking skill, call `monaiq_journal get_state`. If no journal exists, call `monaiq_journal init`, apply the returned file operations, then call `skill_started`.

If state exists, read the bounded resume packet. When `currentSkill` is set, summarize the current skill, current step, scenario, last checkpoint, open todos, open questions, and recent activity before continuing.
</state-detection>

<checkpoint-protocol>
When a step declares a `CHECKPOINT-*` checkpoint, call `monaiq_journal save_checkpoint` with the checkpoint name, summary, question, and options. Present the returned envelope through the host-native ask mechanism, such as `vscode_askQuestions`, Claude `ask`, or Codex `AskUserQuestion`.

Do not treat an envelope as approval. Approval exists only after the user responds and a follow-up `save_checkpoint` call records the `result` payload. Required checkpoint examples include `CHECKPOINT-STATE-CONFLICT`, `CHECKPOINT-FRAMEWORK-CHOICE`, `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, `CHECKPOINT-FEATURE-SELECTION`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`, and `CHECKPOINT-SKILL-COMPLETE`.
</checkpoint-protocol>

<resume-and-reconciliation>
Backend wins for catalog, profile, license, billing, product, offering, and credential facts. Local wins for journey progress, implementation decisions, todos, questions, checkpoints, user preferences, and handoff notes.

When backend state and local journal state diverge, update the journal to match backend-canonical facts and raise `CHECKPOINT-STATE-CONFLICT` before proceeding. Never let `.monaiq` override backend truth.
</resume-and-reconciliation>

<credential-safety>
`.monaiq/` is committed by default so teams and future agent sessions can understand the implementation journey. Opt-out via `.gitignore` is possible but is not the default.

Credential safety is prose-only in this phase. Do not journal ApiKeys, EncodedCredential strings, JWT values, Stripe keys, PEM blocks, private keys, license credentials, user secrets, `.env` contents, or secret-bearing file contents. Mention paths and decisions, never raw credential values. This phase does not implement a server-side credential leak filter.
</credential-safety>

<file-operation-contract>
`monaiq_journal` is deterministic C# MCP/file-operation logic. It does not write arbitrary server filesystem paths, does not add a local transport, and does not operate outside `.monaiq/`. The calling agent applies returned operations locally after validating they target `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/CHECKPOINTS/*.md`, or `.monaiq/config.json`.

Do not add Semantic Kernel, Agent Framework, OpenAI, model calls, model-generated summarization, or hidden LLM orchestration inside `monaiq_journal`. The outer host LLM interprets tool responses.
</file-operation-contract>

<handoff-contract>
Before a skill exits, call `record_file_changes` with changed paths only, record any unresolved todos or questions, call `save_checkpoint` for `CHECKPOINT-SKILL-COMPLETE` when meaningful, then call `skill_completed`. If routing to another skill, the next skill calls `get_state` and `skill_started`; do not duplicate that specialist skill's protocol or implementation instructions.
</handoff-contract>