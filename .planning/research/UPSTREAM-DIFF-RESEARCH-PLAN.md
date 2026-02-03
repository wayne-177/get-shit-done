# Upstream Diff Research Plan

> **Purpose:** Compare local GSD fork (based on v1.9.13) against current glittercowboy/get-shit-done upstream. Identify portable features, conflicts, and development methodology.
>
> **Date:** 2026-02-01
> **Status:** PROMPTS DRAFTED -- awaiting execution after session limit reset
> **Methodology:** Per METHODOLOGY_AND_SOP.md from prompt_studio

---

## Research Structure

**Orchestrator:** Opus (main context) -- synthesizes agent outputs, produces final deliverables
**Sub-agents:** Sonnet -- bounded comparison tasks with clear inputs/outputs
**Write pattern:** Incremental to disk after each section

### Deliverables

1. `.planning/research/01-upstream-changelog-analysis.md` -- What changed since v1.9.13
2. `.planning/research/02-file-diff-comparison.md` -- File-by-file diff of key components
3. `.planning/research/03-portability-assessment.md` -- What can be ported, what conflicts, priority ranking
4. `.planning/research/04-dev-methodology-assessment.md` -- Glittercowboy's development approach analysis
5. `.planning/research/04b-user-autonomy-audit.md` -- Where GSD overrides user direction (both codebases)
6. `.planning/research/05-gsd-porting-phase-prompt.md` -- Ready-to-use prompt for `/gsd:plan-phase`

---

## Agent 1: Upstream Changelog Analysis

**Model:** Sonnet
**Output:** `.planning/research/01-upstream-changelog-analysis.md`

### Prompt

```
You are a research agent analyzing the upstream GSD (Get Shit Done) repository changelog and release history.

**Context:**
- Our local fork is based on glittercowboy/get-shit-done v1.9.13
- The upstream repo is at: https://github.com/glittercowboy/get-shit-done
- We need to identify everything that changed AFTER v1.9.13

**Tasks:**
1. Fetch the CHANGELOG.md from the upstream repo (use the raw GitHub URL or gh CLI)
2. Fetch the package.json to identify the current upstream version
3. Extract and categorize ALL changes after v1.9.13:
   - New features/commands
   - Bug fixes
   - Breaking changes
   - Refactored/renamed components
   - New agent types or modified agent behaviors
   - New or modified skill definitions
   - Configuration changes
   - Dependency updates

4. For each change, note:
   - Version it was introduced
   - Category (feature/fix/breaking/refactor)
   - Affected files/components (if identifiable from the changelog)
   - Potential impact on our fork (HIGH/MEDIUM/LOW)

**Output format:** Markdown with sections per version, tables for categorized changes.

**IMPORTANT:** Write to disk after completing each version's analysis. Do NOT accumulate everything in memory.

**Output file:** /Users/macuser/code/get-shit-done/.planning/research/01-upstream-changelog-analysis.md
```

---

## Agent 2: File-by-File Diff Comparison

**Model:** Sonnet
**Depends on:** Agent 1 (needs to know which files changed)
**Output:** `.planning/research/02-file-diff-comparison.md`

### Prompt

```
You are a research agent comparing our local GSD fork against the upstream glittercowboy/get-shit-done repository.

**Context:**
- Read `.planning/research/01-upstream-changelog-analysis.md` first -- this tells you what changed upstream
- Our local fork is at: /Users/macuser/code/get-shit-done/
- The upstream repo is at: https://github.com/glittercowboy/get-shit-done

**Tasks:**
1. For each file/component identified in the changelog analysis, fetch the upstream version
2. Compare against our local version
3. Categorize differences into:
   - **Upstream-only changes:** New code/features we don't have
   - **Local-only changes:** Our agentic additions that upstream doesn't have
   - **Diverged:** Both sides changed the same file differently
   - **Identical:** No meaningful difference

4. Focus especially on:
   - Skill definition files (the .md prompts that define agent behaviors)
   - Core orchestration logic
   - Agent type definitions
   - Configuration/settings handling
   - Any new files upstream that we don't have at all

5. For diverged files, describe the nature of the divergence:
   - Is it a conflict (mutually exclusive changes)?
   - Is it additive (both sides added different things, could coexist)?
   - Is it a refactor (upstream restructured what we have differently)?

**IMPORTANT:** Write findings to disk after every 3-5 file comparisons. Do NOT wait until the end.

**Output file:** /Users/macuser/code/get-shit-done/.planning/research/02-file-diff-comparison.md
```

---

## Agent 3: Portability Assessment

**Model:** Sonnet
**Depends on:** Agent 1 + Agent 2
**Output:** `.planning/research/03-portability-assessment.md`

### Prompt

