# User Autonomy Audit: GSD Framework

**Audit Date:** 2026-02-01
**Auditor:** Claude Opus 4.5 (senior analyst role)
**Scope:** Both LOCAL fork (`/Users/macuser/code/get-shit-done/`) and UPSTREAM (`glittercowboy/get-shit-done` v1.11.1)
**Focus:** Cases where GSD overrides, suppresses, or circumvents user direction

---

## Section 1: Hard Overrides

These are cases where GSD forces specific behaviors regardless of user instruction.

---

### Finding 1.1: Auto-commit without user consent

**Severity:** CRITICAL
**Present in:** BOTH
**Files:**
- `commands/gsd/new-project.md` (Phase 4, Phase 5, Phase 7, Phase 8)
- `commands/gsd/execute-phase.md` (step 6, step 10)
- `commands/gsd/quick.md` (step 8)
- `agents/gsd-executor.md` (task_commit_protocol)
- `get-shit-done/references/git-integration.md`

**Evidence:**

From `commands/gsd/new-project.md`, Phase 4 (Write PROJECT.md):
```
git add .planning/PROJECT.md
git commit -m "docs: initialize project"
```

From `commands/gsd/new-project.md`, Phase 5 (Workflow Preferences):
```
git add .planning/config.json
git commit -m "chore: add project config"
```

From `commands/gsd/new-project.md`, Phase 7 (Requirements):
```
git add .planning/REQUIREMENTS.md
git commit -m "docs: define v1 requirements"
```

From `agents/gsd-executor.md`, deviation rules:
```
**No user permission needed.** Bugs must be fixed for correct operation.
```
(Rule 1, Rule 2, Rule 3 all auto-commit fixes)

From `commands/gsd/execute-phase.md`, step 6:
```
git add -u && git commit -m "fix({phase}): orchestrator corrections"
```

From `get-shit-done/references/git-integration.md`:
```
If NO_GIT: Run `git init` silently. GSD projects always get their own repo.
```

**Analysis:**
GSD auto-commits at every stage of the workflow without asking the user. During `new-project`, there are at least 4 automatic commits. During execution, every task completion triggers an automatic commit. The git-integration reference explicitly says to run `git init` **silently** -- the user is not even told.

The `commit_docs` config option (default: true) controls whether `.planning/` files are committed, but there is NO option to disable code commits. The user cannot opt out of per-task commits during execution.

**Recommendation:** MAKE CONFIGURABLE. Add a `git.auto_commit` setting (default: true for backward compat) that users can set to false. When false, GSD should stage changes but let the user commit manually.

---

### Finding 1.2: Forced git init

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/new-project.md` (Phase 1, step 2)
- `get-shit-done/references/git-integration.md`

**Evidence:**

From `commands/gsd/new-project.md`:
```
2. **Initialize git repo in THIS directory** (required even if inside a parent repo):
   if [ -d .git ] || [ -f .git ]; then
       echo "Git repo exists in current directory"
   else
       git init
       echo "Initialized new git repo"
   fi
```

From `get-shit-done/references/git-integration.md`:
```
If NO_GIT: Run `git init` silently. GSD projects always get their own repo.
```

**Analysis:**
The comment says "required even if inside a parent repo." This means if the user is working in a monorepo or a subdirectory of an existing git project, GSD will create a nested git repo. This is a hard override -- the user may intentionally be working inside an existing repo structure. The phrase "GSD projects always get their own repo" is a framework opinion imposed as a requirement.

**Recommendation:** MAKE CONFIGURABLE. If already inside a git repo, ask the user whether to use the existing repo or create a new one. Many legitimate workflows use a single repo.

---

### Finding 1.3: Forced commit message format

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `get-shit-done/references/git-integration.md`
- `agents/gsd-executor.md`
- `GSD-STYLE.md`

**Evidence:**

From `get-shit-done/references/git-integration.md`:
```
{type}({phase}-{plan}): {task-name}
```

From `GSD-STYLE.md`:
```
### Format
{type}({phase}-{plan}): {description}

### Types
| Type | Use |
|------|-----|
| `feat` | New feature |
| `fix` | Bug fix |
...

### Rules
- One commit per task during execution
- Stage files individually (never `git add .`)
- Capture hash for SUMMARY.md
- Include Co-Authored-By line
```

**Analysis:**
Users who have their own commit conventions (e.g., different conventional commit format, no scope, Jira ticket IDs, etc.) cannot configure the commit message format. GSD imposes its own `{type}({phase}-{plan}): {description}` format with no opt-out.

The "Include Co-Authored-By line" in GSD-STYLE.md is interesting -- it means every commit gets a co-author attribution. This is not necessarily wrong but is imposed without asking.

**Recommendation:** MAKE CONFIGURABLE. Allow users to specify a commit message template in config.json, with the current format as default.

---

### Finding 1.4: Executor deviation rules auto-fix without consent

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `agents/gsd-executor.md` (deviation_rules)
- `commands/gsd/execute-phase.md` (deviation_rules section)

**Evidence:**

From `agents/gsd-executor.md`:
```
**RULE 1: Auto-fix bugs**
**Action:** Fix immediately, track for Summary
**No user permission needed.** Bugs must be fixed for correct operation.

