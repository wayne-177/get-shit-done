# Portability Assessment: Upstream Changes to Local Fork

**Assessment Date:** 2026-02-01
**Analyst:** Research Agent (Claude Sonnet 4.5)
**Our Fork Base:** v1.9.13
**Target Upstream:** v1.11.1
**Method:** Line-by-line diff analysis + changelog review

---

## Executive Summary

**Total Changes Analyzed:** 15 distinct changes from v1.9.12 through v1.11.1

**Portability Breakdown:**
- **CLEAN PORT:** 4 changes (can be applied directly)
- **MERGE REQUIRED:** 3 changes (need manual integration)
- **CONFLICT:** 0 changes (no fundamental conflicts found)
- **SKIP:** 8 changes (not worth porting)

**Critical Finding:** Our fork is based on v1.9.13 which doesn't exist in upstream. Despite this, file comparison shows we're remarkably close to upstream - most changes are missing upstream features we never had, not divergent implementations.

**Key Insight:** Our "agentic additions" are NOT in the core GSD files we compared. The changes are either:
1. Completely separate files we added (requirements verification, interface verification)
2. Files in `.planning/` directory
3. Not yet implemented

This means upstream changes port cleanly - there are NO conflicts with our custom work because our custom work is in different files.

---

## Portability Categories Defined

### CLEAN PORT
Can be brought in with no modifications. Criteria:
- No file conflicts with our custom code
- Additive feature that doesn't touch our additions
- Bug fix that applies cleanly to identical code

### MERGE REQUIRED
Can be ported but needs manual integration. Criteria:
- File exists but we might have path style differences
- Feature needs testing to ensure compatibility
- Low/Medium/High complexity estimate

### CONFLICT
Cannot be directly ported. Criteria:
- Fundamental architectural difference
- Would break our custom functionality
- Requires resolution plan

### SKIP
Not worth porting. Criteria:
- We already have equivalent or better
- Change is irrelevant to our use case
- Upstream approach is inferior

---

## Priority 1: High-Value Clean Ports

**Do these FIRST - high impact, zero risk**

### 1.1 CONTEXT.md Flow Fix (v1.11.1)

**What it does:**
Fixes critical bug where CONTEXT.md from `/gsd:discuss-phase` wasn't properly flowing to downstream agents (researcher, planner, checker, revision loop). Now loads CONTEXT.md early in Step 4 and passes it to ALL agents with proper usage instructions.

**Files affected:**
- `commands/gsd/plan-phase.md`
- `agents/gsd-plan-checker.md`

**Portability:** CLEAN PORT

**Rationale:**
- File comparison shows our local versions are MISSING these features, not diverged
- No conflicts - these are pure additions to existing logic
- We already have CONTEXT.md support in agents, just missing the orchestration
- Our custom additions (requirements/interface verification) are in different files

**Estimated effort:** 2 hours
- Update plan-phase.md Step 4 to load CONTEXT.md early
- Add CONTEXT.md to all agent spawn prompts (researcher, planner, checker, revision)
- Add success criteria updates

**Risk level:** LOW
- Changes are additive
- No intersection with our custom verification files
- Well-documented in upstream

**Dependencies:** None

**Impact:** CRITICAL
- Without this, user decisions from discuss-phase don't constrain downstream work
- Plans can contradict user's explicit requests
- Context compliance verification (Priority 1.2) depends on this

**Testing:**
1. Run `/gsd:discuss-phase` → verify CONTEXT.md created
2. Run `/gsd:plan-phase` → verify CONTEXT.md loaded in Step 4
3. Check agent prompts → verify CONTEXT.md passed with instructions
4. Verify plans honor locked decisions from CONTEXT.md

---

### 1.2 Context Compliance Verification (v1.11.1)

**What it does:**
Adds "Dimension 7: Context Compliance" to plan checker agent. Validates that plans don't contradict user decisions from CONTEXT.md. Checks:
- Locked decisions have implementing tasks
- No tasks contradict locked decisions
- Deferred ideas not included in plans
- Discretion areas handled appropriately

**Files affected:**
- `agents/gsd-plan-checker.md`

**Portability:** CLEAN PORT

**Rationale:**
- Our local gsd-plan-checker.md only has 6 dimensions, stops at "Dimension 6: Verification Derivation"
- This is a completely new dimension, no overlap with existing code
- No conflicts with our requirements/interface verification (those are separate files)

