<purpose>
Parse GOAL_PROPOSALS.md and update CLAUDE.md with a dynamic suggestions section.

Surfaces prioritized goal proposals at session start so Claude is aware of suggested improvements without user intervention. Includes staleness detection to indicate when refresh is recommended.
</purpose>

<prerequisites>
Before updating CLAUDE.md, check for proposals:

```bash
# Check if GOAL_PROPOSALS.md exists
if [ ! -f .planning/GOAL_PROPOSALS.md ]; then
  echo "No GOAL_PROPOSALS.md found - skipping CLAUDE.md update"
  echo ""
  echo "Run /gsd:suggest or /gsd:analyze-codebase to generate suggestions."
  exit 0
fi

echo "GOAL_PROPOSALS.md found - proceeding with CLAUDE.md update"
```

**If GOAL_PROPOSALS.md missing:** Skip silently (workflow is optional, no error).
**If CLAUDE.md missing:** Create new file with suggestions section only.
**If CLAUDE.md exists:** Preserve all content, update only "## Suggested Goals" section.
</prerequisites>

<process>

<step name="parse_goal_proposals">
**Extract key fields from GOAL_PROPOSALS.md**

Parse the Executive Summary and prioritized suggestions:

```bash
PROPOSALS_FILE=".planning/GOAL_PROPOSALS.md"

# Extract total suggestions count
TOTAL_SUGGESTIONS=$(grep '^\*\*Total Suggestions:\*\*' "$PROPOSALS_FILE" | grep -oE '[0-9]+' | head -1)
if [ -z "$TOTAL_SUGGESTIONS" ]; then
  TOTAL_SUGGESTIONS=0
fi

# Extract quick wins count
QUICK_WINS=$(grep '^\*\*Estimated Quick Wins:\*\*' "$PROPOSALS_FILE" | grep -oE '[0-9]+' | head -1)
if [ -z "$QUICK_WINS" ]; then
  QUICK_WINS=0
fi

# Extract priority breakdown
PRIORITY_BREAKDOWN=$(grep '^\*\*By Priority:\*\*' "$PROPOSALS_FILE" | sed 's/\*\*By Priority:\*\* *//')

# Extract Generated timestamp
PROPOSALS_TIMESTAMP=$(grep '^\*\*Generated:\*\*' "$PROPOSALS_FILE" | sed 's/\*\*Generated:\*\* *//')

# Extract Analysis Report timestamp (for staleness comparison)
ANALYSIS_TIMESTAMP_IN_PROPOSALS=$(grep '^\*\*Analysis Report:\*\*' "$PROPOSALS_FILE" | sed 's/\*\*Analysis Report:\*\* *//')

echo "Total suggestions: $TOTAL_SUGGESTIONS"
echo "Quick wins: $QUICK_WINS"
echo "Priority breakdown: $PRIORITY_BREAKDOWN"
echo "Generated: $PROPOSALS_TIMESTAMP"
```

**Extract top suggestions (max 5):**

```bash
# Extract suggestion titles with scores (numbered headers)
# Format: ### N. [Title]
# Then parse **Score:** and **Effort:** from each section

SUGGESTIONS=()
SUGGESTION_SCORES=()
SUGGESTION_EFFORTS=()
SUGGESTION_COMMANDS=()

# Get all suggestion headers
HEADERS=$(grep -E '^### [0-9]+\. ' "$PROPOSALS_FILE" | head -5)

while IFS= read -r header; do
  if [ -z "$header" ]; then continue; fi

  # Extract rank and title
  RANK=$(echo "$header" | grep -oE '^### [0-9]+' | grep -oE '[0-9]+')
  TITLE=$(echo "$header" | sed 's/^### [0-9]*\. //')

  # Extract the section for this suggestion (from this header to next ### or end)
  SECTION=$(sed -n "/^### $RANK\. /,/^### [0-9]*\. /p" "$PROPOSALS_FILE" | head -n -1)
  if [ -z "$SECTION" ]; then
    # Last suggestion - get to end of file
    SECTION=$(sed -n "/^### $RANK\. /,/^## /p" "$PROPOSALS_FILE" | head -n -1)
  fi

  # Extract score
  SCORE=$(echo "$SECTION" | grep '^\*\*Score:\*\*' | grep -oE '[0-9]+' | head -1)

  # Extract effort
  EFFORT=$(echo "$SECTION" | grep '^\*\*Effort:\*\*' | sed 's/\*\*Effort:\*\* *//')

  # Extract command (from code block after Suggested Action)
  COMMAND=$(echo "$SECTION" | sed -n '/\*\*Suggested Action:\*\*/,/```$/p' | grep -v 'Suggested Action' | grep -v '```' | head -1 | xargs)

  SUGGESTIONS+=("$TITLE")
  SUGGESTION_SCORES+=("$SCORE")
  SUGGESTION_EFFORTS+=("$EFFORT")
  SUGGESTION_COMMANDS+=("$COMMAND")

done <<< "$HEADERS"

echo "Parsed ${#SUGGESTIONS[@]} top suggestions"
```

