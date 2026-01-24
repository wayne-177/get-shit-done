# Execution Dashboard Template

Template for `.planning/EXECUTION_DASHBOARD.md` - real-time monorepo execution status.

---

## File Template

```markdown
# Execution Dashboard

**Status:** {status}
**Started:** {started_timestamp}
**Completed:** {completed_timestamp}
**Packages:** {completed_count}/{total_count} complete

---

## Package Status

```mermaid
graph LR
  classDef pending fill:#9ca3af,stroke:#6b7280,color:#fff
  classDef running fill:#fbbf24,stroke:#d97706,color:#000
  classDef success fill:#22c55e,stroke:#16a34a,color:#fff
  classDef failed fill:#ef4444,stroke:#dc2626,color:#fff
  classDef skipped fill:#f97316,stroke:#ea580c,color:#fff

  {mermaid_nodes}

  {mermaid_edges}
```

---

## Execution Log

| # | Package | Status | Duration | Started | Completed | Details |
|---|---------|--------|----------|---------|-----------|---------|
{execution_log_rows}

---

## Summary

- **Pending:** {pending_count}
- **Running:** {running_count}
- **Success:** {success_count}
- **Failed:** {failed_count}
- **Skipped:** {skipped_count}

---

## Errors

{errors_section}

---

*Dashboard updated: {last_updated}*
*Run: /gsd:execute-monorepo --status to refresh*
```

---

## Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{status}` | Overall execution status | `In Progress`, `Complete`, `Failed` |
| `{started_timestamp}` | ISO 8601 execution start time | `2026-01-08T12:00:00Z` |
| `{completed_timestamp}` | ISO 8601 execution end time | `2026-01-08T12:15:00Z` or `-` |
| `{completed_count}` | Number of packages finished | `3` |
| `{total_count}` | Total packages to execute | `5` |
| `{mermaid_nodes}` | Package nodes with status classes | See below |
| `{mermaid_edges}` | Dependency edges between packages | See below |
| `{execution_log_rows}` | Table rows for each package | See below |
| `{pending_count}` | Packages not yet started | `2` |
| `{running_count}` | Packages currently executing | `1` |
| `{success_count}` | Packages completed successfully | `2` |
| `{failed_count}` | Packages that failed | `0` |
| `{skipped_count}` | Packages skipped due to deps | `0` |
| `{errors_section}` | Error details or "None" | See below |
| `{last_updated}` | ISO 8601 last update time | `2026-01-08T12:05:30Z` |

---

## Mermaid Status Classes

The Mermaid graph uses CSS classes to indicate package status:

