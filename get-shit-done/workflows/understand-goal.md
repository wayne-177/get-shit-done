# Workflow: Understand Goal

Systematic process for converting vague user goals into concrete, executable specifications through analysis and clarification.

## Purpose

Accept ambiguous goals and transform them into:
- Concrete goal statements with success criteria
- Proposed phase breakdown
- Ready-to-execute PROJECT.md and ROADMAP.md

## Process

<step name="validate_input" priority="first">

**Check if goal requires clarification:**

If the goal is already concrete and specific (includes technology choices, measurable outcomes, clear scope), this workflow is unnecessary.

**Vague (proceed with workflow):**
- "add authentication"
- "make it faster"
- "improve UX"
- "add payments"

**Concrete (suggest /gsd:new-project or /gsd:add-phase instead):**
- "Add JWT authentication with refresh tokens using NextAuth.js"
- "Reduce API latency from 800ms to 200ms by adding Redis caching"

If concrete, inform user and exit. If vague, continue.

</step>

<step name="analyze_context">

**Determine project type and load context:**

```bash
# Check if this is an existing project
if [ -f .planning/PROJECT.md ]; then
    echo "EXISTING_PROJECT"
    cat .planning/PROJECT.md
    cat .planning/ROADMAP.md 2>/dev/null
    cat .planning/STATE.md 2>/dev/null
else
    echo "NEW_PROJECT"
fi

# Check for codebase analysis
if [ -d .planning/codebase ]; then
    echo "CODEBASE_MAPPED"
    ls .planning/codebase/*.md
else
    echo "NO_CODEBASE_MAP"
fi
```

**Context gathered:**
- Existing project: PROJECT.md, ROADMAP.md, STATE.md (constraints and decisions)
- Codebase mapped: ARCHITECTURE.md, STACK.md, CONVENTIONS.md (patterns and tech)
- New project: No context yet (greenfield)

**Next step input:**
- User's vague goal
- Project context (existing vs new)
- Codebase context (if available)

</step>

<step name="detect_patterns">

**If existing project (has .planning/ or detectable source files):**

Run pattern detection to generate domain-specific interpretations.

**Priority: Use existing codebase analysis if available**

```bash
# Check if codebase already mapped
if [ -d .planning/codebase ]; then
    echo "CODEBASE_MAPPED - Using existing analysis"
    # Will pass STACK.md and ARCHITECTURE.md to agent
else
    echo "NO_CODEBASE_MAP - Running pattern detection"
fi
```

**If .planning/codebase/ exists:**
- Skip pattern detection (already analyzed)
- Pass STACK.md and ARCHITECTURE.md to agent for better accuracy

**If no codebase map exists, run pattern detection:**

Detect patterns in parallel using Grep tool (NOT bash grep commands):

**Auth patterns:**
```
Use Grep tool to search:
- Pattern: "jwt|jsonwebtoken|jose" in src/ (*.js, *.ts files)
- Pattern: "bcrypt|argon2" in src/ (*.js, *.ts files)
- Pattern: "express-session|cookie-session|iron-session" in package.json
```

**Database patterns:**
```
Use Grep tool to search:
- Pattern: "prisma|@prisma/client" in src/ (*.ts, *.js files)
- Pattern: "typeorm|@typeorm|drizzle|mongoose" in package.json
- Pattern: "SELECT|INSERT|UPDATE" in src/ (*.ts, *.js files) for raw SQL
```

**API patterns:**
```
Use Glob tool to check:
- Pattern: "src/app/api/**/*" (Next.js app router)
- Pattern: "pages/api/**/*" (Next.js pages router)

Use Grep tool to search:
- Pattern: "express|fastify|hono" in package.json
- Pattern: "graphql|apollo|@apollo" in package.json
- Pattern: "trpc|@trpc" in package.json
```

**Frontend patterns:**
```
Use Grep tool to search:
- Pattern: "react|next|vue|svelte" in package.json
- Pattern: "redux|zustand|jotai|recoil" in package.json
- Pattern: "tailwind|styled-components|emotion" in package.json
```

**Aggregate findings into pattern summary:**

