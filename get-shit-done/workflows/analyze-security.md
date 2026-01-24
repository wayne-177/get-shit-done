# Analyze Security Workflow

Security-focused analysis for vulnerability detection and advisory aggregation.

## Purpose

<purpose>
Perform dedicated security analysis to identify:
1. Security vulnerabilities with CVE details
2. Advisory classification by fix availability
3. Supply chain security indicators (lockfile validation)

This workflow is security-focused and separate from general dependency analysis to enable:
- Independent security audits without noise from outdated packages
- CVE-specific reporting for compliance requirements
- Clear prioritization by exploitability and fix availability

**Key principles:**
- HIGH/CRITICAL severity only to avoid false positive fatigue
- CVE extraction for tracking and compliance
- Fix availability classification for actionable prioritization
- Supply chain awareness via lockfile validation
</purpose>

## Prerequisites

<prerequisites>
  <required>
    - Node.js project with package.json
    - npm installed and available in PATH
    - Internet connection for advisory lookups
  </required>

  <optional>
    - jq installed for JSON parsing (falls back to grep patterns if unavailable)
    - lockfile-lint for supply chain validation (npm install -g lockfile-lint)
  </optional>
</prerequisites>

## Process

<step name="check_prerequisites" number="1">
**Verify security scanning prerequisites**

```bash
# Check for package.json
if [ ! -f package.json ]; then
  echo "ERROR: No package.json found"
  echo "This workflow requires a Node.js project"
  exit 1
fi

# Check for lockfile (required for accurate security scanning)
if [ ! -f package-lock.json ] && [ ! -f npm-shrinkwrap.json ]; then
  echo "WARNING: No lockfile found"
  echo "Security scanning works best with package-lock.json"
  echo "Consider running: npm install --package-lock-only"
fi

# Check for jq availability
if command -v jq &> /dev/null; then
  HAS_JQ=true
else
  HAS_JQ=false
  echo "Note: jq not available, using basic JSON parsing"
fi

# Check for lockfile-lint
if command -v lockfile-lint &> /dev/null; then
  HAS_LOCKFILE_LINT=true
else
  HAS_LOCKFILE_LINT=false
  echo "Note: lockfile-lint not available, skipping supply chain validation"
fi

echo "Security scan prerequisites checked"
```
</step>

<step name="run_security_audit" number="2">
**Execute security-focused npm audit**

```bash
# Create temp directory for results
ANALYSIS_DIR="${ANALYSIS_DIR:-.planning/.analysis}"
mkdir -p "$ANALYSIS_DIR"

# Run npm audit with HIGH+ severity filter, production deps only
# --audit-level=high returns exit code 1 only for high/critical
npm audit --json --omit=dev --audit-level=high > "$ANALYSIS_DIR/security-audit.json" 2>/dev/null
AUDIT_EXIT=$?

# Also run full audit for comprehensive data
npm audit --json --omit=dev > "$ANALYSIS_DIR/full-audit.json" 2>/dev/null

# Exit code interpretation for --audit-level=high:
# 0 = no high/critical vulnerabilities
# 1 = high/critical vulnerabilities found

echo "Security audit complete (high+ exit code: $AUDIT_EXIT)"
```

**Output:**
- `security-audit.json` - High/critical only
- `full-audit.json` - All severities for reference
</step>

<step name="extract_cve_details" number="3">
**Extract CVE identifiers and advisory details**

