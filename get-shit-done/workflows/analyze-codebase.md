# Analyze Codebase Workflow

Master orchestration for comprehensive codebase analysis.

## Purpose

<purpose>
Orchestrate all codebase analyzers and aggregate findings into a single report.

This workflow:
1. Runs all three analyzers in parallel for speed
2. Collects and prioritizes findings across categories
3. Generates unified ANALYSIS_REPORT.md in .planning/

**Priority hierarchy:**
1. CRITICAL security > HIGH security > outdated-major > tech-debt

**Output contract:**
The ANALYSIS_REPORT.md serves as the contract between Phase 11 (analysis) and Phase 12 (suggestion engine). The suggestion engine reads this file to generate goal proposals.
</purpose>

## Prerequisites

<prerequisites>
  <required>
    - Node.js project with package.json
    - npm installed and available in PATH
    - .planning/ directory exists (GSD project initialized)
  </required>

  <optional>
    - jq for JSON aggregation (falls back to bash parsing)
    - Knip for tech debt detection
    - lockfile-lint for supply chain validation
  </optional>
</prerequisites>

## Process

<step name="check_prerequisites" number="1">
**Verify project setup**

```bash
# Check for package.json
if [ ! -f package.json ]; then
  echo "ERROR: No package.json found"
  echo "This workflow requires a Node.js project"
  echo ""
  echo "For non-Node.js projects, codebase analysis is not yet supported."
  exit 1
fi

# Check for .planning directory
if [ ! -d .planning ]; then
  echo "ERROR: No .planning/ directory found"
  echo "Initialize GSD project first with /gsd:new-project"
  exit 1
fi

# Check for jq availability
if command -v jq &> /dev/null; then
  HAS_JQ=true
else
  HAS_JQ=false
  echo "Note: jq not available, using basic JSON parsing"
fi

# Record start time
ANALYSIS_START=$(date +%s)
ANALYSIS_START_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "=== Codebase Analysis Starting ==="
echo "Project: $(basename $(pwd))"
echo "Time: $ANALYSIS_START_TIME"
echo ""
```
</step>

<step name="setup_analysis_directory" number="2">
**Create analysis output directory**

```bash
# Create analysis directory
ANALYSIS_DIR=".planning/.analysis"
mkdir -p "$ANALYSIS_DIR"

# Check for ignore file
if [ -f ".codebase-analysis-ignore" ]; then
  echo "Found .codebase-analysis-ignore - exclusions will be applied"
  HAS_IGNORE_FILE=true
else
  HAS_IGNORE_FILE=false
fi

# Initialize flags file for aggregation
> "$ANALYSIS_DIR/flags.env"

echo "Analysis directory: $ANALYSIS_DIR"
```
</step>

<step name="run_analyzers_parallel" number="3">
**Run all analyzers in parallel**

Each analyzer writes its findings to `$ANALYSIS_DIR/`:
- `dependency-findings.json` from analyze-dependencies
- `security-findings.json` from analyze-security
- `tech-debt-findings.json` from analyze-tech-debt