Create summary structure:
```
Detected Patterns:
- Auth: [JWT via jose library, bcrypt password hashing] or [none]
- Database: [Prisma ORM with PostgreSQL] or [none]
- API: [Next.js /app/api routes, REST-style] or [none]
- Frontend: [React via Next.js 14, Tailwind CSS] or [none]
```

**Evidence requirements (avoid false positives):**
- Only report if: package.json entry OR 3+ file matches
- Mark as "none" if insufficient evidence
- Example: 1-2 files mentioning "jwt" = not reported (could be comments)
- Example: package.json has "jsonwebtoken" = reported (clear evidence)

**Error handling:**
- If no source files found: Mark as "New project - no patterns detected"
- If Grep/Glob fails: Continue with generic analysis (no patterns)
- Pass pattern summary (or "no patterns") to agent

**Example pattern summaries:**

For existing Next.js + Prisma project:
```
Detected Patterns:
- Auth: JWT (jsonwebtoken in package.json), bcrypt hashing (5+ files)
- Database: Prisma ORM (package.json + src/lib/prisma.ts)
- API: Next.js /app/api routes (15 route files found)
- Frontend: React via Next.js 14, Tailwind CSS
```

For new project:
```
Detected Patterns: None (new project - no source files)
```

Continue to spawn_goal_analyzer with pattern summary.

</step>

<step name="spawn_goal_analyzer">

**Use Task tool to analyze goal and generate interpretations:**

```
Use Task tool with subagent_type="general-purpose"
```

**Prompt for goal analyzer:**

```markdown
You are analyzing a vague user goal to produce concrete, actionable interpretations.

**User's Goal:** "[vague goal from user]"

**Project Context:**
[If existing project: Include PROJECT.md, constraints, tech stack]
Project type: [New project / Existing project]

**Detected Patterns:**
[If .planning/codebase/ exists:]
@.planning/codebase/STACK.md
@.planning/codebase/ARCHITECTURE.md

[If no codebase map, but patterns detected:]
Detected patterns:
- Auth: [findings or "none"]
- Database: [findings or "none"]
- API: [findings or "none"]
- Frontend: [findings or "none"]

[If new project:]
Detected patterns: None (new project - no source files)

**Your Task:**

Create `.planning/GOAL_ANALYSIS.md` using the template at `~/.claude/get-shit-done/templates/goal-analysis.md`.

**Requirements:**

1. **Domain Understanding Section:**
   - Document detected stack (from patterns or codebase docs)
   - Note "New project" if no patterns detected
   - Identify existing patterns that apply
   - Extract constraints from PROJECT.md (if exists)

2. **Interpretations Section:**
   - Generate 3-5 concrete interpretations of the vague goal
   - **CRITICAL: Make interpretations specific to detected stack**
   - Each option must include:
     - Concrete goal statement (leveraging detected patterns)
     - Rationale (why this makes sense for THIS stack)
     - Proposed phases (2-5 phases with clear goals)
     - Complexity assessment (Low/Medium/High)
     - Expected impact (measurable outcomes)
   - Options should represent meaningfully different approaches

3. **Questions Section:**
   - Identify 3-5 clarifying questions
   - Mark priority: "high" (blocking - affects interpretation), "medium" (optimization)
   - Questions should help choose between interpretations

4. **Recommendation:**
   - Recommend one option with rationale
   - Consider impact vs complexity tradeoff

**Examples of stack-specific interpretations:**

Vague: "make it faster"
Stack: Next.js + Prisma + PostgreSQL

Option A:
Goal: Reduce API response time from 800ms to <200ms via database query optimization
Phases:
  1. Audit Prisma queries for N+1 problems
  2. Add strategic indexes to PostgreSQL
  3. Implement query result caching with Redis
Complexity: Medium
Impact: 4x faster API responses, better UX

Option B:
Goal: Implement Next.js ISR for public pages to reduce server load
Phases:
  1. Identify static/semi-static pages
  2. Add revalidate config to getStaticProps
  3. Set up webhook-based revalidation
Complexity: Low
Impact: 90% reduction in API calls for public content

**Pattern-to-interpretation mapping:**

| Detected Pattern | Vague Goal | Smart Interpretation |
|------------------|------------|---------------------|
| JWT + bcrypt | "make secure" | "Implement refresh tokens + rotate secrets + add rate limiting" |
| Prisma + PostgreSQL | "make faster" | "Optimize Prisma queries + add indexes + implement caching" |
| Next.js /app/api | "add auth" | "Implement NextAuth.js with JWT strategy (integrates with existing API routes)" |
| React + no state mgmt | "improve UX" | "Add Zustand for client state + optimistic updates + loading states" |
| No patterns (new) | "build app" | "Choose stack: Option A (Next.js+Prisma), Option B (Astro+Turso), Option C (SvelteKit+Supabase)" |

**IMPORTANT:**
- ONLY suggest interpretations that make sense given detected patterns
- If no patterns detected (new project), offer stack choices
- If patterns detected, leverage them (don't suggest replacing existing stack)
- Be specific with technology names (e.g., "NextAuth.js" not "auth library")

**Output:** Create GOAL_ANALYSIS.md with Status: "analyzing"
```