**Estimated effort:** 1 hour
- Add `<upstream_input>` section explaining CONTEXT.md structure
- Add "Dimension 7: Context Compliance" after Dimension 6
- Update verification process to extract CONTEXT.md sections
- Add success criteria for context compliance

**Risk level:** LOW
- Pure addition to existing agent
- Well-defined verification dimension
- Follows same pattern as existing dimensions

**Dependencies:**
- **REQUIRED:** Priority 1.1 (CONTEXT.md flow fix) must be done first
- Without 1.1, CONTEXT.md won't be passed to checker

**Impact:** HIGH
- Quality improvement - prevents plans from contradicting user vision
- Completes the CONTEXT.md workflow from discuss-phase → plan-phase → verification
- Makes `/gsd:discuss-phase` decisions actually meaningful

**Testing:**
1. Create CONTEXT.md with locked decisions and deferred ideas
2. Run plan-phase and intentionally create plan that violates decisions
3. Verify checker flags the violation in Dimension 7
4. Fix plan and verify checker passes

---

### 1.3 Context Bar Display Fix (v1.10.0)

**What it does:**
Fixes context bar to show 100% when you hit Claude Code's 80% limit (instead of showing 80%). Adds scaling logic: `displayed = (raw_usage / 80) * 100`. Adjusts color thresholds accordingly.

**Files affected:**
- `hooks/gsd-statusline.js`

**Portability:** CLEAN PORT

**Rationale:**
- Pure JavaScript calculation fix
- No dependencies on other GSD features
- Our file is identical to upstream except missing this fix
- No custom code in statusline hook

**Estimated effort:** 30 minutes
- Replace lines 21-40 in gsd-statusline.js
- Update context calculation to use scaling
- Adjust color thresholds (50/65/80 → 63/81/95)
- Add comment explaining 80% limit

**Risk level:** VERY LOW
- Isolated change in display logic
- No impact on functionality, only UX
- Easy to revert if issues

**Dependencies:** None

**Impact:** MEDIUM
- UX improvement - more intuitive context usage display
- Users see "100%" when they hit the limit, not "80%"
- Better matches user mental model

**Testing:**
1. Fill context to ~40% real usage → should show ~50% (green)
2. Fill context to ~65% real usage → should show ~81% (orange)
3. Fill context to ~80% real usage → should show ~100% (red)
4. Verify color transitions at new thresholds

---

### 1.4 Context File Detection Fix (v1.9.8)

**What it does:**
Improves context file detection to match filename variants. Handles both `CONTEXT.md` and `{phase}-CONTEXT.md` patterns correctly.

**Files affected:**
- File detection logic across codebase (grep for file detection patterns)

**Portability:** CLEAN PORT (if we can identify the file)

**Rationale:**
- Bug fix for pattern matching
- Should be isolated change
- No conflicts expected

**Estimated effort:** Unknown (need to find the file first)
- Identify which file contains context detection logic
- Compare local vs upstream
- Apply pattern matching fix

**Risk level:** LOW
- Bug fix, not feature change
- Pattern matching improvement

**Dependencies:** None

**Impact:** MEDIUM
- Prevents context file detection failures
- Makes CONTEXT.md workflow more robust

**Note:** Need to identify specific file location. Likely in commands or workflows that read CONTEXT.md.

---

## Priority 1 Summary

**Total changes:** 4
**Total effort:** ~4 hours
**Risk:** LOW to VERY LOW
**Impact:** CRITICAL to MEDIUM