```bash
echo "Running analyzers in parallel..."
echo ""

# Track analyzer status
DEPS_STATUS="RUNNING"
SECURITY_STATUS="RUNNING"
TECHDEBT_STATUS="RUNNING"

# Function to run dependency analysis
run_deps_analysis() {
  echo "[1/3] Dependency analysis..."

  # Run npm audit
  npm audit --json --omit=dev > "$ANALYSIS_DIR/audit.json" 2>/dev/null || true

  # Run npm outdated
  npm outdated --json > "$ANALYSIS_DIR/outdated.json" 2>/dev/null || true

  # Parse results (simplified inline - full logic in analyze-dependencies.md)
  if [ "$HAS_JQ" = true ]; then
    CRITICAL=$(jq '[.vulnerabilities[] | select(.severity == "critical")] | length' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    HIGH=$(jq '[.vulnerabilities[] | select(.severity == "high")] | length' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    OUTDATED=$(jq 'keys | length' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")

    # Calculate major updates
    MAJOR_UPDATES=$(jq -r 'to_entries[] | select(
      (.value.current | split(".")[0]) != (.value.latest | split(".")[0])
    ) | .key' "$ANALYSIS_DIR/outdated.json" 2>/dev/null | wc -l | tr -d ' ' || echo "0")
  else
    CRITICAL=$(grep -c '"severity":"critical"' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    HIGH=$(grep -c '"severity":"high"' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    OUTDATED=$(grep -c '"current":' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")
    MAJOR_UPDATES="?"
  fi

  # Determine status
  if [ "$CRITICAL" -gt 0 ]; then
    DEP_STATUS="CRITICAL"
    DEP_PRIORITY="IMMEDIATE"
  elif [ "$HIGH" -gt 0 ]; then
    DEP_STATUS="ACTION_REQUIRED"
    DEP_PRIORITY="HIGH"
  else
    DEP_STATUS="OK"
    DEP_PRIORITY="NORMAL"
  fi

  # Write findings
  cat > "$ANALYSIS_DIR/dependency-findings.json" << DEPEOF
{
  "timestamp": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "type": "dependency-analysis",
  "summary": {
    "vulnerabilities": {
      "critical": $CRITICAL,
      "high": $HIGH
    },
    "outdated": {
      "total": $OUTDATED,
      "major": $MAJOR_UPDATES
    }
  },
  "status": "$DEP_STATUS",
  "priority": "$DEP_PRIORITY"
}
DEPEOF

  echo "[1/3] Dependency analysis complete: $DEP_STATUS"
}

# Function to run security analysis
run_security_analysis() {
  echo "[2/3] Security analysis..."

  # Run npm audit (reuse if exists from deps analysis)
  if [ ! -f "$ANALYSIS_DIR/audit.json" ]; then
    npm audit --json --omit=dev > "$ANALYSIS_DIR/audit.json" 2>/dev/null || true
  fi

  # Parse for security-specific metrics
  if [ "$HAS_JQ" = true ]; then
    SEC_CRITICAL=$(jq '[.vulnerabilities[] | select(.severity == "critical")] | length' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    SEC_HIGH=$(jq '[.vulnerabilities[] | select(.severity == "high")] | length' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")

    # Count fix availability
    AUTO_FIX=$(jq '[.vulnerabilities[] | select(.severity == "critical" or .severity == "high") | select(.fixAvailable == true)] | length' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
  else
    SEC_CRITICAL=$(grep -c '"severity":"critical"' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    SEC_HIGH=$(grep -c '"severity":"high"' "$ANALYSIS_DIR/audit.json" 2>/dev/null || echo "0")
    AUTO_FIX="?"
  fi

  # Run lockfile-lint if available
  if command -v lockfile-lint &> /dev/null && [ -f package-lock.json ]; then
    lockfile-lint --path package-lock.json --type npm --validate-https --allowed-hosts npm > "$ANALYSIS_DIR/lockfile-lint.txt" 2>&1
    if [ $? -eq 0 ]; then
      LOCKFILE_STATUS="PASS"
    else
      LOCKFILE_STATUS="FAIL"
    fi
  else
    LOCKFILE_STATUS="SKIPPED"
  fi

  # Determine status
  if [ "$SEC_CRITICAL" -gt 0 ]; then
    SEC_STATUS="CRITICAL"
    SEC_PRIORITY="IMMEDIATE"
  elif [ "$SEC_HIGH" -gt 0 ]; then
    SEC_STATUS="HIGH"
    SEC_PRIORITY="URGENT"
  elif [ "$LOCKFILE_STATUS" = "FAIL" ]; then
    SEC_STATUS="WARNING"
    SEC_PRIORITY="REVIEW"
  else
    SEC_STATUS="OK"
    SEC_PRIORITY="NONE"
  fi

  # Write findings
  cat > "$ANALYSIS_DIR/security-findings.json" << SECEOF
{
  "timestamp": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "type": "security-analysis",
  "summary": {
    "vulnerabilities": {
      "critical": $SEC_CRITICAL,
      "high": $SEC_HIGH,
      "total_actionable": $((SEC_CRITICAL + SEC_HIGH))
    },
    "fixes": {
      "auto_fixable": $AUTO_FIX
    },
    "supply_chain": {
      "lockfile_validation": "$LOCKFILE_STATUS"
    }
  },
  "status": "$SEC_STATUS",
  "priority": "$SEC_PRIORITY"
}
SECEOF

  echo "[2/3] Security analysis complete: $SEC_STATUS"
}

# Function to run tech debt analysis
run_techdebt_analysis() {
  echo "[3/3] Tech debt analysis..."

  # Run npm outdated (reuse if exists)
  if [ ! -f "$ANALYSIS_DIR/outdated.json" ]; then
    npm outdated --json > "$ANALYSIS_DIR/outdated.json" 2>/dev/null || true
  fi

  # Initialize counters
  UNUSED_FILES=0
  UNUSED_DEPS=0
  UNUSED_EXPORTS=0

  # Run Knip if available
  if npx knip --version &> /dev/null 2>&1; then
    timeout 120 npx knip --reporter json > "$ANALYSIS_DIR/knip.json" 2>/dev/null || true

    if [ "$HAS_JQ" = true ] && [ -s "$ANALYSIS_DIR/knip.json" ]; then
      UNUSED_FILES=$(jq '.files | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
      UNUSED_DEPS=$(jq '.dependencies | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
      UNUSED_EXPORTS=$(jq '.exports | length' "$ANALYSIS_DIR/knip.json" 2>/dev/null || echo "0")
    fi
  fi

  # Count outdated packages
  if [ "$HAS_JQ" = true ]; then
    OUTDATED_COUNT=$(jq 'keys | length' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")
    MAJOR_OUTDATED=$(jq -r 'to_entries[] | select(
      (.value.current | split(".")[0]) != (.value.latest | split(".")[0])
    ) | .key' "$ANALYSIS_DIR/outdated.json" 2>/dev/null | wc -l | tr -d ' ' || echo "0")
  else
    OUTDATED_COUNT=$(grep -c '"current":' "$ANALYSIS_DIR/outdated.json" 2>/dev/null || echo "0")
    MAJOR_OUTDATED="0"
  fi

  # Calculate tech debt score
  # Weights: deps (10), files (5), exports (2), major outdated (3)
  DEBT_SCORE=$((UNUSED_DEPS * 10 + UNUSED_FILES * 5 + UNUSED_EXPORTS * 2 + MAJOR_OUTDATED * 3))

  # Determine level
  if [ $DEBT_SCORE -ge 100 ]; then
    DEBT_LEVEL="HIGH"
    DEBT_PRIORITY="RECOMMENDED"
  elif [ $DEBT_SCORE -ge 50 ]; then
    DEBT_LEVEL="MEDIUM"
    DEBT_PRIORITY="SUGGESTED"
  elif [ $DEBT_SCORE -gt 0 ]; then
    DEBT_LEVEL="LOW"
    DEBT_PRIORITY="OPTIONAL"
  else
    DEBT_LEVEL="CLEAN"
    DEBT_PRIORITY="NONE"
  fi

  # Write findings
  cat > "$ANALYSIS_DIR/tech-debt-findings.json" << TDEOF
{
  "timestamp": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "type": "tech-debt-analysis",
  "summary": {
    "unused_code": {
      "files": $UNUSED_FILES,
      "dependencies": $UNUSED_DEPS,
      "exports": $UNUSED_EXPORTS
    },
    "outdated": {
      "total": $OUTDATED_COUNT,
      "major": $MAJOR_OUTDATED
    }
  },
  "score": $DEBT_SCORE,
  "level": "$DEBT_LEVEL",
  "priority": "$DEBT_PRIORITY"
}
TDEOF

  echo "[3/3] Tech debt analysis complete: $DEBT_LEVEL (score: $DEBT_SCORE)"
}

# Run all analyzers (sequentially for shell compatibility, but independent)
run_deps_analysis &
DEPS_PID=$!

run_security_analysis &
SECURITY_PID=$!

run_techdebt_analysis &
TECHDEBT_PID=$!

# Wait for all to complete
wait $DEPS_PID 2>/dev/null || DEPS_STATUS="FAILED"
wait $SECURITY_PID 2>/dev/null || SECURITY_STATUS="FAILED"
wait $TECHDEBT_PID 2>/dev/null || TECHDEBT_STATUS="FAILED"

echo ""
echo "All analyzers complete."
```