**Wait for Task completion and GOAL_ANALYSIS.md creation.**

**Validation:**
After agent completes, verify GOAL_ANALYSIS.md exists and has:
- At least 3 interpretations
- Each interpretation has concrete goal + phases + complexity + impact
- Questions section populated (even if just "Confirm which option matches your intent")
- Recommendation included

If validation fails: Retry Task tool with more explicit prompt.

</step>

<step name="present_interpretations">

Read `.planning/GOAL_ANALYSIS.md` generated by pattern detection.

**Two-stage selection process** (fits AskUserQuestion 4-option limit):

---

### Stage 1: Capture Intent

First, understand WHY the user wants this goal. Intent determines which options are most relevant.

**Use AskUserQuestion:**
```
AskUserQuestion({
  header: "Intent",
  question: "What's driving this goal?",
  multiSelect: false,
  options: [
    {
      label: "Fix critical vulnerabilities",
      description: "Address immediate security risks. Fastest path to safer code."
    },
    {
      label: "Prepare for production",
      description: "Ensure production-readiness with security best practices."
    },
    {
      label: "Meet compliance requirements",
      description: "SOC2, GDPR, HIPAA, or other regulatory requirements."
    },
    {
      label: "Comprehensive review",
      description: "Systematic security audit covering all aspects."
    }
  ]
})
```

**Intent-to-priority mapping:**

| Intent | Prioritize Options That... |
|--------|---------------------------|
| Fix critical vulnerabilities | Address immediate risks, low complexity, high impact |
| Prepare for production | Cover authentication, API security, standard hardening |
| Meet compliance requirements | Include audit logging, data protection, access control |
| Comprehensive review | Systematic coverage, can be higher complexity |

---

### Stage 2: Present Prioritized Options

Based on intent, show 3 most relevant options + "Show all options" escape hatch.

**Prioritization logic:**

```
For each option in GOAL_ANALYSIS.md:
  - Score based on intent match (complexity, impact, scope alignment)
  - Rank by score descending
  - Take top 3 for display
  - Always include "Show all options" as 4th choice
```

**Example for "Fix critical vulnerabilities" intent:**
```
AskUserQuestion({
  header: "Approach",
  question: "Which approach best fits? (Prioritized for quick fixes)",
  multiSelect: true,
  options: [
    {
      label: "Option A: Authentication Hardening",
      description: "Fix JWT vulnerability, add refresh tokens. Medium complexity, highest impact."
    },
    {
      label: "Option C: API Security Hardening",
      description: "Rate limiting, security headers. Low complexity, prevents attacks."
    },
    {
      label: "Option B: Access Control (RBAC)",
      description: "Role-based permissions. Medium complexity."
    },
    {
      label: "Show all options",
      description: "See complete list of all [N] interpretations from analysis."
    }
  ]
})
```

**If "Show all options" selected:**

Present remaining options in next AskUserQuestion call, or output full list as text:

```markdown
## All Interpretations

**Option A: [Goal]**
[Rationale] | Complexity: [X] | Impact: [Y]

**Option B: [Goal]**
[Rationale] | Complexity: [X] | Impact: [Y]

... (all options from GOAL_ANALYSIS.md)
```