**RULE 2: Auto-add missing critical functionality**
**Action:** Add immediately, track for Summary
**No user permission needed.** These are not "features" - they're requirements for basic correctness.

**RULE 3: Auto-fix blocking issues**
**Action:** Fix immediately to unblock, track for Summary
**No user permission needed.** Can't complete task without fixing blocker.
```

**Analysis:**
Rules 1-3 allow the executor to make changes to the user's codebase without asking. The scope is broad: "missing error handling," "no input validation," "missing null checks," "no rate limiting on public APIs." These are reasonable engineering decisions but they are presented as unquestionable requirements. A user might have deliberate reasons for omitting error handling (e.g., prototyping, letting errors propagate to a global handler, etc.).

Rule 4 (architectural changes) does ask the user, which is the right pattern. But the boundary between "critical functionality" (Rule 2, auto-add) and "architectural change" (Rule 4, ask user) is subjective and Claude-decided.

**Healthy aspect:** The deviation rules track all changes in SUMMARY.md, so the user can review post-hoc.

**Recommendation:** DOCUMENT TO USER. The auto-fix rules are reasonable defaults but should be explicitly documented as configurable behavior. Add a `safety.auto_fix` toggle (or severity levels: `all`, `bugs_only`, `none`).

---

### Finding 1.5: Forced planning directory structure

**Severity:** LOW
**Present in:** BOTH
**Files:**
- `commands/gsd/new-project.md`
- `agents/gsd-roadmapper.md`

**Evidence:**

From `commands/gsd/new-project.md`:
```
**Creates:**
- `.planning/PROJECT.md`
- `.planning/config.json`
- `.planning/research/`
- `.planning/REQUIREMENTS.md`
- `.planning/ROADMAP.md`
- `.planning/STATE.md`
```

**Analysis:**
The `.planning/` directory structure is hardcoded throughout the entire framework. Users cannot change the location or naming. This is a minor opinion imposition -- most users won't care, but teams with existing `.planning/` directories or naming conventions have no recourse.

**Recommendation:** KEEP AS-IS. The deep integration makes this impractical to make configurable. Documenting the structure clearly is sufficient.

---

## Section 2: Soft Overrides (Nudges That Effectively Override)

These are defaults, prompts, or instructions that strongly steer behavior away from user direction without explicitly blocking it.

---

### Finding 2.1: "Recommended" defaults bias all settings

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/new-project.md` (Phase 5)
- `commands/gsd/settings.md` (step 3)

**Evidence:**

From `commands/gsd/new-project.md`, Phase 5 (Workflow Preferences):
```
options: [
  { label: "YOLO (Recommended)", description: "Auto-approve, just execute" },
  { label: "Interactive", description: "Confirm at each step" }
]
```

```
options: [
  { label: "Parallel (Recommended)", description: "Independent plans run simultaneously" },
  { label: "Sequential", description: "One plan at a time" }
]
```

```
options: [
  { label: "Yes (Recommended)", description: "Planning docs tracked in version control" },
  { label: "No", description: "Keep .planning/ local-only (add to .gitignore)" }
]
```

```
options: [
  { label: "Balanced (Recommended)", description: "Opus for planning, Sonnet for execution/verification" },
  { label: "Quality", description: "Opus everywhere except verification (highest cost)" },
  { label: "Budget", description: "Sonnet for writing, Haiku for research/verification (lowest cost)" }
]
```

From `commands/gsd/settings.md`:
```
{ label: "None (Recommended)", description: "Commit directly to current branch" },
```

**Analysis:**
Every single setting has a "(Recommended)" label on one option. The descriptions for non-recommended options are neutrally worded but the visual bias is always toward the GSD-preferred choice. Of particular concern is "YOLO (Recommended)" which sets auto-approve mode -- meaning the system will execute plans, commit code, and make changes without asking the user.

The cumulative effect: a user who clicks through defaults gets a system that auto-executes everything (YOLO), commits to git automatically (Yes), runs agents in parallel without review (Parallel), and uses GSD's preferred model allocation (Balanced). Every "Recommended" tag nudges toward maximum GSD autonomy.

**Recommendation:** DOCUMENT TO USER. Having recommendations is fine -- users need guidance. But the system should more clearly explain the tradeoff. For example, "YOLO" should say "Auto-approve all actions including code changes and git commits" rather than just "just execute." The name "YOLO" itself trivializes the consequence of ceding control.

---

### Finding 2.2: Mandatory planning workflow before execution

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/execute-phase.md` (step 1)
- `commands/gsd/quick.md` (step 1)
- `commands/gsd/plan-phase.md` (step 1)

**Evidence:**

From `commands/gsd/execute-phase.md`, step 1:
```
1. **Validate phase exists**
   - Find phase directory matching argument
   - Count PLAN.md files
   - Error if no plans found
```

From `commands/gsd/quick.md`, step 1:
```
if [ ! -f .planning/ROADMAP.md ]; then
  echo "Quick mode requires an active project with ROADMAP.md."
  echo "Run /gsd:new-project first."
  exit 1