**Categorize by effort:**

```bash
# Separate quick wins from requires-planning
QUICK_WIN_LIST=()
REQUIRES_PLANNING_LIST=()

for i in "${!SUGGESTIONS[@]}"; do
  EFFORT="${SUGGESTION_EFFORTS[$i]}"

  if [ "$EFFORT" = "quick-fix" ]; then
    QUICK_WIN_LIST+=("$i")
  else
    REQUIRES_PLANNING_LIST+=("$i")
  fi
done

echo "Quick wins: ${#QUICK_WIN_LIST[@]}"
echo "Requires planning: ${#REQUIRES_PLANNING_LIST[@]}"
```
</step>

<step name="detect_staleness">
**Compare timestamps to detect stale proposals**

```bash
STALENESS_INDICATOR=""
STALENESS_MESSAGE=""

# Check 1: Compare proposals timestamp with current ANALYSIS_REPORT.md
if [ -f .planning/ANALYSIS_REPORT.md ]; then
  CURRENT_ANALYSIS_TS=$(grep '^\*\*Generated:\*\*' .planning/ANALYSIS_REPORT.md | sed 's/\*\*Generated:\*\* *//')

  if [ -n "$ANALYSIS_TIMESTAMP_IN_PROPOSALS" ] && [ -n "$CURRENT_ANALYSIS_TS" ]; then
    if [ "$ANALYSIS_TIMESTAMP_IN_PROPOSALS" != "$CURRENT_ANALYSIS_TS" ]; then
      STALENESS_INDICATOR="stale"
      STALENESS_MESSAGE="Analysis updated - refresh suggested"
    fi
  fi
fi

# Check 2: File age check
PROPOSALS_AGE_DAYS=""
if [ "$(uname)" = "Darwin" ]; then
  # macOS
  PROPOSALS_MTIME=$(stat -f %m "$PROPOSALS_FILE" 2>/dev/null)
  CURRENT_TIME=$(date +%s)
  if [ -n "$PROPOSALS_MTIME" ]; then
    PROPOSALS_AGE_DAYS=$(( (CURRENT_TIME - PROPOSALS_MTIME) / 86400 ))
  fi
else
  # Linux
  PROPOSALS_AGE_CHECK=$(find "$PROPOSALS_FILE" -mtime +7 2>/dev/null)
  if [ -n "$PROPOSALS_AGE_CHECK" ]; then
    PROPOSALS_AGE_DAYS=8  # At least 7+ days
  fi
fi

# Set staleness based on age if not already stale from timestamp mismatch
if [ -z "$STALENESS_INDICATOR" ] && [ -n "$PROPOSALS_AGE_DAYS" ]; then
  if [ "$PROPOSALS_AGE_DAYS" -gt 14 ]; then
    STALENESS_INDICATOR="very-stale"
    STALENESS_MESSAGE="run \`/gsd:suggest\` to refresh"
  elif [ "$PROPOSALS_AGE_DAYS" -gt 7 ]; then
    STALENESS_INDICATOR="stale"
    STALENESS_MESSAGE="consider refreshing"
  elif [ "$PROPOSALS_AGE_DAYS" -gt 1 ]; then
    STALENESS_MESSAGE="(${PROPOSALS_AGE_DAYS} days ago)"
  fi
fi

# Format the staleness display
STALENESS_DISPLAY=""
case "$STALENESS_INDICATOR" in
  "very-stale")
    STALENESS_DISPLAY=" - $STALENESS_MESSAGE"
    ;;
  "stale")
    STALENESS_DISPLAY=" - $STALENESS_MESSAGE"
    ;;
  *)
    if [ -n "$STALENESS_MESSAGE" ]; then
      STALENESS_DISPLAY=" $STALENESS_MESSAGE"
    fi
    ;;
esac

echo "Staleness: ${STALENESS_INDICATOR:-fresh}"
echo "Display: $STALENESS_DISPLAY"
```
</step>

<step name="format_suggestions_section">
**Generate compact suggestions section for CLAUDE.md**

