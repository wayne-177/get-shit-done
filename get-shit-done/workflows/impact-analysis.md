# Workflow: Impact Analysis

<purpose>
Analyze git changes to determine which monorepo packages are affected.

This workflow:
- Gets changed files since branch diverged from main
- Maps files to their containing packages
- Computes transitive affected packages via dependency graph
- Outputs build order for affected packages only

Output: AFFECTED_PACKAGES variable (JSON) with direct/transitive distinction
</purpose>

<prerequisites>
<required>
- MONOREPO_INFO variable OR .planning/MONOREPO_INFO.md exists
- Git repository with remote origin configured
- Changes to analyze (uncommitted or committed since fork point)
</required>

<optional>
- origin/main branch (falls back to main, master)
</optional>

**If MONOREPO_INFO not available:**
Run discover-monorepo workflow first:
```
ERROR: No monorepo information available.
Run discovery first: /gsd:discover-monorepo
```
</prerequisites>

<process>

<step name="get_changed_files" number="1">
## Step 1: Get Changed Files

Get all files changed since the branch diverged from main.

**Use merge-base (handles rebases correctly):**

```bash
get_changed_files() {
  # Try fork-point first (handles rebased branches)
  local merge_base
  merge_base=$(git merge-base --fork-point origin/main HEAD 2>/dev/null)

  if [ -z "$merge_base" ]; then
    # Fallback to regular merge-base
    merge_base=$(git merge-base origin/main HEAD 2>/dev/null)
  fi

  if [ -z "$merge_base" ]; then
    # Try main without origin
    merge_base=$(git merge-base main HEAD 2>/dev/null)
  fi

  if [ -z "$merge_base" ]; then
    # Try master
    merge_base=$(git merge-base origin/master HEAD 2>/dev/null || git merge-base master HEAD 2>/dev/null)
  fi

  if [ -z "$merge_base" ]; then
    echo "ERROR: Could not determine merge base. Ensure origin/main exists." >&2
    return 1
  fi

  echo "MERGE_BASE=$merge_base"

  # Get changed files (committed changes)
  local committed_changes
  committed_changes=$(git diff --name-only "$merge_base" HEAD 2>/dev/null)

  # Get uncommitted changes (staged + unstaged)
  local uncommitted_changes
  uncommitted_changes=$(git diff --name-only HEAD 2>/dev/null)
  uncommitted_changes="$uncommitted_changes $(git diff --name-only --cached 2>/dev/null)"

  # Combine and deduplicate
  echo "$committed_changes $uncommitted_changes" | tr ' ' '\n' | sort -u | grep -v '^$'
}

CHANGED_FILES=$(get_changed_files)
echo "Changed files:"
echo "$CHANGED_FILES" | head -20
if [ $(echo "$CHANGED_FILES" | wc -l) -gt 20 ]; then
  echo "... and $(($(echo "$CHANGED_FILES" | wc -l) - 20)) more"
fi
```

**Result:**
- MERGE_BASE: commit hash of fork point
- CHANGED_FILES: newline-separated list of changed file paths
</step>

<step name="map_files_to_packages" number="2">
## Step 2: Map Files to Packages

For each changed file, find the containing workspace package.

**Algorithm:**
1. For each changed file path
2. Traverse up directory tree
3. Find nearest package.json
4. Extract package name
5. Build set of directly affected packages

