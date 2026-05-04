# Completion Workflow

Before a skill exits or hands off to another skill:

1. Call `monaiq_journal record_file_changes` with changed paths and intent only. Do not include file contents or secret values.
2. Call `monaiq_journal update_checklist_progress` only for gates owned by the current skill, and only after source skill, relevant MCP tool, canonical resources, and checkpoint/journal evidence are recorded.
3. Save `CHECKPOINT-SKILL-COMPLETE` when the user-visible journey state changed or a downstream skill needs durable proof.
4. Include a no-secret `proofOfDone` packet in `CHECKPOINT-SKILL-COMPLETE`.
5. Apply returned file operations locally and verify expected `.monaiq` files.
6. Call `monaiq_journal skill_completed`.
7. Recommend the next specialist skill or stop with unresolved risks.

## proofOfDone Schema

```json
{
  "backendReadBack": ["backend facts verified through MCP tools"],
  "filesChanged": ["workspace-relative paths affected"],
  "commandsRun": ["build/test/check commands and results"],
  "behaviorVerified": ["allow/denied/expired/over-limit/checkout states checked"],
  "unverifiedRisks": ["what remains untested or deferred"],
  "nextRecommendedSkill": "skill name or null"
}
```

`manage-catalog` proof must include read-back verification. Code skills must include build/test commands and behavior states checked.