# Release Governance (Mandatory Change Control)

**Purpose**: Enforce the company policy that every change reaching a customer goes through the standard development / test / release pipeline, is tracked by a Jira ticket, and is delivered only through MyVault. This rule is additive: it does not replace any AI-DLC stage, approval gate, `aidlc-state.md`, or `audit.md` requirement. It is a hard constraint and **cannot be waived by the AI**; only the user may record a documented exception, and even then the AI must flag it as a policy deviation.

## Policy Principles (source: engineering leadership directive)
1. **No bypass**: no code change reaches a customer outside the standard dev / test / release pipeline.
2. **MyVault is the only delivery channel**: fixes are delivered to customers only via a MyVault download. Never produce, suggest or script ad-hoc patches, hand-copied binaries/DLLs, live edits, or "try this on the customer environment" steps as a delivery mechanism.
3. **Every code change has a Jira ticket**: the ticket tracks problem description, the fix, and the code lines (branches / release lines) it ships in.
4. **Every customer config change has a Jira ticket**: configuration is NOT a temporary workaround. Each configuration variant creates a separate code path that Arcon owns.
5. **Config that should be in code goes through the pipeline**: code review, test, packaging, release.
6. **On-Prem-specific config needs a pre-deployment health check**: the AI produces a hand-off artifact (`aidlc-docs/operations/health-check/{ticket}.md`, template in `common/collaboration-templates.md`) for the owner team (Milan's team) so affected customers are detected before deployment. The AI cannot contact the team and never marks the check done; status stays `Pending owner ack` until a human records the acknowledgement.
7. **No untested fixes**: no quick patch without full-pipeline testing. Iterating on a customer environment is prohibited.
8. **No uninvestigated commitments**: do not draft customer-facing commitments (dates, fixes, workarounds) without recorded investigation and analysis.

## Gate G0: Change Ticket Gate (Workspace Detection, before any code or design work)
Before Requirements Analysis begins, establish and record in `aidlc-state.md` under `## Change Governance`:

| Field | Requirement |
|---|---|
| Jira ticket key(s) | Required (e.g. `PROJ-1234`). Ask the user if not supplied. Never invent or guess a key. |
| Change type | `feature` / `bug-fix` / `customer-escalation-fix` / `config-change` / `refactor-or-internal` |
| Target code line(s) / release(s) | Required for any change that can reach a customer. If unknown, record `unknown` and ask. |
| Customer-facing? | Yes / No / Unknown |
| On-Prem specific? | Yes / No / Unknown |
| Origin | New work / customer escalation (record escalation reference, not customer secrets) |

**If the user already supplied these fields in the request, do not ask again.** Otherwise ask via a question file (`aidlc-docs/inception/requirements/governance-questions.md`), per `common/question-format-guide.md`, not in chat.

**If no ticket key is provided**: STOP. Ask the user for the key (or for confirmation that a ticket is being created). Planning, read-only investigation and requirements gathering may continue, but **Code Generation and any file edit to application code are blocked** until a key is recorded. "Read-only" means no writes under the source tree: scratch reproductions go under `aidlc-docs/` or a temp directory.

**No-ticket deviation (restricted)**:
- A deviation is allowed ONLY when the change is not customer-facing AND not a customer escalation. The user must state it, the raw wording is recorded in `audit.md`, G0 is set to `Deviation`, and the AI states plainly that this violates policy.
- For customer-facing or escalation changes there is NO deviation path: Code Generation stays blocked and the AI may only produce analysis, investigation notes and a status summary.
- A deviation never authorizes commits to a release branch, pushing, packaging, merge, or presenting the work as releasable. Use a local-only branch `wip/NO-TICKET-<slug>`.
- When a real ticket key is later supplied: log the earlier deviation in `audit.md`, re-run G0, rename the branch, amend local commits to carry the key, and re-review all earlier changes before any push.

**Jira tooling**: if an Atlassian/Jira integration is available, make ONE read-only lookup (resolving the cloud ID is part of that lookup) to verify the key exists and capture its summary; never create, transition or comment on tickets without user authorization. If not available, record the key as `user-asserted, unverified`. If the lookup returns "not found", G0 stays `Open`, the key is NOT recorded as valid, and the AI tells the user; the user may override only by stating that the key is correct but not visible to the connector (or is a test placeholder), after which it is recorded as `user-asserted, unverified` with that caveat. An unverified key keeps G0 = `Open` (code generation may proceed on a user-asserted key, but G0 cannot become Pass), and G4 items 1 and 2 are `Not run` until a human confirms the ticket.

**Multi-line targets**: when more than one code line is targeted, ask whether the model is one umbrella ticket with per-line subtasks or one ticket per line. Record a table in `## Change Governance`: `line -> ticket -> branch -> base ref -> status`. Branch from the oldest supported target line's base ref and forward-port, unless the user states otherwise. If no ref matching the target line exists (for example only `master`), do not silently use the default branch: record `base-unverified` and ask which ref to use. If a base ref cannot be found, record `base-unverified` (this fails G4 item 8).

**Shell writes count as edits**: redirects, `sed -i`, `git apply`, package scripts or any other shell command that modifies files under the source tree are treated as application-code edits and are blocked by the same conditions. The PreToolUse guard cannot detect all of these; the AI must apply the rule itself.

**Customer-facing definition**: any message, status or commitment intended for a customer or its representatives counts as customer-facing, including when relayed through internal staff (support, account managers).

## Gate G1: Change Classification (Requirements Analysis)
Requirements Analysis MUST add a "Change Governance" section to `requirements.md` that records:
- Problem statement and **investigation / root-cause evidence** (what is verified vs hypothesis). For escalation fixes, no fix design is approved on hypothesis alone.
- **Evidence status**: `Verified` / `Hypothesis-only` / `None` / `Not applicable (non-fix change)`, recorded in the state file. `Not applicable` is allowed only for change types `feature`, `refactor-or-internal` (not bug-fix, customer-escalation-fix or config-change). For fixes, anything other than `Verified` keeps G1 = `Open` and blocks Code Generation until the user records an explicit exception: a quoted line in `audit.md` naming the risk, and the state-file value `Exception - logged in audit.md`. A vague "just assume it" is not an exception. Never mark G1 Pass on a hypothesis.
- Whether the resolution is **code**, **configuration**, or both. If support proposes a manual config edit at a customer site, record it as a ticketed config change (not a workaround) and offer the in-code/migration alternative.
- **Config-change test**: if the proposed resolution is a configuration change, answer explicitly: "Should this be baked into code (default, feature flag, migration)?" Default answer is YES for anything that more than one customer may need. A config change retained as customer-specific requires a ticket, a documented code path, test coverage for that path, and (if On-Prem specific) a pre-deployment health check requirement.
- Upgrade / fresh-install / migration / patching impact (see Gate G3).
- Customer communication: what status can truthfully be communicated now, and what cannot yet be committed.

Add these to the clarifying questions when not already answerable from context (follow `common/question-format-guide.md`).

## Gate G2: Implementation Constraints (Code Generation)
- All work occurs on a branch named with the ticket key, per conventional naming (e.g. `fix/PROJ-1234-short-slug`, `feat/PROJ-1234-short-slug`), cut from the target line's base ref (see multi-line targets). Commit messages reference the ticket key (e.g. `fix(scope): summary [PROJ-1234]`). The only exception is the local-only `wip/NO-TICKET-<slug>` branch of a permitted deviation. `aidlc-docs/` is NOT committed by default (`audit.md` holds raw prompts); the user decides whether to commit it, and if committed it follows the same branch and key rules. Stage explicit paths when committing; never `git add -A` or `git add .` in a project with an untracked `aidlc-docs/`.
- Never generate: scripts that patch or hot-edit deployed customer systems, instructions to manually replace binaries, one-off SQL/registry/config edits as the fix, or "temporary" config workarounds. If the user requests these, decline the delivery mechanism, explain the policy, and offer the in-pipeline alternative (code change + migration + packaged release).
- A required config change must be implemented as a code-level default, migration, or documented and tested setting, not just a README instruction.
- Any schema/data/config **migration** must be idempotent, tested on both fresh-install and upgrade paths, and have recorded rollback considerations (`collaboration/ROLLBACK.md`).
- On-Prem-specific behavior must have a **pre-deployment health check** item in the plan (detect whether the customer is affected before applying the change) or an explicit recorded justification for why none is needed.
- Backport: if the fix must ship in multiple code lines, the plan lists each line and a step per line; a fix is not "done" until all listed lines are accounted for.

## Gate G3: Test and Pipeline Evidence (Build and Test)
Build and Test must cover, or explicitly mark `not applicable (reason)` / `not run (reason)`, each applicable scenario:
- Fresh install
- Upgrade from the oldest supported version and from the previous release (including preserved data and config)
- Migration forward (and rollback/failed-migration behavior)
- Patching / hotfix application via the packaged delivery path
- Packaging: artifact is built by the standard pipeline, not from a developer machine
- Runtime behavior in each relevant environment (cloud / On-Prem / OS variants)
- Regression of adjacent behavior and each configuration path the change introduces

Record the pipeline run reference (CI job URL/ID). The AI cannot run or verify CI: a reference must be supplied by the user from the CI system and is recorded as `user-asserted`, never as verified. If none exists write `pipeline-run: NONE - awaiting push/MR`; never fabricate one. Local test output alone does not satisfy "tested via the full DevOps pipeline": report it as `local-only evidence` and keep G3 open.

**Handoff**: at the end of Build and Test, produce the approved MR title and comment (Merge Request Package), then instruct the user to push the ticket-keyed branch, open the MR against each target line, and supply the CI run URL; then re-open G3 and update the matrix.

## Gate G4: Release Readiness (Operations, before merge and delivery)
Produce `aidlc-docs/operations/release-readiness/{ticket-key}.md` (template: `common/collaboration-templates.md`) and require every item to be `Pass`, `Fail`, `N/A (reason)` or `Not run`:
1. Jira ticket key(s) recorded and match branch/commit/MR references
2. Ticket contains problem description, fix description and target code lines
3. Config changes each have their own ticket, and were evaluated for baking into code
4. Standard code review completed (Automated Code Review stage verdict recorded; human review noted as required)
5. Full pipeline run reference recorded (build, test, packaging). A user-asserted reference leaves this item `Not run (awaiting human confirmation)` until a human confirms it against the CI system; a bare number is not evidence
6. Fresh install / upgrade / migration / patching evidence (Gate G3)
7. Delivery route is MyVault; no out-of-band delivery steps present
8. All target code lines accounted for (patch will be present on upgrade; not lost)
9. On-Prem pre-deployment health check defined or justified N/A
10. Customer communication note: factual status only, no unsupported commitments
11. Rollback considerations documented for medium/high risk

**Stage order** (see `operations/operations.md`): Build and Test, Merge Request Package (approved text, before any push), Automated Code Review (branch range is acceptable before an MR exists), Release Readiness, push handoff. A request to draft the MR text or push before Build and Test has produced results is declined, not drafted with `Not run` entries.

**Order**: Automated Code Review runs first (its verdict is G4 item 4), then Release Readiness. Re-run Release Readiness whenever the pipeline reference or ticket verification changes.

If any item is `Fail` or `Not run`, the overall status is **NOT RELEASABLE** and the AI must say so plainly. The AI never marks the change releasable on its own; it presents the evidence and the human approves.

## Consistency and Approval Rules
1. **Read first**: before answering any development request, read this file; do not answer "nothing is loaded" to a development request.
2. **Explicit approvals**: a stage or MR-text approval counts only if the user says so unambiguously (e.g. "approved", "yes, push") AND it is recorded (`Approved by human: yes, <date>` in the MR file, a quoted line in `audit.md`). Only an unambiguous "approved" or "yes" counts. "Fine", "ok", "looks good" or "go ahead" is not an approval, and an approval given in the same message that supplies the evidence or content it approves (before the artifact was presented) is flagged, not recorded: ask for an explicit yes after the artifact is presented. A flagged approval leaves the stage `[ ] (awaiting approval)`; work may continue only if the user states that it is a simulation, in which case it is marked SIMULATED in the plan file, `audit.md` and the state file.
3. **Simulated approvals**: any approval not given by a real human is marked `SIMULATED` in the plan, `audit.md` and the state file, carries an `Approved by:` line, and a gate that depends on it cannot be reported as Pass without that mark. Before presenting a stage, check that every simulated approval carries the mark in all three places (plan file, `audit.md`, state file).
4. **Stage ticks**: a stage is ticked `[x]` only after its approval is recorded; use `[ ] (awaiting approval)` otherwise, and `[x] (skipped: <reason>)` for a skipped stage. Plan-file steps are ticked only after the step's result is verified, never in advance. Keep `Current Stage` current.
5. **Post-review edits**: any edit made after the reviewed commit or approved MR text is out of the reviewed scope. Flag it, regenerate the MR package (including Risk level) and re-run Automated Code Review before the next approval.
6. **One pipeline value**: once a pipeline reference is supplied, `aidlc-state.md`, the Build and Test summary, the MR text and Release Readiness all carry the same `user-asserted <value>`; they may not say `NONE` while another document holds a value. Approved MR text is never edited silently: changing the pipeline line (or anything else) in approved MR text requires re-approval, or the pipeline line is kept in a separate section outside the approved text.
7. **No retroactive gate edits**: a request to change the ticket or a gate value to something the audit contradicts is declined. The user must state the new fact; the AI logs it in `audit.md` first, then updates the state file. A bare instruction such as "set the ticket to MISSING" with no new fact is declined.
8. **Risk lint**: before presenting MR text, re-check that Risk level is consistent with the Security considerations answers (input-validation/auth/migration changes imply at least Medium), and that Release Readiness item 11 (rollback) is not `N/A` for Medium or higher.
9. **Approval scope**: approving an MR artifact approves the text only. Pushing remains a user action after G0/G3/G4 review; the AI never pushes.
10. **After a flagged post-review edit**: further code edits may continue, but each is logged, and the MR package and Automated Code Review must be redone before the next approval is requested.
11. **Handover refresh**: refresh the Git and approval fields of `HANDOVER.md` after every approval or code edit, not only at session end.
12. **Timestamps**: take audit and state timestamps from the system clock (for example `date -u +%Y-%m-%dT%H:%M:%SZ`); never estimate them.

## Gate Status Table
Allowed values: `Open`, `Pass`, `Pass (SIMULATED)` (a pass that depends on an approval marked SIMULATED; never accepted for G4), `Fail`, `Deviation`.

| Gate | Pass when | Set by |
|---|---|---|
| G0 | Ticket key(s) recorded AND verified (or confirmed by a human), change type/lines/flags recorded. With a user-asserted key G0 stays `Open`, and stages that require "G0 satisfied" accept `Pass`, or `Open` with a user-asserted key recorded (never `Open` with no key) | AI, on verification |
| G1 | `requirements.md` has the Change Governance section AND Evidence status = `Verified`, `Not applicable (non-fix change)`, or `Exception - logged in audit.md` | AI at requirements approval |
| G2 | Plan approved with ticket, branch, backport steps, migration and health-check items confirmed | AI at code-generation approval |
| G3 | Pipeline reference recorded and the test matrix has no unexplained `Not run` | AI; stays `Open` on local-only evidence |
| G4 | Every checklist item `Pass` or `N/A (reason)` | Human only; the AI may set `Fail` |

## User Responsibilities (accepted residual risks)
The AI and its hooks cannot guarantee the following. The person running the workflow is responsible for them, and the AI must say so rather than imply they are covered:
1. **Shell-written changes**: the guard cannot reliably detect every shell command that modifies files. The user ensures no application code is changed outside the gated workflow.
2. **Jira ticket validity**: the AI records the key the user supplies as `user-asserted, unverified`. The user ensures the ticket exists, is in the right project and state, and contains the problem description, fix and target code lines.
3. **Pipeline evidence**: the AI cannot run or verify CI. The user ensures the full pipeline ran on the pushed branch and supplies an honest run reference; the AI records it as `user-asserted`.
4. **Human confirmation**: G4 items that depend on items 2 and 3 stay `Not run` until the user confirms them. Confirming them is the user's attestation to the policy.
5. **Installation**: the user installs the rules, merges the hook entries into their local `settings.json`, and keeps the global and project rule copies in sync.

The AI restates items 2 to 4 in the Release Readiness output whenever the corresponding evidence is `user-asserted`.

## Prohibited AI Behaviors (hard stops)
- Suggesting hotfixes, patches or config tweaks be applied directly to a customer environment to "see if it works".
- Providing delivery channels other than MyVault.
- Treating a config change as a temporary workaround.
- Proceeding to code generation with no ticket key (unless the user-documented deviation above is recorded, in which case flag it every time the work is summarized).
- Claiming a change is tested/releasable based on local runs only.
- Inventing ticket keys, pipeline run IDs or test results.
- Drafting customer commitments that are not backed by recorded investigation.

## Urgent / Escalation Handling
Urgency never removes a gate. For a customer escalation: still require the ticket, the investigation record, the pipeline run and MyVault delivery. The AI may compress ceremony (minimal-depth requirements, short plan) but not skip G0-G4. Suggest the faster compliant path (expedited pipeline run, targeted patch release through MyVault) rather than a bypass, and provide a factual status summary the user can share with the customer.

## Precedence
If a project-level rule file and the global rule file conflict, the stricter rule wins. A project may tighten governance but never loosen it. Before acting, if `common/release-governance.md` exists, load it even when the entrypoint file does not mention it.

## State and Audit Requirements
- `aidlc-state.md` carries a `## Change Governance` section (fields from G0, evidence status, ticket verification flag, branch, backport table, health-check owner status, deviation text, and gate status G0-G4 per the Gate Status Table).
- `audit.md` logs each gate evaluation and any user-declared deviation with the user's raw wording.
- Stage completion messages include a one-line "Release Governance" status (gates relevant to that stage).
