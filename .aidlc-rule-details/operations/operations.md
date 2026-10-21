# Operations

## Purpose
**Pre-merge automated review, plus a placeholder for future deployment/monitoring workflows**

The Operations phase currently has active stages - Merge Request Package, Automated Code Review and Release Readiness - and otherwise remains a placeholder for future expansion (deployment planning, monitoring/observability, incident response, maintenance workflows, production readiness checklists).

---

# Merge Request Package (before any push)

## Purpose
**Produce the MR title and MR comment for human review BEFORE the branch is pushed to remote**

## Prerequisites
- Build and Test steps 1-8 complete: results (even if `local-only`) are available. This step runs as Build and Test Step 8.5, before its approval prompt
- Gate G0 satisfied: ticket key recorded in `aidlc-state.md`

## Operations stage order
1. Merge Request Package (this step; approved text before any push)
2. Automated Code Review (against the branch range when no MR exists yet; Critical/High fixes that change code require regenerating and re-approving the MR text)
3. Release Readiness
4. User pushes, opens the MR with the approved text and supplies the CI run URL; then re-run Release Readiness

## Steps
1. Create `aidlc-docs/operations/merge-request/{ticket-key}.md` containing the MR title and MR comment below. Generate from the actual diff (`git diff <base>...HEAD`), the Build and Test summary and `aidlc-state.md`. Never invent files, results or tickets.
2. **MR title** format exactly: `[<ticket identifier>]: <title/summary>` (e.g. `[IDV-1234]: Add validation for vault create request`). Use the verified or user-asserted ticket key; use a short imperative summary; for multi-line targets create one title/comment per MR/target line.
3. **MR comment** format exactly (headings and order unchanged; use `Not run`, `No` or `Not performed` rather than omitting items):

```markdown
## What changed
[1-3 sentences describing the change]

## Why
[Problem or requirement, tied to the ticket; root-cause evidence status if a fix]

## Files touched
- [path from the actual diff, one per line, excluding generated/lock files only if noted]

## Risk level
- [Low | Medium | High] (one-line justification)

## Tests run
- Unit tests: [what was added/run and result, or Not run]
- Integration tests: [result or Not run]
- Manual checks: [what was checked, or Not performed]

## Security considerations
- Secrets changed? [Yes/No]
- Auth/authz changed? [Yes/No]
- Input validation changed? [Yes/No]
- Sensitive data exposed? [Yes/No]

## Deployment notes
- DB migration: [Yes/No (reference migration file)]
- Config change: [Yes/No (reference config ticket if Yes)]
- Rollback needed: [Yes/No (reference ROLLBACK.md if Yes)]

## Reviewer notes
[Specific things a reviewer should check; open risks; pipeline-run status such as `pipeline-run: NONE - awaiting push/MR`; known gaps]
```

4. Consistency rules (check before presenting):
   - Files touched equals the actual diff file list as of MR approval, excluding `aidlc-docs/operations/**`, `aidlc-docs/audit.md` and `aidlc-docs/aidlc-state.md` (these change after approval); state this exclusion in Reviewer notes only if `aidlc-docs/` is committed. Check `git diff --name-only` for generated files (e.g. `__pycache__`, build output) before listing; flag them instead of listing them
   - Test lines match the Build and Test summary and name the exact command used; `local-only evidence` is stated as such; never claim pipeline-tested without a user-supplied run reference
   - A migration or config change in the diff must show `Yes` under Deployment notes, with the config ticket key for config changes
   - Risk level is at least Medium for migrations, auth changes, input-validation changes or multi-line backports
   - No secrets, customer names or private data in the text
5. Present the title and comment to the user in chat (copyable block) and as the saved file. Wait for explicit approval of the text and record it (`Approved by human: yes, <date>` in the MR file and a quoted line in `audit.md`; see Consistency and Approval Rules in `common/release-governance.md`). **Do not push, and do not tell the user the push is ready, until this approval is recorded.** After approval, remind the user: push the ticket-keyed branch, open the MR with this title/comment, supply the CI run URL.
6. Log the presented text and the user's raw response in audit.md; mark the step complete in aidlc-state.md.

If code changes after the MR package is approved, regenerate it and obtain approval again.

---

# Release Readiness

## Purpose
**Prove a change is safe to deliver through the standard pipeline and MyVault, with traceability**

## Prerequisites
- Build and Test complete; Automated Code Review verdict recorded (or documented as not applicable because no MR exists yet, which makes item 4 `Not run`)
- Order: Automated Code Review first, then Release Readiness

Execute for any change that can reach a customer. Load `common/release-governance.md` (Gate G4).

## Steps
1. Create `aidlc-docs/operations/release-readiness/{ticket-key}.md` using the Release Readiness template in `common/collaboration-templates.md`
2. Fill each of the 11 checklist items with Pass / Fail / N/A (reason) / Not run and a link or reference as evidence. Never invent pipeline run IDs, ticket contents or test results; verify the Jira ticket via the Atlassian tools when available
3. Cross-check that branch name, commits and MR reference the same ticket key
4. Overall status is **NOT RELEASABLE** if any item is Fail or Not run. State this plainly with the list of open items
   - For every item resting on `user-asserted` evidence (ticket validity, pipeline run, shell-written changes), restate the matching item from User Responsibilities in `common/release-governance.md` so the user knows the attestation is theirs
