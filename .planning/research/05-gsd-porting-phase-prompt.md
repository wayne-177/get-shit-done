# GSD Porting Phase Prompt

> **Purpose:** Ready-to-use prompt for `/gsd:plan-phase` or `/gsd:new-milestone` to plan the porting of upstream GSD improvements into our fork.
>
> **Generated:** 2026-02-01
> **Source Research:** Agents 1-4b (`.planning/research/01-*` through `04b-*`)
> **Synthesized by:** Opus orchestrator

---

## Milestone Goal

Port selected upstream improvements from glittercowboy/get-shit-done v1.11.1 into our local fork (based on v1.9.13), while preserving our custom agentic additions and addressing identified user autonomy issues.

---

## Phase Breakdown

### Phase 1: Critical Upstream Ports (Priority 1)

**Goal:** Bring in the 4 highest-value upstream changes that have zero conflict risk.

**Tasks:**

1. **Context bar scaling fix** (`hooks/gsd-statusline.js`)
   - Replace raw percentage calculation with upstream's 80%-scaled logic
   - Adjust color thresholds to match scaled display (63%/81%/95% instead of 50%/65%/80%)
   - Acceptance: Bar shows 100% when hitting Claude Code's 80% limit

2. **CONTEXT.md flow fix** (`commands/gsd/plan-phase.md`)
   - Add CONTEXT.md loading in Step 4 (before spawning agents)
   - Add `<phase_context>` section to researcher spawn prompt
   - Add `Phase Context` section to planner spawn prompt
   - Add CONTEXT.md to checker spawn prompt
   - Add context to revision loop prompt
   - Update success criteria to include CONTEXT.md flow verification
   - Acceptance: User decisions from `/gsd:discuss-phase` reach all downstream agents

3. **Context compliance verification** (`agents/gsd-plan-checker.md`)
   - Add `<upstream_input>` section explaining CONTEXT.md usage
   - Add "Dimension 7: Context Compliance" verification
   - Add red flags for: decision without implementing task, task contradicting decision, deferred idea inclusion
   - Update success criteria
   - Acceptance: Plans that violate user decisions are flagged by checker
   - Dependency: Task 2 must complete first

4. **Context file detection improvement** (location TBD -- research during execution)
   - Improve glob patterns for finding CONTEXT.md variants
   - Acceptance: CONTEXT.md found regardless of naming variations

**Estimated effort:** ~4 hours total
**Risk:** LOW -- all changes are additive, no conflicts with our custom code

---

### Phase 2: User Autonomy Improvements

**Goal:** Address the 3 CRITICAL autonomy findings from the audit (04b) to ensure GSD serves the user rather than overriding them.

**Tasks:**

1. **Make auto-commit configurable**
   - Add `git.auto_commit` boolean to `get-shit-done/templates/config.json`
   - Add auto-commit question to `/gsd:settings` (commands/gsd/settings.md)
   - Update `get-shit-done/references/git-integration.md` to respect the setting
   - When `false`: stage files but prompt user before committing
   - Acceptance: User can set `git.auto_commit: false` and GSD will not auto-commit

2. **Soften "NON-NEGOTIABLE" directives in executor** (LOCAL ONLY fix)
   - In `agents/gsd-executor.md`, change `verify_interfaces` from "NON-NEGOTIABLE" to "STRONGLY RECOMMENDED"
   - Change `verify_requirement_coverage` from "NON-NEGOTIABLE" to "STRONGLY RECOMMENDED"
   - Add config toggles: `safety.verify_interfaces` and `safety.verify_requirements`
   - Acceptance: User can disable these checks via config when appropriate (e.g., rapid prototyping)

3. **Add user-override mechanism**
   - Add `safety` section to config.json template with toggleable guardrails
   - Document override capabilities in a reference doc
   - Ensure agents check config before enforcing non-safety guardrails
   - Acceptance: Users have a documented escape hatch for workflow guardrails

**Estimated effort:** ~6 hours total
**Risk:** MEDIUM -- changes touch core workflow files, need careful testing

---

### Phase 3: Git Branching Strategy (Priority 2)

**Goal:** Port upstream's git branching configuration and evaluate implementation completeness.

**Tasks:**

1. **Research: Locate branching implementation**
   - Search upstream for all references to `branching_strategy`
   - Identify which files consume the config (execute-phase? executor agent? git-integration?)
   - Determine if squash merge at milestone is implemented or planned
   - Output: List of all files needed for complete feature