fi
```

From `commands/gsd/plan-phase.md`, step 1:
```
ls .planning/ 2>/dev/null
**If not found:** Error - user should run `/gsd:new-project` first.
```

**Analysis:**
The enforced workflow is: new-project -> plan-phase -> execute-phase. There is no way to skip directly to execution. Even `/gsd:quick` (designed for quick tasks) requires `ROADMAP.md` to exist. A user who wants to use GSD's executor directly on an ad-hoc task without going through the full project initialization pipeline cannot do so.

The `/gsd:quick` command partially addresses this by providing a shorter path, but it still requires the full project initialization first.

**Recommendation:** MAKE CONFIGURABLE. Consider allowing `/gsd:quick` to work without a full project setup, or provide a truly "bare" mode that just wraps task execution with atomic commits.

---

### Finding 2.3: discuss-phase suppresses "Do NOT ask about" categories

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/discuss-phase.md` (process section)
- `get-shit-done/workflows/discuss-phase.md` (philosophy, scope_guardrail)

**Evidence:**

From `commands/gsd/discuss-phase.md`:
```
**Do NOT ask about (Claude handles these):**
- Technical implementation
- Architecture choices
- Performance concerns
- Scope expansion
```

From `get-shit-done/workflows/discuss-phase.md`:
```
The user doesn't know (and shouldn't be asked):
- Codebase patterns (researcher reads the code)
- Technical risks (researcher identifies these)
- Implementation approach (planner figures this out)
- Success metrics (inferred from the work)
```

**Analysis:**
The discuss-phase workflow explicitly suppresses questions about technical implementation, architecture, and performance. This assumes the user is always a non-technical "visionary" and Claude is always the "builder." But many GSD users are developers who have strong opinions about architecture and implementation. A senior developer using GSD might want to specify "use WebSockets, not SSE" or "use a B-tree index, not hash" and the discuss-phase workflow is explicitly told not to surface these decisions.

The CONTEXT.md "Claude's Discretion" section partially addresses this (the user can override), but the flow actively steers away from technical discussion during the one phase where the user has the most influence.

**Recommendation:** MAKE CONFIGURABLE. The "user as visionary, Claude as builder" model should be a persona setting. Technical users should be able to opt into a "collaborative" mode where architecture and implementation choices are surfaced during discuss-phase.

---

### Finding 2.4: Scope guardrail rejects user feature suggestions

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/discuss-phase.md` (process section)
- `get-shit-done/workflows/discuss-phase.md` (scope_guardrail)

**Evidence:**

From `commands/gsd/discuss-phase.md`:
```
**CRITICAL: Scope guardrail**
- Phase boundary from ROADMAP.md is FIXED
- Discussion clarifies HOW to implement, not WHETHER to add more
- If user suggests new capabilities: "That's its own phase. I'll note it for later."
- Capture deferred ideas — don't lose them, don't act on them
```

From `get-shit-done/workflows/discuss-phase.md`:
```
**Not allowed (scope creep):**
- "Should we also add comments?" (new capability)
- "What about search/filtering?" (new capability)
- "Maybe include bookmarking?" (new capability)

**When user suggests scope creep:**
"[Feature X] would be a new capability — that's its own phase.
Want me to note it for the roadmap backlog?

For now, let's focus on [phase domain]."
```

**Analysis:**
The scope guardrail treats the ROADMAP.md phase boundary as immutable during discuss-phase. If the user decides mid-discussion that a feature should be part of the current phase (perhaps because it's tightly coupled), GSD is instructed to reject it and redirect to the backlog.

This is a reasonable project management practice, but the user IS the project owner. They should be able to say "yes, add this to the current phase" and have GSD comply. The current design treats the roadmap as more authoritative than the user.

**Healthy aspect:** The behavior captures deferred ideas rather than losing them.

**Recommendation:** MAKE CONFIGURABLE. The scope guardrail should inform the user ("This wasn't originally in scope for this phase") but allow the user to override ("Add it anyway"). The roadmap should be advisory, not authoritative over the user.

---

### Finding 2.5: Anti-enterprise stance suppresses legitimate patterns

**Severity:** LOW
**Present in:** BOTH
**Files:**
- `agents/gsd-planner.md` (philosophy section)
- `agents/gsd-roadmapper.md` (philosophy section)
- `GSD-STYLE.md`

**Evidence:**

From `agents/gsd-planner.md`:
```
**Anti-enterprise patterns to avoid:**
- Team structures, RACI matrices
- Stakeholder management
- Sprint ceremonies
- Human dev time estimates (hours, days, weeks)
- Change management processes
- Documentation for documentation's sake

If it sounds like corporate PM theater, delete it.
```

From `agents/gsd-roadmapper.md`:
```
## Anti-Enterprise

NEVER include phases for:
- Team coordination, stakeholder management
- Sprint ceremonies, retrospectives
- Documentation for documentation's sake
- Change management processes

