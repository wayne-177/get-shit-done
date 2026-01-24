# CLAUDE.md Template

Template for `CLAUDE.md` - project context file with auto-updated suggestions section.

---

## File Template

```markdown
# Project: [project-name]

## Quick Context
[Brief project description - user fills this in]
[Key technologies, architecture, or domain notes]

## Suggested Goals

Last analyzed: [timestamp] [staleness-indicator]

**Quick Wins ([N] suggestions):**
1. [Title] - `[command]` (score: N)
2. [Title] - `[command]` (score: N)

**Requires Planning ([N] suggestions):**
- [Title] (score: N, significant effort)
- [Title] (score: N, moderate effort)

Run `/gsd:suggest` to refresh or `/gsd:analyze-codebase` for new analysis.

## Development Notes
[User's custom notes - preserved across updates]
[Code conventions, testing requirements, deployment notes]
```

---

## Section Descriptions

### Project Title
```markdown
# Project: [project-name]
```
Uses the directory name by default. User can customize.

### Quick Context
```markdown
## Quick Context
[User-provided project description]
```
- Filled in by user or during `/gsd:new-project`
- Never auto-updated by workflows
- Contains project-specific context for Claude

### Suggested Goals (Auto-Updated)
```markdown
## Suggested Goals

Last analyzed: 2026-01-08T10:30:00Z

**Quick Wins (3 suggestions):**
1. Fix critical security vulnerabilities - `npm audit fix` (score: 130)
2. Update outdated dependencies - `npm update` (score: 90)
3. Remove unused exports - review and clean (score: 65)

**Requires Planning (2 suggestions):**
- Refactor authentication module (score: 60, significant effort)
- Migrate to TypeScript (score: 45, significant effort)

Run `/gsd:suggest` to refresh or `/gsd:analyze-codebase` for new analysis.
```

**This section is auto-managed by `update-claude-md.md` workflow.**
- Created from `.planning/GOAL_PROPOSALS.md`
- Updated on `/gsd:suggest` or `/gsd:analyze-codebase`
- All other sections are preserved during updates

### Development Notes
```markdown
## Development Notes
[User's custom content]
```
- Optional user section for project-specific notes
- Never touched by auto-update workflows
- Can contain anything relevant to the project

---

## Staleness Indicators

The "Last analyzed" line shows when suggestions were generated and indicates freshness:

### Fresh (within 24 hours)
```markdown
Last analyzed: 2026-01-08T10:30:00Z
```
No indicator needed - suggestions are current.

### Recent (1-7 days old)
```markdown
Last analyzed: 2026-01-05T10:30:00Z (3 days ago)
```
Shows age in parentheses. Still relatively current.

### Stale (7-14 days old)
```markdown
Last analyzed: 2026-01-01T10:30:00Z - consider refreshing
```
Suggests refreshing but not urgent.

### Very Stale (>14 days old)
```markdown
Last analyzed: 2025-12-20T10:30:00Z - run `/gsd:suggest` to refresh
```
Explicit call to action to regenerate suggestions.

### Analysis Updated (Mismatch Detected)
```markdown
Last analyzed: 2026-01-08T10:30:00Z - Analysis updated - refresh suggested
```
Shown when ANALYSIS_REPORT.md has been regenerated but GOAL_PROPOSALS.md hasn't been updated to match.

---

## Integration

### When Created

CLAUDE.md can be created by:

1. **`/gsd:new-project`** (optional)
   - Creates initial file with Quick Context from goal analysis
   - Suggestions section added if analysis is run

2. **`/gsd:analyze-codebase`** (if auto_suggest enabled)
   - Creates file if missing
   - Populates Suggested Goals section

3. **`/gsd:suggest`** (explicit command)
   - Creates file if missing
   - Updates Suggested Goals section

4. **Manual creation**
   - User creates file manually
   - Suggested Goals section added on first suggestion generation

### When Updated

The Suggested Goals section is updated by the `update-claude-md.md` workflow when:

1. `/gsd:suggest` command completes
2. `/gsd:analyze-codebase` completes (if auto_suggest enabled)
3. `/gsd:complete-milestone` can optionally refresh

**Update behavior:**
- Only the `## Suggested Goals` section is modified
- All content before and after is preserved exactly
- If section doesn't exist, it's appended to end of file

### Section Replacement Logic

The workflow uses this logic to update the file:

1. Find `## Suggested Goals` header
2. Capture everything until next `## ` header (or EOF)
3. Replace captured content with new suggestions
4. Preserve all other content unchanged

```bash
# Conceptual replacement logic
awk '
  /^## Suggested Goals/ { in_section=1; print new_section; next }
  /^## / && in_section { in_section=0 }
  !in_section { print }
'
```

---

## Examples

### Example 1: Active Development Project

