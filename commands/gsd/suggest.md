---
description: Generate prioritized goal suggestions from codebase analysis (user)
allowed-tools:
  - Read
  - Bash
  - Write
  - Edit
---

<objective>

Generate or refresh goal suggestions based on codebase analysis findings and knowledge base patterns.

This command orchestrates the suggestion generation workflow to produce `.planning/GOAL_PROPOSALS.md` with prioritized, actionable improvement opportunities.

Use this command:
- Before starting work to see what improvements are available
- After running `/gsd:analyze-codebase` to generate suggestions
- Anytime you want to refresh the prioritized goal list
- When deciding what to work on next

</objective>

<execution_context>

@~/.claude/get-shit-done/workflows/generate-suggestions.md

</execution_context>

<context>

**Check for required analysis report:**
@.planning/ANALYSIS_REPORT.md

**Check for existing proposals:**
@.planning/GOAL_PROPOSALS.md

**Load configuration:**
@.planning/config.json

</context>

<process>

<step name="check_prerequisites">

**Verify analysis report exists**

```bash
# Check for .planning directory
if [ ! -d .planning ]; then
  echo "ERROR: No .planning/ directory found"
  echo ""
  echo "Initialize GSD project first:"
  echo "  /gsd:new-project"
  exit 1
fi

# Check for analysis report
if [ ! -f .planning/ANALYSIS_REPORT.md ]; then
  echo "ERROR: No analysis report found at .planning/ANALYSIS_REPORT.md"
  echo ""
  echo "Run codebase analysis first to generate findings:"
  echo "  /gsd:analyze-codebase"
  exit 1
fi

# Get analysis timestamp
ANALYSIS_TIMESTAMP=$(grep '^\*\*Generated:\*\*' .planning/ANALYSIS_REPORT.md | sed 's/.*: *//')
echo "Analysis report found (generated: $ANALYSIS_TIMESTAMP)"
```

**If prerequisites not met:**
Display clear error message with guidance.

**If prerequisites met:**
Continue to staleness check.

</step>

<step name="check_staleness">

**Determine if regeneration is needed**

```bash
# Check if proposals already exist
if [ -f .planning/GOAL_PROPOSALS.md ]; then
  # Get proposals timestamp
  PROPOSALS_TIMESTAMP=$(grep '^\*\*Generated:\*\*' .planning/GOAL_PROPOSALS.md | sed 's/.*: *//')
  PROPOSALS_ANALYSIS_TS=$(grep '^\*\*Analysis Report:\*\*' .planning/GOAL_PROPOSALS.md | sed 's/.*: *//')

  echo "Existing proposals found (generated: $PROPOSALS_TIMESTAMP)"

  # Compare analysis timestamps
  if [ "$PROPOSALS_ANALYSIS_TS" = "$ANALYSIS_TIMESTAMP" ]; then
    echo ""
    echo "Suggestions are up-to-date with current analysis."
    echo ""

    # Show summary
    TOTAL=$(grep '^\*\*Total Suggestions:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+' || echo "0")
    CRITICAL=$(grep '^\*\*By Priority:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+ critical' | grep -oE '[0-9]+' || echo "0")
    HIGH=$(grep '^\*\*By Priority:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+ high' | grep -oE '[0-9]+' || echo "0")
    QUICK_WINS=$(grep '^\*\*Estimated Quick Wins:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+' || echo "0")

    echo "Current suggestions:"
    echo "  Total: $TOTAL"
    echo "  Critical: $CRITICAL"
    echo "  High: $HIGH"
    echo "  Quick wins: $QUICK_WINS"
    echo ""
    echo "Options:"
    echo "  - View proposals: cat .planning/GOAL_PROPOSALS.md"
    echo "  - Force refresh: /gsd:suggest --force"
    echo "  - New analysis: /gsd:analyze-codebase"

    PROPOSALS_UP_TO_DATE="true"
  else
    echo "Analysis has been updated since last suggestions."
    echo "Regenerating proposals..."
    PROPOSALS_UP_TO_DATE="false"
  fi
else
  echo "No existing proposals found."
  echo "Generating suggestions from analysis report..."
  PROPOSALS_UP_TO_DATE="false"
fi
```

**Staleness logic:**
- If proposals exist AND match current analysis → Already up to date (skip unless --force)
- If proposals exist but analysis is newer → Regenerate
- If no proposals exist → Generate fresh

</step>

<step name="check_analysis_age">

**Warn if analysis report is stale**

```bash
# Check analysis report age
if [ -f .planning/ANALYSIS_REPORT.md ]; then
  # Get file modification time in days
  ANALYSIS_AGE=$(find .planning/ANALYSIS_REPORT.md -mtime +7 2>/dev/null)

  if [ -n "$ANALYSIS_AGE" ]; then
    echo ""
    echo "NOTE: Analysis report is older than 7 days."
    echo "Consider running /gsd:analyze-codebase for fresh findings."
    echo ""
  fi
fi
```