**Parallel execution pattern:**
- Each analyzer runs in background (&)
- Main process waits for all to complete
- Individual failures don't block others
</step>

<step name="aggregate_findings" number="4">
**Aggregate findings from all analyzers**

```bash
echo ""
echo "Aggregating findings..."

# Read findings from each analyzer
if [ "$HAS_JQ" = true ]; then
  # Dependency findings
  DEP_STATUS=$(jq -r '.status' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null || echo "UNKNOWN")
  DEP_PRIORITY=$(jq -r '.priority' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null || echo "UNKNOWN")
  DEP_CRITICAL=$(jq -r '.summary.vulnerabilities.critical' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null || echo "0")
  DEP_HIGH=$(jq -r '.summary.vulnerabilities.high' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null || echo "0")
  DEP_OUTDATED=$(jq -r '.summary.outdated.total' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null || echo "0")
  DEP_MAJOR=$(jq -r '.summary.outdated.major' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null || echo "0")

  # Security findings
  SEC_STATUS=$(jq -r '.status' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null || echo "UNKNOWN")
  SEC_PRIORITY=$(jq -r '.priority' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null || echo "UNKNOWN")
  SEC_CRITICAL=$(jq -r '.summary.vulnerabilities.critical' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null || echo "0")
  SEC_HIGH=$(jq -r '.summary.vulnerabilities.high' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null || echo "0")
  SEC_AUTO_FIX=$(jq -r '.summary.fixes.auto_fixable' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null || echo "0")
  SEC_LOCKFILE=$(jq -r '.summary.supply_chain.lockfile_validation' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null || echo "SKIPPED")

  # Tech debt findings
  TD_LEVEL=$(jq -r '.level' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null || echo "UNKNOWN")
  TD_PRIORITY=$(jq -r '.priority' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null || echo "UNKNOWN")
  TD_SCORE=$(jq -r '.score' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null || echo "0")
  TD_UNUSED_FILES=$(jq -r '.summary.unused_code.files' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null || echo "0")
  TD_UNUSED_DEPS=$(jq -r '.summary.unused_code.dependencies' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null || echo "0")
  TD_UNUSED_EXPORTS=$(jq -r '.summary.unused_code.exports' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null || echo "0")
else
  # Fallback: grep parsing
  DEP_STATUS=$(grep -o '"status":"[^"]*"' "$ANALYSIS_DIR/dependency-findings.json" 2>/dev/null | cut -d'"' -f4 || echo "UNKNOWN")
  SEC_STATUS=$(grep -o '"status":"[^"]*"' "$ANALYSIS_DIR/security-findings.json" 2>/dev/null | cut -d'"' -f4 || echo "UNKNOWN")
  TD_LEVEL=$(grep -o '"level":"[^"]*"' "$ANALYSIS_DIR/tech-debt-findings.json" 2>/dev/null | cut -d'"' -f4 || echo "UNKNOWN")
  # Other fields use defaults
  DEP_CRITICAL=0; DEP_HIGH=0; DEP_OUTDATED=0; DEP_MAJOR=0
  SEC_CRITICAL=0; SEC_HIGH=0; SEC_AUTO_FIX=0; SEC_LOCKFILE="UNKNOWN"
  TD_SCORE=0; TD_UNUSED_FILES=0; TD_UNUSED_DEPS=0; TD_UNUSED_EXPORTS=0
fi

# Determine overall status (priority: CRITICAL > HIGH > MEDIUM > LOW)
if [ "$SEC_STATUS" = "CRITICAL" ] || [ "$DEP_STATUS" = "CRITICAL" ]; then
  OVERALL_STATUS="CRITICAL"
  OVERALL_PRIORITY="IMMEDIATE"
elif [ "$SEC_STATUS" = "HIGH" ] || [ "$DEP_STATUS" = "ACTION_REQUIRED" ]; then
  OVERALL_STATUS="ACTION_REQUIRED"
  OVERALL_PRIORITY="URGENT"
elif [ "$TD_LEVEL" = "HIGH" ]; then
  OVERALL_STATUS="MAINTENANCE_NEEDED"
  OVERALL_PRIORITY="RECOMMENDED"
elif [ "$SEC_STATUS" = "WARNING" ] || [ "$TD_LEVEL" = "MEDIUM" ]; then
  OVERALL_STATUS="REVIEW_RECOMMENDED"
  OVERALL_PRIORITY="SUGGESTED"
else
  OVERALL_STATUS="HEALTHY"
  OVERALL_PRIORITY="NONE"
fi

# Count total issues by category
TOTAL_CRITICAL=$((SEC_CRITICAL))
TOTAL_HIGH=$((SEC_HIGH))
TOTAL_MEDIUM=$((DEP_MAJOR + TD_UNUSED_DEPS))
TOTAL_LOW=$((DEP_OUTDATED - DEP_MAJOR + TD_UNUSED_FILES + TD_UNUSED_EXPORTS))

echo "Overall status: $OVERALL_STATUS"
echo "Priority: $OVERALL_PRIORITY"
```
</step>

