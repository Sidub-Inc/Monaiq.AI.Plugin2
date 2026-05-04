# Checkpoint Workflow

Use this workflow for every hard checkpoint.

1. **Prompt:** call `monaiq_journal save_checkpoint` with `checkpoint`, `summary`, `question`, intended change, user/business impact, affected entities/files/areas, secret policy, risks, validation plan, and safe context. This prompt call returns no file operations.
2. **Ask:** present the checkpoint through the host-native ask mechanism with concise options from `_shared/gate-prompts.md`.
3. **Approval + write:** after the user answers, call `monaiq_journal save_checkpoint` again with the same `checkpoint`, `summary`, `question`, and a `result` payload such as `{ "answer": "approved" }`.
4. **Apply:** apply returned `.monaiq/*` file operations locally after validating each path against the `.monaiq` allowlist.
5. **Verify:** verify the expected `.monaiq/CHECKPOINTS/*` file exists before claiming the checkpoint is recorded.

Use `checkpoint` in tool payloads. Do not use `checkpointName` except as route-packet display metadata.

Failure to complete prompt, ask, approval write, operation application, and local verification means the checkpoint has not been durably recorded. Do not continue into catalog mutation, credential/config writes, application behavior edits, purchase-flow changes, or validation remediation until the checkpoint is recorded or the user selects a stop/revise option.

Each checkpoint prompt must include: summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options. Default options are `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, and `stop`.

Hard checkpoints include `CHECKPOINT-WORKFLOW-START`, `CHECKPOINT-PRE-CATALOG-MUTATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, and `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`.