```bash
format_suggestions_section() {
  local timestamp="$PROPOSALS_TIMESTAMP"
  local staleness="$STALENESS_DISPLAY"

  # Start section
  echo "## Suggested Goals"
  echo ""
  echo "Last analyzed: $timestamp$staleness"
  echo ""

  # Handle empty suggestions case
  if [ "$TOTAL_SUGGESTIONS" -eq 0 ]; then
    echo "No suggestions - codebase is healthy."
    echo ""
    echo "Run \`/gsd:analyze-codebase\` after major changes to check for new recommendations."
    return
  fi

  # Quick Wins section
  if [ ${#QUICK_WIN_LIST[@]} -gt 0 ]; then
    echo "**Quick Wins (${#QUICK_WIN_LIST[@]} suggestions):**"
    local num=1
    for idx in "${QUICK_WIN_LIST[@]}"; do
      local title="${SUGGESTIONS[$idx]}"
      local score="${SUGGESTION_SCORES[$idx]}"
      local cmd="${SUGGESTION_COMMANDS[$idx]}"

      if [ -n "$cmd" ]; then
        echo "$num. $title - \`$cmd\` (score: $score)"
      else
        echo "$num. $title (score: $score)"
      fi
      ((num++))
    done
    echo ""
  fi

  # Requires Planning section
  if [ ${#REQUIRES_PLANNING_LIST[@]} -gt 0 ]; then
    echo "**Requires Planning (${#REQUIRES_PLANNING_LIST[@]} suggestions):**"
    for idx in "${REQUIRES_PLANNING_LIST[@]}"; do
      local title="${SUGGESTIONS[$idx]}"
      local score="${SUGGESTION_SCORES[$idx]}"
      local effort="${SUGGESTION_EFFORTS[$idx]}"

      echo "- $title (score: $score, $effort effort)"
    done
    echo ""
  fi

  # Footer with refresh command
  echo "Run \`/gsd:suggest\` to refresh or \`/gsd:analyze-codebase\` for new analysis."
}

# Generate the section content
SUGGESTIONS_SECTION=$(format_suggestions_section)
echo "Generated suggestions section (${#SUGGESTIONS_SECTION} chars)"
```
</step>

<step name="update_claude_md">
**Update or create CLAUDE.md with suggestions section**

```bash
CLAUDE_MD="CLAUDE.md"

update_claude_md() {
  if [ ! -f "$CLAUDE_MD" ]; then
    # Create new CLAUDE.md with just suggestions section
    echo "# Project: $(basename "$(pwd)")"
    echo ""
    echo "$SUGGESTIONS_SECTION"
    return
  fi

  # CLAUDE.md exists - need to replace or append suggestions section

  # Check if ## Suggested Goals section exists
  if grep -q '^## Suggested Goals' "$CLAUDE_MD"; then
    # Replace existing section
    # Find start and end of section (from ## Suggested Goals to next ## or end)

    # Use awk to replace the section
    awk -v section="$SUGGESTIONS_SECTION" '
      BEGIN { in_section = 0; printed = 0 }
      /^## Suggested Goals/ {
        in_section = 1
        print section
        printed = 1
        next
      }
      /^## / && in_section {
        in_section = 0
        print ""
        print $0
        next
      }
      !in_section { print }
      END { if (!printed) print "\n" section }
    ' "$CLAUDE_MD"
  else
    # Append section to end of file
    cat "$CLAUDE_MD"
    echo ""
    echo "$SUGGESTIONS_SECTION"
  fi
}

# Generate updated content
UPDATED_CONTENT=$(update_claude_md)

# Write to file
echo "$UPDATED_CONTENT" > "$CLAUDE_MD"

echo "Updated $CLAUDE_MD with suggestions section"
```

**Alternative approach using temp file for complex replacements:**

```bash
update_claude_md_safe() {
  local temp_file=$(mktemp)
  local in_section=0
  local section_replaced=0

  if [ ! -f "$CLAUDE_MD" ]; then
    # Create new file
    {
      echo "# Project: $(basename "$(pwd)")"
      echo ""
      echo "$SUGGESTIONS_SECTION"
    } > "$CLAUDE_MD"
    echo "Created new $CLAUDE_MD"
    return
  fi

  # Process existing file line by line
  while IFS= read -r line || [ -n "$line" ]; do
    if echo "$line" | grep -q '^## Suggested Goals'; then
      # Start of suggestions section
      in_section=1
      echo "$SUGGESTIONS_SECTION" >> "$temp_file"
      section_replaced=1
      continue
    fi

    if [ $in_section -eq 1 ]; then
      # Check if we've reached next section
      if echo "$line" | grep -q '^## '; then
        in_section=0
        echo "" >> "$temp_file"
        echo "$line" >> "$temp_file"
      fi
      # Skip lines within old section
      continue
    fi

    echo "$line" >> "$temp_file"
  done < "$CLAUDE_MD"

  # If section wasn't replaced, append it
  if [ $section_replaced -eq 0 ]; then
    echo "" >> "$temp_file"
    echo "$SUGGESTIONS_SECTION" >> "$temp_file"
  fi

  # Replace original file
  mv "$temp_file" "$CLAUDE_MD"
  echo "Updated $CLAUDE_MD"
}

update_claude_md_safe
```
</step>

