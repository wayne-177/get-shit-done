# Phase 1: Context Flow & Display - Research

**Researched:** 2026-02-01
**Domain:** Upstream port (markdown/JS file editing)
**Confidence:** HIGH

## Summary

This phase ports 4 critical upstream changes from GSD v1.11.1 and v1.10.0 to fix CONTEXT.md flow and context bar display. The changes are pure ADDITIONS with zero conflicts — our local fork is missing features, not diverged.

**What was researched:**
- Existing research at `.planning/research/02-file-diff-comparison.md` (line-by-line diffs)
- Local file state for all 4 files requiring changes
- Upstream code snippets and their integration points
- Dependencies between changes

**What the standard approach is:**
This is a straightforward file editing task. All changes are additive code additions to existing markdown prompts and one JavaScript hook. No library installations, no architecture changes, no build steps.

**Key recommendations:**
1. Changes can be made in any order (no dependencies between files)
2. Each file has a single, well-defined addition location
3. No conflicts with existing code — all insertions
4. Test after all changes: run `/gsd:discuss-phase` → `/gsd:plan-phase` to verify CONTEXT.md flow

**Primary recommendation:** Edit files in order of impact (plan-phase.md → plan-checker.md → statusline.js). Test CONTEXT.md flow before committing.

## Standard Stack

This is NOT a library installation task. The "stack" is GSD's existing markdown prompt files and JavaScript hooks.

### Files to Modify (4 total)

| File | Lines Changed | Type | Purpose |
|------|---------------|------|---------|
| `commands/gsd/plan-phase.md` | ~100 additions | Markdown | Load CONTEXT.md in Step 4, pass to all agents |
| `agents/gsd-plan-checker.md` | ~85 additions | Markdown | Add Dimension 7: Context Compliance verification |
| `hooks/gsd-statusline.js` | ~15 changes | JavaScript | Scale context bar (80% → 100%), adjust colors |
| `commands/gsd/settings.md` | NOT IN PHASE 1 | Markdown | Deferred to Phase 3 (git branching) |

### Tools Required

| Tool | Purpose | Already Available |
|------|---------|-------------------|
| Read | Read local files to understand current state | ✓ |
| Edit | Make precise string replacements | ✓ |
| Bash | Test changes (run commands to verify) | ✓ |

### Verification Method

After changes:
1. Run `/gsd:discuss-phase 1` to create CONTEXT.md
2. Run `/gsd:plan-phase 1` to verify:
   - Step 4 loads CONTEXT.md
   - Researcher receives it with instructions
   - Planner receives it with instructions
   - Checker receives it with instructions
3. Check statusline shows scaled context percentage

## Architecture Patterns

### Pattern 1: Early Context Loading (Step 4 Enhancement)

**What:** Load CONTEXT.md immediately after phase directory exists, before spawning any agents.

