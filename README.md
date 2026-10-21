# AI-DLC Claude Code collaboration integration

This repository represents the **global Claude Code configuration**. Its tracked files are intentionally a small allowlist; do not add or commit runtime history, credentials or `settings.json`.

## Installation / activation

- **Global (this repository):** install `CLAUDE.md` and `.aidlc-rule-details/` together in `~/.claude/`. On Windows the default home is `%USERPROFILE%\.claude`. Do not replace existing private runtime files. `AGENTS.md` is the equivalent entrypoint for compatible agents but is not a replacement for `CLAUDE.md` in Claude Code.
- **Project-local:** alternatively put `CLAUDE.md` and `.aidlc-rule-details/` together at the project root. The Claude entrypoint searches project-local rules first and the global rules as a fallback. For other supported tool layouts refer to its path list.
- **Run Claude from the actual project working directory**, not the `.claude` config directory. Project artifacts are generated under `<project>/aidlc-docs/`.
- `CLAUDE.md` must actually be loaded by your client/session. If the rules are not picked up, explicitly ask Claude to read it and check the resolved rule path.

## Lifecycle hook (optional, read-only)

`scripts/aidlc-session-reminder.py` inspects the current project and prints a reminder on session start or context compaction. It **does not create, edit or approve files**, and never substitutes for task/hand-over checkpoints. It accepts hook input on stdin and uses its `cwd` when supplied. It emits only short paths/status; no private file content.

To opt in, merge the entries in `templates/claude-settings-hooks.example.json` into your **existing local** `~/.claude/settings.json` (do not overwrite that file). The sample assumes a global installation and the `python` command is on PATH; on Windows use an appropriate Python launcher if needed. Claude Code hook schemas can change, so validate the resulting settings against your installed Claude Code version. No hooks are installed by this patch.

## Pickup and acceptance test

1. In a disposable project, start Claude Code, ask it to identify which AI-DLC rule directory was loaded and which project is active.
2. Begin a small feature and inspect its plan and required approval. Check the active-task record and `aidlc-docs/aidlc-state.md` without assuming every template must be created.
3. Make one approved edit, run a test and inspect the real diff. Require explicit `passed`, `failed` or `not run` evidence.
4. Checkpoint `aidlc-docs/collaboration/HANDOVER.md`; exit, then start a **fresh Claude session** from the same project.
5. Verify that Claude recovers the actual stage, task, blockers and next step, reconciles them with the repository and requests any pending approval.
6. Interrupt a subsequent task midway. Verify the next session does not incorrectly mark it complete. Lifecycle reminders alone do not prove handover persistence.

For validation without Claude, `python scripts/aidlc-session-reminder.py --self-test` runs a limited no-write smoke check. Full Claude runtime behavior requires an actual two-session test.

## Release governance

`.aidlc-rule-details/common/release-governance.md` encodes the company change-control policy as gates G0-G4: Jira ticket required before code (G0), change classification and config-vs-code decision (G1), implementation constraints (G2), full-pipeline test evidence (G3), and a Release Readiness checklist before merge (G4). Delivery is MyVault only; urgency never removes a gate.

Optional enforcement: `scripts/aidlc-governance-guard.py` (PreToolUse hook, blocks application-code edits until a Jira key is recorded; entry in `templates/claude-settings-hooks.example.json`) and `templates/git-hooks/commit-msg` (requires a ticket key in commit messages). Run `python scripts/aidlc-governance-guard.py --self-test` to smoke-test.

Strict mode (optional): set the environment variable `AIDLC_STRICT=1` and the guard also blocks source edits in any git repository that has no `aidlc-docs/aidlc-state.md`. It is off by default so unrelated repositories are unaffected.
# Modded-AWS-AIDLC