<step name="generate_analysis_report" number="5">
**Generate unified ANALYSIS_REPORT.md**

```bash
ANALYSIS_END=$(date +%s)
ANALYSIS_DURATION=$((ANALYSIS_END - ANALYSIS_START))
ANALYSIS_END_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Create the analysis report
cat > ".planning/ANALYSIS_REPORT.md" << EOF
# Codebase Analysis Report

**Project:** $(basename $(pwd))
**Generated:** $ANALYSIS_END_TIME
**Duration:** ${ANALYSIS_DURATION}s

---

## Executive Summary

**Overall Status:** $OVERALL_STATUS
**Action Priority:** $OVERALL_PRIORITY

| Category | Critical | High | Medium | Low |
|----------|----------|------|--------|-----|
| Security | $SEC_CRITICAL | $SEC_HIGH | 0 | 0 |
| Dependencies | 0 | 0 | $DEP_MAJOR | $((DEP_OUTDATED - DEP_MAJOR)) |
| Tech Debt | 0 | 0 | $TD_UNUSED_DEPS | $((TD_UNUSED_FILES + TD_UNUSED_EXPORTS)) |
| **Total** | **$TOTAL_CRITICAL** | **$TOTAL_HIGH** | **$TOTAL_MEDIUM** | **$TOTAL_LOW** |

---

## Security Issues

**Status:** $SEC_STATUS
**Priority:** $SEC_PRIORITY

### Vulnerabilities

| Severity | Count | Fix Available |
|----------|-------|---------------|
| Critical | $SEC_CRITICAL | - |
| High | $SEC_HIGH | - |
| **Total Actionable** | **$((SEC_CRITICAL + SEC_HIGH))** | $SEC_AUTO_FIX auto-fixable |

### Supply Chain

- **Lockfile validation:** $SEC_LOCKFILE

EOF

# Add security recommendations
if [ "$SEC_CRITICAL" -gt 0 ] || [ "$SEC_HIGH" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
### Recommended Actions

1. **IMMEDIATE:** Run \`npm audit fix\` to address auto-fixable vulnerabilities
2. Review \`npm audit\` output for vulnerabilities requiring manual intervention
3. Consider package alternatives for unfixable vulnerabilities

EOF
fi

cat >> ".planning/ANALYSIS_REPORT.md" << EOF

---

## Dependency Issues

**Status:** $DEP_STATUS
**Priority:** $DEP_PRIORITY

### Outdated Packages

| Category | Count |
|----------|-------|
| Major updates available | $DEP_MAJOR |
| Total outdated | $DEP_OUTDATED |

EOF

if [ "$DEP_MAJOR" != "?" ] && [ "$DEP_MAJOR" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
### Recommended Actions

1. Review changelogs for major version updates
2. Run \`npx npm-check-updates -i\` for interactive updates
3. Test thoroughly after major version bumps

EOF
fi

cat >> ".planning/ANALYSIS_REPORT.md" << EOF

---

## Tech Debt

**Level:** $TD_LEVEL
**Score:** $TD_SCORE
**Priority:** $TD_PRIORITY

### Unused Code

| Category | Count |
|----------|-------|
| Unused files | $TD_UNUSED_FILES |
| Unused dependencies | $TD_UNUSED_DEPS |
| Unused exports | $TD_UNUSED_EXPORTS |

EOF

if [ "$TD_LEVEL" = "HIGH" ] || [ "$TD_LEVEL" = "MEDIUM" ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
### Recommended Actions

1. Remove unused dependencies to reduce install time and attack surface
2. Delete unused files to improve codebase navigability
3. Clean up unused exports when touching affected files

EOF
fi

cat >> ".planning/ANALYSIS_REPORT.md" << EOF

---

## Recommendations for Suggestion Engine

<!-- Phase 12 suggestion engine reads this section -->

### Prioritized Action Items

EOF

# Generate prioritized recommendations
ITEM_NUM=1

if [ "$SEC_CRITICAL" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
$ITEM_NUM. **[CRITICAL] Fix critical security vulnerabilities**
   - Count: $SEC_CRITICAL
   - Effort: quick-fix (if auto-fixable) / moderate (if manual)
   - Command: \`npm audit fix\`
   - Goal type: security-fix

EOF
  ITEM_NUM=$((ITEM_NUM + 1))
fi

if [ "$SEC_HIGH" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
$ITEM_NUM. **[HIGH] Address high-severity vulnerabilities**
   - Count: $SEC_HIGH
   - Effort: moderate
   - Command: \`npm audit\` for details
   - Goal type: security-fix

EOF
  ITEM_NUM=$((ITEM_NUM + 1))
fi

if [ "$DEP_MAJOR" != "?" ] && [ "$DEP_MAJOR" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
$ITEM_NUM. **[MEDIUM] Update major version dependencies**
   - Count: $DEP_MAJOR packages
   - Effort: significant (breaking changes possible)
   - Command: \`npx npm-check-updates -i\`
   - Goal type: dependency-update

EOF
  ITEM_NUM=$((ITEM_NUM + 1))
fi

if [ "$TD_UNUSED_DEPS" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
$ITEM_NUM. **[MEDIUM] Remove unused dependencies**
   - Count: $TD_UNUSED_DEPS packages
   - Effort: quick-fix
   - Command: Review Knip output, then \`npm uninstall <packages>\`
   - Goal type: tech-debt-cleanup

EOF
  ITEM_NUM=$((ITEM_NUM + 1))
fi

if [ "$TD_UNUSED_FILES" -gt 0 ]; then
  cat >> ".planning/ANALYSIS_REPORT.md" << EOF
$ITEM_NUM. **[LOW] Delete unused files**
   - Count: $TD_UNUSED_FILES files
   - Effort: quick-fix
   - Command: Review \`unused-files.txt\`, then delete
   - Goal type: tech-debt-cleanup

EOF
  ITEM_NUM=$((ITEM_NUM + 1))
fi

if [ "$ITEM_NUM" -eq 1 ]; then
  echo "**All clear!** No significant issues detected." >> ".planning/ANALYSIS_REPORT.md"
fi

cat >> ".planning/ANALYSIS_REPORT.md" << EOF

---

## Metadata

**Analysis Configuration:**
- jq available: $HAS_JQ
- Ignore file: $HAS_IGNORE_FILE

**Tool Outputs:**
- Dependency findings: \`.planning/.analysis/dependency-findings.json\`
- Security findings: \`.planning/.analysis/security-findings.json\`
- Tech debt findings: \`.planning/.analysis/tech-debt-findings.json\`

**Timing:**
- Started: $ANALYSIS_START_TIME
- Completed: $ANALYSIS_END_TIME
- Duration: ${ANALYSIS_DURATION}s

---

*Generated by analyze-codebase orchestrator*
*Report location: .planning/ANALYSIS_REPORT.md*
EOF

echo ""
echo "=== Analysis Complete ==="
echo "Report: .planning/ANALYSIS_REPORT.md"
echo "Status: $OVERALL_STATUS"
echo "Duration: ${ANALYSIS_DURATION}s"
```
</step>

