# Phase 2: MCP Abstraction Layer - Research

**Researched:** 2026-01-17
**Domain:** Graphite MCP/CLI abstraction for GSD agent orchestration
**Confidence:** MEDIUM (MCP tool naming convention unverified, but patterns established)

## Summary

This research investigates how to create a unified abstraction layer for Graphite operations that can route to MCP tools when available, fall back to `gt` CLI when MCP is unavailable, and ultimately fall back to `git` when neither is available.

The key insight is that GSD is a **markdown-based prompt system**, not a code library. The abstraction layer will be implemented as **reference documentation and workflow patterns** that GSD agents read and follow, not as executable code. Agents will check backend availability using bash commands and MCP tool calls, then follow the appropriate operation pattern.

**Primary recommendation:** Create a `graphite-operations.md` reference document that defines: (1) detection protocol for backend availability, (2) operation mappings for each backend, and (3) fallback chain logic. Agents include this reference and follow its patterns.

## Standard Stack

This phase doesn't introduce new libraries - it creates markdown documentation patterns.

### Core Patterns
| Pattern | Purpose | Why Standard |
|---------|---------|--------------|
| Reference document | Defines operation abstractions | GSD pattern - agents `@include` references |
| Bash detection | Check CLI availability | Already used in new-project.md for Graphite detection |
| MCP tool calls | Use `mcp__graphite__*` when available | MCP naming convention for Claude Code |
| Config-based routing | Read from `.planning/config.json` | GSD pattern - config drives behavior |

### Supporting Tools
| Tool | Version | Purpose | When to Use |
|------|---------|---------|-------------|
| `gt` CLI | 1.6.7+ | Graphite branch/commit operations | When MCP unavailable |
| `git` CLI | 2.38+ | Basic version control fallback | When neither MCP nor gt available |
| Graphite MCP | embedded in gt 1.6.7+ | AI-native Graphite operations | When configured and connected |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Reference doc pattern | Inline instructions | Reference is reusable, inline is duplicative |
| Config-based detection | Environment detection only | Config respects user choice, env can be wrong |
| Lazy detection | Eager detection | Lazy avoids overhead when operations not needed |

## Architecture Patterns

### Recommended Document Structure
```
get-shit-done/
├── references/
│   ├── git-integration.md          # Existing - commit patterns
│   └── graphite-operations.md      # NEW - abstraction layer
├── workflows/
│   └── execute-plan.md             # Modified - include graphite-operations
└── templates/
    └── config.json                 # Already has versionControl section
```

### Pattern 1: Detection Protocol

**What:** Check backend availability at first operation, cache result for session
**When to use:** Before any Graphite/git operation
**Source:** User decisions from CONTEXT.md

```markdown
## Backend Detection Protocol

### Step 1: Check Config Authority
Read `.planning/config.json` versionControl.type:
- If "git" -> Use git-only mode (skip Graphite detection)
- If "graphite" -> Proceed to MCP detection

### Step 2: Check MCP Availability (if Graphite mode)
Attempt MCP tool call:
```
mcp__graphite__status
```
- If returns successfully -> MCP backend available
- If tool not found or error -> Fall back to CLI detection

### Step 3: Check CLI Availability (if MCP unavailable)
```bash
gt --version >/dev/null 2>&1 && echo "gt" || echo "none"
```
- If "gt" -> CLI backend available
- If "none" -> Fall back to git-only (warn user)

### Step 4: Set Session Backend
Store result for session (no re-detection):
- backend: "mcp" | "cli" | "git"
- If downgraded from graphite config to git: warn user
```

### Pattern 2: Operation Abstraction

**What:** Map logical operations to backend-specific commands
**When to use:** Any branch/commit operation
**Source:** Graphite documentation

```markdown
## Operation Mappings

### Create Branch
| Backend | Command |
|---------|---------|
| MCP | mcp__graphite__create_branch with name, message |
| CLI | gt create "{name}" -m "{message}" |
| Git | git checkout -b "{name}" && git commit -m "{message}" |

### Modify/Commit
| Backend | Command |
|---------|---------|
| MCP | mcp__graphite__modify with message, amend flag |
| CLI | gt modify -cam "{message}" OR gt modify -am "{message}" |
| Git | git add -A && git commit -m "{message}" OR git commit --amend -m "{message}" |

### Submit/Push
| Backend | Command |
|---------|---------|
| MCP | mcp__graphite__submit with flags |
| CLI | gt submit --stack |
| Git | git push -u origin HEAD |
```

### Pattern 3: Fallback Chain in Prompts

**What:** Agent prompt structure for graceful degradation
**When to use:** In agent workflow references

