# Session Continuity Templates

## Welcome Back Prompt Template
When a user returns to continue work on an existing AI-DLC project, present this prompt:

```markdown
**Welcome back! I can see you have an existing AI-DLC project in progress.**

Based on your aidlc-state.md, here's your current status:
- **Project**: [project-name]
- **Current Phase**: [INCEPTION/CONSTRUCTION/OPERATIONS]
- **Current Stage**: [Stage Name]
- **Last Completed**: [Last completed step]
- **Next Step**: [Next step to work on]

**What would you like to work on today?**

A) Continue where you left off ([Next step description])

B) Review a previous stage ([Show available stages])

[Answer]:
```

## MANDATORY: Session Continuity Instructions
1. **Always read aidlc-state.md first** when detecting existing project
2. **Parse current status** from the workflow file to populate the prompt
3. **MANDATORY: Load Previous Stage Artifacts** - Before resuming any stage, automatically read all relevant artifacts from previous stages:
   - **Reverse Engineering**: Read architecture.md, code-structure.md, api-documentation.md
   - **Requirements Analysis**: Read requirements.md, requirement-verification-questions.md
   - **User Stories**: Read stories.md, personas.md, story-generation-plan.md
   - **Application Design**: Read application-design artifacts (components.md, component-methods.md, services.md)
   - **Design (Units)**: Read unit-of-work.md, unit-of-work-dependency.md, unit-of-work-story-map.md
   - **Per-Unit Design**: Per-unit artifacts live under `aidlc-docs/construction/{unit-name}/` in
     `functional-design/`, `nfr-requirements/`, `nfr-design/`, and `infrastructure-design/`
     subdirectories. On resume, determine the in-progress unit from `aidlc-state.md` and load that
     unit's design artifacts, plus the design artifacts of any units it depends on (per
     `unit-of-work-dependency.md`). The exact files in each subdirectory are enumerated by the
     corresponding construction stage rules.
   - **Code Stages**: Read all code files, plans, AND all previous artifacts
4. **Smart Context Loading by Stage**:
   - **Early Stages (Workspace Detection, Reverse Engineering)**: Load workspace analysis
   - **Requirements/Stories**: Load reverse engineering + requirements artifacts
   - **Design Stages**: Load requirements + stories + architecture + design artifacts
   - **Code Stages**: Load ALL artifacts + existing code files
5. **Adapt options** based on architectural choice and current phase
6. **Show specific next steps** rather than generic descriptions
7. **Log the continuity prompt** in audit.md with timestamp
8. **Context Summary**: After loading artifacts, provide brief summary of what was loaded for user awareness
9. **Asking questions**: ALWAYS ask clarification or user feedback questions by placing them in .md files. DO NOT place the multiple-choice questions in-line in the chat session.

## Error Handling
If artifacts are missing or corrupted during session resumption, see [error-handling.md](error-handling.md) for guidance on recovery procedures.
## Collaboration pickup checkpoint (additive)
After reading `aidlc-state.md` and before offering next steps:
1. If present, read `aidlc-docs/collaboration/HANDOVER.md`, then its linked active task note and any relevant decision/flow/constraint records. If absent, treat the project as not yet initialized for collaboration; do not invent prior history.
1a. **Release governance resume check**: read `## Change Governance` in `aidlc-state.md`. Restate to the user the ticket key(s) and verification flag, evidence status, backport table, pipeline reference and gates G0-G4 including any `Deviation` or `unverified` flag. Block application-code edits unless G0 is satisfied (Pass, Open with a user-asserted key, or a permitted non-customer-facing deviation) AND G1 is satisfied (Pass, N/A for non-fix changes, or a logged exception). See the Gate Status Table in `common/release-governance.md`.
1b. On resume, also check: the MR file records `Approved by human: yes` before any push is considered; edits after the last reviewed commit are flagged as outside the MR/review scope; the pipeline reference is identical in every document; `Current Stage` and stage ticks match the real state. Report each mismatch.
2. Cross-check the reported stage and pending steps against `aidlc-state.md`, stage plans, repository status/diff and relevant code. The state file and approvals take precedence over narrative handovers.
3. Report the last *verified* result, outstanding approvals, blockers, and exact next safe action. If a check was not executed, report `not run` rather than success.
4. If work is resumed after an interrupted session, record the interruption and uncertainties in the task record before performing changes; do not retroactively mark work complete.
5. Before a planned context switch or compact, checkpoint the current active task and handover, without modifying the approval gate or implying human approval.
