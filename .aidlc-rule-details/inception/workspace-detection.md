# Workspace Detection

**Purpose**: Determine workspace state and check for existing AI-DLC projects

## Step 1: Check for Existing AI-DLC Project

Check if `aidlc-docs/aidlc-state.md` exists:
- **If exists**: Resume from last phase (load context from previous phases)
- **If not exists**: Continue with new project assessment

## Step 2: Scan Workspace for Existing Code

**Determine if workspace has existing code:**
- Scan workspace for source code files (.java, .py, .js, .ts, .jsx, .tsx, .kt, .kts, .scala, .groovy, .go, .rs, .rb, .php, .c, .h, .cpp, .hpp, .cc, .cs, .fs, etc.)
- Check for build files (pom.xml, package.json, build.gradle, etc.)
- Look for project structure indicators
- Identify workspace root directory (NOT aidlc-docs/)

**Record findings:**
```markdown
## Workspace State
- **Existing Code**: [Yes/No]
- **Programming Languages**: [List if found]
- **Build System**: [Maven/Gradle/npm/etc. if found]
- **Project Structure**: [Monolith/Microservices/Library/Empty]
- **Workspace Root**: [Absolute path]
```

## Step 3: Determine Next Phase

**IF workspace is empty (no existing code)**:
- Set flag: `brownfield = false`
- Next phase: Requirements Analysis

**IF workspace has existing code**:
- Set flag: `brownfield = true`
- Check for existing reverse engineering artifacts in `aidlc-docs/inception/reverse-engineering/`
- **IF reverse engineering artifacts exist**:
    - Check if artifacts are stale (compare artifact timestamps against codebase's last significant modification)
    - **IF artifacts are current**: Load them, skip to Requirements Analysis
    - **IF artifacts are stale**: Next phase is Reverse Engineering (rerun to refresh artifacts)
    - **IF user explicitly requests rerun**: Next phase is Reverse Engineering regardless of staleness
- **IF no reverse engineering artifacts**: Next phase is Reverse Engineering

## Step 4: Create Initial State File

**ORDER (MANDATORY)**: before asking ANY question (including `governance-questions.md`) or reporting a blocker, do these in order: (1) display the FULL welcome message from `common/welcome-message.md` once (not an abbreviation), (2) create/append `audit.md` with the user's raw request, (3) create `aidlc-state.md` with `Jira Ticket(s): MISSING` if no key is known yet. Blocking on a missing ticket never excuses skipping these. Update `Current Stage` in `aidlc-state.md` at every stage change.

If `aidlc-docs/audit.md` does not exist, create it with header `# AI-DLC Audit Log` and make the user's raw request the first entry (append-only afterwards). Mark any simulated or assumed approval as `SIMULATED` in the audit entry.

Create `aidlc-docs/aidlc-state.md`:

```markdown
# AI-DLC State Tracking

## Project Information
- **Project Type**: [Greenfield/Brownfield]
- **Start Date**: [ISO timestamp]
- **Current Stage**: INCEPTION - Workspace Detection

## Workspace State
- **Existing Code**: [Yes/No]
- **Reverse Engineering Needed**: [Yes/No]
- **Workspace Root**: [Absolute path]

## Change Governance
- **Jira Ticket(s)**: [key(s) or MISSING]
- **Ticket Verification**: [verified / user-asserted, unverified]
- **Origin**: [new work / customer escalation (reference only, no secrets)]
- **Branch**: [name, base ref, or base-unverified]
- **Evidence Status (G1)**: [Verified / Hypothesis-only / None / Not applicable (non-fix change) / Exception - logged in audit.md]
- **Backport Table**: [line -> ticket -> branch -> base ref -> status]
- **Health-Check Owner Status**: [N/A / Pending owner ack / Acknowledged by <name>]
- **Deviation**: [none / raw wording reference in audit.md]
- **Pipeline Run**: [NONE - awaiting push/MR / user-asserted <url>]
- **Change Type**: [feature/bug-fix/customer-escalation-fix/config-change/refactor-or-internal]
- **Target Code Line(s)**: [list or unknown]
- **Customer-Facing**: [Yes/No/Unknown]
- **On-Prem Specific**: [Yes/No/Unknown]
- **Gates** (Open/Pass/Fail/Deviation): G0 [Open] | G1 [Open] | G2 [Open] | G3 [Open] | G4 [Open]

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Extension Configuration
(Written at Requirements Analysis: one row per extension with Enabled and Decided At; if opt-ins are skipped, every extension is `No` with `Skipped - logged in audit.md`.)

## Stage Progress
A stage is ticked `[x]` only after its approval is recorded; until then mark it `[ ] (awaiting approval)`.

### 🔵 INCEPTION PHASE
- [ ] Workspace Detection
- [ ] Reverse Engineering
- [ ] Requirements Analysis
- [ ] User Stories
- [ ] Workflow Planning
- [ ] Application Design
- [ ] Units Generation

### 🟢 CONSTRUCTION PHASE
- [ ] Functional Design
- [ ] NFR Requirements
- [ ] NFR Design
- [ ] Infrastructure Design
- [ ] Code Generation
- [ ] Build and Test

### 🟡 OPERATIONS PHASE
- [ ] Merge Request Package
- [ ] Automated Code Review
- [ ] Release Readiness
- [ ] Operations - PLACEHOLDER
```

## Step 4.5: Release Governance Gate G0 (Change Ticket)

Load `common/release-governance.md`. Using values from the user's request when present (otherwise via `governance-questions.md`; never guess), fill the `## Change Governance` section of `aidlc-state.md`: Jira ticket key(s), change type, target code line(s), customer-facing, On-Prem specific, origin, ticket verification flag, branch. Follow the governance rule for missing tickets, deviations, unverified keys and multi-line targets. Create or switch to a branch named with the ticket key from the target base ref; if that is not possible, record `base-unverified`.

## Step 5: Present Completion Message

**For Brownfield Projects:**
```markdown
# 🔍 Workspace Detection Complete

Workspace analysis findings:
• **Project Type**: Brownfield project
• [AI-generated summary of workspace findings in bullet points]
• **Next Step**: Proceeding to **Reverse Engineering** to analyze existing codebase...
```

**For Greenfield Projects:**
```markdown
# 🔍 Workspace Detection Complete

Workspace analysis findings:
• **Project Type**: Greenfield project
• **Next Step**: Proceeding to **Requirements Analysis**...
```

## Step 6: Automatically Proceed

- **No user approval required** - this is informational only
- Automatically proceed to next phase:
  - **Brownfield**: Reverse Engineering (if no existing artifacts) or Requirements Analysis (if artifacts exist)
  - **Greenfield**: Requirements Analysis