Then ask: "Which option(s) would you like to proceed with?" (free-form or follow-up AskUserQuestion)

---

### Handle Responses

- **Single selection**: Use that interpretation as base
- **Multiple selections**: Combine into hybrid interpretation (handled in refine_analysis step)
- **"Other" selected**: Treat custom text as refinement constraint
- **"Show all" then selection**: Use selected option(s) from full list

---

### Update GOAL_ANALYSIS.md

Add section after `</questions>`:
```xml
<user_selections>
  <intent>[User's selected intent]</intent>
  <selected>option-a, option-c</selected>
  <custom_refinement>[If "Other" was selected, include user's custom text]</custom_refinement>
  <showed_all_options>[true/false]</showed_all_options>
</user_selections>
```

Update status in frontmatter: `analyzing` → `clarifying`

Update iteration counter:
```
*Analysis iterations: 1*
```

</step>

<step name="gather_clarifications">

Read `<questions>` section from GOAL_ANALYSIS.md.

**Filter to high-priority questions only** (limit 2-3 to avoid fatigue):
- If 1-3 high-priority questions: Use all
- If >3 high-priority questions: Take top 3 based on blocking impact
- If 0 high-priority: Skip to step 6 (no clarification needed)

**Format questions for AskUserQuestion:**

For each high-priority question, create AskUserQuestion call:

```
AskUserQuestion({
  header: "Clarification",
  question: "[Question from GOAL_ANALYSIS.md]",
  multiSelect: false,  // Usually single choice for clarity
  options: [
    {
      label: "[Option 1 based on question context]",
      description: "[Implications of this choice]"
    },
    {
      label: "[Option 2]",
      description: "[Implications]"
    },
    // "Other" for free-form answer (auto-added)
  ]
})
```

**Example questions by goal type:**

| Vague Goal | Generated Question | Options |
|------------|-------------------|---------|
| "make it faster" | "Which performance aspect matters most?" | API response time / Page load time / Build time |
| "add auth" | "What authentication method?" | Email+password / OAuth (Google/GitHub) / Magic links |
| "improve UX" | "Which UX issue to prioritize?" | Loading states / Error handling / Responsive design |

**Update GOAL_ANALYSIS.md:**

Add section after `<user_selections>`:
```xml
<clarifications>
  <answer question="[Question text]">[User's answer]</answer>
  <answer question="[Question 2]">[Answer 2]</answer>
</clarifications>
```

**Iteration tracking:**

Increment iteration counter in frontmatter:
```
*Analysis iterations: 1*
```

**What to avoid and WHY:**
- Don't ask questions already answered by pattern detection (wastes time) - if JWT detected, don't ask "do you want JWT?"
- Don't ask more than 3 questions per round (user fatigue kills momentum)
- Don't present options without tradeoff context (user can't decide blind) - always include "description" explaining implications

</step>

<step name="refine_analysis">

Spawn Task tool with subagent_type="general-purpose":

**Prompt structure:**
```
You are refining a goal analysis based on user feedback.

**Context:**
@.planning/GOAL_ANALYSIS.md (contains original interpretations + user selections + clarifications)

**User selections:**
- Selected interpretations: [option IDs from user_selections]
- Clarification answers: [answers from clarifications section]
- Custom refinement: [if "Other" was provided]

**Task:**
Update GOAL_ANALYSIS.md with refined interpretation:

1. If single selection: Enhance that interpretation with clarification answers
2. If multiple selections: Combine into hybrid interpretation
3. If custom refinement: Incorporate constraints into interpretation

**Update these sections:**

<refined_goal>
[Concrete goal statement incorporating feedback]

Success Criteria:
- [Measurable outcome 1 - informed by clarifications]
- [Measurable outcome 2]

Proposed Phases:
1. [Phase name]: [Goal - adjusted based on answers]
2. [Phase name]: [Goal]

Complexity: [Low/Medium/High - may change with scope adjustment]
Impact: [Expected benefit - quantified where possible]

Notes:
- [How user selections affected the interpretation]
- [What tradeoffs were made if combining options]
</refined_goal>

**If information gaps remain:**
Generate new high-priority questions in `<questions>` section.

**If goal is now concrete enough:**
Clear questions section (mark as "None - goal is concrete").

Status: clarifying → refined (or back to analyzing if more questions needed)
```