| Class | Color | Description |
|-------|-------|-------------|
| `pending` | Gray (#9ca3af) | Not yet started, waiting |
| `running` | Yellow (#fbbf24) | Currently executing |
| `success` | Green (#22c55e) | All tasks completed successfully |
| `failed` | Red (#ef4444) | One or more tasks failed |
| `skipped` | Orange (#f97316) | Skipped due to dependency failure |

**Node format:**
```mermaid
pkg_id["@scope/package-name"]:::status_class
```

**Example:**
```mermaid
shared["@scope/shared"]:::success
utils["@scope/utils"]:::running
api["@scope/api"]:::pending
```

---

## Execution Log Row Format

Each row in the execution log follows this format:

```markdown
| 1 | @scope/shared | success | 45s | 12:00:00 | 12:00:45 | All tasks passed |
| 2 | @scope/utils | running | - | 12:00:46 | - | Executing build... |
| 3 | @scope/api | pending | - | - | - | Waiting |
```

**Status values:**
- `pending`: Waiting to start
- `running`: Currently executing
- `success`: Completed successfully
- `failed`: Execution failed
- `skipped`: Skipped (dependency failed)

**Duration format:**
- `{X}ms` for < 1 second
- `{X}.{Y}s` for < 60 seconds
- `{X}m {Y}s` for >= 60 seconds
- `{X}h {Y}m` for >= 60 minutes
- `-` for pending/running

---

## Errors Section

**When no errors:**
```markdown
None
```

**When errors exist:**
```markdown
### @scope/package-name

**Task:** build
**Exit Code:** 1
**Output:**
\`\`\`
Error: Cannot find module './missing'
    at Object.<anonymous> (/path/to/file.js:1:1)
\`\`\`

### @scope/another-package

**Task:** test
**Exit Code:** 2
**Output:**
\`\`\`
Test suite failed: 3 tests failed
\`\`\`
```

---

## Real-Time Update Instructions

The coordinated-execution.md workflow updates this dashboard during execution:

### 1. Initialize Dashboard

Create dashboard with all packages in `pending` status:

```bash
initialize_dashboard() {
  local packages="$1"  # Newline-separated package names
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Generate Mermaid nodes (all pending)
  local mermaid_nodes=""
  local num=1
  while IFS= read -r pkg; do
    [ -z "$pkg" ] && continue
    local safe_id=$(echo "$pkg" | tr '@/-' '_' | tr -cd '[:alnum:]_')
    mermaid_nodes="${mermaid_nodes}  ${safe_id}[\"$pkg\"]:::pending\n"
    num=$((num + 1))
  done <<< "$packages"

  # Write initial dashboard...
}
```

### 2. Update Package Status

When a package starts or completes:

```bash
update_package_status() {
  local pkg_name="$1"
  local new_status="$2"
  local duration="$3"
  local details="$4"

  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Update Mermaid node class
  local safe_id=$(echo "$pkg_name" | tr '@/-' '_' | tr -cd '[:alnum:]_')
  sed -i.bak "s/${safe_id}\[.*\]:::[a-z]*/${safe_id}[\"$pkg_name\"]:::$new_status/" .planning/EXECUTION_DASHBOARD.md

  # Update table row
  # Find row by package name, update status/duration/times

  # Update summary counts
  update_summary_counts

  # Update last_updated timestamp
  sed -i.bak "s/^\*Dashboard updated:.*/\*Dashboard updated: $timestamp\*/" .planning/EXECUTION_DASHBOARD.md

  rm -f .planning/EXECUTION_DASHBOARD.md.bak
}
```

### 3. Finalize Dashboard

When execution completes:

```bash
finalize_dashboard() {
  local overall_status="$1"  # success | failed | partial

  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Update header status
  local status_text="Complete"
  [ "$overall_status" = "failed" ] && status_text="Failed"
  [ "$overall_status" = "partial" ] && status_text="Partial"

  sed -i.bak "s/^\*\*Status:\*\*.*/\*\*Status:\*\* $status_text/" .planning/EXECUTION_DASHBOARD.md
  sed -i.bak "s/^\*\*Completed:\*\*.*/\*\*Completed:\*\* $timestamp/" .planning/EXECUTION_DASHBOARD.md

  # Set running count to 0
  sed -i.bak "s/^- \*\*Running:\*\*.*/- **Running:** 0/" .planning/EXECUTION_DASHBOARD.md

  rm -f .planning/EXECUTION_DASHBOARD.md.bak
}
```

---

## Example: Complete Dashboard

```markdown
# Execution Dashboard

**Status:** Complete
**Started:** 2026-01-08T12:00:00Z
**Completed:** 2026-01-08T12:02:34Z
**Packages:** 5/5 complete

---

## Package Status

\`\`\`mermaid
graph LR
  classDef pending fill:#9ca3af,stroke:#6b7280,color:#fff
  classDef running fill:#fbbf24,stroke:#d97706,color:#000
  classDef success fill:#22c55e,stroke:#16a34a,color:#fff
  classDef failed fill:#ef4444,stroke:#dc2626,color:#fff
  classDef skipped fill:#f97316,stroke:#ea580c,color:#fff

  shared["@scope/shared"]:::success
  utils["@scope/utils"]:::success
  ui["@scope/ui"]:::failed
  api["@scope/api"]:::success
  app["@scope/app"]:::skipped

  shared --> utils
  shared --> ui
  utils --> api
  ui --> app
  api --> app
\`\`\`

---

## Execution Log

| # | Package | Status | Duration | Started | Completed | Details |
|---|---------|--------|----------|---------|-----------|---------|
| 1 | @scope/shared | success | 12.3s | 12:00:00 | 12:00:12 | All tasks passed |
| 2 | @scope/utils | success | 8.7s | 12:00:13 | 12:00:22 | All tasks passed |
| 3 | @scope/ui | failed | 45.2s | 12:00:22 | 12:01:07 | Test failed |
| 4 | @scope/api | success | 23.1s | 12:01:08 | 12:01:31 | All tasks passed |
| 5 | @scope/app | skipped | - | - | - | Blocked by @scope/ui |

---

## Summary

- **Pending:** 0
- **Running:** 0
- **Success:** 3
- **Failed:** 1
- **Skipped:** 1

---

## Errors

### @scope/ui

**Task:** test
**Exit Code:** 1
**Output:**
\`\`\`
FAIL src/Button.test.tsx
  Button component
    ✕ should render correctly (15ms)

Expected: "Click me"
Received: "Click"
\`\`\`

---

*Dashboard updated: 2026-01-08T12:02:34Z*
*Run: /gsd:execute-monorepo --status to refresh*
```

---

## Integration with coordinated-execution.md

This template is used by `~/.claude/get-shit-done/workflows/coordinated-execution.md`:

1. **Step 2 (initialize_dashboard):** Creates dashboard with all packages pending
2. **Step 3 (execute_per_package):** Updates status as each package runs/completes
3. **Step 5 (generate_results):** Finalizes dashboard with overall status

The dashboard file is created at `.planning/EXECUTION_DASHBOARD.md` and updated in real-time during execution, allowing users to monitor progress.

---

*Execution dashboard template for GSD monorepo support*
*Created: 2026-01-08*