2. **Port git branching config UI** (`commands/gsd/settings.md`)
   - Add 5th question: branching strategy (none/phase/milestone)
   - Add git section to config.json update logic
   - Update confirmation display
   - Acceptance: `/gsd:settings` shows branching option, saves to config

3. **Port git branching execution** (files TBD from research)
   - Apply branching logic during phase/milestone execution
   - Test with our custom verification steps
   - Acceptance: Branches created per strategy, our verifications still work

4. **Port squash merge** (if implementation exists)
   - Apply squash merge option to milestone completion
   - Acceptance: User can choose squash vs. full history at milestone completion

**Estimated effort:** 4-6 hours (depends on research findings)
**Risk:** MEDIUM -- need to discover implementation location first
**Blocker:** Task 1 must complete before Tasks 2-4 can be scoped

---

## What NOT to Port

Per portability assessment (03), skip these 8 upstream changes:
- Gemini CLI support (v1.10.0, v1.10.1) -- multi-LLM complexity not needed
- Discord integration (v1.9.9, v1.9.10) -- community feature, not relevant
- OpenCode fixes (v1.9.7) -- not in scope
- `--all` install flag (v1.10.0) -- multi-platform, not needed
- CI/CD changes (v1.9.11, v1.9.12) -- infrastructure, separate process
- Command removal (v1.9.12) -- already N/A
- Discord badge (v1.9.11) -- documentation only

---

## What NOT to Blindly Port (Autonomy Audit Findings)

Per autonomy audit (04b), do NOT port upstream behaviors that override user direction:
- Do not adopt upstream's pattern of auto-committing without consent unless behind a config toggle
- Do not port scope guardrails without adding override capability
- When porting the plan checker's Dimension 7 (context compliance), ensure it routes back to user when a locked decision is technically infeasible rather than silently failing

---

## Testing Strategy

### Per-Phase Verification:

**Phase 1 tests:**
- Fill context to ~40%, ~65%, ~80% -- verify bar displays scaled correctly
- Run `/gsd:discuss-phase` with explicit decisions, then `/gsd:plan-phase` -- verify CONTEXT.md appears in all agent prompts
- Create a plan that violates a user decision -- verify checker flags it in Dimension 7
- Create a plan that includes a deferred idea -- verify checker flags it

**Phase 2 tests:**
- Set `git.auto_commit: false` -- run `/gsd:execute-phase` -- verify staging without auto-commit
- Set `safety.verify_interfaces: false` -- run executor -- verify step is skipped
- Verify safety controls can be toggled independently

**Phase 3 tests:**
- Set branching to "phase" -- execute a phase -- verify branch created as `gsd/phase-{N}-{slug}`
- Set branching to "milestone" -- verify branch created as `gsd/{version}-{slug}`
- Set branching to "none" -- verify commits go to current branch (existing behavior)
- Run with our custom verifications enabled -- verify no regressions

---

## Risk Mitigation

1. **Backup branch before any changes:** `git checkout -b backup-pre-upstream-merge`
2. **Feature branch per phase:** `feat/upstream-phase-1`, `feat/upstream-phase-2`, etc.
3. **Merge Phase 1 first, use for several days, then proceed to Phase 2**
4. **Phase 3 blocked on research** -- do not start until implementation location is confirmed

---

## Success Criteria

- [ ] All 4 Priority 1 changes ported and working (context bar, CONTEXT.md flow, compliance, detection)
- [ ] 3 CRITICAL autonomy issues addressed (auto-commit configurable, NON-NEGOTIABLE softened, override mechanism exists)
- [ ] Git branching config ported (execution porting conditional on research)
- [ ] Zero regressions in existing workflows (requirements verification, interface verification still work)
- [ ] All changes documented
- [ ] Clean git history (atomic commits per logical change)

---

## Reference Documents

| Document | Path | Purpose |
|----------|------|---------|
| Changelog Analysis | `.planning/research/01-upstream-changelog-analysis.md` | What changed upstream since v1.9.13 |
| File Diff Comparison | `.planning/research/02-file-diff-comparison.md` | Line-level differences per file |
| Portability Assessment | `.planning/research/03-portability-assessment.md` | What can be ported, priority ranking, change cards |
| Dev Methodology | `.planning/research/04-dev-methodology-assessment.md` | How glittercowboy develops, strengths/weaknesses |
| Autonomy Audit | `.planning/research/04b-user-autonomy-audit.md` | Where GSD overrides user direction |
| Research Plan | `.planning/research/UPSTREAM-DIFF-RESEARCH-PLAN.md` | Original research plan and agent prompts |

---

*Generated: 2026-02-01*
*Ready for: `/gsd:plan-phase` or `/gsd:new-milestone`*