<step name="display_summary" number="6">
**Display summary to user**

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "CODEBASE ANALYSIS COMPLETE"
echo "═══════════════════════════════════════════════════"
echo ""
echo "Overall Status: $OVERALL_STATUS"
echo "Action Priority: $OVERALL_PRIORITY"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "FINDINGS SUMMARY"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Security:     $SEC_STATUS (Critical: $SEC_CRITICAL, High: $SEC_HIGH)"
echo "Dependencies: $DEP_STATUS (Outdated: $DEP_OUTDATED, Major: $DEP_MAJOR)"
echo "Tech Debt:    $TD_LEVEL (Score: $TD_SCORE)"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "NEXT STEPS"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

case "$OVERALL_STATUS" in
  "CRITICAL")
    echo "CRITICAL issues found. Immediate action required."
    echo "Review: .planning/ANALYSIS_REPORT.md"
    echo "Run: npm audit fix"
    ;;
  "ACTION_REQUIRED")
    echo "Security issues found. Review and address soon."
    echo "Review: .planning/ANALYSIS_REPORT.md"
    ;;
  "MAINTENANCE_NEEDED")
    echo "Tech debt accumulating. Schedule cleanup."
    echo "Review: .planning/ANALYSIS_REPORT.md"
    ;;
  "REVIEW_RECOMMENDED")
    echo "Minor issues detected. Review when convenient."
    echo "Review: .planning/ANALYSIS_REPORT.md"
    ;;
  "HEALTHY")
    echo "Codebase is healthy. No urgent issues."
    ;;