```bash
map_file_to_package() {
  local file="$1"
  local dir
  dir=$(dirname "$file")

  # Traverse up to find package.json
  while [ "$dir" != "." ] && [ "$dir" != "/" ]; do
    if [ -f "$dir/package.json" ]; then
      # Extract package name
      local pkg_name
      pkg_name=$(node -e "console.log(require('./$dir/package.json').name || '')" 2>/dev/null)

      if [ -n "$pkg_name" ]; then
        echo "$pkg_name"
        return 0
      fi
    fi
    dir=$(dirname "$dir")
  done

  # Check root package.json
  if [ -f "package.json" ]; then
    # Root package.json - check if relevant fields changed
    if echo "$file" | grep -qE '^package\.json$'; then
      # Only count if dependencies/workspaces changed
      # Filter out root changes that don't affect builds
      echo "__ROOT__"
    fi
  fi

  return 0
}

get_direct_affected() {
  local changed_files="$1"
  local workspace_packages="$2"  # From MONOREPO_INFO

  local direct_affected=""

  while IFS= read -r file; do
    [ -z "$file" ] && continue

    local pkg
    pkg=$(map_file_to_package "$file")

    if [ -n "$pkg" ] && [ "$pkg" != "__ROOT__" ]; then
      # Verify it's a workspace package
      if echo "$workspace_packages" | grep -q "^$pkg$"; then
        direct_affected="$direct_affected $pkg"
      fi
    fi
  done <<< "$changed_files"

  # Deduplicate
  echo "$direct_affected" | tr ' ' '\n' | sort -u | grep -v '^$'
}

DIRECT_AFFECTED=$(get_direct_affected "$CHANGED_FILES" "$WORKSPACE_PACKAGES")
echo "Directly affected packages:"
echo "$DIRECT_AFFECTED"
```

**Root package.json handling:**

Changes to root package.json are filtered unless:
- `dependencies` or `devDependencies` changed (affects all packages)
- `workspaces` field changed (affects structure)

```bash
check_root_package_impact() {
  if echo "$CHANGED_FILES" | grep -q '^package\.json$'; then
    # Check if dependency fields changed
    local dep_changed
    dep_changed=$(git diff "$MERGE_BASE" HEAD -- package.json | grep -E '^\+.*"(dependencies|devDependencies|workspaces)"')

    if [ -n "$dep_changed" ]; then
      echo "WARNING: Root package.json dependencies changed - may affect all packages"
      return 0  # Flag for full rebuild
    fi
  fi
  return 1
}
```

**Result:** DIRECT_AFFECTED - list of package names with file changes
</step>

<step name="compute_transitive_affected" number="3">
## Step 3: Compute Transitive Affected Packages

Find all packages that depend on directly affected packages.

**Algorithm (transitive closure):**
1. Start with DIRECT_AFFECTED set
2. For each package in set:
   - Find all packages that depend on it (dependantsOf)
   - Add to affected set
3. Repeat until no new packages found
4. Result is transitive closure

```bash
get_transitive_affected() {
  local direct_affected="$1"
  local dependency_graph="$2"  # JSON from MONOREPO_INFO

  # Using Claude reasoning to compute transitive closure:
  #
  # Given:
  #   - direct_affected: ["@scope/shared", "@scope/pkg-a"]
  #   - graph: {
  #       "@scope/shared": [],
  #       "@scope/pkg-a": ["@scope/shared"],
  #       "@scope/pkg-b": ["@scope/shared", "@scope/pkg-a"],
  #       "@scope/app": ["@scope/pkg-b"]
  #     }
  #
  # Algorithm:
  #   1. Start: affected = direct_affected
  #   2. For each pkg in affected:
  #      - Find dependants (packages where pkg appears in their deps)
  #      - @scope/shared is in pkg-a, pkg-b deps → add pkg-a, pkg-b
  #      - @scope/pkg-a is in pkg-b deps → add pkg-b
  #      - pkg-b is in app deps → add app
  #   3. Continue until no new packages added
  #   4. Result: affected = [shared, pkg-a, pkg-b, app]

  # Conceptual implementation:
  local affected="$direct_affected"
  local prev_count=0
  local curr_count=$(echo "$affected" | wc -w)

  while [ "$curr_count" -gt "$prev_count" ]; do
    prev_count=$curr_count

    # For each affected package, find its dependants
    for pkg in $affected; do
      # Find packages where this pkg appears in dependencies
      # (Using Claude reasoning on the graph)
      local dependants
      dependants=$(find_dependants_of "$pkg" "$dependency_graph")
      affected="$affected $dependants"
    done

    # Deduplicate
    affected=$(echo "$affected" | tr ' ' '\n' | sort -u | grep -v '^$' | tr '\n' ' ')
    curr_count=$(echo "$affected" | wc -w)
  done

  # Return only transitive (exclude direct)
  local transitive_only=""
  for pkg in $affected; do
    if ! echo "$direct_affected" | grep -q "^$pkg$"; then
      transitive_only="$transitive_only $pkg"
    fi
  done

  echo "$transitive_only" | tr ' ' '\n' | sort -u | grep -v '^$'
}

# Helper: find packages that depend on given package
find_dependants_of() {
  local target="$1"
  local graph="$2"  # JSON

  # Parse graph and find entries where target appears in dependencies
  # Conceptually: for each pkg in graph, if target in graph[pkg], emit pkg

  node -e "
    const graph = $graph;
    const target = '$target';
    Object.entries(graph).forEach(([pkg, deps]) => {
      if (deps.includes(target)) {
        console.log(pkg);
      }
    });
  " 2>/dev/null
}

TRANSITIVE_AFFECTED=$(get_transitive_affected "$DIRECT_AFFECTED" "$DEPENDENCY_GRAPH")
echo "Transitively affected packages:"
echo "$TRANSITIVE_AFFECTED"
```