```
You are a research agent assessing which upstream GSD changes can be ported into our local fork.

**Context:**
- Read these files first (do NOT re-research what they cover):
  - `.planning/research/01-upstream-changelog-analysis.md`
  - `.planning/research/02-file-diff-comparison.md`
- Our fork has significant agentic additions (30+ workflows, 8+ commands, enhanced execution)
- We WANT upstream improvements but CANNOT break our custom additions

**Tasks:**
1. For each upstream change, assess portability:

   **CLEAN PORT** -- Can be brought in with no modifications
   - No file conflicts
   - Additive feature that doesn't touch our custom code
   - Bug fix that applies cleanly

   **MERGE REQUIRED** -- Can be ported but needs manual integration
   - File is diverged but changes are in different sections
   - Feature needs adaptation to work with our additions
   - Estimate complexity: LOW (< 30 min), MEDIUM (30-120 min), HIGH (> 120 min)

   **CONFLICT** -- Cannot be directly ported
   - Fundamental architectural difference
   - Upstream refactored something we also refactored differently
   - Would break our custom functionality
   - Note: what would be needed to resolve

   **SKIP** -- Not worth porting
   - We already have equivalent or better functionality
   - Change is irrelevant to our use case
   - The upstream approach is inferior to what we have

2. Produce a priority-ranked list:
   - Priority 1: High-value clean ports (do these first)
   - Priority 2: High-value merges worth the effort
   - Priority 3: Nice-to-have clean ports
   - Priority 4: Low-priority merges
   - Conflicts and skips listed separately with rationale

**IMPORTANT:** Write incrementally after each priority tier is assessed.

**Output file:** /Users/macuser/code/get-shit-done/.planning/research/03-portability-assessment.md
```

---

## Agent 4: Development Methodology Assessment

**Model:** Opus (this requires judgment and synthesis, not mechanical comparison)
**Depends on:** Agent 1 + Agent 2 (for evidence)
**Output:** `.planning/research/04-dev-methodology-assessment.md`

### Prompt

```
You are a senior software engineering analyst evaluating the development methodology and evolution of the GSD (Get Shit Done) project by glittercowboy.

**Context:**
- Read these files first:
  - `.planning/research/01-upstream-changelog-analysis.md` (version history and changes)
  - `.planning/research/02-file-diff-comparison.md` (code evolution evidence)
  - The upstream CHANGELOG.md (fetch from https://github.com/glittercowboy/get-shit-done)
  - The upstream README.md and any contributing/development docs
- Also examine the commit history and release cadence from the GitHub repo

**Analysis Framework:**

### 1. Development Methodology
- What methodology is glittercowboy following? (Agile, iterative, chaotic, etc.)
- Release cadence -- how frequently, how consistent?
- Commit patterns -- large monolithic changes vs. small incremental?
- Is there evidence of planning (roadmaps, milestones) or reactive development?
- Testing strategy -- evidence of tests, CI/CD, quality gates?
- Documentation approach -- inline, separate docs, changelog discipline?

### 2. Paradigm Shifts
- Identify any major architectural or philosophical shifts between versions
- Did the project change direction? When and why (if discernible)?
- Are there signs of learning/maturation in the approach?
- How has the agent/skill model evolved over time?

### 3. What He's Doing Right
- Specific practices that show good engineering judgment
- Smart architectural decisions
- Good developer experience choices
- Community/ecosystem awareness

### 4. What He's Doing Wrong (or Suboptimally)
- Anti-patterns or technical debt being accumulated
- Missed opportunities
- Architectural decisions that will cause pain at scale
- Documentation or communication gaps

### 5. What's Indifferent (Neutral Choices)
- Decisions that are neither clearly good nor bad
- Style/preference choices that don't materially affect quality
- Areas where multiple valid approaches exist

### 6. What Could Be Improved
- Concrete, actionable improvements (not vague suggestions)
- Things our fork does better that upstream could learn from
- Industry best practices being missed

**Evidence standard:** Every claim must reference specific versions, commits, files, or patterns. No unsupported opinions.

**IMPORTANT:** Write to disk after each major section (1-6). Do NOT accumulate.

**Output file:** /Users/macuser/code/get-shit-done/.planning/research/04-dev-methodology-assessment.md
```

---

## Agent 4b: User Autonomy Override Audit

**Model:** Opus (requires nuanced judgment about intent vs. behavior)
**Depends on:** Agent 1 + Agent 2 (for evidence from both codebases)
**Output:** `.planning/research/04b-user-autonomy-audit.md`
**Can run in parallel with:** Agent 3 and Agent 4

### Prompt