**Current state (lines 109-120):**
```markdown
## 4. Ensure Phase Directory Exists

```bash
PHASE_DIR=$(ls -d .planning/phases/${PHASE}-* 2>/dev/null | head -1)
if [ -z "$PHASE_DIR" ]; then
  PHASE_NAME=$(grep "Phase ${PHASE}:" .planning/ROADMAP.md | sed 's/.*Phase [0-9]*: //' | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  mkdir -p ".planning/phases/${PHASE}-${PHASE_NAME}"
  PHASE_DIR=".planning/phases/${PHASE}-${PHASE_NAME}"
fi
```
```

**Upstream addition (after the bash block, before "## 5. Handle Research"):**
```markdown
Load CONTEXT.md now — before any agents spawn:

```bash
# Load CONTEXT.md immediately - this informs ALL downstream agents
CONTEXT_CONTENT=$(cat "${PHASE_DIR}"/*-CONTEXT.md 2>/dev/null)
```

**CRITICAL:** Store `CONTEXT_CONTENT` now. It must be passed to:
- **Researcher** — constrains what to research (locked decisions vs Claude's discretion)
- **Planner** — locked decisions must be honored, not revisited
- **Checker** — verifies plans respect user's stated vision
- **Revision** — context for targeted fixes

If CONTEXT.md exists, display: `Using phase context from: ${PHASE_DIR}/*-CONTEXT.md`
```

**Why this works:**
- Loads once, early, before any agent spawning
- Stored in `CONTEXT_CONTENT` variable for reuse
- Step 7 already reads it again (line 246), but that's for the planner only
- This ensures ALL agents get it with proper instructions

### Pattern 2: Enhanced Agent Prompts with Context Guidance

**What:** Pass CONTEXT.md to agents with explicit instructions on how to use it.

**Location 1 - Research prompt (after line 199):**

Current state shows generic phase_context variable. Upstream adds structured guidance:

```markdown
<phase_context>
**IMPORTANT:** If CONTEXT.md exists below, it contains user decisions from /gsd:discuss-phase.

- **Decisions section** = Locked choices — research THESE deeply, don't explore alternatives
- **Claude's Discretion section** = Your freedom areas — research options, make recommendations
- **Deferred Ideas section** = Out of scope — ignore completely

{context_content}
</phase_context>
```

**Location 2 - Planner prompt (after line 283, replace existing "Phase Context" line):**

Current state (line 282-283):
```markdown
**Phase Context (if exists):**
{context_content}
```

Upstream replacement:
```markdown
**Phase Context (if exists):**

IMPORTANT: If phase context exists below, it contains USER DECISIONS from /gsd:discuss-phase.
- **Decisions** = LOCKED — honor these exactly, do not revisit or suggest alternatives
- **Claude's Discretion** = Your freedom — make implementation choices here
- **Deferred Ideas** = Out of scope — do NOT include in this phase

{context_content}
```

**Location 3 - Checker prompt (after line 377, NEW section):**

Current state: Requirements only. No CONTEXT.md at all.

Upstream addition:
```markdown
**Phase Context (if exists):**

IMPORTANT: If phase context exists below, it contains USER DECISIONS from /gsd:discuss-phase.
Plans MUST honor these decisions. Flag as issue if plans contradict user's stated vision.

- **Decisions** = LOCKED — plans must implement these exactly
- **Claude's Discretion** = Freedom areas — plans can choose approach
- **Deferred Ideas** = Out of scope — plans must NOT include these

{context_content}
```

**Location 4 - Revision prompt (likely around line 420+, need to verify):**

Search for the revision prompt section and add similar context guidance.

### Pattern 3: Context Compliance Dimension (Plan Checker)

**What:** New verification dimension (Dimension 7) that validates plans honor user decisions.

**Current state:** Plan checker has 6 dimensions. Ends at Dimension 6.

**Upstream addition:** After Dimension 6 (around line 236+), add entire new dimension:

```markdown
## Dimension 7: Context Compliance (if CONTEXT.md exists)

**Question:** Do plans honor user decisions from /gsd:discuss-phase?

**Only check this dimension if CONTEXT.md was provided in the verification context.**

**Process:**
1. Parse CONTEXT.md sections: Decisions, Claude's Discretion, Deferred Ideas
2. For each locked Decision, find task(s) that implement it
3. Verify no tasks implement Deferred Ideas (scope creep)
4. Verify Discretion areas are handled (planner's choice is valid)

**Red flags:**
- Locked decision has no implementing task
- Task contradicts a locked decision (e.g., user said "cards layout", plan says "table layout")
- Task implements something from Deferred Ideas
- Plan ignores user's stated preference

**Example issue:**
```yaml
issue:
  dimension: context_compliance
  severity: blocker
  description: "User locked decision: 'use existing API library', but Task 03 creates custom API client"
  plan: "08-02"
  fix_hint: "Update task to use existing library per user decision"
```
```

**Also need:** Add `<upstream_input>` section at top of plan-checker.md (after line 26):

```markdown
<upstream_input>
**CONTEXT.md** (if exists) — User decisions from `/gsd:discuss-phase`

| Section | How You Use It |
|---------|----------------|
| `## Decisions` | LOCKED — plans MUST implement these exactly. Flag if contradicted. |
| `## Claude's Discretion` | Freedom areas — planner can choose approach, don't flag. |
| `## Deferred Ideas` | Out of scope — plans must NOT include these. Flag if present. |

If CONTEXT.md exists, add a verification dimension: **Context Compliance**
- Do plans honor locked decisions?
- Are deferred ideas excluded?
- Are discretion areas handled appropriately?
</upstream_input>
```

### Pattern 4: Context Bar Scaling (Statusline)

**What:** Scale context usage to show 100% at Claude Code's 80% limit.

**Current calculation (lines 21-25):**
```javascript
// Context window display (shows USED percentage)
let ctx = '';
if (remaining != null) {
  const rem = Math.round(remaining);
  const used = Math.max(0, Math.min(100, 100 - rem));
```

**Upstream replacement:**
```javascript
// Context window display (shows USED percentage scaled to 80% limit)
// Claude Code enforces an 80% context limit, so we scale to show 100% at that point
let ctx = '';
if (remaining != null) {
  const rem = Math.round(remaining);
  const rawUsed = Math.max(0, Math.min(100, 100 - rem));
  // Scale: 80% real usage = 100% displayed
  const used = Math.min(100, Math.round((rawUsed / 80) * 100));
```

**Current color thresholds (lines 32-40):**
```javascript
// Color based on usage
if (used < 50) {
  ctx = ` \x1b[32m${bar} ${used}%\x1b[0m`;
} else if (used < 65) {
  ctx = ` \x1b[33m${bar} ${used}%\x1b[0m`;
} else if (used < 80) {
  ctx = ` \x1b[38;5;208m${bar} ${used}%\x1b[0m`;
```

**Upstream replacement:**
```javascript
// Color based on scaled usage (thresholds adjusted for new scale)
if (used < 63) {        // ~50% real
  ctx = ` \x1b[32m${bar} ${used}%\x1b[0m`;
} else if (used < 81) { // ~65% real
  ctx = ` \x1b[33m${bar} ${used}%\x1b[0m`;
} else if (used < 95) { // ~76% real
  ctx = ` \x1b[38;5;208m${bar} ${used}%\x1b[0m`;
```

**Why:** When Claude Code stops you at 80%, the bar now shows 100% instead of 80%. More intuitive UX.

### Anti-Patterns to Avoid

**DON'T:** Try to refactor or improve the upstream code during porting
- This is a port task, not a refactor task
- Keep upstream code exactly as-is to maintain compatibility

**DON'T:** Change variable names or locations beyond what's documented
- Keep `CONTEXT_CONTENT` as variable name (already used in line 246)
- Keep insertion locations as documented in diff comparison

**DON'T:** Skip testing after changes
- CONTEXT.md flow is critical — must verify it works end-to-end
- Test with actual `/gsd:discuss-phase` → `/gsd:plan-phase` workflow

## Don't Hand-Roll

This is a port task — all code already exists upstream. Nothing to hand-roll.

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| CONTEXT.md loading | Custom bash logic | Upstream's proven approach | Already handles both `CONTEXT.md` and `{phase}-CONTEXT.md` patterns |
| Context scaling | Custom percentage calculation | Upstream's formula | Already tested, matches Claude Code's 80% enforcement |
| Verification dimension | Custom compliance checks | Upstream's Dimension 7 structure | Follows established dimension pattern |

## Common Pitfalls

### Pitfall 1: Missing Revision Prompt Enhancement

**What goes wrong:** Research, planner, and checker get CONTEXT.md, but revision loop doesn't.

**Why it happens:** Revision prompt is harder to locate in plan-phase.md (around line 420-450).

**How to avoid:**
1. Search for "revision" or "Spawn gsd-planner with revision prompt"
2. Find the revision prompt template
3. Add CONTEXT.md section similar to planner prompt

**Warning signs:** Diff comparison mentions revision prompt but you can't find it in current file.

### Pitfall 2: Wrong Insertion Location for Step 4

**What goes wrong:** CONTEXT.md loading added in wrong place, doesn't execute before agent spawning.

**Why it happens:** Multiple bash blocks in Step 4, easy to insert in wrong one.

**How to avoid:**
- Insert AFTER the existing bash block that creates PHASE_DIR
- Insert BEFORE the "## 5. Handle Research" heading
- Should be new section with its own explanation text

**Warning signs:** If inserted inside the existing bash block, variable won't be available to later steps.

### Pitfall 3: Inconsistent Context Content Variable

**What goes wrong:** Use different variable names in different places (CONTEXT_CONTENT vs context_content).

**Why it happens:** Bash uses uppercase, prompt templates use lowercase placeholders.

**How to avoid:**
- Bash variable: `CONTEXT_CONTENT` (uppercase, already used in line 246)
- Prompt placeholder: `{context_content}` (lowercase, for substitution)
- Keep consistent with existing patterns

**Warning signs:** Grep for "CONTEXT_CONTENT" should show it in Step 4, Step 7, and all agent spawn sections.

### Pitfall 4: Incomplete Dimension 7 Addition

**What goes wrong:** Add Dimension 7 but forget the `<upstream_input>` section at file top.

**Why it happens:** Two separate locations to update in gsd-plan-checker.md.

**How to avoid:**
1. First add `<upstream_input>` section after line 26 (after `</role>` tag)
2. Then add Dimension 7 after Dimension 6 (after line 236)
3. Both are necessary for full context compliance verification

**Warning signs:** Plan checker mentions Dimension 7 but agent definition doesn't explain what CONTEXT.md is.

### Pitfall 5: Context Bar Formula Error

**What goes wrong:** Incorrect scaling formula causes percentage over 100% or negative values.

**Why it happens:** Math is tricky, easy to mess up the order of operations.

**How to avoid:**
- Use EXACT upstream formula: `Math.min(100, Math.round((rawUsed / 80) * 100))`
- Don't try to optimize or simplify
- `Math.min(100, ...)` ensures cap at 100%

**Warning signs:** Test by checking statusline at different context levels — should never exceed 100%.

## Code Examples

All examples are from upstream v1.11.1 and v1.10.0, verified in diff comparison.

### Example 1: Step 4 CONTEXT.md Loading

**File:** `commands/gsd/plan-phase.md`
**Location:** After line 120, before "## 5. Handle Research"

```markdown
Load CONTEXT.md now — before any agents spawn:

```bash
# Load CONTEXT.md immediately - this informs ALL downstream agents
CONTEXT_CONTENT=$(cat "${PHASE_DIR}"/*-CONTEXT.md 2>/dev/null)
```

**CRITICAL:** Store `CONTEXT_CONTENT` now. It must be passed to:
- **Researcher** — constrains what to research (locked decisions vs Claude's discretion)
- **Planner** — locked decisions must be honored, not revisited
- **Checker** — verifies plans respect user's stated vision
- **Revision** — context for targeted fixes

If CONTEXT.md exists, display: `Using phase context from: ${PHASE_DIR}/*-CONTEXT.md`
```

### Example 2: Research Prompt Enhancement

**File:** `commands/gsd/plan-phase.md`
**Location:** After line 199 (after `{phase_context}` line in research prompt)

Replace the simple `{phase_context}` section with:

```markdown
<phase_context>
**IMPORTANT:** If CONTEXT.md exists below, it contains user decisions from /gsd:discuss-phase.

- **Decisions section** = Locked choices — research THESE deeply, don't explore alternatives
- **Claude's Discretion section** = Your freedom areas — research options, make recommendations
- **Deferred Ideas section** = Out of scope — ignore completely

{context_content}
</phase_context>
```

### Example 3: Checker Prompt Enhancement

**File:** `commands/gsd/plan-phase.md`
**Location:** After line 377 (after requirements_content in checker prompt)

Add new section to verification_context:

```markdown
**Phase Context (if exists):**

IMPORTANT: If phase context exists below, it contains USER DECISIONS from /gsd:discuss-phase.
Plans MUST honor these decisions. Flag as issue if plans contradict user's stated vision.

- **Decisions** = LOCKED — plans must implement these exactly
- **Claude's Discretion** = Freedom areas — plans can choose approach
- **Deferred Ideas** = Out of scope — plans must NOT include these

{context_content}
```

### Example 4: Context Compliance Dimension

**File:** `agents/gsd-plan-checker.md`
**Location:** After Dimension 6 (around line 236+)

```markdown
## Dimension 7: Context Compliance (if CONTEXT.md exists)

**Question:** Do plans honor user decisions from /gsd:discuss-phase?

**Only check this dimension if CONTEXT.md was provided in the verification context.**

**Process:**
1. Parse CONTEXT.md sections: Decisions, Claude's Discretion, Deferred Ideas
2. For each locked Decision, find task(s) that implement it
3. Verify no tasks implement Deferred Ideas (scope creep)
4. Verify Discretion areas are handled (planner's choice is valid)

**Red flags:**
- Locked decision has no implementing task
- Task contradicts a locked decision (e.g., user said "cards layout", plan says "table layout")
- Task implements something from Deferred Ideas
- Plan ignores user's stated preference

**Example issue:**
```yaml
issue:
  dimension: context_compliance
  severity: blocker
  description: "User locked decision: 'use existing API library', but Task 03 creates custom API client"
  plan: "08-02"
  fix_hint: "Update task to use existing library per user decision"
```
```

### Example 5: Context Bar Scaling

**File:** `hooks/gsd-statusline.js`
**Location:** Replace lines 21-40

```javascript
// Context window display (shows USED percentage scaled to 80% limit)
// Claude Code enforces an 80% context limit, so we scale to show 100% at that point
let ctx = '';
if (remaining != null) {
  const rem = Math.round(remaining);
  const rawUsed = Math.max(0, Math.min(100, 100 - rem));
  // Scale: 80% real usage = 100% displayed
  const used = Math.min(100, Math.round((rawUsed / 80) * 100));

  // Build progress bar (10 segments)
  const filled = Math.floor(used / 10);
  const bar = '█'.repeat(filled) + '░'.repeat(10 - filled);

  // Color based on scaled usage (thresholds adjusted for new scale)
  if (used < 63) {        // ~50% real
    ctx = ` \x1b[32m${bar} ${used}%\x1b[0m`;
  } else if (used < 81) { // ~65% real
    ctx = ` \x1b[33m${bar} ${used}%\x1b[0m`;
  } else if (used < 95) { // ~76% real
    ctx = ` \x1b[38;5;208m${bar} ${used}%\x1b[0m`;
  } else {
    ctx = ` \x1b[5;31m💀 ${bar} ${used}%\x1b[0m`;
  }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| CONTEXT.md loaded only in Step 7 | CONTEXT.md loaded early in Step 4 | v1.11.1 (2026-01-31) | All agents now receive user decisions |
| Generic phase_context variable | Structured guidance per agent role | v1.11.1 (2026-01-31) | Agents know HOW to use CONTEXT.md |
| 6 verification dimensions | 7 dimensions (added Context Compliance) | v1.11.1 (2026-01-31) | Plans validated against user decisions |
| Context bar shows raw % | Context bar scaled to 80% limit | v1.10.0 (2026-01-29) | UX matches Claude Code's enforcement |
| Color thresholds at 50/65/80 | Color thresholds at 63/81/95 | v1.10.0 (2026-01-29) | Adjusted for scaled display |

**Nothing deprecated** — all changes are additive enhancements.

## File-Specific Change Details

### File 1: commands/gsd/plan-phase.md

**Current state:**
- Step 4 creates phase directory but doesn't load CONTEXT.md
- Step 7 (line 246) reads CONTEXT.md for planner only
- Research prompt has generic phase_context variable
- Planner prompt has generic phase_context line
- Checker prompt doesn't receive CONTEXT.md at all
- Revision prompt (need to verify) likely missing CONTEXT.md

**Changes needed:**
1. **After line 120** (end of Step 4): Add CONTEXT.md loading section
2. **Around line 199** (research prompt): Replace generic context with structured guidance
3. **Around line 283** (planner prompt): Replace generic context line with structured guidance
4. **After line 377** (checker prompt): Add new CONTEXT.md section with guidance
5. **Around line 420+** (revision prompt): Add CONTEXT.md section (need to find exact location)

**Total additions:** ~100 lines (mostly guidance text)

### File 2: agents/gsd-plan-checker.md

**Current state:**
- No `<upstream_input>` section explaining CONTEXT.md
- Only 6 verification dimensions (ends at Dimension 6)
- Line 264 references old "BRIEF.md" instead of CONTEXT.md
- Success criteria don't mention context compliance

**Changes needed:**
1. **After line 26** (after `</role>` tag): Add `<upstream_input>` section (30 lines)
2. **After line 236** (after Dimension 6): Add Dimension 7: Context Compliance (~50 lines)
3. **Update line 264:** Change "BRIEF.md" reference to "CONTEXT.md"
4. **Add to success criteria:** Context compliance checklist items

**Total additions:** ~85 lines

### File 3: hooks/gsd-statusline.js

**Current state:**
- Lines 21-25: Raw percentage calculation
- Lines 32-40: Color thresholds at 50/65/80

**Changes needed:**
1. **Replace lines 21-27:** Add scaling calculation with comments
2. **Replace lines 32-40:** Adjust color thresholds to 63/81/95

**Total changes:** ~15 lines (replacement, not addition)

### File 4: commands/gsd/settings.md (NOT IN PHASE 1)

This file adds git branching configuration but is deferred to Phase 3 per roadmap.

## Open Questions

**None.** All changes are well-documented in existing research with exact line-by-line diffs.

The diff comparison at `.planning/research/02-file-diff-comparison.md` provides complete details including:
- Exact upstream code snippets
- Exact insertion locations
- Reasoning for each change

## Testing Strategy

### Test 1: CONTEXT.md Flow End-to-End

**Steps:**
1. Run `/gsd:discuss-phase 1` to create test CONTEXT.md with Decisions/Discretion/Deferred sections
2. Run `/gsd:plan-phase 1` and verify:
   - Step 4 displays "Using phase context from: ..."
   - Researcher mentions checking CONTEXT.md sections
   - Planner mentions honoring locked decisions
   - Checker validates context compliance (Dimension 7)

**Expected outcome:** All agents acknowledge CONTEXT.md and use it to constrain their work.

### Test 2: Context Bar Scaling

**Steps:**
1. Start fresh conversation (low context usage) — should show green, low percentage
2. Work until approaching 80% context — should show orange, high percentage (90%+)
3. Hit Claude Code's 80% limit — should show 100% (not 80%)

**Expected outcome:** Context bar scales correctly, shows 100% at enforcement limit.

### Test 3: Context Compliance Verification

**Steps:**
1. Create CONTEXT.md with specific locked decision (e.g., "use library X")
2. Create plan that contradicts it (e.g., "implement custom X")
3. Run plan checker — should flag as Dimension 7 issue

**Expected outcome:** Plan checker catches context violations.

## Sources

### Primary (HIGH confidence)

- `.planning/research/02-file-diff-comparison.md` — Line-by-line diffs with exact upstream code
- `.planning/research/01-SUMMARY.md` — Executive summary of upstream changes
- Local file reads of current state (commands/gsd/plan-phase.md, agents/gsd-plan-checker.md, hooks/gsd-statusline.js)

All changes verified against:
- Upstream GSD v1.11.1 (2026-01-31) — CONTEXT.md flow fix, context compliance verification
- Upstream GSD v1.10.0 (2026-01-29) — Context bar scaling fix

### No Secondary or Tertiary Sources

This research is based entirely on:
1. Existing diff analysis (already completed by prior research agent)
2. Local file inspection (current state verification)
3. Upstream changelog (official release notes)

No web searches, no external documentation needed — pure code analysis.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - File editing with Read/Edit tools, already available
- Architecture: HIGH - All patterns documented in existing research with exact locations
- Pitfalls: HIGH - Based on direct code analysis, not speculation
- Code examples: HIGH - Exact upstream code from diff comparison

**Research date:** 2026-02-01
**Valid until:** 90+ days (stable codebase, no fast-moving dependencies)

**Why high confidence:**
- Not greenfield research — porting existing, tested code
- Exact diffs already provided by prior research
- Local files verified to match expected state
- All changes are additive (no conflicts to resolve)
- Test strategy is straightforward (run actual commands)

**Research scope note:**
This research is intentionally narrow. The objective is "what do I need to know to PLAN this phase well", not "design a new context flow system from scratch". Since extensive prior research exists with exact diffs, this research focuses on:
1. Verifying prior research is accurate
2. Understanding current local file state
3. Identifying any ordering dependencies or pitfalls
4. Providing code examples for planner to reference

The planner has everything needed to create specific, executable tasks.