**Result:** TRANSITIVE_AFFECTED - packages affected via dependencies (not directly)
</step>

<step name="compute_build_order" number="4">
## Step 4: Compute Build Order for Affected Packages

Get topological sort of affected packages only.

**Algorithm:**
1. Combine direct + transitive affected
2. Filter build order to only include affected packages
3. Preserve topological ordering

```bash
compute_affected_build_order() {
  local direct_affected="$1"
  local transitive_affected="$2"
  local full_build_order="$3"  # From MONOREPO_INFO

  # Combine all affected
  local all_affected="$direct_affected $transitive_affected"
  all_affected=$(echo "$all_affected" | tr ' ' '\n' | sort -u | grep -v '^$')

  # Filter full build order to only affected packages
  local affected_order=""
  for pkg in $full_build_order; do
    if echo "$all_affected" | grep -q "^$pkg$"; then
      affected_order="$affected_order $pkg"
    fi
  done

  echo "$affected_order" | tr ' ' '\n' | grep -v '^$'
}

AFFECTED_BUILD_ORDER=$(compute_affected_build_order "$DIRECT_AFFECTED" "$TRANSITIVE_AFFECTED" "$BUILD_ORDER")
echo "Affected build order:"
local num=1
for pkg in $AFFECTED_BUILD_ORDER; do
  echo "  $num. $pkg"
  num=$((num + 1))
done
```

**Result:** AFFECTED_BUILD_ORDER - topologically sorted list of affected packages
</step>

<step name="generate_output" number="5">
## Step 5: Generate Output

Create JSON output with complete impact analysis.

**JSON Output (AFFECTED_PACKAGES variable):**

```json
{
  "merge_base": "abc123def456",
  "analysis_date": "2026-01-08T12:00:00Z",
  "changed_files_count": 5,
  "changed_files": [
    "packages/shared/src/index.ts",
    "packages/shared/src/utils.ts",
    "packages/pkg-a/package.json"
  ],
  "affected": {
    "direct": ["@scope/shared", "@scope/pkg-a"],
    "transitive": ["@scope/pkg-b", "@scope/app"],
    "total": 4
  },
  "build_order": ["@scope/shared", "@scope/pkg-a", "@scope/pkg-b", "@scope/app"],
  "root_package_impact": false
}
```