**Recommended order:**
1. Context bar fix (30 min, very low risk, standalone)
2. CONTEXT.md flow fix (2 hrs, foundational for #3)
3. Context compliance verification (1 hr, depends on #2)
4. Context file detection fix (unknown, after identifying file)

---

## Priority 2: High-Value Merges Worth the Effort

**Do these AFTER Priority 1 - high value but need manual work**

### 2.1 Git Branching Strategy Configuration (v1.11.1)

**What it does:**
Adds git branching strategy configuration to `/gsd:settings`. Three options:
- `none` (default): Commit directly to current branch
- `phase`: Create branch per phase (`gsd/phase-{N}-{slug}`)
- `milestone`: Create branch per milestone (`gsd/{version}-{slug}`)

Stores in config.json under `git.branching_strategy`. Integrates with execution workflow and milestone completion.

**Files affected:**
- `commands/gsd/settings.md`
- Configuration system (config.json updated dynamically)
- Potentially: `commands/gsd/complete-milestone.md` (for branch merging)
- Potentially: `get-shit-done/workflows/execute-phase.md` (for branch creation)

**Portability:** MERGE REQUIRED

**Complexity:** LOW

**Rationale:**
- Our settings.md is missing the git branching question (has 4 questions, upstream has 5)
- Need to add 5th question to AskUserQuestion
- Need to add git section to config.json update
- Need to add git setting to confirmation display
- File comparison shows complete-milestone.md and execute-phase.md are IDENTICAL between local/upstream
  - This suggests git branching logic might be implemented dynamically during execution
  - Or upcoming in a file we haven't compared yet
  - Or the feature is partially implemented (config exists but not used yet)

**Estimated effort:** 2 hours
- Add git branching question to settings.md (~30 min)
- Add git config writing logic (~30 min)
- Test settings flow (~30 min)
- Research where branching strategy is USED during execution (~30 min)
  - Need to check if there are other files that consume this config
  - May need to port additional logic beyond settings.md

**Risk level:** MEDIUM
- Config change is low risk (additive)
- BUT: We don't know if our custom additions interact with git workflow
- Need to verify our requirements/interface verification doesn't conflict with git branching
- Need to test that branching doesn't break atomic commits

**Dependencies:**
- None for config UI
- Unknown dependencies for actual branching implementation

**Impact:** HIGH
- Major workflow feature for team-based usage
- Enables cleaner git history management
- Per-phase branches useful for PRs and code review

**Testing:**
1. Run `/gsd:settings` → verify git branching question appears
2. Select each option (none/phase/milestone) → verify saved to config.json
3. Run execution with each strategy → verify branches created correctly
4. Complete milestone → verify branch handling
5. Verify requirements/interface verification still works with branching

**Unknown factors:**
- Where is git branching strategy CONSUMED? (execute-phase.md is identical local/upstream but should have branching logic)
- Does squash merge option (v1.11.1) integrate with this? (Agent 1 mentioned it but file comparison shows no difference)
- Are there additional files we need to port for full feature?

**Action before merge:**
- Search local and upstream codebases for `branching_strategy` usage
- Identify ALL files that reference this config
- Determine if we're missing implementation files

---

### 2.2 Squash Merge Option (v1.11.1)

**What it does:**
At milestone completion, offers squash merge (recommended) or merge-with-history as alternatives for integrating milestone branch back to main.

**Files affected:**
- `commands/gsd/complete-milestone.md` (supposedly)
- `get-shit-done/workflows/complete-milestone.md` (supposedly)

**Portability:** MERGE REQUIRED (or SKIP - unclear)

**Complexity:** UNKNOWN

**Rationale:**
- Agent 1's changelog analysis says this feature exists in v1.11.1
- But file comparison shows complete-milestone.md and workflow are IDENTICAL local/upstream
- This suggests either:
  1. Feature implemented dynamically during execution (not in markdown)
  2. Feature not yet in the files we compared
  3. Feature is in a different file
  4. Changelog was misleading

**Estimated effort:** UNKNOWN
- Need to identify where this feature actually exists

**Risk level:** UNKNOWN

**Dependencies:**
- Likely depends on git branching strategy (Priority 2.1)

**Impact:** MEDIUM (if we can find it)
- Cleaner git history with squash merge
- Optional feature - doesn't break anything if skipped

**Action before assessment:**
- Search upstream for "squash" keyword
- Search upstream for milestone completion logic
- Determine if feature exists or was planned but not implemented

**Current status:** BLOCKED - Need to locate feature

---

## Priority 2 Summary

**Total changes:** 2
**Total effort:** 2+ hours (one blocked)
**Risk:** MEDIUM to UNKNOWN
**Impact:** HIGH to MEDIUM

**Recommended order:**
1. Git branching strategy config (after researching usage)
2. Squash merge option (after finding it)

**Blockers:**
- Need to identify where git branching is actually USED
- Need to locate squash merge implementation

---

## Priority 3: Nice-to-Have Clean Ports

**Do these if time permits - low impact, low effort**

### 3.1 Uninstall Flag (v1.9.8)

**What it does:**
Adds `--uninstall` flag to installer for clean removal of GSD from global or local installations.

**Files affected:**
- `bin/install.js`

**Portability:** CLEAN PORT (probably)

**Rationale:**
- Installer feature, unlikely to conflict with anything
- We haven't modified the installer for our fork
- Should be simple addition to install.js

**Estimated effort:** 15 minutes
- Review upstream bin/install.js for --uninstall logic
- Copy to local
- Test uninstall flow

**Risk level:** VERY LOW
- Installer only, no runtime impact
- Easy to test and revert

**Dependencies:** None

**Impact:** LOW
- Convenience feature for users
- Nice QoL improvement
- Not critical functionality

---

### 3.2 Discord Community Link in Installer (v1.9.10)

**What it does:**
Shows Discord community link in installer completion message.

**Files affected:**
- `bin/install.js`

**Portability:** SKIP

**Rationale:**
- We're a fork, not the official GSD
- Discord link would point to upstream community
- Not relevant for our users unless we have our own Discord

**Impact:** NONE

---

## Priority 3 Summary

**Total changes:** 2 (1 port, 1 skip)
**Total effort:** ~15 minutes
**Risk:** VERY LOW
**Impact:** LOW

---

## Priority 4: Low-Priority Merges and SKIPs

**Do these last or never - low value**

### 4.1 `/gsd:join-discord` Command (v1.9.9)

**What it does:**
Adds command to show Discord community invite link.

**Files affected:**
- `commands/gsd/join-discord.md`

**Portability:** SKIP

**Rationale:**
- Community feature for upstream
- Not relevant for our fork
- Would need to maintain separate Discord or remove command

**Impact:** NONE for our fork

---

### 4.2 Gemini CLI Support (v1.10.0)

**What it does:**
Adds native Gemini CLI support. Install with `--gemini` flag or select from interactive menu.

**Files affected:**
- `bin/install.js`
- CLI integration code
- Agent loading

**Portability:** SKIP

**Rationale:**
- We're focused on Claude Code integration
- Gemini support adds complexity without value for our use case
- Our fork's agentic additions are Claude-specific

**Impact:** NONE (unless we want multi-LLM support)

---

### 4.3 Gemini CLI Bug Fixes (v1.10.1)

**What it does:**
Fixes errors that prevented Gemini commands from executing.

**Files affected:**
- Gemini CLI integration
- Agent loading

**Portability:** SKIP

**Rationale:**
- Depends on Gemini CLI support (4.2) which we're skipping
- No value without Gemini integration

**Impact:** NONE

---

### 4.4 `--all` Installation Flag (v1.10.0)

**What it does:**
Install for Claude Code, OpenCode, and Gemini simultaneously.

**Files affected:**
- `bin/install.js`

**Portability:** SKIP

**Rationale:**
- Multi-platform feature
- Our fork is Claude Code specific
- Adds unnecessary complexity

**Impact:** NONE

---

### 4.5 OpenCode Path Fixes (v1.9.7)

**What it does:**
Fixes OpenCode installer to use XDG-compliant paths (`~/.config/opencode/`) and correct command structure.

**Files affected:**
- `bin/install.js` (OpenCode-specific logic)

**Portability:** SKIP

**Rationale:**
- Our fork targets Claude Code
- OpenCode support not in scope
- No OpenCode users in our fork

**Impact:** NONE

---

### 4.6 `/gsd:whats-new` Command Removal (v1.9.12)

**What it does:**
Removes `/gsd:whats-new` command, replaced by `/gsd:update` (which shows changelog with cancel option).

**Files affected:**
- `commands/gsd/whats-new.md` (deleted)
- `commands/gsd/update.md` (enhanced)

**Portability:** SKIP (probably already handled)

**Rationale:**
- Our fork is based on v1.9.13 (post v1.9.12)
- We likely never had whats-new command
- Update command consolidation already happened

**Impact:** NONE (likely already N/A)

**Action:** Verify we don't have whats-new.md file

---

### 4.7 CI/CD Changes (v1.9.11, v1.9.12)

**What it does:**
- v1.9.11: Switch to manual npm publish workflow
- v1.9.12: Restore GitHub Actions auto-release workflow

**Files affected:**
- `.github/workflows/` (CI/CD configs)
- `scripts/build-hooks.js`

**Portability:** SKIP

**Rationale:**
- Development infrastructure
- Our fork has separate release process
- Upstream CI/CD not relevant to runtime

**Impact:** NONE

---

### 4.8 Discord Badge Fix (v1.9.11)

**What it does:**
Updates Discord badge in README to use static format for reliable rendering.

**Files affected:**
- `README.md`

**Portability:** SKIP

**Rationale:**
- Documentation change
- Our fork has separate README
- Discord badge not relevant

**Impact:** NONE

---

## Priority 4 Summary

**Total changes:** 8
**Ports:** 0
**Skips:** 8
**Rationale:** All are either community features, multi-platform support, or infrastructure changes irrelevant to our Claude Code-focused fork.

---

## Conflicts and Blockers

### No Direct Conflicts Found

**Critical Finding:** Zero architectural conflicts between upstream changes and our fork.

**Why?**
1. Our "agentic additions" are in SEPARATE files:
   - `.planning/issues/002-requirements-verification.md`
   - Interface verification files (referenced but not compared)
   - Custom workflows we added

2. Upstream changes are in CORE GSD files:
   - `commands/gsd/*.md`
   - `agents/*.md`
   - `get-shit-done/workflows/*.md`
   - `hooks/*.js`

3. No overlap = No conflicts

**Path style difference (NOT a conflict):**
- Our fork uses absolute paths: `/Users/macuser/.claude/...`
- Upstream uses tilde paths: `~/.claude/...`
- This is cosmetic, not functional
- Likely from our local installation path
- Can keep as-is or normalize to tilde

---

### Unknown Factors (Research Needed)

**1. Git Branching Implementation Location**

**Issue:** Settings.md adds config for git branching, but WHERE is it consumed?

**Evidence:**
- `execute-phase.md` is IDENTICAL local/upstream
- `complete-milestone.md` is IDENTICAL local/upstream
- Yet changelog says branching feature exists in v1.11.1

**Hypothesis:**
1. Feature is partially implemented (config exists but not used)
2. Implementation is in files we didn't compare
3. Dynamic execution in Claude's runtime (not in markdown)

**Action required:**
- Search upstream for `branching_strategy` or `git.branching`
- Check executor.md, other agent files
- May need to port additional files beyond what Agent 2 compared

**Risk if we skip:** LOW
- Config UI works standalone
- Users can set preference even if not used
- Execution still works (defaults to "none" behavior)

---

**2. Squash Merge Feature Location**

**Issue:** Changelog says v1.11.1 added squash merge at milestone completion, but files are identical.

**Evidence:**
- `complete-milestone.md` command: IDENTICAL
- `workflows/complete-milestone.md`: IDENTICAL
- Both are 903 lines, same content

**Hypothesis:**
1. Feature added to changelog but not implemented
2. Feature in different file
3. Dynamic behavior during milestone completion

**Action required:**
- Search upstream for "squash"
- Check if feature exists or was planned
- May be false positive from changelog

**Risk if we skip:** NONE
- Feature may not exist yet
- Or exists but we can't find it
- Not blocking other changes

---

### Verification Needed: Custom Additions Compatibility

**Our custom additions (not in upstream):**
1. Requirements verification (002-requirements-verification.md)
2. Interface verification (mentioned in git status)

**Questions:**
1. Do our verification files integrate with plan-phase workflow?
2. Will CONTEXT.md flow changes affect our verifications?
3. Do git branching changes impact our atomic commit requirements?

**Testing required:**
1. Port Priority 1 changes
2. Test with requirements verification enabled
3. Test with interface verification enabled
4. Verify no regressions in our custom workflows

**Expected result:** NO conflicts
- Our verifications appear to be additional checks in planning
- CONTEXT.md flow happens in plan-phase orchestration
- Our checks likely run separately
- But need to verify

---

## Priority-Ranked Implementation List

### Do First (Priority 1: High-Value Clean Ports)

| # | Change | Files | Effort | Risk | Impact | Dependencies |
|---|--------|-------|--------|------|--------|--------------|
| 1.3 | Context bar fix | `hooks/gsd-statusline.js` | 30m | VERY LOW | MEDIUM | None |
| 1.1 | CONTEXT.md flow | `commands/gsd/plan-phase.md` | 2h | LOW | CRITICAL | None |
| 1.2 | Context compliance | `agents/gsd-plan-checker.md` | 1h | LOW | HIGH | 1.1 |
| 1.4 | Context file detection | TBD | TBD | LOW | MEDIUM | None |

**Total Priority 1:** ~4 hours, CRITICAL impact

---

### Do Second (Priority 2: High-Value Merges)

| # | Change | Files | Effort | Risk | Impact | Dependencies |
|---|--------|-------|--------|------|--------|--------------|
| 2.1 | Git branching config | `commands/gsd/settings.md` + TBD | 2h+ | MEDIUM | HIGH | Research needed |
| 2.2 | Squash merge | TBD | TBD | UNKNOWN | MEDIUM | 2.1, Research needed |

**Total Priority 2:** 2+ hours, HIGH impact
**Blockers:** Need to locate implementation files

---

### Do If Time (Priority 3: Nice-to-Have)

| # | Change | Files | Effort | Risk | Impact | Dependencies |
|---|--------|-------|--------|------|--------|--------------|
| 3.1 | Uninstall flag | `bin/install.js` | 15m | VERY LOW | LOW | None |

**Total Priority 3:** 15 minutes, LOW impact

---

### Skip (Priority 4: Not Worth Porting)

| # | Change | Reason |
|---|--------|--------|
| 4.1 | `/gsd:join-discord` | Community feature, not relevant to fork |
| 4.2 | Gemini CLI support | Multi-LLM complexity, not needed |
| 4.3 | Gemini bug fixes | Depends on 4.2 |
| 4.4 | `--all` install flag | Multi-platform, not in scope |
| 4.5 | OpenCode fixes | OpenCode support not in scope |
| 4.6 | Remove `/gsd:whats-new` | Already N/A (post-v1.9.12) |
| 4.7 | CI/CD changes | Infrastructure, separate release process |
| 4.8 | Discord badge | Documentation, fork has own README |

**Total Priority 4:** 8 changes skipped

---

## Detailed Change Cards

### For easy reference during implementation

---

#### Change Card: 1.1 CONTEXT.md Flow Fix

**Category:** CLEAN PORT
**Priority:** 1 (CRITICAL)
**Effort:** 2 hours
**Risk:** LOW

**What:** Fix critical bug where CONTEXT.md from discuss-phase doesn't flow to downstream agents.

**Why port:** Without this, user decisions from `/gsd:discuss-phase` are ignored by planner and checker.

**Files to modify:**
- `/Users/macuser/code/get-shit-done/commands/gsd/plan-phase.md`

**Changes needed:**
1. **Step 4:** Add CONTEXT.md loading
   ```bash
   # Load CONTEXT.md immediately
   CONTEXT_CONTENT=$(cat "${PHASE_DIR}"/*-CONTEXT.md 2>/dev/null)
   ```

2. **Research prompt (Line ~182):** Add phase_context section
   ```markdown
   <phase_context>
   **IMPORTANT:** If CONTEXT.md exists, it contains user decisions.
   - Decisions = Locked, research these deeply
   - Claude's Discretion = Research options
   - Deferred Ideas = Ignore
   {context_content}
   </phase_context>
   ```

3. **Planner prompt (Line ~282):** Add Phase Context section
   ```markdown
   **Phase Context (if exists):**
   IMPORTANT: User decisions from /gsd:discuss-phase
   - Decisions = LOCKED
   - Claude's Discretion = Your freedom
   - Deferred Ideas = Out of scope
   {context_content}
   ```

4. **Checker prompt (Line ~367):** Add CONTEXT.md
   ```markdown
   **Phase Context (if exists):**
   Plans MUST honor these decisions.
   {context_content}
   ```

5. **Revision prompt (Line ~427):** Add context
   ```markdown
   **Phase Context:** Revisions must honor user decisions.
   {context_content}
   ```

6. **Success criteria:** Add CONTEXT.md flow checks

**Testing:**
1. Run `/gsd:discuss-phase` with decisions
2. Run `/gsd:plan-phase`
3. Verify CONTEXT.md loaded in Step 4 output
4. Check researcher prompt has phase_context
5. Check planner prompt has Phase Context
6. Check plans honor locked decisions

**Dependencies:** None

---

#### Change Card: 1.2 Context Compliance Verification

**Category:** CLEAN PORT
**Priority:** 1 (HIGH)
**Effort:** 1 hour
**Risk:** LOW

**What:** Add "Dimension 7: Context Compliance" to plan checker to verify plans don't contradict user decisions.

**Why port:** Quality improvement - prevents plans from violating user's explicit requests.

**Files to modify:**
- `/Users/macuser/code/get-shit-done/agents/gsd-plan-checker.md`

**Changes needed:**
1. **After line 24:** Add to role description
   ```markdown
   - Plans contradict user decisions from CONTEXT.md
   ```

2. **After line 26:** Add `<upstream_input>` section
   ```markdown
   <upstream_input>
   **CONTEXT.md** (if exists) — User decisions from /gsd:discuss-phase

   | Section | How You Use It |
   |---------|----------------|
   | Decisions | LOCKED — plans MUST implement exactly |
   | Claude's Discretion | Freedom — don't flag |
   | Deferred Ideas | Out of scope — flag if present |

   If CONTEXT.md exists, add verification dimension: Context Compliance
   </upstream_input>
   ```

3. **After Dimension 6:** Add entire Dimension 7
   ```markdown
   ## Dimension 7: Context Compliance (if CONTEXT.md exists)

   **Question:** Do plans honor user decisions?

   **Process:**
   1. Parse CONTEXT.md sections
   2. Verify decisions have implementing tasks
   3. Verify no deferred ideas included
   4. Verify discretion areas handled

   **Red flags:**
   - Decision has no task
   - Task contradicts decision
   - Task implements deferred idea
   [... full dimension text from upstream]
   ```

4. **Step 1 (Line ~244):** Update to mention CONTEXT.md parsing

5. **Success criteria:** Add context compliance checks

**Testing:**
1. Create CONTEXT.md with:
   - Decision: "Use cards layout"
   - Deferred: "Add dark mode"
2. Create plan that violates:
   - Uses table layout (contradicts decision)
   - Includes dark mode (deferred idea)
3. Run checker → verify flags both issues in Dimension 7
4. Fix plan → verify checker passes

**Dependencies:** Requires 1.1 (CONTEXT.md flow) first

---

#### Change Card: 1.3 Context Bar Display Fix

**Category:** CLEAN PORT
**Priority:** 1 (MEDIUM)
**Effort:** 30 minutes
**Risk:** VERY LOW

**What:** Scale context bar to show 100% at 80% limit (Claude Code's enforcement point).

**Why port:** Better UX - users see "100%" when they hit the limit, not "80%".

**Files to modify:**
- `/Users/macuser/code/get-shit-done/hooks/gsd-statusline.js`

**Changes needed:**
Replace lines 21-40 with upstream scaling logic:

```javascript
// Context window display (shows USED percentage scaled to 80% limit)
// Claude Code enforces an 80% context limit, so we scale to show 100% at that point
let ctx = '';
if (remaining != null) {
  const rem = Math.round(remaining);
  const rawUsed = Math.max(0, Math.min(100, 100 - rem));
  // Scale: 80% real usage = 100% displayed
  const used = Math.min(100, Math.round((rawUsed / 80) * 100));

  const bar = '█'.repeat(Math.floor(used / 5));

  // Color based on scaled usage (thresholds adjusted for new scale)
  if (used < 63) {        // ~50% real
    ctx = ` \x1b[32m${bar} ${used}%\x1b[0m`;
  } else if (used < 81) { // ~65% real
    ctx = ` \x1b[33m${bar} ${used}%\x1b[0m`;
  } else if (used < 95) { // ~76% real
    ctx = ` \x1b[38;5;208m${bar} ${used}%\x1b[0m`;
  } else {                // approaching limit
    ctx = ` \x1b[31m${bar} ${used}%\x1b[0m`;
  }
}
```

**Testing:**
1. Fill context to ~40% real → expect ~50% displayed (green)
2. Fill context to ~65% real → expect ~81% displayed (orange)
3. Fill context to ~80% real → expect ~100% displayed (red)
4. Verify color transitions correct

**Dependencies:** None

---

#### Change Card: 2.1 Git Branching Strategy Config

**Category:** MERGE REQUIRED
**Priority:** 2 (HIGH)
**Effort:** 2+ hours
**Risk:** MEDIUM
**Complexity:** LOW

**What:** Add git branching strategy configuration to settings command.

**Why port:** Major workflow feature for cleaner git history and team collaboration.

**Files to modify:**
- `/Users/macuser/code/get-shit-done/commands/gsd/settings.md`
- Potentially others (need to find where branching_strategy is used)

**Changes needed:**
1. **Step 2:** Add git.branching_strategy parsing
2. **Step 3:** Add 5th question to AskUserQuestion:
   ```javascript
   {
     question: "Git branching strategy?",
     header: "Branching",
     options: [
       { label: "None (Recommended)", description: "Commit to current branch" },
       { label: "Per Phase", description: "Branch per phase (gsd/phase-{N}-{name})" },
       { label: "Per Milestone", description: "Branch per milestone (gsd/{version}-{name})" }
     ]
   }
   ```
3. **Step 4:** Add git section to config update
4. **Step 5:** Add git branching to confirmation display

**Testing:**
1. Run `/gsd:settings`
2. Verify git question appears
3. Select each option
4. Verify saved to config.json under `git.branching_strategy`
5. Test execution to see if branches created
6. Verify our requirements/interface verification works with branching

**Blockers:**
- Need to find where `branching_strategy` is consumed
- May need to port additional implementation files
- Risk of conflict with our atomic commit workflow

**Action before merge:**
- Search codebase for `branching_strategy` references
- Identify ALL files needed for complete feature
- Plan integration with our custom verifications

**Dependencies:** None for config, unknown for execution

---

#### Change Card: 3.1 Uninstall Flag

**Category:** CLEAN PORT
**Priority:** 3 (LOW)
**Effort:** 15 minutes
**Risk:** VERY LOW

**What:** Add `--uninstall` flag to cleanly remove GSD.

**Why port:** Convenience for users, QoL improvement.

**Files to modify:**
- `/Users/macuser/code/get-shit-done/bin/install.js`

**Changes needed:**
1. Review upstream install.js for --uninstall logic
2. Copy logic to local install.js
3. Test uninstall flow

**Testing:**
1. Install GSD locally
2. Run with `--uninstall`
3. Verify clean removal
4. Reinstall to verify no issues

**Dependencies:** None

---

## Final Recommendations

### Immediate Actions (Week 1)

**Priority 1 - CRITICAL fixes:**
1. Port context bar fix (30 min) ← Start here, low risk
2. Port CONTEXT.md flow fix (2 hrs) ← Critical feature
3. Port context compliance verification (1 hr) ← Completes CONTEXT.md workflow
4. Test all three together with requirements/interface verification

**Total effort:** ~4 hours
**Impact:** Unlocks full `/gsd:discuss-phase` → `/gsd:plan-phase` workflow

---

### Research Phase (Week 1-2)

**Before porting Priority 2:**
1. Search for `branching_strategy` in upstream
2. Identify where git branching is implemented
3. Search for "squash" to locate merge feature
4. Create list of additional files needed
5. Assess impact on our custom verifications

**Estimated effort:** 2-3 hours investigation

---

### Later Actions (Week 2-3)

**Priority 2 - After research:**
1. Port git branching config (settings.md)
2. Port git branching implementation (TBD files)
3. Port squash merge if found
4. Extensive testing with our custom workflows

**Total effort:** 4-6 hours (depends on research findings)

---

### Optional (Week 3+)

**Priority 3:**
1. Port uninstall flag (15 min)

---

### Never Do

**Priority 4 (8 changes):**
- All Gemini/OpenCode/Discord/CI features
- Total skips: 8

---

## Risk Mitigation

### Before ANY merge:

1. **Create backup branch**
   ```bash
   git checkout -b backup-pre-upstream-merge
   git push origin backup-pre-upstream-merge
   ```

2. **Feature branch for each priority**
   ```bash
   git checkout -b feat/upstream-context-flow
   # Port Priority 1 changes
   git commit
   git checkout -b feat/upstream-git-branching
   # Port Priority 2 changes
   ```

3. **Test thoroughly**
   - Run existing workflows
   - Test new features
   - Verify no regressions in custom verifications
   - Check atomic commits still work

4. **Merge incrementally**
   - Merge Priority 1 first
   - Use for a few days
   - Then merge Priority 2 if stable

---

## Success Metrics

### Priority 1 Success:
- [ ] Context bar shows 100% at 80% limit
- [ ] CONTEXT.md loaded in plan-phase Step 4
- [ ] Researcher receives CONTEXT.md with instructions
- [ ] Planner receives CONTEXT.md with instructions
- [ ] Checker receives CONTEXT.md and validates compliance
- [ ] Revision loop receives CONTEXT.md
- [ ] Plan checker has Dimension 7: Context Compliance
- [ ] Plans that violate decisions are flagged
- [ ] Requirements verification still works
- [ ] Interface verification still works

### Priority 2 Success:
- [ ] Settings shows git branching question
- [ ] Config.json stores git.branching_strategy
- [ ] Branches created according to strategy (if implementation found)
- [ ] Milestone completion handles branches (if squash merge found)
- [ ] Our custom workflows compatible with branching

### Overall Success:
- [ ] All Priority 1 features working
- [ ] No regressions in existing workflows
- [ ] No conflicts with custom additions
- [ ] Git history clean (atomic commits preserved)
- [ ] Documentation updated for new features

---

**Analysis Complete**
**Total Changes Assessed:** 15
**Clean Ports:** 4
**Merge Required:** 3 (2 blocked on research)
**Conflicts:** 0
**Skips:** 8

**Recommended First Step:** Port Priority 1.3 (context bar fix) - lowest risk, immediate UX win