<step name="handle_edge_cases">
**Handle parsing failures and edge cases gracefully**

```bash
handle_edge_cases() {
  # Edge case 1: No GOAL_PROPOSALS.md
  # Already handled in prerequisites - exit 0 (skip silently)

  # Edge case 2: Empty suggestions (Total: 0)
  # Handled in format_suggestions_section - shows "codebase is healthy" message

  # Edge case 3: Parsing failures
  # Use fallback values if parsing fails
  if [ -z "$TOTAL_SUGGESTIONS" ]; then
    TOTAL_SUGGESTIONS=0
  fi

  if [ -z "$PROPOSALS_TIMESTAMP" ]; then
    PROPOSALS_TIMESTAMP="Unknown"
  fi

  # Edge case 4: CLAUDE.md write failure
  # Check if write succeeded
  if [ ! -f "$CLAUDE_MD" ]; then
    echo "WARNING: Failed to create/update CLAUDE.md"
    echo "Suggestions section not written, but workflow continues."
    exit 0  # Don't fail the workflow
  fi

  # Edge case 5: Corrupted GOAL_PROPOSALS.md
  # If we can't parse any suggestions but file exists, show generic message
  if [ ${#SUGGESTIONS[@]} -eq 0 ] && [ "$TOTAL_SUGGESTIONS" -gt 0 ]; then
    echo "WARNING: Could not parse individual suggestions from GOAL_PROPOSALS.md"
    echo "CLAUDE.md will show summary only."
    # Continue with summary-only section
  fi
}

handle_edge_cases
```

**Graceful degradation priorities:**
1. Never crash - always exit cleanly
2. Never corrupt existing CLAUDE.md content
3. Preserve other sections when updating
4. Show helpful messages when skipping
</step>

</process>

<output>
Updates `CLAUDE.md` in project root with:

**New file created (if CLAUDE.md doesn't exist):**
```markdown
# Project: [project-name]

## Suggested Goals

Last analyzed: [timestamp]

**Quick Wins ([N] suggestions):**
1. [Title] - `[command]` (score: N)

**Requires Planning ([N] suggestions):**
- [Title] (score: N, significant effort)

Run `/gsd:suggest` to refresh or `/gsd:analyze-codebase` for new analysis.
```

**Existing file updated (preserves all other content):**
- Only `## Suggested Goals` section is modified
- All other sections preserved exactly
- Section appended if not present

**Success message:**
```
Updated CLAUDE.md with suggestions section
- Total suggestions: N
- Quick wins: N
- Staleness: fresh | stale | very-stale
```

**Skip cases:**
- No GOAL_PROPOSALS.md: Silent skip with guidance message
- Empty suggestions: "No suggestions - codebase is healthy"
- Parse errors: Warning + continue with available data
</output>

<staleness_indicators>
**Fresh (within 24 hours):**
No indicator - just the timestamp

**Recent (1-7 days):**
`Last analyzed: 2026-01-08T10:30:00Z (3 days ago)`

**Stale (7-14 days):**
`Last analyzed: 2026-01-01T10:30:00Z - consider refreshing`

**Very stale (>14 days):**
`Last analyzed: 2025-12-20T10:30:00Z - run `/gsd:suggest` to refresh`

**Analysis updated:**
`Last analyzed: 2026-01-08T10:30:00Z - Analysis updated - refresh suggested`

Staleness is detected by:
1. Comparing GOAL_PROPOSALS.md "Analysis Report:" with ANALYSIS_REPORT.md "Generated:" timestamps
2. Checking file modification age
</staleness_indicators>

<integration>
This workflow is called by:
- `/gsd:suggest` command (after generating GOAL_PROPOSALS.md)
- `/gsd:analyze-codebase` (if auto_suggest enabled, after generating proposals)
- `/gsd:new-project` (optionally, to initialize CLAUDE.md)

This workflow depends on:
- `generate-suggestions.md` workflow (produces GOAL_PROPOSALS.md)
- `goal-proposals.md` template (defines parsing contract)
</integration>

<related_files>
- `@get-shit-done/templates/goal-proposals.md` - Input format contract
- `@get-shit-done/templates/claude-md.md` - Output format specification
- `@get-shit-done/workflows/generate-suggestions.md` - Produces input file
- `@commands/gsd/suggest.md` - User-facing command that calls this workflow
</related_files>