**Agent validation:**
After agent completes:
- Read updated GOAL_ANALYSIS.md
- Check if `<refined_goal>` has concrete statement + phases
- Check if questions section is empty OR iteration count >= 3

**Iteration decision:**
```javascript
if (questionsRemain && iterationCount < 3) {
  return to step 5 (gather_clarifications) for next round
} else {
  proceed to step 7 (confirm_final_goal)
}
```

**If max iterations (3) reached with questions:**
Escalate to user with best current interpretation:
"I've refined the goal through 3 rounds but still have open questions. Here's my best interpretation based on what we've discussed. Proceed with this, or start over for different approach?"

</step>

<step name="confirm_final_goal">

Read `<refined_goal>` from GOAL_ANALYSIS.md.

Present to user for confirmation:

**Use AskUserQuestion:**
```
AskUserQuestion({
  header: "Final Confirmation",
  question: "Does this goal capture your intent?",
  multiSelect: false,
  options: [
    {
      label: "Yes, this is exactly what I want",
      description: "Proceed to generate PROJECT.md and ROADMAP.md"
    },
    {
      label: "Close, but needs adjustment",
      description: "Provide specific feedback to refine further (if under 3 iterations)"
    },
    {
      label: "Start over",
      description: "Clear analysis and restart from scratch"
    }
  ]
})
```

**Handle responses:**

1. **"Yes"**:
   - Update GOAL_ANALYSIS.md status: refined → confirmed
   - Proceed to step 8 (generate_documents)

2. **"Close, but needs adjustment"**:
   - If iterationCount < 3: Capture feedback, return to step 6 with refinement prompt
   - If iterationCount >= 3: "Maximum iterations reached. Proceeding with current interpretation or start over?"

3. **"Start over"**:
   - Clear GOAL_ANALYSIS.md
   - Reset iteration counter to 0
   - Return to step 3 (spawn_goal_analyzer)

**Update GOAL_ANALYSIS.md:**
```
Status: confirmed
*Analysis iterations: [final count]*
*Last updated: [timestamp]*
```

**Iteration safety:**
- Hard limit of 3 clarification rounds
- Each round: present → clarify → refine → confirm
- Escape hatches: "start over" always available, "proceed anyway" after max iterations

</step>

<step name="generate_documents">

Read `.planning/GOAL_ANALYSIS.md` (status must be "confirmed").

Extract from `<refined_goal>`:
- Concrete goal statement
- Success criteria
- Proposed phases
- Complexity estimate
- Impact description

**Check if new project or existing:**

```bash
if [ -f .planning/PROJECT.md ]; then
  echo "Existing project - skip PROJECT.md generation"
  EXISTING_PROJECT=true
else
  echo "New project - generate PROJECT.md"
  EXISTING_PROJECT=false
fi
```

**Part 1: PROJECT.md Generation (new projects only)**

**If new project (no PROJECT.md):**

Use template from `get-shit-done/templates/project.md`:

Create PROJECT.md with these sections:

**Title:** `# [Project Name from refined goal]`

**What This Is:**
- Concrete goal statement from refined_goal
- If domain_understanding.stack exists: "This project uses [detected stack] and focuses on [goal focus area]."

**Core Value:**
- Extract from impact description - what's the primary benefit

Examples:
- If goal is "Reduce API latency": Core value is "Fast, responsive user experience through optimized backend performance"
- If goal is "Add JWT auth": Core value is "Secure user authentication and authorization with industry-standard tokens"

**Requirements - Validated:**
- `(None yet — ship to validate)`

**Requirements - Active:**
- Convert success criteria into checkable requirements

Example from GOAL_ANALYSIS.md:
```
Success Criteria:
- API response time < 200ms
- All endpoints cached appropriately

Becomes:
- [ ] API response time < 200ms for all endpoints
- [ ] Redis caching layer for frequent queries
- [ ] Query optimization eliminates N+1 problems
```

**Requirements - Out of Scope:**
- Items explicitly ruled out during clarification, or obvious non-goals