**Age check:**
- Warn if analysis is >7 days old
- Continue with generation (don't block)
- User can choose to refresh analysis first

</step>

<step name="generate_suggestions">

**Run suggestion generation workflow**

Follow `@~/.claude/get-shit-done/workflows/generate-suggestions.md`:

1. Parse recommendations from ANALYSIS_REPORT.md
2. Query knowledge base for patterns (if adaptive.enabled)
3. Score suggestions using priority + effort + patterns
4. Generate GOAL_PROPOSALS.md with prioritized list

**Workflow execution:**
Follow all steps in the generate-suggestions workflow exactly as specified.

**Output:** `.planning/GOAL_PROPOSALS.md`

</step>

<step name="display_summary">

**Display suggestions summary**

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "GOAL SUGGESTIONS GENERATED"
echo "═══════════════════════════════════════════════════"
echo ""

# Read summary values
TOTAL=$(grep '^\*\*Total Suggestions:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+' || echo "0")
CRITICAL=$(grep '^\*\*By Priority:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+ critical' | grep -oE '[0-9]+' || echo "0")
HIGH=$(grep '^\*\*By Priority:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+ high' | grep -oE '[0-9]+' || echo "0")
MEDIUM=$(grep '^\*\*By Priority:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+ medium' | grep -oE '[0-9]+' || echo "0")
LOW=$(grep '^\*\*By Priority:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+ low' | grep -oE '[0-9]+' || echo "0")
QUICK_WINS=$(grep '^\*\*Estimated Quick Wins:\*\*' .planning/GOAL_PROPOSALS.md | grep -oE '[0-9]+' || echo "0")

# Check for pattern insights
PATTERN_STATUS=$(grep '^\*\*Pattern Insights:\*\*' .planning/GOAL_PROPOSALS.md | sed 's/.*: *//')

echo "Total Suggestions: $TOTAL"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "BY PRIORITY"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  Critical:  $CRITICAL"
echo "  High:      $HIGH"
echo "  Medium:    $MEDIUM"
echo "  Low:       $LOW"
echo ""
echo "Quick Wins:  $QUICK_WINS (quick-fix effort)"
echo ""
echo "Pattern Insights: $PATTERN_STATUS"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "NEXT STEPS"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

# Status-specific guidance
if [ "$CRITICAL" -gt 0 ]; then
  echo "CRITICAL issues found. Address security vulnerabilities first."
  echo ""
  echo "View top suggestion:"
  echo "  grep -A 20 '^### 1\\. ' .planning/GOAL_PROPOSALS.md"
elif [ "$HIGH" -gt 0 ]; then
  echo "HIGH priority issues found. Review and address soon."
  echo ""
elif [ "$QUICK_WINS" -gt 0 ]; then
  echo "Quick wins available! Start with easy improvements."
  echo ""
else
  echo "Codebase is healthy. No urgent issues."
  echo ""
fi

echo "Full proposals: .planning/GOAL_PROPOSALS.md"
echo "Refresh analysis: /gsd:analyze-codebase"
echo ""
echo "═══════════════════════════════════════════════════"
```

</step>

<step name="handle_healthy_codebase">

**Handle case where no suggestions exist**

If GOAL_PROPOSALS.md shows 0 suggestions:

```
═══════════════════════════════════════════════════
CODEBASE HEALTHY
═══════════════════════════════════════════════════

No actionable suggestions found.

Your codebase is in good health:
- No critical or high-severity security vulnerabilities
- Dependencies are reasonably up to date
- Tech debt is within acceptable levels

Next review recommended after:
- Major dependency updates
- New feature implementations
- Before milestone completions

═══════════════════════════════════════════════════
```

</step>

</process>

<success_criteria>

- Prerequisites checked (analysis report exists)
- Staleness handled (skip if up-to-date, regenerate if stale)
- GOAL_PROPOSALS.md created or confirmed current
- Summary displayed to user
- Next steps clear

</success_criteria>

<examples>

**Example 1: Fresh generation**

```
$ /gsd:suggest

Analysis report found (generated: 2026-01-08T10:25:00Z)
No existing proposals found.
Generating suggestions from analysis report...

═══════════════════════════════════════════════════
GOAL SUGGESTIONS GENERATED
═══════════════════════════════════════════════════

Total Suggestions: 5

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BY PRIORITY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Critical:  1
  High:      2
  Medium:    1
  Low:       1

Quick Wins:  3 (quick-fix effort)

Pattern Insights: 12 patterns consulted

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NEXT STEPS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CRITICAL issues found. Address security vulnerabilities first.

View top suggestion:
  grep -A 20 '^### 1\. ' .planning/GOAL_PROPOSALS.md

Full proposals: .planning/GOAL_PROPOSALS.md
Refresh analysis: /gsd:analyze-codebase

═══════════════════════════════════════════════════
```

---

**Example 2: Already up to date**

```
$ /gsd:suggest

Analysis report found (generated: 2026-01-08T10:25:00Z)
Existing proposals found (generated: 2026-01-08T10:30:00Z)

Suggestions are up-to-date with current analysis.

Current suggestions:
  Total: 5
  Critical: 1
  High: 2
  Quick wins: 3

Options:
  - View proposals: cat .planning/GOAL_PROPOSALS.md
  - Force refresh: /gsd:suggest --force
  - New analysis: /gsd:analyze-codebase
```

---

**Example 3: No analysis report**

```
$ /gsd:suggest

ERROR: No analysis report found at .planning/ANALYSIS_REPORT.md

Run codebase analysis first to generate findings:
  /gsd:analyze-codebase
```

---

**Example 4: Healthy codebase**

```
$ /gsd:suggest

Analysis report found (generated: 2026-01-08T10:25:00Z)
Generating suggestions from analysis report...

═══════════════════════════════════════════════════
CODEBASE HEALTHY
═══════════════════════════════════════════════════

No actionable suggestions found.

Your codebase is in good health:
- No critical or high-severity security vulnerabilities
- Dependencies are reasonably up to date
- Tech debt is within acceptable levels

Next review recommended after:
- Major dependency updates
- New feature implementations
- Before milestone completions

═══════════════════════════════════════════════════
```

</examples>