```
You are a senior analyst auditing the GSD (Get Shit Done) framework for user autonomy violations -- cases where GSD overrides, suppresses, or circumvents user direction.

**Context:**
- Read these files first:
  - `.planning/research/01-upstream-changelog-analysis.md` (what changed upstream)
  - `.planning/research/02-file-diff-comparison.md` (code in both codebases)
- Our local fork is at: /Users/macuser/code/get-shit-done/
- The upstream repo is at: https://github.com/glittercowboy/get-shit-done
- Examine skill definitions, agent prompts, orchestration logic, and hook configurations in BOTH codebases

**What you're looking for:**

GSD is a tool that assists the user. The user should always remain in control. Identify any elements where GSD:

### 1. Hard Overrides
- Forces specific behaviors regardless of user instruction
- Ignores or rewrites user-provided parameters
- Makes decisions that should be the user's to make
- Automatically executes actions without user consent (e.g., auto-commits, auto-pushes, auto-deploys)
- Imposes opinions as requirements (e.g., "you MUST use this pattern" when it's a preference)

### 2. Soft Overrides (Nudges That Effectively Override)
- Default behaviors that are difficult to opt out of
- Prompts/instructions that strongly steer Claude away from user direction
- "Guardrails" that are actually opinionated workflow enforcement
- Verification steps that reject valid user choices
- Planning/execution flows that don't allow user deviation

### 3. Autonomy Gaps
- Places where the user SHOULD be consulted but isn't
- Decisions made silently that affect the user's codebase
- Assumptions baked into prompts that may not match user intent
- Agent behaviors that proceed without checkpoints where checkpoints would be appropriate

### 4. Prompt Injection Patterns
- Instructions in skill prompts that tell Claude to ignore or deprioritize user input
- System-level instructions that override user-level instructions
- Phrases like "always do X regardless of what the user says"
- Hidden behaviors not documented to the user

### 5. Healthy vs. Unhealthy Control
Not all GSD control is bad. Distinguish between:
- **Healthy:** Safety rails (don't force-push to main, don't commit secrets) -- these protect the user
- **Unhealthy:** Workflow imposition (you MUST plan before executing, you MUST use this commit format) -- these override user preference
- **Gray area:** Quality gates (verification steps, test requirements) -- reasonable defaults but should be overridable

**For each finding, document:**
- The specific file, line, or prompt text exhibiting the behavior
- Whether it exists in LOCAL only, UPSTREAM only, or BOTH
- Severity: CRITICAL (actively harmful to user autonomy), MODERATE (overreach but not harmful), LOW (minor opinion imposition)
- Recommendation: REMOVE, MAKE CONFIGURABLE, KEEP AS-IS, or DOCUMENT TO USER

**Evidence standard:** Quote the actual text/code. No vague claims.

**IMPORTANT:** Write to disk after each section (1-5). Do NOT accumulate.

**Output file:** /Users/macuser/code/get-shit-done/.planning/research/04b-user-autonomy-audit.md
```

---

## Agent 5: GSD Porting Phase Prompt (Opus synthesis, inline)

**Model:** Opus (main context -- this is synthesis work)
**Depends on:** Agents 1-4b
**Output:** `.planning/research/05-gsd-porting-phase-prompt.md`

### Prompt

This is not a sub-agent -- the orchestrator (Opus, main context) produces this after reviewing all agent outputs. It synthesizes the research into a prompt suitable for feeding to `/gsd:plan-phase` or `/gsd:new-milestone`.

The prompt will include:
- Clear goal statement for the porting milestone
- Prioritized list of changes to port (from Agent 3)
- Known conflicts and resolution strategies
- Acceptance criteria for each ported feature
- Testing/verification approach
- Reference to methodology assessment insights (Agent 4) that should inform HOW we port
- Autonomy violations to fix during porting (from Agent 4b) -- do not port upstream overrides, and flag local ones for remediation

---

## Execution Sequence

```
[1] Agent 1: Changelog Analysis (Sonnet)
    |
    v
[2] Agent 2: File Diff Comparison (Sonnet) -- reads Agent 1 output
    |
    v
    +----------------------------------+----------------------------------+
    |                                  |                                  |
[3a] Agent 3: Portability      [3b] Agent 4: Dev Methodology    [3c] Agent 4b: User Autonomy
     Assessment (Sonnet)              (Opus)                           Audit (Opus)
     reads Agent 1+2                  reads Agent 1+2                  reads Agent 1+2
    |                                  |                                  |
    +----------------------------------+----------------------------------+
    |
    v
[4] Agent 5: GSD Prompt Synthesis (Opus, inline) -- reads ALL outputs (1, 2, 3, 4, 4b)
```

**Estimated token budget:** 350K-550K tokens total
- Agent 1: ~30-50K (changelog fetch + categorization)
- Agent 2: ~80-120K (multiple file fetches + comparisons)
- Agent 3: ~40-60K (assessment based on prior research)
- Agent 4: ~60-100K (deep analysis + web fetches for context)
- Agent 4b: ~80-120K (deep code reading across both codebases + judgment)
- Agent 5: ~20-30K (synthesis, no new research)

**Note:** Agent 4b is budget-heavy because it needs to read actual prompt/skill file contents
from both codebases, not just summarized diffs. This is intentional -- surface-level
analysis would miss the subtle override patterns.

---

## How to Execute

When session limits reset, resume this conversation (or start fresh and point to this file) and say:

> Execute the upstream diff research plan at `.planning/research/UPSTREAM-DIFF-RESEARCH-PLAN.md`

The orchestrator will launch agents in the sequence above, monitor outputs, and produce the final synthesis.

---

*Drafted: 2026-02-01*
*Status: AWAITING EXECUTION*