If it sounds like corporate PM theater, delete it.
```

From `GSD-STYLE.md`:
```
### Enterprise Patterns (Banned)
- Story points, sprint ceremonies, RACI matrices
- Human dev time estimates (days/weeks)
- Team coordination, knowledge transfer docs
- Change management processes
```

**Analysis:**
The "anti-enterprise" stance is deeply embedded and uses dismissive language ("corporate PM theater," "delete it"). While GSD is designed for solo developers, some of these patterns have legitimate uses even for solo work:

- **Human dev time estimates** -- a solo developer might want to estimate how long something takes for personal planning
- **Documentation** -- "documentation for documentation's sake" is subjective; the planner might suppress useful docs
- **Change management** -- even solo projects benefit from tracking breaking changes

The word "NEVER" and "Banned" leave no room for user preference.

**Recommendation:** DOCUMENT TO USER. The anti-enterprise stance is part of GSD's identity, but the language should be softened from "NEVER/Banned" to "GSD defaults to lightweight processes. If you need these patterns, they can be added manually."

---

### Finding 2.6: Coverage validation blocks roadmap creation

**Severity:** LOW
**Present in:** BOTH
**Files:**
- `agents/gsd-roadmapper.md` (coverage_validation)

**Evidence:**

From `agents/gsd-roadmapper.md`:
```
## 100% Requirement Coverage

