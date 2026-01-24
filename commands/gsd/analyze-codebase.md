---
description: Run comprehensive codebase analysis for security, dependencies, and tech debt
allowed-tools:
  - Read
  - Bash
  - Write
  - Edit
---

<objective>

Run comprehensive codebase analysis to identify security vulnerabilities, outdated dependencies, and tech debt.

This command orchestrates all codebase analyzers (security, dependencies, tech debt) and generates a unified analysis report at `.planning/ANALYSIS_REPORT.md`. The report is used by the suggestion engine (Phase 12) to propose improvement goals.

Use this command:
- Before starting significant work to know the codebase state
- After dependency changes to check for new vulnerabilities
- Before milestone completion to assess health
- Anytime you want a comprehensive codebase audit

</objective>

<execution_context>

@~/.claude/get-shit-done/workflows/analyze-codebase.md

</execution_context>

<context>

**Check for project setup:**
@.planning/config.json

**Reference output template:**
@~/.claude/get-shit-done/templates/analysis-report.md

</context>

<process>

<step name="check_prerequisites">

**Verify project setup**

1. Check for package.json:
   ```bash
   if [ ! -f package.json ]; then
     echo "ERROR: No package.json found"
     echo ""
     echo "This command analyzes Node.js projects."
     echo "For non-Node.js projects, codebase analysis is not yet supported."
     exit 1
   fi
   ```

2. Check for .planning directory:
   ```bash
   if [ ! -d .planning ]; then
     echo "ERROR: No .planning/ directory found"
     echo ""
     echo "Initialize GSD project first:"
     echo "  /gsd:new-project"
     exit 1
   fi
   ```

**If prerequisites not met:**
Display clear error message and exit.

**If prerequisites met:**
Continue to analysis.

</step>

<step name="run_analysis">

**Execute analyze-codebase workflow**

Follow the `analyze-codebase.md` workflow which:

1. Sets up analysis directory (`.planning/.analysis/`)
2. Runs all analyzers in parallel:
   - Dependency analysis (npm audit, npm outdated)
   - Security analysis (CVE extraction, lockfile validation)
   - Tech debt analysis (Knip for unused code)
3. Aggregates findings across all analyzers
4. Generates unified `.planning/ANALYSIS_REPORT.md`
5. Displays summary to user

**Execute workflow:**
Follow all steps in `@~/.claude/get-shit-done/workflows/analyze-codebase.md`

</step>

<step name="report_results">

**Display analysis results**

After workflow completes, show summary:

```
═══════════════════════════════════════════════════
CODEBASE ANALYSIS COMPLETE
═══════════════════════════════════════════════════

Overall Status: [STATUS]
Action Priority: [PRIORITY]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FINDINGS SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Security:     [STATUS] (Critical: [N], High: [N])
Dependencies: [STATUS] (Outdated: [N], Major: [N])
Tech Debt:    [LEVEL] (Score: [N])

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NEXT STEPS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Context-appropriate guidance based on status]

Full report: .planning/ANALYSIS_REPORT.md
═══════════════════════════════════════════════════
```

**Status-specific guidance:**

- **CRITICAL:** "CRITICAL issues found. Immediate action required."
- **ACTION_REQUIRED:** "Security issues found. Review and address soon."
- **MAINTENANCE_NEEDED:** "Tech debt accumulating. Schedule cleanup."
- **REVIEW_RECOMMENDED:** "Minor issues detected. Review when convenient."
- **HEALTHY:** "Codebase is healthy. No urgent issues."

</step>

</process>

<success_criteria>

- Command identifies project type (Node.js or error)
- All analyzers execute (with graceful degradation for missing tools)
- `.planning/ANALYSIS_REPORT.md` generated with all sections
- User sees clear summary with actionable guidance
- Exit cleanly even if some analyzers fail

</success_criteria>

<examples>

**Example 1: Healthy project**

```
$ /gsd:analyze-codebase

=== Codebase Analysis Starting ===
Project: my-app
Time: 2026-01-08T10:30:00Z

Running analyzers in parallel...
[1/3] Dependency analysis complete: OK
[2/3] Security analysis complete: OK
[3/3] Tech debt analysis complete: CLEAN (score: 0)

═══════════════════════════════════════════════════
CODEBASE ANALYSIS COMPLETE
═══════════════════════════════════════════════════

Overall Status: HEALTHY
Action Priority: NONE

Codebase is healthy. No urgent issues.

Full report: .planning/ANALYSIS_REPORT.md
```

---

**Example 2: Security issues found**

```
$ /gsd:analyze-codebase

=== Codebase Analysis Starting ===
Project: legacy-app
Time: 2026-01-08T10:30:00Z

Running analyzers in parallel...
[1/3] Dependency analysis complete: ACTION_REQUIRED
[2/3] Security analysis complete: CRITICAL
[3/3] Tech debt analysis complete: MEDIUM (score: 75)

═══════════════════════════════════════════════════
CODEBASE ANALYSIS COMPLETE
═══════════════════════════════════════════════════

Overall Status: CRITICAL
Action Priority: IMMEDIATE

Security:     CRITICAL (Critical: 1, High: 3)
Dependencies: ACTION_REQUIRED (Outdated: 25, Major: 8)
Tech Debt:    MEDIUM (Score: 75)

CRITICAL issues found. Immediate action required.
Review: .planning/ANALYSIS_REPORT.md
Run: npm audit fix

Full report: .planning/ANALYSIS_REPORT.md
```

---

**Example 3: Non-Node.js project**

```
$ /gsd:analyze-codebase

ERROR: No package.json found

This command analyzes Node.js projects.
For non-Node.js projects, codebase analysis is not yet supported.
```

</examples>