Example:
- If user selected "API performance" over "Frontend optimization": Out of scope includes "UI performance, client-side caching"

**Context:**

**Project Type:** [New greenfield / Existing project enhancement]

**Current State:** [From domain_understanding if existing, or "Starting from scratch" if new]

If existing project:
**What Exists:**
[List detected patterns from GOAL_ANALYSIS.md domain_understanding section]

**Critical Gaps:**
[What the vague goal identified as missing - the "make it X" motivation]

**Constraints:**
- Extract from domain_understanding.constraints if exists, or add standard constraints

Standard constraints:
- Tech Stack: [Detected stack from pattern detection, or "TBD - to be chosen in Phase 1"]
- [Any constraints mentioned during clarification rounds]

**Key Decisions:**

Leave table empty with note:
*Decisions will be tracked as implementation progresses*

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| [TBD] | [TBD] | [TBD] |

---
*Created via /gsd:understand: [ISO date]*

**Generate PROJECT.md:**
```bash
# Use Write tool to create .planning/PROJECT.md
# Populate all sections from GOAL_ANALYSIS.md
# Ensure markdown syntax is valid
```

**Commit:**
```bash
git add .planning/PROJECT.md .planning/GOAL_ANALYSIS.md
git commit -m "$(cat <<'EOF'
docs: understand goal - [goal summary]

Converted vague goal "[original vague input]" into concrete specification.

Analysis iterations: [N]
Final interpretation: [refined goal one-liner]

Generated PROJECT.md ready for roadmap creation.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
EOF
)"
```

**If existing project:**

Skip PROJECT.md generation (already exists).

Update GOAL_ANALYSIS.md status: confirmed → integrated

Log in GOAL_ANALYSIS.md:
```
PROJECT.md: Skipped (existing project)
Proceeding to ROADMAP.md phase addition
```

**Part 2: ROADMAP.md Generation**

Read `<refined_goal>` proposed phases from GOAL_ANALYSIS.md.

**Check if ROADMAP.md exists:**

```bash
if [ -f .planning/ROADMAP.md ]; then
  echo "Existing roadmap - append phases"
  APPEND_MODE=true
else
  echo "New roadmap - create from scratch"
  APPEND_MODE=false
fi
```

**If new roadmap:**

Use template from `get-shit-done/templates/roadmap.md`:

Create ROADMAP.md:

```markdown
# Roadmap: [Project Name]

## Overview

[Refined goal statement]

This roadmap breaks down the goal into [N] sequential phases:

[One-line summary of transformation]
Example: "Transform slow API (800ms) into fast API (<200ms) through query optimization, caching, and indexing"

## Domain Expertise

[If domain-specific resources were consulted during analysis, list them here]
[Otherwise: "None"]

## Phases

[For each phase from GOAL_ANALYSIS.md proposed phases:]

### Phase [N]: [Phase Name]

**Goal**: [Phase goal from refined_goal]

**Depends on**: [Phase N-1 if N>1, otherwise "None"]

**Research**: [Likely / Maybe / No]
[If research: **Research topics**: [Specific topics based on phase]]

Plans:
- [ ] [N]-01: TBD (run /gsd:plan-phase [N] to break down)

[Repeat for all phases]

## Progress

| Phase | Plans | Status | Completed |
|-------|-------|--------|-----------|
[One row per phase with 0/? plans, Not started status]

---

**Next:** Plan first phase with `/gsd:plan-phase 1`
```

**Phase derivation logic:**

Example GOAL_ANALYSIS.md refined_goal:
```
Proposed Phases:
1. Audit and optimize Prisma queries
2. Add PostgreSQL indexes for frequent lookups
3. Implement Redis caching layer
```

Becomes ROADMAP.md:
```
### Phase 1: Query Optimization Audit

**Goal**: Identify and eliminate N+1 queries, optimize Prisma select/include patterns

**Depends on**: None

**Research**: Maybe
**Research topics**: Prisma performance best practices, query profiling tools

Plans:
- [ ] 01-01: TBD

### Phase 2: Database Indexing

**Goal**: Add strategic indexes to PostgreSQL for high-frequency queries identified in Phase 1

**Depends on**: Phase 1 (need query analysis results)

**Research**: No

Plans:
- [ ] 02-01: TBD

### Phase 3: Caching Implementation

**Goal**: Implement Redis caching layer with TTL strategies for expensive queries

**Depends on**: Phase 2 (optimal queries before caching)

**Research**: Likely
**Research topics**: Redis patterns for Next.js, cache invalidation strategies

Plans:
- [ ] 03-01: TBD
```