esac

echo ""
echo "Full report: .planning/ANALYSIS_REPORT.md"
echo "═══════════════════════════════════════════════════"
```
</step>

<step name="generate_suggestions" number="7" optional="true">
**Optionally generate goal suggestions from analysis**

Check configuration for auto-suggest behavior:

```bash
# Default: auto-suggest enabled
AUTO_SUGGEST="true"

# Check config for explicit setting
if [ -f .planning/config.json ]; then
  CONFIG_SUGGEST=$(grep -o '"auto_suggest": *[^,}]*' .planning/config.json 2>/dev/null | grep -o 'true\|false')
  if [ "$CONFIG_SUGGEST" = "false" ]; then
    AUTO_SUGGEST="false"
  fi
fi

echo ""
echo "Auto-suggest: $AUTO_SUGGEST"
```

**If auto-suggest enabled:**

```bash
if [ "$AUTO_SUGGEST" = "true" ]; then
  echo ""
  echo "Generating goal suggestions..."

  # Follow generate-suggestions.md workflow to create GOAL_PROPOSALS.md
  # The workflow:
  # 1. Parses recommendations from ANALYSIS_REPORT.md
  # 2. Queries knowledge base for patterns (if adaptive.enabled)
  # 3. Scores suggestions using priority + effort + patterns
  # 4. Generates GOAL_PROPOSALS.md

  # Check if GOAL_PROPOSALS.md was created
  if [ -f .planning/GOAL_PROPOSALS.md ]; then
    SUGGESTION_COUNT=$(grep '^\*\*Total Suggestions:\*\*' .planning/GOAL_PROPOSALS.md 2>/dev/null | grep -oE '[0-9]+' || echo "0")
    QUICK_WINS=$(grep '^\*\*Estimated Quick Wins:\*\*' .planning/GOAL_PROPOSALS.md 2>/dev/null | grep -oE '[0-9]+' || echo "0")

    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "SUGGESTIONS"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""
    echo "Generated $SUGGESTION_COUNT suggestions"
    echo "Quick wins: $QUICK_WINS"
    echo ""
    echo "View: .planning/GOAL_PROPOSALS.md"
  fi