```bash
generate_impact_output() {
  local merge_base="$1"
  local changed_files="$2"
  local direct_affected="$3"
  local transitive_affected="$4"
  local build_order="$5"

  local timestamp
  timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  local changed_count
  changed_count=$(echo "$changed_files" | grep -c . || echo "0")

  local direct_count
  direct_count=$(echo "$direct_affected" | grep -c . || echo "0")

  local transitive_count
  transitive_count=$(echo "$transitive_affected" | grep -c . || echo "0")

  local total_count
  total_count=$((direct_count + transitive_count))

  # Generate JSON
  cat << EOF
{
  "merge_base": "$merge_base",
  "analysis_date": "$timestamp",
  "changed_files_count": $changed_count,
  "changed_files": $(echo "$changed_files" | head -20 | jq -R . | jq -s .),
  "affected": {
    "direct": $(echo "$direct_affected" | jq -R . | jq -s .),
    "transitive": $(echo "$transitive_affected" | jq -R . | jq -s .),
    "total": $total_count
  },
  "build_order": $(echo "$build_order" | jq -R . | jq -s .)
}
EOF
}

AFFECTED_PACKAGES=$(generate_impact_output "$MERGE_BASE" "$CHANGED_FILES" "$DIRECT_AFFECTED" "$TRANSITIVE_AFFECTED" "$AFFECTED_BUILD_ORDER")
echo "$AFFECTED_PACKAGES"
```

**Display Summary:**

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "IMPACT ANALYSIS COMPLETE"
echo "═══════════════════════════════════════════════════"
echo ""
echo "Merge Base: $MERGE_BASE"
echo "Changed Files: $CHANGED_FILES_COUNT"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "AFFECTED PACKAGES"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Direct ($DIRECT_COUNT):"
for pkg in $DIRECT_AFFECTED; do
  echo "  - $pkg"
done
echo ""
echo "Transitive ($TRANSITIVE_COUNT):"
for pkg in $TRANSITIVE_AFFECTED; do
  echo "  - $pkg (depends on changed package)"
done
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "BUILD ORDER (affected only)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
local num=1
for pkg in $AFFECTED_BUILD_ORDER; do
  local marker=""
  if echo "$DIRECT_AFFECTED" | grep -q "^$pkg$"; then
    marker=" [direct]"
  else
    marker=" [transitive]"
  fi
  echo "  $num. $pkg$marker"
  num=$((num + 1))
done
echo ""
echo "═══════════════════════════════════════════════════"
```

**Result:** AFFECTED_PACKAGES JSON variable set
</step>

</process>

<output>
## Output Format

**Variables set:**
- `AFFECTED_PACKAGES`: JSON string with complete impact analysis

**JSON structure:**
```typescript
interface AffectedPackages {
  merge_base: string;           // Git commit hash
  analysis_date: string;        // ISO timestamp
  changed_files_count: number;
  changed_files: string[];      // First 20 files (truncated for large diffs)
  affected: {
    direct: string[];           // Packages with file changes
    transitive: string[];       // Packages depending on direct
    total: number;
  };
  build_order: string[];        // Topological sort of affected only
}
```

**Integration with MONOREPO_INFO.md:**

The impact analysis results can be appended to .planning/MONOREPO_INFO.md in an "Impact Analysis" section (see monorepo-info.md template).
</output>

<error_handling>
## Error Handling

**No merge base found:**
- Check for origin/main, main, origin/master, master
- If none found, suggest: "Ensure remote origin is configured"

**No changes detected:**
```
No changes detected since $MERGE_BASE.
Either:
- Working directory is clean
- All changes already merged to main
```

**MONOREPO_INFO missing:**
- Display clear error with guidance to run discover-monorepo first

**File outside workspace:**
- Skip files that don't map to any workspace package
- Log: "Skipping non-workspace file: $file"

**Empty affected set:**
- This is valid (e.g., only docs changed)
- Output empty arrays, don't error
</error_handling>

<integration>
## Integration Points

**Used by:**
- `/gsd:discover-monorepo --impact` command
- Coordinated execution (Phase 19)

**Requires:**
- discover-monorepo.md workflow output (MONOREPO_INFO)
- OR existing .planning/MONOREPO_INFO.md file

**Outputs to:**
- AFFECTED_PACKAGES variable
- Can update Impact Analysis section of MONOREPO_INFO.md
</integration>

---

*Impact analysis workflow for GSD monorepo support*
*Created: 2026-01-08*