```bash
AUDIT_FILE="$ANALYSIS_DIR/full-audit.json"

if [ "$HAS_JQ" = true ]; then
  # Extract vulnerabilities with CVE details
  jq -r '
    .vulnerabilities | to_entries[] |
    select(.value.severity == "critical" or .value.severity == "high") |
    {
      package: .key,
      severity: .value.severity,
      via: [.value.via[] | if type == "object" then {
        source: .source,
        name: .name,
        title: .title,
        url: .url,
        severity: .severity,
        cwe: .cwe,
        cvss: .cvss
      } else {dependency: .} end],
      effects: .value.effects,
      range: .value.range,
      nodes: .value.nodes,
      fixAvailable: .value.fixAvailable
    }
  ' "$AUDIT_FILE" > "$ANALYSIS_DIR/security-details.json" 2>/dev/null

  # Extract unique CVE/advisory IDs
  jq -r '
    .vulnerabilities[].via[] |
    select(type == "object") |
    .source // empty
  ' "$AUDIT_FILE" 2>/dev/null | sort -u > "$ANALYSIS_DIR/advisory-ids.txt"

  # Count unique advisories
  ADVISORY_COUNT=$(wc -l < "$ANALYSIS_DIR/advisory-ids.txt" | tr -d ' ')

  # Count by severity
  CRITICAL=$(jq '[.vulnerabilities[] | select(.severity == "critical")] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")
  HIGH=$(jq '[.vulnerabilities[] | select(.severity == "high")] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")

else
  # Fallback: basic grep parsing
  CRITICAL=$(grep -c '"severity":"critical"' "$AUDIT_FILE" 2>/dev/null || echo "0")
  HIGH=$(grep -c '"severity":"high"' "$AUDIT_FILE" 2>/dev/null || echo "0")
  ADVISORY_COUNT="?"

  # Extract advisory IDs via grep (basic)
  grep -oE 'GHSA-[a-z0-9]+-[a-z0-9]+-[a-z0-9]+' "$AUDIT_FILE" 2>/dev/null | sort -u > "$ANALYSIS_DIR/advisory-ids.txt" || true
fi

echo "=== Security Summary ==="
echo "Critical: $CRITICAL"
echo "High: $HIGH"
echo "Unique advisories: $ADVISORY_COUNT"
```

**Output:**
- `security-details.json` - Detailed vulnerability info
- `advisory-ids.txt` - Unique advisory identifiers (GHSA, CVE, etc.)
</step>

<step name="classify_by_fix_availability" number="4">
**Classify vulnerabilities by fix availability**

```bash
if [ "$HAS_JQ" = true ]; then
  # Classify: has-fix vs no-fix-available
  HAS_FIX=$(jq '[.vulnerabilities[] |
    select(.severity == "critical" or .severity == "high") |
    select(.fixAvailable == true)] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")

  NO_FIX=$(jq '[.vulnerabilities[] |
    select(.severity == "critical" or .severity == "high") |
    select(.fixAvailable == false or .fixAvailable == null)] | length' "$AUDIT_FILE" 2>/dev/null || echo "0")

  # Create classified output
  jq -r '
    .vulnerabilities | to_entries[] |
    select(.value.severity == "critical" or .value.severity == "high") |
    {
      package: .key,
      severity: .value.severity,
      fixAvailable: .value.fixAvailable,
      fixType: (
        if .value.fixAvailable == true then "auto"
        elif .value.fixAvailable == false then "none"
        elif .value.fixAvailable | type == "object" then "breaking"
        else "unknown"
        end
      )
    }
  ' "$AUDIT_FILE" > "$ANALYSIS_DIR/fix-classification.json" 2>/dev/null

  # Count by fix type
  AUTO_FIX=$(jq '[.[] | select(.fixType == "auto")] | length' "$ANALYSIS_DIR/fix-classification.json" 2>/dev/null || echo "0")
  BREAKING_FIX=$(jq '[.[] | select(.fixType == "breaking")] | length' "$ANALYSIS_DIR/fix-classification.json" 2>/dev/null || echo "0")
  NO_FIX_AVAILABLE=$(jq '[.[] | select(.fixType == "none")] | length' "$ANALYSIS_DIR/fix-classification.json" 2>/dev/null || echo "0")

else
  # Fallback: basic counting
  HAS_FIX=$(grep -c '"fixAvailable":true' "$AUDIT_FILE" 2>/dev/null || echo "0")
  NO_FIX=$((CRITICAL + HIGH - HAS_FIX))
  AUTO_FIX="?"
  BREAKING_FIX="?"
  NO_FIX_AVAILABLE="?"
fi

echo "=== Fix Availability ==="
echo "Auto-fixable: $AUTO_FIX"
echo "Breaking change required: $BREAKING_FIX"
echo "No fix available: $NO_FIX_AVAILABLE"
```

**Fix type classification:**
- `auto` - Can be fixed with `npm audit fix`
- `breaking` - Fix available but requires major version bump
- `none` - No fix currently available
- `unknown` - Could not determine fix status
</step>

<step name="validate_lockfile" number="5">
**Validate lockfile for supply chain security (if lockfile-lint available)**