```markdown
# Project: my-api-service

## Quick Context
REST API service for user management built with Express.js and PostgreSQL.
Uses JWT for authentication, Redis for caching.

## Suggested Goals

Last analyzed: 2026-01-08T10:30:00Z

**Quick Wins (3 suggestions):**
1. Fix critical security vulnerabilities - `npm audit fix` (score: 130)
2. Update express to 4.19.0 - `npm update express` (score: 90)
3. Add missing input validation - review endpoints (score: 75)

**Requires Planning (2 suggestions):**
- Migrate from callbacks to async/await (score: 60, significant effort)
- Add comprehensive API documentation (score: 45, moderate effort)

Run `/gsd:suggest` to refresh or `/gsd:analyze-codebase` for new analysis.

## Development Notes
- Tests: `npm test` (requires local PostgreSQL)
- Local dev: `npm run dev` starts on port 3000
- Deployment: GitHub Actions → AWS ECS
```

### Example 2: Healthy Codebase

```markdown
# Project: well-maintained-app

## Quick Context
Production-ready React application with CI/CD pipeline.
Regular maintenance keeps dependencies current.

## Suggested Goals

Last analyzed: 2026-01-08T14:00:00Z

No suggestions - codebase is healthy.

Run `/gsd:analyze-codebase` after major changes to check for new recommendations.

## Development Notes
- All dependencies reviewed monthly
- Security scanning in CI pipeline
- TypeScript strict mode enabled
```

### Example 3: Stale Suggestions

```markdown
# Project: legacy-migration

## Quick Context
Legacy PHP application being migrated to Node.js.
Multi-year effort with phased approach.

## Suggested Goals

Last analyzed: 2025-12-15T09:00:00Z - run `/gsd:suggest` to refresh

**Quick Wins (1 suggestion):**
1. Update Node.js from 14 to 18 LTS - migration guide available (score: 85)

**Requires Planning (3 suggestions):**
- Complete authentication module migration (score: 70, significant effort)
- Migrate database layer to Prisma (score: 65, significant effort)
- Add TypeScript to new modules (score: 50, moderate effort)

Run `/gsd:suggest` to refresh or `/gsd:analyze-codebase` for new analysis.

## Development Notes
- PHP code in /legacy, Node.js in /src
- Both apps run in parallel during migration
- Feature flags control traffic routing
```

---

## Guidelines

<guidelines>

### File Location

Always at project root: `CLAUDE.md`

This is the standard location Claude Code reads for project context at session start.

### Section Ownership

| Section | Owner | Updated By |
|---------|-------|------------|
| `# Project:` | User / new-project | Manual or init |
| `## Quick Context` | User | Manual only |
| `## Suggested Goals` | Workflow | `update-claude-md.md` |
| `## Development Notes` | User | Manual only |
| Other sections | User | Manual only |

**Only `## Suggested Goals` is auto-updated. Everything else is user-controlled.**

### Adding Custom Sections

Users can add any sections they want:

```markdown
## Architecture
[Custom architecture notes]

## API Endpoints
[Custom API documentation]

## Known Issues
[Current issues and workarounds]
```

Custom sections are preserved during Suggested Goals updates.

### Best Practices

1. **Keep Quick Context brief** - Claude reads this every session
2. **Don't duplicate GOAL_PROPOSALS.md** - Suggested Goals is a summary
3. **Use Development Notes for actionable info** - Commands, conventions, gotchas
4. **Let staleness guide refreshes** - Don't manually refresh if not stale
5. **Trust the scoring** - Higher scores = higher priority

### Suggested Goals Format

The compact format prioritizes scannability:

**Quick Wins format:**
```
N. [Title] - `[command]` (score: N)
```
- Numbered for priority
- Command inline if available
- Score indicates relative importance

**Requires Planning format:**
```
- [Title] (score: N, [effort] effort)
```
- Bullet points (not numbered)
- Effort level shown
- No inline commands (need planning first)

### When to Refresh

Refresh suggestions (`/gsd:suggest`) when:
- Staleness indicator shows "consider refreshing" or stronger
- After major dependency updates
- After fixing security vulnerabilities
- Before starting new development phase

### Relationship to GOAL_PROPOSALS.md

| File | Purpose | Detail Level |
|------|---------|--------------|
| GOAL_PROPOSALS.md | Full analysis with evidence | High - all details |
| CLAUDE.md | Session-start awareness | Low - summary only |

CLAUDE.md surfaces top suggestions so Claude is aware of them.
For full details, read `.planning/GOAL_PROPOSALS.md`.

</guidelines>

---

## Related Files

- `@get-shit-done/workflows/update-claude-md.md` - Workflow that updates this file
- `@get-shit-done/templates/goal-proposals.md` - Source data format
- `@get-shit-done/workflows/generate-suggestions.md` - Produces GOAL_PROPOSALS.md
- `@commands/gsd/suggest.md` - User command to refresh suggestions

---

*Template for CLAUDE.md with auto-updating suggestions section*
