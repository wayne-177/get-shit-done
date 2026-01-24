---
description: Convert vague goals into concrete, executable plans through autonomous analysis and clarification
allowed-tools:
  - Task
  - AskUserQuestion
  - Read
  - Glob
  - Grep
  - Write
---

<objective>

Convert ambiguous user goals into concrete, executable project specifications through systematic analysis and clarification.

This command accepts high-level, vague goals (like "add auth", "make it faster", "improve UX") and transforms them through:
- Codebase analysis to understand existing patterns and constraints
- Domain research to identify interpretation options
- Iterative clarification questions to refine intent
- Multi-option analysis with concrete phase breakdowns

Creates PROJECT.md and ROADMAP.md ready for execution with `/gsd:plan-phase`.

</objective>

<execution_context>

@~/.claude/get-shit-done/workflows/understand-goal.md

</execution_context>

<context>

# For existing projects (brownfield):
@.planning/PROJECT.md
@.planning/codebase/*.md

# For new projects (greenfield):
# No context yet - workflow will create initial structure

</context>

<process>

<step name="validate_input">

**Check if goal is actually vague:**

The goal should be high-level and require clarification. If it's already concrete and specific, suggest the appropriate command instead.

**Examples of vague goals (appropriate for this command):**
- "add authentication"
- "make it faster"
- "improve the UI/UX"
- "add payments"
- "modernize the codebase"
- "add real-time features"

**Examples of concrete goals (suggest different command):**
- "Add JWT authentication with refresh tokens using NextAuth.js" → Suggest `/gsd:new-project` or `/gsd:add-phase`
- "Reduce API latency from 800ms to 200ms by adding Redis caching" → Suggest `/gsd:new-project` or `/gsd:add-phase`
- "Implement Stripe payment integration with webhook handling" → Suggest `/gsd:new-project` or `/gsd:add-phase`

If the goal is already concrete, inform the user and suggest the appropriate command.

</step>

<step name="load_workflow">

**Load the understand-goal workflow:**

Read `~/.claude/get-shit-done/workflows/understand-goal.md` which contains the 8-step process for goal analysis and clarification.

</step>

<step name="execute_workflow">

**Execute understand-goal.md workflow:**

The workflow handles:
1. Input validation
2. Context analysis (existing project vs new)
3. Goal analysis via Task tool
4. Presenting interpretation options
5. Gathering clarifications through questions
6. Refining analysis based on user feedback
7. Confirming final goal specification
8. Generating PROJECT.md and ROADMAP.md

Follow the workflow exactly as specified.

</step>

</process>

<output>

- `.planning/GOAL_ANALYSIS.md` - Analysis artifact with interpretations and refinements
- `.planning/PROJECT.md` - Project specification (created or updated)
- `.planning/ROADMAP.md` - Roadmap with proposed phases (created or updated)

</output>

<success_criteria>

- [ ] Vague goal successfully clarified
- [ ] User confirmed final interpretation
- [ ] PROJECT.md reflects refined goal
- [ ] ROADMAP.md has concrete phases
- [ ] All documents committed to git
- [ ] User knows next step (`/gsd:plan-phase 1`)

</success_criteria>