5. Present the result with: "**Release readiness: <RELEASABLE-PENDING-HUMAN-APPROVAL | NOT RELEASABLE>. Ready to proceed?**" and wait for explicit approval
6. Log in audit.md, update Change Governance gates in aidlc-state.md (G4)

Delivery to customers happens only via MyVault by the release process; this stage never produces out-of-band delivery steps.

---

# Automated Code Review

Runs BEFORE Release Readiness.

## Purpose
**Apply the organization's DevSec CI Pipeline ruleset to the MR/PR diff before merge**

This stage replicates the same review the live CI pipeline (`claude_server.py` / `pipeline.py` / `rules.json`, DevSec CI Pipeline) performs automatically on every MR, so issues can be caught and fixed in-session rather than only discovered after a push triggers the real pipeline.

## Prerequisites
- Build and Test stage is complete
- A merge request (or pull request) exists for this work, OR the user has pointed at a specific branch/commit range to review
- The ruleset is available at `rules.json` in this same directory (`.aidlc-rule-details/operations/rules.json`) (25 rules, IDs R001-R025, with severities Critical/High/Medium/Low and categories Security/Code Quality/Performance/Style), plus its `general_instructions` array

## Overview
Diff the branch against its target (same convention as `/code-review`: three-dot diff against the merge-base, e.g. `git diff origin/<target>...HEAD`), exclude the same file types the live pipeline skips, and apply every rule plus the general instructions to what remains. Produce findings in the exact output format the pipeline's Claude Code Review step uses, so the output is a faithful preview of what the real CI comment would say.

## Steps to Execute

### Step 1: Resolve the Diff Scope
- Identify source branch/commit and target branch (ask the user if not already known from context)
- Confirm both refs resolve and the diff is non-empty before proceeding
- Exclude files matching the pipeline's `SKIP_EXTENSIONS`: `package-lock.json`, `.csproj`, `.sln`, `.min.js`, `.min.css`, `.map`, `.svg`, `.png`, `.jpg`, `.jpeg`, `.gif`, `.ico`, `.woff`, `.woff2`, `.ttf`, `.eot`

### Step 2: Load the Ruleset
- Read `rules.json`'s `rules` array (R001-R025) and `general_instructions` array in full
- Do not paraphrase or summarize the rules - apply them as written, including each rule's `patterns` and `example`

### Step 3: Apply Rules to the Diff
For each reviewable file's diff:
- Check every rule R001-R025 against both added and removed lines (removed lines matter too - per `general_instructions`, a historical security issue revealed by a removed line may still exist in git history)
- Follow every general instruction literally, in particular:
  - Don't flag a pattern match alone without considering full context
  - Don't flag test files/mock data with the same severity as production code
  - When in doubt, downgrade severity - prefer accuracy over noise
  - Credentials/sensitive data as URL query parameters is always Critical
  - Skip a file entirely if its change is trivial or harmless

### Step 4: Produce Findings in the Pipeline's Own Format
For every finding, emit one block:
```
ISSUE_START
File: <exact filename>
Line: <integer>
Severity: <Critical|High|Medium|Low>
Category: <Security|Code Quality|Performance|Style>
Issue: <description>
Suggestion: <concrete fix>
ISSUE_END
```
After all findings, emit exactly one summary block:
```
SUMMARY_START
Verdict: <Approve|Request Changes|Needs Discussion>
Summary: <2-3 sentences>
SUMMARY_END
```
Verdict rules (matching the pipeline's own aggregation logic):
- **Request Changes** - any Critical or High severity finding, or any finding that represents a real exploitable issue
- **Needs Discussion** - only Medium/Low findings, or genuine ambiguity a human should weigh in on
- **Approve** - no findings, or only trivial/non-blocking Low findings explicitly called out as such

### Step 5: Fix Blocking Findings
- If the verdict is **Request Changes**, OR any individual finding is Critical/High:
  - Fix the issue directly in code (do not just report it and stop)
  - Add or update tests covering the fix where the codebase's testing conventions call for it
  - Rerun the full build and test suite
  - Re-run Steps 1-4 against the updated diff before presenting completion, so the user sees a clean (or at least improved) verdict
- Medium/Low findings that are genuinely non-blocking (per the ruleset's own severity guide) may be presented without a mandatory fix - note them in the summary and let the user decide

### Step 6: Present Completion Message
- Present findings and the final verdict to the user, in the same `ISSUE_START`/`SUMMARY_START` format
- If any findings were fixed per Step 5, say so explicitly (what was fixed, what tests were added/rerun) rather than only showing the final clean state
- End with:

```markdown
**Automated code review complete (Verdict: <verdict>). Ready to proceed?**
```

### Step 7: Wait for Explicit Approval
- Do not proceed until the user explicitly confirms
- If the user disagrees with a finding or severity, discuss and adjust rather than silently re-asserting the ruleset's output

### Step 8: Record Approval and Update Progress
- Log the full findings and verdict in audit.md with timestamp
- Record the user's response with timestamp
- Save findings to `aidlc-docs/operations/code-review/{mr-or-branch-name}-findings.md`
- Mark Automated Code Review stage complete in aidlc-state.md

## Notes
- This stage is a local preview/fix loop, not a replacement for the live CI pipeline - the real `pipeline.py` still runs on push and is the actual merge gate
- `rules.json` is the single source of truth for what counts as a finding here; if the live pipeline's ruleset changes, re-sync this stage's copy of `rules.json` rather than hand-editing divergent rules into this file