**Research flag heuristics:**
- "Likely" if phase involves new technology not in domain_understanding.stack
- "Maybe" if phase uses existing stack but complex patterns
- "No" if straightforward application of known patterns

**If existing roadmap (APPEND_MODE):**

Parse existing ROADMAP.md to find highest phase number.

Add new phases starting at next number:

```bash
LAST_PHASE=$(grep "^### Phase" .planning/ROADMAP.md | tail -1 | sed 's/### Phase \([0-9]*\):.*/\1/')
NEXT_PHASE=$((LAST_PHASE + 1))
```

Append phases to "## Phases" section.
Update "## Progress" table with new rows.

**Create STATE.md if new project:**

```markdown
# Project State

## Current Position

Phase: 1 of [N]
Status: Ready to plan
Last activity: [ISO date] - Goal understood and roadmap created

Progress: ░░░░░░░░░░ 0%

## Accumulated Context

### Decisions

None yet - planning phase 1

### Deferred Issues

None

### Blockers/Concerns

None - ready to begin planning

## Session Continuity

Last session: [ISO date]
Stopped at: Roadmap creation via /gsd:understand
Resume: Run /gsd:plan-phase 1 to break down first phase
```

**Commit:**
```bash
git add .planning/ROADMAP.md .planning/STATE.md
git commit -m "$(cat <<'EOF'
docs: create roadmap ([N] phases)

Derived from goal analysis:
- [One-line goal summary]

Phases:
1. [Phase 1 name]
2. [Phase 2 name]
[...]

Ready for /gsd:plan-phase 1

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
EOF
)"
```

**Final output to user:**

```
✓ Goal understanding complete

Created:
- .planning/PROJECT.md ([Project Name])
- .planning/ROADMAP.md ([N] phases)
- .planning/STATE.md (tracking)
- .planning/GOAL_ANALYSIS.md (analysis record)

Commits:
- [commit hash]: docs: understand goal - [summary]
- [commit hash]: docs: create roadmap ([N] phases)

---

## Next Up

**Phase 1: [Phase Name]** - [Goal]

`/gsd:plan-phase 1`

<sub>`/clear` first - fresh context window</sub>

---

**Also available:**
- Review goal analysis: `cat .planning/GOAL_ANALYSIS.md`
- Adjust roadmap: Edit .planning/ROADMAP.md before planning
- Map codebase: `/gsd:map-codebase` for brownfield projects

---
```

</step>

## Error Handling

**Goal already concrete:**
- Detect in step 1 (validate_input)
- Inform user: "Your goal is already concrete. Use `/gsd:new-project` for new projects or `/gsd:add-phase` to extend existing roadmap."
- Exit workflow

**Max iterations reached (3 refinement cycles):**
- Present current best interpretation
- Offer: "We've refined this 3 times. Shall we proceed with this version and validate through execution? You can always adjust after Phase 1."
- If user declines, escalate: Suggest breaking into smaller goal or consulting domain expert

**User says "start over" more than once:**
- After second "start over", ask: "Let's take a different approach. Can you describe what's missing from these interpretations?"
- Collect free-form context, then create a focused single interpretation instead of multiple options

**GOAL_ANALYSIS.md creation fails:**
- Retry once with more explicit prompt
- If fails again, fall back to inline questioning (skip analysis document, build PROJECT.md directly)

## Success Criteria

- [ ] Vague goal successfully transformed into concrete specification
- [ ] User confirmed final goal captures their intent
- [ ] PROJECT.md reflects concrete goal and success criteria
- [ ] ROADMAP.md has actionable phases with clear goals
- [ ] All documents committed to git
- [ ] User knows next step (`/gsd:plan-phase 1`)
- [ ] Goal transformation preserved in GOAL_ANALYSIS.md for reference
