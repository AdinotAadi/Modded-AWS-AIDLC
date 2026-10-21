# Collaboration Traceability and Human Review

This rule extends AI-DLC, not replaces it. Existing stage execution, approval gates, extension choices, plan checkbox updates, and `aidlc-docs/audit.md` remain authoritative. These controls apply proportionately to the task's impact.

## Project-local records
All generated documentation belongs in `<workspace>/aidlc-docs/`, never the global `.claude` folder. Prefer existing AI-DLC reverse-engineering/architecture documents and link to them rather than duplicating. Maintain the following where relevant:

- `collaboration/HANDOVER.md`: current state, status, active changes, problems, next action; update at session end and before context handoff.
- `collaboration/DECISIONS.md`: meaningful accepted decisions, alternatives, rationale, tradeoffs, impacts, decision-maker, date, and model/tool if known. Never present AI suggestions as human-approved decisions.
- `collaboration/FLOW.md`: actual entry points, function/module call chain, state/data flow, impacted branches and errors, with source paths; distinguish verified paths from hypotheses.
- `collaboration/CONSTRAINTS.md`: project-specific guardrails approved by the user; do not invent them or silently weaken them.
- `collaboration/TEST-CHECKLIST.md`: runnable commands, scenarios, expected outcomes, actual outcomes and evidence. Unexecuted tests must remain unverified.
- `collaboration/ROLLBACK.md`: rollback steps/checkpoints and data migration recovery for medium/high-risk tasks; do not presume reverting code reverses persistent state.
- `collaboration/tasks/BUG-<slug>.md` or `FEATURE-<slug>.md`: symptom or requirement, scope, evidence, hypotheses, implementation attempts, affected files, test results, open questions and next actions. Use one record per meaningful task rather than one mutable global bug file.

Do not create every file blindly. For minor edits, a short task note/checklist within existing AI-DLC artifacts may suffice; for complex or risky work, create the relevant files. Existing `aidlc-state.md` remains the stage status source of truth, while `HANDOVER.md` is a concise narrative pointer to it. Existing `audit.md` remains the mandatory raw interaction log under the original AI-DLC rules; do not copy full prompts into handovers/decisions.

## At session start / resumption
1. Follow original AI-DLC workspace detection and stage-resumption instructions.
2. Read `aidlc-state.md`, existing handover and active task record where present; consult architecture, constraints, decisions, and flow only to the depth needed.
3. State confirmed facts vs unknowns; do not assert that documentation proves current code behavior without checking relevant source.
4. Confirm scope and applicable approval gate before mutating code.

## Before implementation
1. Identify entry points, call path, affected files, data/security implications and adjacent behavior; document verified flow for cross-file/risky changes.
2. Propose an understandable plan and reasons, alternatives when material, scope, tests and rollback considerations.
3. Follow all existing AI-DLC required approvals. Approval to proceed is not blanket permission for unrelated modifications or destructive commands.
4. Keep each implementation step logically bounded and mark existing plan checkboxes in the same interaction.

## During implementation
- Change only the approved scope. Ask before adding dependencies, migrations, destructive actions or unrelated refactors unless already authorized.
- Add comments to non-obvious intent, invariants, callers or lifecycle behavior; avoid line-by-line narration.
- Track major attempts, failures and decisions without falsely claiming verification.
- Do not overwrite or undo user changes; inspect workspace changes before writing.

## Verification and review
- Inspect the **actual diff** (including unexpected and generated changes) before claiming completion; summary alone is insufficient.
- Execute applicable checks and record command, result and limitations; distinguish not-run, failed and passed.
- Validate changed behavior and adjacent regression paths; use risk-appropriate security and failure-case tests.
- For risky changes, verify rollback instructions and record prerequisites; do not execute rollback without appropriate authorization.
- Explain final code behavior sufficiently for human review. Human approval remains essential even with passing checks.

## End of session
Update the short handover: objective, completed work, pending work, risks/blockers, next step, plus links to task/decision/flow records. Include tool/model/version only when reliably available; write `unknown` otherwise. Do not represent a planned step as completed. Keep project records free of secrets; follow organizational data-handling policy, especially for the original AI-DLC raw audit logs.

## Precedence / conflicts
Follow higher-priority instructions and explicit user authorization. Within AI-DLC, original stage/approval/audit/extension requirements remain in place; this file adds supporting traceability/review only. If project constraints conflict with a plan, stop and request clarification. Never silently substitute this handover for `aidlc-state.md` or a task record for `audit.md`.

## Operational checkpoints and first-use initialization
- **First use:** If `aidlc-docs/aidlc-state.md` exists but collaboration records do not, initialize `collaboration/HANDOVER.md` using `common/collaboration-templates.md` with only facts verified from existing state, plans and repository. Use `unknown` or `not run` for missing evidence. Avoid creating placeholder task, decision, flow, and rollback files until applicable.
- **Before changes:** Inspect `git status --short` and the relevant source paths; record the active unit/task and approval status. Do not overwrite existing user work.
- **During work:** Update task notes at meaningful milestones (after a tested edit, before an approval gate, and before a long context switch), not only at session termination. Record commits/paths when available, commands executed, test results and the next executable step.
- **Before completion:** Reconcile `HANDOVER.md` with `aidlc-state.md` and plan checkboxes. Tests and review evidence must be recorded as passed/failed/not run. Do not complete a stage on documentation alone.
- **If interrupted:** The next session must verify the last recorded status against the live workspace, identify incomplete work and ask for approval where required. Lifecycle hooks may surface reminders but cannot safely guarantee writes during termination.
- **Privacy:** Never write secrets, full credentials, access tokens or copied private prompts into collaboration records or hook output.

## Lightweight completion evidence
For a code-changing task, verify and record: (1) approved scope, (2) actual changed-file diff, (3) runnable tests with result or explicit not-run reason, (4) adjacent regression and security implications, (5) rollback considerations when warranted, (6) next step/owner, and (7) consistent AI-DLC state, active task, and handover. If any required item is missing, report the task as unverified/incomplete rather than implicitly passing it.