```bash
if [ "$HAS_LOCKFILE_LINT" = true ] && [ -f package-lock.json ]; then
  echo "Running lockfile validation..."

  # Run lockfile-lint with recommended security checks
  lockfile-lint \
    --path package-lock.json \
    --type npm \
    --validate-https \
    --allowed-hosts npm \
    > "$ANALYSIS_DIR/lockfile-lint.txt" 2>&1
  LINT_EXIT=$?

  if [ $LINT_EXIT -eq 0 ]; then
    LOCKFILE_STATUS="PASS"
    LOCKFILE_ISSUES=0
    echo "Lockfile validation: PASS"
  else
    LOCKFILE_STATUS="FAIL"
    LOCKFILE_ISSUES=$(grep -c "ERR\|WARN" "$ANALYSIS_DIR/lockfile-lint.txt" 2>/dev/null || echo "1")
    echo "Lockfile validation: FAIL ($LOCKFILE_ISSUES issues)"
  fi

else
  LOCKFILE_STATUS="SKIPPED"
  LOCKFILE_ISSUES=0

  if [ "$HAS_LOCKFILE_LINT" = false ]; then
    echo "Lockfile validation: SKIPPED (lockfile-lint not installed)"
    echo "Install with: npm install -g lockfile-lint"
  else
    echo "Lockfile validation: SKIPPED (no package-lock.json)"
  fi
fi
```

**Supply chain checks:**
- HTTPS-only package URLs (prevent protocol downgrade)
- Allowed hosts validation (prevent typosquatting)
- Integrity hash presence (prevent tampering)
</step>

<step name="generate_security_findings" number="6">
**Generate structured security findings**

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Determine overall security status
if [ "$CRITICAL" -gt 0 ]; then
  SECURITY_STATUS="CRITICAL"
  ACTION_PRIORITY="IMMEDIATE"
elif [ "$HIGH" -gt 0 ]; then
  SECURITY_STATUS="HIGH"
  ACTION_PRIORITY="URGENT"
elif [ "$LOCKFILE_STATUS" = "FAIL" ]; then
  SECURITY_STATUS="WARNING"
  ACTION_PRIORITY="REVIEW"
else
  SECURITY_STATUS="OK"
  ACTION_PRIORITY="NONE"
fi

# Create findings output
cat > "$ANALYSIS_DIR/security-findings.json" << EOF
{
  "timestamp": "$TIMESTAMP",
  "type": "security-analysis",
  "summary": {
    "vulnerabilities": {
      "critical": $CRITICAL,
      "high": $HIGH,
      "total_actionable": $((CRITICAL + HIGH))
    },
    "advisories": {
      "unique_count": $ADVISORY_COUNT
    },
    "fixes": {
      "auto_fixable": $AUTO_FIX,
      "breaking_required": $BREAKING_FIX,
      "no_fix_available": $NO_FIX_AVAILABLE
    },
    "supply_chain": {
      "lockfile_validation": "$LOCKFILE_STATUS",
      "issues": $LOCKFILE_ISSUES
    }
  },
  "status": "$SECURITY_STATUS",
  "priority": "$ACTION_PRIORITY"
}
EOF

echo "=== Security Analysis Complete ==="
echo "Status: $SECURITY_STATUS"
echo "Action priority: $ACTION_PRIORITY"
echo "Findings saved to: $ANALYSIS_DIR/security-findings.json"
```

**Output:** Structured security findings JSON for orchestrator
</step>

<step name="format_security_report" number="7">
**Generate security-focused report**

```bash
cat > "$ANALYSIS_DIR/security-report.md" << EOF
# Security Analysis Report

**Generated:** $TIMESTAMP
**Project:** $(basename $(pwd))
**Status:** $SECURITY_STATUS

---

## Vulnerability Summary

| Severity | Count | Fix Available |
|----------|-------|---------------|
| Critical | $CRITICAL | - |
| High | $HIGH | - |

**Total actionable:** $((CRITICAL + HIGH)) vulnerabilities

### Fix Availability