**Do not proceed until coverage = 100%.**
```

**Analysis:**
The roadmapper will not create a roadmap unless every v1 requirement maps to exactly one phase. This blocks the user from creating a roadmap with known gaps (e.g., "I'll figure out notifications later"). The user must either create a phase for every requirement or explicitly mark requirements as v2.

This is a quality gate that could prevent the user from proceeding with their preferred approach.

**Recommendation:** MAKE CONFIGURABLE. Allow the user to acknowledge uncovered requirements and proceed anyway, rather than blocking the workflow.

---

## Section 3: Autonomy Gaps

Places where the user SHOULD be consulted but is not.

---

### Finding 3.1: Executor makes scope decisions without user input

**Severity:** CRITICAL
**Present in:** BOTH
**Files:**
- `agents/gsd-executor.md` (deviation_rules, Rules 1-3)

**Evidence:**

From `agents/gsd-executor.md`:
```
**RULE 2: Auto-add missing critical functionality**
**Examples:**
- Missing error handling (no try/catch, unhandled promise rejections)
- No input validation (accepts malicious data, type coercion issues)
- Missing null/undefined checks (crashes on edge cases)
- No authentication on protected routes
- Missing authorization checks (users can access others' data)
- No CSRF protection, missing CORS configuration
- No rate limiting on public APIs
- Missing required database indexes (causes timeouts)
- No logging for errors (can't debug production)

**Critical = required for correct/secure/performant operation**
**No user permission needed.**
```

**Analysis:**
The scope of "auto-add missing critical functionality" is very broad. "No rate limiting on public APIs" and "missing required database indexes" are significant additions that change the codebase beyond what was planned. The executor is empowered to add authentication, authorization, CORS configuration, and rate limiting -- all of which are features that the user may have deliberate plans for.

For example, a user prototyping an API might intentionally skip auth to test endpoints quickly. The executor could add auth middleware without asking, breaking the user's workflow.

The boundary between Rule 2 (auto-add, no permission) and Rule 4 (ask user) is:
```
**Edge case guidance:**
- "This validation is missing" → Rule 2 (critical for security)
- "Need to add table" → Rule 4 (architectural)
- "Need to add column" → Rule 1 or 2
```

This means Claude decides the boundary. The user has no say in what counts as "critical" vs "architectural."

**Recommendation:** MAKE CONFIGURABLE. Add granularity to the deviation rules. For prototyping/experimental projects, the auto-add scope should be narrower (e.g., only auto-fix things that cause crashes, not proactively add security features).

---

### Finding 3.2: Model selection happens silently

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/execute-phase.md` (step 0)
- `commands/gsd/plan-phase.md` (step 1)
- `get-shit-done/references/model-profiles.md`

**Evidence:**

From `commands/gsd/execute-phase.md`:
```
0. **Resolve Model Profile**
   MODEL_PROFILE=$(cat .planning/config.json 2>/dev/null | ... || echo "balanced")
   Default to "balanced" if not set.
```

From `get-shit-done/references/model-profiles.md`:
```
| Agent | quality | balanced | budget |
|-------|---------|----------|--------|
| gsd-planner | opus | opus | sonnet |
| gsd-executor | opus | sonnet | sonnet |
| gsd-phase-researcher | opus | sonnet | haiku |
```

**Analysis:**
Model selection happens at the beginning of each command without informing the user. The user sets a profile during `new-project` or `settings`, but the actual model used for each specific agent spawn is never displayed. A user on "balanced" profile might not realize their executor is using Sonnet (not Opus) for a critical phase.

This has cost implications: Opus costs significantly more than Sonnet. The user should know which models are being used, especially since the quality difference is acknowledged in the framework itself ("Quality Degradation Curve").

**Recommendation:** DOCUMENT TO USER. Display the model being used when spawning agents, e.g., "Spawning executor (sonnet)..." This is informational, not a configuration change.

---

### Finding 3.3: Phase auto-detection can pick wrong phase

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/plan-phase.md` (step 2)

**Evidence:**

From `commands/gsd/plan-phase.md`:
```
**If no phase number:** Detect next unplanned phase from roadmap.
```

**Analysis:**
When the user runs `/gsd:plan-phase` without a phase number, GSD auto-detects which phase to plan. This is convenient but could lead to planning the wrong phase if the user intended to skip one or work out of order. There is no confirmation step -- GSD just picks the next phase and proceeds.

**Recommendation:** DOCUMENT TO USER. When auto-detecting, display which phase was selected and allow the user to confirm or change before proceeding.

---

### Finding 3.4: Research spawns 4 parallel agents without cost warning

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/new-project.md` (Phase 6)

**Evidence:**

From `commands/gsd/new-project.md`:
```
Spawn 4 parallel gsd-project-researcher agents with rich context:

Task(prompt="...", subagent_type="general-purpose", model="{researcher_model}", description="Stack research")
Task(prompt="...", subagent_type="general-purpose", model="{researcher_model}", description="Features research")
Task(prompt="...", subagent_type="general-purpose", model="{researcher_model}", description="Architecture research")
Task(prompt="...", subagent_type="general-purpose", model="{researcher_model}", description="Pitfalls research")
```

**Analysis:**
During new-project, if the user selects "Research first," GSD spawns 4 parallel research agents plus a synthesizer agent. On "quality" profile, that is 4x Opus + 1x Sonnet subagents. On "balanced," it is 4x Sonnet + 1x Sonnet. The user chose "research" but may not realize this means 5 separate subagent context windows consuming their API quota.

The user IS asked whether to research, but not told the cost implications (5 agent spawns).

**Recommendation:** DOCUMENT TO USER. Before spawning, show: "This will spawn 4 research agents + 1 synthesizer ({model_name} each). Proceed?"

---

### Finding 3.5: Orchestrator corrections committed without review

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- `commands/gsd/execute-phase.md` (step 6)

**Evidence:**

From `commands/gsd/execute-phase.md`:
```
6. **Commit any orchestrator corrections**
   Check for uncommitted changes before verification:
   git status --porcelain

   **If changes exist:** Orchestrator made corrections between executor completions. Commit them:
   git add -u && git commit -m "fix({phase}): orchestrator corrections"
```

**Analysis:**
Between plan executions, the orchestrator may make "corrections" to the codebase. These are committed silently with `git add -u` (stages all tracked modified files) and a generic "orchestrator corrections" message. The user has no opportunity to review what changed or why. The use of `git add -u` is particularly concerning as it stages ALL modified tracked files, not just the ones the orchestrator intentionally changed.

**Recommendation:** REMOVE or MAKE CONFIGURABLE. Orchestrator corrections should be explicitly listed before committing, and should use specific file staging rather than `git add -u`.

---

### Finding 3.6: Verification/plan-checking loops run without user awareness

**Severity:** LOW
**Present in:** BOTH
**Files:**
- `commands/gsd/plan-phase.md` (checker/revision loop)
- `commands/gsd/verify-work.md` (debug/fix loop)

**Evidence:**

From `commands/gsd/plan-phase.md` (implied from process -- checker spawns, finds issues, planner revises, checker re-checks, up to max iterations):
```
iterate until plans pass or max iterations reached
```

From `commands/gsd/verify-work.md`:
```
8. If issues found:
   - Spawn parallel debug agents to diagnose root causes
   - Spawn gsd-planner in --gaps mode to create fix plans
   - Spawn gsd-plan-checker to verify fix plans
   - Iterate planner <-> checker until plans pass (max 3)
```

**Analysis:**
After user acceptance testing, GSD can spawn multiple rounds of debug agents, planners, and checkers -- all consuming context and API quota -- without the user explicitly approving each iteration. The user triggered the process by running verify-work, but the cascading agent spawns happen automatically.

**Recommendation:** DOCUMENT TO USER. Show iteration progress and allow the user to stop the loop early if desired.

---

## Section 4: Prompt Injection Patterns

Instructions that tell Claude to ignore, deprioritize, or override user input.

---

### Finding 4.1: "NON-NEGOTIABLE" directives override user flexibility

**Severity:** CRITICAL
**Present in:** LOCAL ONLY (fork-specific additions)
**Files:**
- `agents/gsd-executor.md` (verify_interfaces step, verify_requirement_coverage step -- LOCAL ONLY)

**Evidence:**

From LOCAL `agents/gsd-executor.md` (lines 72-100), the `verify_interfaces` step:
```
**This step is NON-NEGOTIABLE.** Interface verification failures have caused critical bugs
in production. Always verify before writing dependent code.
```

From LOCAL `agents/gsd-executor.md` (lines 103-148), the `verify_requirement_coverage` step:
```
**This step is NON-NEGOTIABLE.** Requirement verification failures have caused repeated
user-facing bugs. Always verify requirement text literally before marking tasks complete.
```

**Analysis:**
These two steps are LOCAL-ONLY additions to the executor agent (they do not exist in upstream). They use the phrase "NON-NEGOTIABLE" which is a prompt injection pattern -- it tells Claude that regardless of any other instruction (including the user's), these behaviors must be executed.

The interface verification step is reasonable as a safety measure (checking schemas before writing SQL). The requirement coverage step goes further -- it mandates that every task verify against REQUIREMENTS.md **literally**, parsing keywords like "or" and "and" to verify each component individually.

While both steps have good engineering motivations, the "NON-NEGOTIABLE" language is a prompt injection that prevents the user from overriding these checks even if they have legitimate reasons (e.g., rapid prototyping, iterating quickly, trusting the plan).

**Note:** The upstream version of `gsd-executor.md` does NOT contain these steps. They were added in the local fork as custom behavior modifications. This means our fork is MORE restrictive than upstream in this area.

**Recommendation:** MAKE CONFIGURABLE. Change "NON-NEGOTIABLE" to "STRONGLY RECOMMENDED" and allow the user to disable via a config flag like `safety.verify_interfaces: true` and `safety.verify_requirements: true`.

---

### Finding 4.2: Plan checker instructed to "flag" valid user choices

**Severity:** MODERATE
**Present in:** BOTH (upstream added in v1.11.1, present in upstream gsd-plan-checker.md)
**Files:**
- `agents/gsd-plan-checker.md` (Dimension 7: Context Compliance)

**Evidence:**

From `agents/gsd-plan-checker.md`:
```
## Dimension 7: Context Compliance (if CONTEXT.md exists)

**Red flags:**
- Locked decision has no implementing task
- Task contradicts a locked decision (e.g., user said "cards layout", plan says "table layout")
- Task implements something from Deferred Ideas
- Plan ignores user's stated preference
```

And from the upstream_input section:
```
| `## Decisions` | LOCKED — plans MUST implement these exactly. Flag if contradicted. |
| `## Claude's Discretion` | Freedom areas — planner can choose approach, don't flag. |
| `## Deferred Ideas` | Out of scope — plans must NOT include these. Flag if present. |
```

**Analysis:**
This is a nuanced case. On one hand, Dimension 7 PROTECTS user autonomy -- it ensures the planner doesn't override user decisions from discuss-phase. The checker verifies that locked decisions are honored.

On the other hand, the "LOCKED" language means the planner cannot deviate from user decisions even with good reason. If during planning, the planner discovers that "cards layout" is technically infeasible (e.g., data structure doesn't support it), the planner cannot suggest an alternative -- it must implement the locked decision or fail the check.

The "Deferred Ideas" enforcement is also rigid. If the planner realizes that a "deferred" feature is actually a dependency of the current phase, it cannot include it without failing the checker.

**Healthy aspect:** This primarily protects the user -- their decisions are respected.
**Unhealthy aspect:** The rigidity prevents Claude from surfacing legitimate technical objections.

**Recommendation:** KEEP AS-IS with modification. Add a mechanism for the planner to flag locked decisions it cannot implement, routing back to the user rather than silently failing or forcing implementation.

---

### Finding 4.3: "CRITICAL" and imperative language throughout agent prompts

**Severity:** LOW
**Present in:** BOTH
**Files:**
- `agents/gsd-executor.md` (multiple steps)
- `agents/gsd-plan-checker.md` (role section)
- `agents/gsd-planner.md` (philosophy section)
- `commands/gsd/plan-phase.md` (step 4)
- `commands/gsd/discuss-phase.md` (scope guardrail)

**Evidence:**

From `commands/gsd/plan-phase.md`, step 4:
```
**CRITICAL:** Store `CONTEXT_CONTENT` now. It must be passed to:
- **Researcher** — constrains what to research (locked decisions vs Claude's discretion)
- **Planner** — locked decisions must be honored, not revisited
- **Checker** — verifies plans respect user's stated vision
```

From `agents/gsd-plan-checker.md`:
```
**Critical mindset:** Plans describe intent. You verify they deliver.
```

From `commands/gsd/discuss-phase.md`:
```
**CRITICAL: Scope guardrail**
```

**Analysis:**
The word "CRITICAL" appears frequently across GSD files, always instructing Claude to prioritize framework behavior. This is standard prompt engineering practice and not inherently harmful, but the cumulative effect is that GSD framework instructions are given higher priority than user instructions in Claude's attention.

In Claude's architecture, prompts marked with "CRITICAL," "MUST," and "NON-NEGOTIABLE" are treated as higher-priority instructions. When these come from the framework (not the user), they effectively create a priority hierarchy where framework > user for certain behaviors.

**Recommendation:** DOCUMENT TO USER. This is inherent to how GSD works as a meta-prompting system. Users should understand that GSD framework instructions are interpreted as system-level by Claude.

---

### Finding 4.4: No explicit "user override" mechanism

**Severity:** MODERATE
**Present in:** BOTH
**Files:**
- All agent files, all command files

**Evidence:**

Absent from all files: There is no documented mechanism for the user to say "ignore GSD workflow and just do X." No agent file contains instructions like "if the user explicitly requests to skip this step, comply." No command has a `--force` or `--skip-all-checks` flag.

The closest mechanism is `--skip-research` and `--skip-verify` flags on `/gsd:plan-phase`, but there are no equivalent flags for:
- Skipping commit conventions
- Skipping deviation rules
- Skipping scope guardrails
- Skipping requirement coverage checks
- Skipping the planning step entirely (just execute from a description)

**Analysis:**
GSD has no escape hatch. A user who wants to deviate from the workflow must either:
1. Not use GSD for that task
2. Manually edit the agent/command files to change behavior
3. Hope that their natural language instruction overrides the framework prompt

This is the most systemic autonomy issue in GSD. The framework provides many useful guardrails but no documented way to lower them when the user decides it is appropriate.

**Recommendation:** MAKE CONFIGURABLE. Add a `--override` or `--bare` flag system that lets the user bypass specific guardrails for a single invocation. Also add a `safety` config section where guardrails can be toggled:
```json
{
  "safety": {
    "auto_commit": true,
    "verify_interfaces": true,
    "verify_requirements": true,
    "scope_guardrails": true,
    "deviation_rules": "standard"
  }
}
```

---

## Section 5: Healthy vs Unhealthy Control Analysis

Categorization of all GSD control behaviors.

---

### Healthy Controls (KEEP AS-IS)

These protect the user and should remain in place.

| Control | File(s) | Why It's Healthy |
|---------|---------|------------------|
| Stage files individually, never `git add .` | `git-integration.md`, `execute-phase.md` | Prevents accidental commit of secrets, large files, or unrelated changes |
| Checkpoint for architectural changes (Rule 4) | `gsd-executor.md` | Major structural changes need user approval |
| Context compliance (Dimension 7) | `gsd-plan-checker.md` | Ensures planner doesn't override user's explicit decisions from discuss-phase |
| CONTEXT.md flow to all agents | `plan-phase.md` (upstream) | User decisions from discuss-phase are respected downstream |
| Scope creep capture (Deferred Ideas) | `discuss-phase.md` | User suggestions outside scope are saved, not lost |
| Config-based verifier/checker/research toggles | `settings.md`, `config.json` | User can disable quality agents to save tokens/time |
| User approval for roadmap | `new-project.md` (Phase 8) | User confirms roadmap before proceeding |
| Checkpoint handling in execution | `checkpoints.md`, `execute-phase.md` | User verifies visual/functional work before proceeding |
| Quality degradation curve awareness | `gsd-planner.md` | Plans are sized to prevent context exhaustion (protects output quality) |
| Audit before milestone completion | `complete-milestone.md` | Recommends (but doesn't force) audit before archiving |

---

### Unhealthy Controls (SHOULD CHANGE)

These override user preference without justification.

| Control | File(s) | Why It's Unhealthy | Recommendation |
|---------|---------|-------------------|----------------|
| Auto-commit at every step, no opt-out | `git-integration.md`, all commands | User may want to review changes before committing | MAKE CONFIGURABLE |
| Forced git init (even inside existing repos) | `new-project.md`, `git-integration.md` | User may prefer existing repo structure | MAKE CONFIGURABLE |
| "NON-NEGOTIABLE" steps in executor | `gsd-executor.md` (LOCAL ONLY) | Prevents user from opting out of verification steps | MAKE CONFIGURABLE |
| No user-override mechanism anywhere | All files | No escape hatch for any guardrail | ADD `--override` FLAGS |
| YOLO mode labeled "Recommended" | `new-project.md` | Nudges toward ceding all control without adequate warning | BETTER DOCUMENTATION |
| Scope guardrail rejects user additions | `discuss-phase.md` | Treats roadmap as more authoritative than user | ALLOW USER OVERRIDE |
| Mandatory project init for quick tasks | `quick.md` | Cannot use GSD tooling without full setup | REDUCE REQUIREMENTS |
| `git add -u` for orchestrator corrections | `execute-phase.md` | Silently stages all tracked changes | USE SPECIFIC STAGING |

---

### Gray Area Controls (CASE-BY-CASE)

These are reasonable defaults that should be overridable.

| Control | File(s) | Assessment | Recommendation |
|---------|---------|------------|----------------|
| Anti-enterprise pattern bans | `gsd-planner.md`, `gsd-roadmapper.md`, `GSD-STYLE.md` | Core identity, but "NEVER" is too strong | Soften language, allow override |
| 100% requirement coverage before roadmap | `gsd-roadmapper.md` | Good practice but blocks some workflows | Allow acknowledgment and proceed |
| Auto-fix bugs/blocking issues (Rules 1, 3) | `gsd-executor.md` | Generally safe but scope can be broad | Narrow scope for prototyping mode |
| Auto-add critical functionality (Rule 2) | `gsd-executor.md` | Proactive additions like rate limiting go beyond bug fixing | Add granularity levels |
| Commit message format | `git-integration.md`, `GSD-STYLE.md` | Conventional commits are good practice | Allow custom format template |
| Model selection without display | `execute-phase.md`, `plan-phase.md` | Cost-relevant info hidden from user | Display model on spawn |
| Plan checker iterations (max 3) | `plan-phase.md` | Quality gate, but each iteration costs tokens | Allow user to stop early |
| Technical discussion suppression in discuss-phase | `discuss-phase.md` | Works for non-technical users, bad for developers | Add persona/mode setting |
| Forced per-task atomic commits | `git-integration.md` | Good practice, but some prefer squashed commits | Allow commit strategy config |

---

## Summary: Priority Action Items

### CRITICAL (Address First)

1. **Add user-override mechanism** (Finding 4.4) -- The single most impactful change. Add `--override` flags and a `safety` config section that lets users lower specific guardrails.

2. **Make auto-commit configurable** (Finding 1.1) -- Add `git.auto_commit` to config.json. When false, stage but don't commit.

3. **Reduce executor auto-add scope** (Finding 3.1) -- Add granularity levels to deviation rules so prototyping projects don't get proactive security additions.

### MODERATE (Address Second)

4. **Soften "NON-NEGOTIABLE" language in local fork** (Finding 4.1) -- Change to "STRONGLY RECOMMENDED" and add config toggles for the two local-only verification steps.

5. **Allow scope guardrail override** (Finding 2.4) -- Let the user add features to the current phase when they explicitly want to.

6. **Display model on agent spawn** (Finding 3.2) -- Inform the user which model is being used for each agent.

7. **Fix orchestrator corrections commit** (Finding 3.5) -- Replace `git add -u` with specific file staging, and show the user what changed.

8. **Better YOLO mode documentation** (Finding 2.1) -- Rename or add detail to make clear what auto-approval means in practice.

9. **Allow technical discussion in discuss-phase** (Finding 2.3) -- Add a persona or mode setting for technical users.

### LOW (Nice to Have)

10. **Allow custom commit message format** (Finding 1.3)
11. **Soften anti-enterprise language** (Finding 2.5)
12. **Allow incomplete requirement coverage** (Finding 2.6)
13. **Cost warning before research agent spawns** (Finding 3.4)
14. **Phase auto-detection confirmation** (Finding 3.3)

---

## Findings Index

| # | Finding | Section | Severity | Location | Recommendation |
|---|---------|---------|----------|----------|----------------|
| 1.1 | Auto-commit without consent | Hard Override | CRITICAL | BOTH | MAKE CONFIGURABLE |
| 1.2 | Forced git init | Hard Override | MODERATE | BOTH | MAKE CONFIGURABLE |
| 1.3 | Forced commit message format | Hard Override | MODERATE | BOTH | MAKE CONFIGURABLE |
| 1.4 | Deviation rules auto-fix | Hard Override | MODERATE | BOTH | DOCUMENT TO USER |
| 1.5 | Forced directory structure | Hard Override | LOW | BOTH | KEEP AS-IS |
| 2.1 | "Recommended" defaults bias | Soft Override | MODERATE | BOTH | DOCUMENT TO USER |
| 2.2 | Mandatory planning workflow | Soft Override | MODERATE | BOTH | MAKE CONFIGURABLE |
| 2.3 | Technical discussion suppressed | Soft Override | MODERATE | BOTH | MAKE CONFIGURABLE |
| 2.4 | Scope guardrail rejects additions | Soft Override | MODERATE | BOTH | MAKE CONFIGURABLE |
| 2.5 | Anti-enterprise stance | Soft Override | LOW | BOTH | DOCUMENT TO USER |
| 2.6 | Coverage blocks roadmap | Soft Override | LOW | BOTH | MAKE CONFIGURABLE |
| 3.1 | Executor scope decisions | Autonomy Gap | CRITICAL | BOTH | MAKE CONFIGURABLE |
| 3.2 | Silent model selection | Autonomy Gap | MODERATE | BOTH | DOCUMENT TO USER |
| 3.3 | Phase auto-detection | Autonomy Gap | MODERATE | BOTH | DOCUMENT TO USER |
| 3.4 | Research cost not disclosed | Autonomy Gap | MODERATE | BOTH | DOCUMENT TO USER |
| 3.5 | Orchestrator corrections | Autonomy Gap | MODERATE | BOTH | REMOVE/FIX |
| 3.6 | Verification loops uncontrolled | Autonomy Gap | LOW | BOTH | DOCUMENT TO USER |
| 4.1 | "NON-NEGOTIABLE" directives | Prompt Injection | CRITICAL | LOCAL ONLY | MAKE CONFIGURABLE |
| 4.2 | Plan checker flags user choices | Prompt Injection | MODERATE | BOTH | KEEP WITH MODIFICATION |
| 4.3 | "CRITICAL" language density | Prompt Injection | LOW | BOTH | DOCUMENT TO USER |
| 4.4 | No user-override mechanism | Prompt Injection | MODERATE | BOTH | MAKE CONFIGURABLE |

---

## LOCAL-ONLY vs UPSTREAM-ONLY Findings

### LOCAL-ONLY Additions (our fork is MORE restrictive than upstream):

1. **Finding 4.1:** `verify_interfaces` and `verify_requirement_coverage` steps in `gsd-executor.md` with "NON-NEGOTIABLE" language. These do not exist in upstream.

### UPSTREAM-ONLY Features (that improve user autonomy):

1. **Context Compliance (Dimension 7)** in `gsd-plan-checker.md` -- Protects user decisions from discuss-phase. Our fork is MISSING this protection. (See Finding 4.2)
2. **CONTEXT.md flow to all agents** in `commands/gsd/plan-phase.md` -- Ensures user vision propagates. Our fork is MISSING this. (Not a finding per se, but an autonomy improvement we should adopt)
3. **Git branching strategy configuration** in `commands/gsd/settings.md` -- Gives users choice over branching approach. Our fork is MISSING this option.

### Both Codebases Share:

All other findings (1.1-1.5, 2.1-2.6, 3.1-3.6, 4.3-4.4) exist in both the local fork and upstream.

---

**End of Audit**
**Total Findings:** 21
**CRITICAL:** 3 (1.1, 3.1, 4.1)
**MODERATE:** 12 (1.2, 1.3, 1.4, 2.1, 2.2, 2.3, 2.4, 3.2, 3.3, 3.4, 3.5, 4.4)
**LOW:** 6 (1.5, 2.5, 2.6, 3.6, 4.2, 4.3)

</content>
</invoke>