fi
```

**If auto-suggest disabled:**

```bash
if [ "$AUTO_SUGGEST" = "false" ]; then
  echo ""
  echo "Suggestion generation skipped (config: auto_suggest=false)"
  echo "Run /gsd:suggest to generate manually"
fi
```

**Configuration:**

Add to `.planning/config.json` to control behavior:

```json
{
  "analysis": {
    "auto_suggest": true  // Generate suggestions after analysis (default: true)
  }
}
```

When `auto_suggest` is:
- `true` (default): Automatically generate GOAL_PROPOSALS.md after analysis
- `false`: Skip suggestion generation, user can run `/gsd:suggest` manually

**Graceful degradation:**
- If generate-suggestions workflow fails, log warning and continue
- Analysis report is still valid even without suggestions
- Never fail the entire analysis due to suggestion generation issues
</step>

## Output Format

<output>
**Primary output:**
- `.planning/ANALYSIS_REPORT.md` - Unified analysis report
- `.planning/GOAL_PROPOSALS.md` - Prioritized suggestions (if auto_suggest enabled)

**Supporting outputs (in .planning/.analysis/):**
- `dependency-findings.json` - Dependency analysis results
- `security-findings.json` - Security analysis results
- `tech-debt-findings.json` - Tech debt analysis results
- `audit.json` - Raw npm audit output
- `outdated.json` - Raw npm outdated output
- `knip.json` - Raw Knip output (if available)

**Report structure:**
1. Executive Summary - Overall status and issue counts
2. Security Issues - Vulnerabilities and supply chain
3. Dependency Issues - Outdated packages
4. Tech Debt - Unused code and maintenance burden
5. Recommendations - Prioritized action items for suggestion engine
6. Metadata - Analysis configuration and timing

**Suggestion engine output (if auto_suggest enabled):**
After generating ANALYSIS_REPORT.md, the workflow optionally runs generate-suggestions.md to produce GOAL_PROPOSALS.md with:
- Prioritized suggestions sorted by score
- Pattern insights from knowledge base (if available)
- Quick wins identification
- Goal type groupings

**Suggestion engine contract:**
The "Recommendations for Suggestion Engine" section provides structured data:
- Priority level: CRITICAL, HIGH, MEDIUM, LOW
- Issue count
- Effort estimate: quick-fix, moderate, significant
- Suggested command
- Goal type for categorization
</output>

## Error Handling

<error_handling>
**Partial failure handling:**
- Individual analyzer failures don't block others
- Failed analyzers report "UNKNOWN" status
- Report still generated with available data

**Graceful degradation:**
- No jq: Fall back to grep parsing
- No Knip: Skip tech debt detection, note in report
- npm audit failure: Report empty vulnerabilities
- Timeout on Knip: Use empty results, note timeout

**Hard failures (workflow exits):**
- No package.json: Cannot analyze non-Node.js projects
- No .planning/ directory: Project not initialized
</error_handling>

## Configuration

<configuration>
**Ignore file (.codebase-analysis-ignore):**
Create this file to exclude paths from analysis:
```
# Exclude test fixtures
test/fixtures/
# Exclude generated code
dist/
build/
```

**Environment variables:**
- `ANALYSIS_DIR` - Override analysis output directory (default: `.planning/.analysis`)

**Auto-suggest configuration (in .planning/config.json):**
```json
{
  "analysis": {
    "auto_suggest": true  // Generate suggestions after analysis (default: true)
  }
}
```

When `auto_suggest` is:
- `true` (default): Automatically generate GOAL_PROPOSALS.md after analysis completes
- `false`: Skip suggestion generation; user can run `/gsd:suggest` manually
</configuration>

## Integration

<integration>
**Invoked by:**
- `/gsd:analyze-codebase` command
- Manual workflow execution
- CI/CD pipelines

**Feeds into:**
- Phase 12 suggestion engine (reads ANALYSIS_REPORT.md)
- CLAUDE.md dynamic suggestions section

**Example CI integration:**
```yaml
- name: Run codebase analysis
  run: |
    # Source or execute the workflow
    # Check overall status
    if grep -q "CRITICAL" .planning/ANALYSIS_REPORT.md; then
      echo "Critical issues found"
      exit 1
    fi
```
</integration>

---

*Master orchestration workflow for GSD codebase analysis*