```markdown
## Fallback Execution Pattern

For each Graphite operation:

1. Read session backend (from detection)
2. Execute operation for that backend
3. If operation fails:
   - MCP error -> retry with CLI
   - CLI error -> retry with git (if safe)
   - All fail -> report error to user

Example in agent action:
"""
Create branch for this phase:

Backend: {session_backend}

If MCP:
  Call mcp__graphite__create_branch(name="{branch}", message="{message}")

If CLI:
  Run: gt create "{branch}" -m "{message}"

If Git:
  Run: git checkout -b "{branch}" && git add -A && git commit --allow-empty -m "{message}"
"""
```

### Anti-Patterns to Avoid

- **Checking backend every operation:** Lazy detection means once per session, not every call
- **Persisting backend choice:** Each session re-detects (config may change, MCP may become available)
- **Hardcoding MCP tool names:** Tool names should be in reference doc, not scattered in prompts
- **Notifying on every fallback:** Silent fallback unless error (per user decisions)
- **Lowest common denominator:** Expose Graphite features when available, not just git subset

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MCP tool listing | Custom detection | `/mcp` command or try-catch | MCP protocol handles discovery |
| CLI detection | Complex version parsing | Simple `command --version` check | Existence is enough |
| Config reading | Custom parser | `cat .planning/config.json` + jq | JSON is standard |
| Branch naming | Custom validation | `git check-ref-format` | Git validates branch names |

**Key insight:** The abstraction is documentation, not code. Agents read patterns and execute them. No need for a "library" - just well-documented patterns.

## Common Pitfalls

### Pitfall 1: MCP Tool Name Uncertainty
**What goes wrong:** Assuming MCP tool names without verification
**Why it happens:** Graphite MCP tool definitions not publicly documented in detail
**How to avoid:** Use try-catch pattern - try MCP, if tool not found fall back to CLI
**Warning signs:** "Unknown tool" errors in agent execution

### Pitfall 2: Over-Engineering Detection
**What goes wrong:** Complex state machines for backend selection
**Why it happens:** Trying to handle every edge case upfront
**How to avoid:** Simple three-step detection (config -> MCP -> CLI -> git)
**Warning signs:** Detection taking significant context, multiple re-detections

### Pitfall 3: Breaking Git Mode
**What goes wrong:** Graphite logic interferes with git-only users
**Why it happens:** Not respecting config.versionControl.type as authoritative
**How to avoid:** Config is gate - if "git", never try Graphite backends
**Warning signs:** Graphite errors in git-mode projects

### Pitfall 4: Verbose Fallback Notifications
**What goes wrong:** User interrupted by "falling back to CLI" messages
**Why it happens:** Over-communicating internal state
**How to avoid:** Silent fallback per user decision (CONTEXT.md)
**Warning signs:** Multiple notifications per operation

### Pitfall 5: Session State Leakage
**What goes wrong:** Backend choice persists across sessions incorrectly
**Why it happens:** Caching in wrong scope
**How to avoid:** Re-detect each session, store only in-memory
**Warning signs:** Stale backend after user changes MCP config

## Code Examples

