# Checkpoint Workflow

Use this workflow for every hard checkpoint. This shared source file is assembled into the multi-target Monaiq plugin output for Claude, GitHub Copilot / VS Code, and Codex, so the host-specific ask behavior must be explicit and portable.

1. **Prompt:** call `monaiq_journal save_checkpoint` with `checkpoint`, `summary`, `question`, intended change, user/business impact, affected entities/files/areas, secret policy, risks, validation plan, and safe context. This prompt call returns no file operations.
2. **Ask:** present the checkpoint through the target host's blocking ask/approval mechanism with concise options from `_shared/gate-prompts.md`. In GitHub Copilot / VS Code, use `vscode_askQuestions`. In Claude and Codex runtimes, use the available blocking approval, confirmation, or question mechanism; if no structured ask tool is exposed, ask one blocking question in chat, require an explicit option/value answer, and do not continue until the answer is recorded in the next `monaiq_journal save_checkpoint` call.
3. **Approval + write:** after the user answers, call `monaiq_journal save_checkpoint` again with the same `checkpoint`, `summary`, `question`, and a `result` payload such as `{ "answer": "approved" }`.
4. **Apply:** apply returned `.monaiq/*` file operations locally after validating each path against the `.monaiq` allowlist.
5. **Verify:** verify the expected `.monaiq/CHECKPOINTS/*` file exists before claiming the checkpoint is recorded.

Use `checkpoint` in tool payloads. Do not use `checkpointName` except as route-packet display metadata.

Failure to complete prompt, ask, approval write, operation application, and local verification means the checkpoint has not been durably recorded. Do not continue into catalog mutation, credential/config writes, application behavior edits, purchase-flow changes, or validation remediation until the checkpoint is recorded or the user selects a stop/revise option.

Do not infer hard approval from casual chat such as "looks good", "i like it", "sure", or similar conversational agreement. Catalog mutation, code/config edits, credential writes, platform-scope expansion, checkout behavior, and validation remediation require an explicit host-native checkpoint answer and a durable checkpoint result.

Each checkpoint prompt must include: summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options. Default options are `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, and `stop`.

Hard checkpoints include `CHECKPOINT-WORKFLOW-START`, `CHECKPOINT-PRE-CATALOG-MUTATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, and `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`.