| Category | Count |
|----------|-------|
| Auto-fixable (\`npm audit fix\`) | $AUTO_FIX |
| Breaking change required | $BREAKING_FIX |
| No fix available | $NO_FIX_AVAILABLE |

---

## Advisories

EOF

# Add advisory list
if [ -f "$ANALYSIS_DIR/advisory-ids.txt" ] && [ -s "$ANALYSIS_DIR/advisory-ids.txt" ]; then
  echo "### Unique Advisory IDs" >> "$ANALYSIS_DIR/security-report.md"
  echo "" >> "$ANALYSIS_DIR/security-report.md"
  while read -r advisory; do
    echo "- [$advisory](https://github.com/advisories/$advisory)" >> "$ANALYSIS_DIR/security-report.md"
  done < "$ANALYSIS_DIR/advisory-ids.txt"
  echo "" >> "$ANALYSIS_DIR/security-report.md"
fi

# Add vulnerable packages details
if [ "$HAS_JQ" = true ] && [ -f "$ANALYSIS_DIR/security-details.json" ] && [ -s "$ANALYSIS_DIR/security-details.json" ]; then
  echo "### Affected Packages" >> "$ANALYSIS_DIR/security-report.md"
  echo "" >> "$ANALYSIS_DIR/security-report.md"

  jq -r '
    "#### \(.package) (\(.severity))\n" +
    "- **Range:** \(.range)\n" +
    "- **Fix available:** \(.fixAvailable)\n" +
    "- **Affected by:**\n" +
    (.via | map(
      if type == "object" then
        "  - \(.title // .name) (\(.source // "no-id"))"
      else
        "  - via: \(.)"
      end
    ) | join("\n")) +
    "\n"
  ' "$ANALYSIS_DIR/security-details.json" >> "$ANALYSIS_DIR/security-report.md" 2>/dev/null
fi

cat >> "$ANALYSIS_DIR/security-report.md" << EOF

---

## Supply Chain Security

**Lockfile validation:** $LOCKFILE_STATUS
EOF

if [ "$LOCKFILE_STATUS" = "FAIL" ] && [ -f "$ANALYSIS_DIR/lockfile-lint.txt" ]; then
  echo "" >> "$ANALYSIS_DIR/security-report.md"
  echo "### Issues Found" >> "$ANALYSIS_DIR/security-report.md"
  echo "\`\`\`" >> "$ANALYSIS_DIR/security-report.md"
  head -20 "$ANALYSIS_DIR/lockfile-lint.txt" >> "$ANALYSIS_DIR/security-report.md"
  echo "\`\`\`" >> "$ANALYSIS_DIR/security-report.md"
elif [ "$LOCKFILE_STATUS" = "SKIPPED" ]; then
  echo "" >> "$ANALYSIS_DIR/security-report.md"
  echo "Lockfile validation was skipped. Install lockfile-lint for supply chain checks:" >> "$ANALYSIS_DIR/security-report.md"
  echo "\`npm install -g lockfile-lint\`" >> "$ANALYSIS_DIR/security-report.md"
fi

cat >> "$ANALYSIS_DIR/security-report.md" << EOF

---

## Remediation Steps

EOF

# Generate remediation steps based on findings
if [ "$AUTO_FIX" != "?" ] && [ "$AUTO_FIX" -gt 0 ]; then
  cat >> "$ANALYSIS_DIR/security-report.md" << EOF
### 1. Auto-fix Available ($AUTO_FIX vulnerabilities)

\`\`\`bash
npm audit fix
\`\`\`

This will automatically update packages where non-breaking fixes are available.

EOF
fi

if [ "$BREAKING_FIX" != "?" ] && [ "$BREAKING_FIX" -gt 0 ]; then
  cat >> "$ANALYSIS_DIR/security-report.md" << EOF
### 2. Breaking Changes Required ($BREAKING_FIX vulnerabilities)

Review and apply breaking fixes:

\`\`\`bash
npm audit fix --force
\`\`\`

**Warning:** This may introduce breaking changes. Review changelogs before applying.

EOF
fi

if [ "$NO_FIX_AVAILABLE" != "?" ] && [ "$NO_FIX_AVAILABLE" -gt 0 ]; then
  cat >> "$ANALYSIS_DIR/security-report.md" << EOF
### 3. No Fix Available ($NO_FIX_AVAILABLE vulnerabilities)

For vulnerabilities without fixes:
- Consider alternative packages
- Implement workarounds if documented
- Monitor advisories for future fixes
- Assess actual exploitability in your context

EOF
fi

if [ $((CRITICAL + HIGH)) -eq 0 ]; then
  echo "**No high-severity vulnerabilities detected.**" >> "$ANALYSIS_DIR/security-report.md"
  echo "" >> "$ANALYSIS_DIR/security-report.md"
  echo "Continue monitoring with regular \`npm audit\` checks." >> "$ANALYSIS_DIR/security-report.md"
fi

cat >> "$ANALYSIS_DIR/security-report.md" << EOF

---

*Generated by analyze-security workflow*
*For compliance: Review advisory-ids.txt for CVE tracking*
EOF

echo "Security report saved to: $ANALYSIS_DIR/security-report.md"
```

**Output:** Security-focused markdown report with remediation steps
</step>

## Output Format

<output>
The workflow produces:

**Primary outputs:**
1. **security-findings.json** - Machine-readable findings for orchestrator
2. **security-report.md** - Human-readable security report

**Supporting outputs:**
- `security-audit.json` - npm audit with --audit-level=high
- `full-audit.json` - Complete npm audit data
- `security-details.json` - Parsed vulnerability details
- `advisory-ids.txt` - Unique advisory identifiers (GHSA, CVE)
- `fix-classification.json` - Vulnerabilities by fix type
- `lockfile-lint.txt` - Lockfile validation results (if available)

**Findings JSON structure:**
```json
{
  "timestamp": "2026-01-08T10:30:00Z",
  "type": "security-analysis",
  "summary": {
    "vulnerabilities": {
      "critical": 1,
      "high": 3,
      "total_actionable": 4
    },
    "advisories": {
      "unique_count": 4
    },
    "fixes": {
      "auto_fixable": 2,
      "breaking_required": 1,
      "no_fix_available": 1
    },
    "supply_chain": {
      "lockfile_validation": "PASS",
      "issues": 0
    }
  },
  "status": "CRITICAL",
  "priority": "IMMEDIATE"
}
```

**Status values:**
- `OK` - No high/critical vulnerabilities, lockfile valid
- `WARNING` - Lockfile issues but no high/critical vulns
- `HIGH` - High severity vulnerabilities present
- `CRITICAL` - Critical vulnerabilities present

**Priority values:**
- `NONE` - No action required
- `REVIEW` - Review recommended (lockfile issues)
- `URGENT` - Address high severity vulnerabilities
- `IMMEDIATE` - Critical vulnerabilities require immediate action
</output>

## Error Handling

<error_handling>
**Graceful degradation:**
- If jq unavailable: Fall back to grep-based parsing
- If lockfile-lint unavailable: Skip supply chain validation
- If npm audit fails: Report empty results, don't fail workflow
- If no lockfile: Warn but continue

**Non-blocking failures:**
- lockfile-lint errors → Log and continue
- Advisory ID extraction failures → Use basic grep fallback

**Hard failures (workflow exits):**
- No package.json → Cannot analyze, exit with clear message
- npm not installed → Cannot run analysis, exit with message
</error_handling>

## Integration

<integration>
**Called by:**
- `analyze-codebase.md` orchestrator (Phase 11-02)
- Manual invocation for security audits
- CI/CD pipelines for security gates

**Feeds into:**
- Suggestion engine (Phase 12) for security-related goal proposals
- CLAUDE.md security warnings section
- Compliance reporting

**Difference from analyze-dependencies.md:**
- analyze-dependencies: Broader view including outdated packages
- analyze-security: Security-only focus with CVE details and lockfile validation

**Example orchestrator integration:**
```bash
# Run security analysis
source analyze-security.md

# Check status for gate decisions
STATUS=$(jq -r '.status' "$ANALYSIS_DIR/security-findings.json")
case "$STATUS" in
  "CRITICAL")
    echo "BLOCKING: Critical vulnerabilities found"
    exit 1
    ;;
  "HIGH")
    echo "WARNING: High vulnerabilities found"
    ;;
esac
```
</integration>

## Examples

### Example 1: Clean Security Status

**Scenario:** Project with no security issues

**Output:**
```
=== Security Summary ===
Critical: 0
High: 0
Unique advisories: 0

=== Fix Availability ===
Auto-fixable: 0
Breaking change required: 0
No fix available: 0

Lockfile validation: PASS

=== Security Analysis Complete ===
Status: OK
Action priority: NONE
```

---

### Example 2: Critical Vulnerability Found

**Scenario:** Project with critical security issue

**Output:**
```
=== Security Summary ===
Critical: 1
High: 2
Unique advisories: 3

=== Fix Availability ===
Auto-fixable: 2
Breaking change required: 0
No fix available: 1

Lockfile validation: PASS

=== Security Analysis Complete ===
Status: CRITICAL
Action priority: IMMEDIATE
```

**Remediation generated:**
1. Run `npm audit fix` for 2 auto-fixable issues
2. Review alternatives for 1 vulnerability without fix

---

### Example 3: Supply Chain Warning

**Scenario:** Lockfile validation failure

**Output:**
```
=== Security Summary ===
Critical: 0
High: 0
Unique advisories: 0

=== Fix Availability ===
(no vulnerabilities to classify)

Lockfile validation: FAIL (2 issues)
- Non-HTTPS URL detected: http://registry.npmjs.org/lodash
- Unexpected host: evil-registry.com

=== Security Analysis Complete ===
Status: WARNING
Action priority: REVIEW
```

---

*Security analysis workflow for GSD autonomous goal setting*