### Detection Pattern (for reference doc)
```markdown
## Detecting Available Backend

### In Agent Prompts

Before your first Graphite operation, detect the backend:

1. **Read config:**
   ```bash
   VC_TYPE=$(cat .planning/config.json 2>/dev/null | grep -o '"type"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4)
   ```

2. **If git mode, use git:**
   If `VC_TYPE` is "git", use git commands only.

3. **If graphite mode, try MCP:**
   Attempt: `mcp__graphite__status` (or similar)
   - Success: Use MCP backend
   - Tool not found: Continue to CLI check

4. **Check CLI:**
   ```bash
   gt --version >/dev/null 2>&1 && echo "gt-available"
   ```
   - "gt-available": Use CLI backend
   - Otherwise: Warn and use git fallback
```

### Operation Pattern (for reference doc)
```markdown
## Creating a Branch

### MCP Backend
If MCP available, call:
```
mcp__graphite__create_branch
  name: "{username}/{feature-name}"
  commit_message: "{message}"
```

### CLI Backend
If CLI available, run:
```bash
gt create "{username}/{feature-name}" -m "{message}"
```

### Git Backend (fallback)
If neither available:
```bash
git checkout -b "{username}/{feature-name}"
git commit --allow-empty -m "{message}"
```

Note: Git fallback loses Graphite stacking benefits. Warn user once per session.
```

### Config Reading Pattern
```bash
# Read version control type from config
VC_TYPE=$(cat .planning/config.json 2>/dev/null | \
  grep -o '"type"[[:space:]]*:[[:space:]]*"[^"]*"' | \
  cut -d'"' -f4)

# Default to git if not set
VC_TYPE=${VC_TYPE:-git}

# Read username for branch naming
VC_USERNAME=$(cat .planning/config.json 2>/dev/null | \
  grep -o '"username"[[:space:]]*:[[:space:]]*"[^"]*"' | \
  cut -d'"' -f4)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| gt CLI only | gt CLI with MCP server | gt 1.6.7 (July 2025) | AI agents can use Graphite natively |
| Manual stacking | `gt submit --stack` | Established | Automated PR stacking |
| Manual branch navigation | `gt up/down/checkout` | Established | Stack-aware navigation |

**Deprecated/outdated:**
- None identified - Graphite MCP is recent (July 2025)

## MCP Tool Discovery

### Known Patterns
Based on MCP protocol and Claude Code conventions:

1. **Tool naming:** `mcp__[server]__[operation]` (e.g., `mcp__graphite__create_branch`)
2. **Server setup:** `claude mcp add graphite gt mcp` adds the Graphite MCP server
3. **Tool listing:** `/mcp` command shows connected servers and available tools
4. **Try-catch detection:** Attempt tool call, handle "unknown tool" error

### Graphite MCP Server
- **Minimum version:** gt CLI 1.6.7+
- **Setup command:** `claude mcp add graphite gt mcp`
- **Status:** Beta (per official docs)

### Likely Tool Names (LOW confidence - need verification)
Based on gt CLI commands and MCP conventions:
- `mcp__graphite__create_branch` - create stacked branch
- `mcp__graphite__modify` - commit/amend changes
- `mcp__graphite__submit` - push and create PRs
- `mcp__graphite__status` - check stack status
- `mcp__graphite__log` - show branch stack

**Note:** Actual tool names should be verified by attempting calls or checking `/mcp` output.

## Graphite CLI Reference

### Key Commands for GSD Operations

| Command | Purpose | GSD Use Case |
|---------|---------|--------------|
| `gt create [name] -m [msg]` | Create branch with commit | Phase branch creation |
| `gt modify -cam [msg]` | Amend + new commit | Task commits |
| `gt modify -am [msg]` | Amend existing commit | Amend mode |
| `gt submit --stack` | Push all in stack | PR creation |
| `gt sync` | Sync with remote | Before starting work |
| `gt log` | Show stack | Stack visualization |
| `gt log --stack` | Current branch stack | Phase progress |

### Flags Reference
- `-a, --all`: Stage all changes
- `-c, --commit`: Create new commit (not amend)
- `-m, --message`: Commit message
- `-p, --patch`: Interactive staging
- `--ai`: AI-generate PR title/description

## Open Questions

Things that couldn't be fully resolved:

1. **Exact Graphite MCP tool names**
   - What we know: MCP server exists, added via `claude mcp add graphite gt mcp`
   - What's unclear: Precise tool function names and parameters
   - Recommendation: Design with try-catch pattern, verify tool names during implementation

2. **MCP error format for "tool not found"**
   - What we know: MCP protocol returns error codes
   - What's unclear: Exact error structure Claude sees
   - Recommendation: Test during implementation, document in reference

3. **Session state scope in Claude Code**
   - What we know: Each agent spawn is fresh context
   - What's unclear: Can detection result be passed between agents?
   - Recommendation: Re-detect in each agent, or pass via prompt context

## Sources

### Primary (HIGH confidence)
- [Graphite Command Reference](https://graphite.com/docs/command-reference) - CLI commands and flags
- [Graphite Install/Auth](https://graphite.com/docs/install-the-cli) - Setup verification
- [MCP Tools Specification](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) - Protocol details
- [Claude Code MCP Docs](https://code.claude.com/docs/en/mcp) - MCP integration in Claude Code

### Secondary (MEDIUM confidence)
- [PulseMCP Graphite Server](https://www.pulsemcp.com/servers/graphite-cli) - MCP server info
- [Graphite GT MCP Docs](https://graphite.com/docs/gt-mcp) - Setup instructions

### Tertiary (LOW confidence)
- MCP tool names inferred from CLI command mapping
- Session state behavior based on GSD workflow analysis

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - GSD patterns well-documented in codebase
- Architecture: HIGH - Reference doc pattern matches existing GSD structure
- Detection protocol: MEDIUM - Based on user decisions, needs implementation verification
- MCP tool names: LOW - Not publicly documented, need try-catch pattern

**Research date:** 2026-01-17
**Valid until:** 2026-02-17 (30 days - stable domain, MCP may evolve)
