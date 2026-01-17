<overview>
Unified interface for Graphite operations in GSD framework.

This reference document provides the abstraction layer that routes Graphite operations to the correct backend: MCP tools when available, `gt` CLI as fallback, and standard `git` as ultimate fallback. Agents include this reference to ensure consistent operation across different environments.

**Key principle:** Config authority first, then detection, then operation. The `.planning/config.json` versionControl.type setting is authoritative - if it says "git", never attempt Graphite backends.
</overview>

<backend_detection_protocol>
## Backend Detection Protocol

Detection happens lazily (on first operation, not at session start) and is sticky per session (no re-detection after determined). Each new session re-detects.

### Step 1: Read Config Authority

Check `.planning/config.json` for the versionControl.type setting:

```bash
VC_TYPE=$(cat .planning/config.json 2>/dev/null | grep -o '"type"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4)
VC_TYPE=${VC_TYPE:-git}  # Default to git if not set
```

**Routing:**
- If `VC_TYPE` is "git" -> Use git-only mode (skip all Graphite detection)
- If `VC_TYPE` is "graphite" -> Proceed to Step 2 (MCP detection)

**Rationale:** Config is authoritative. Users who selected git-only mode should never see Graphite errors or unexpected behavior.

### Step 2: Check MCP Availability (Graphite mode only)

Attempt an MCP tool call to verify the Graphite MCP server is connected:

```
Try: mcp__graphite__status (or similar status check)
```

**Routing:**
- Success (tool returns result) -> Use MCP backend
- Tool not found / error -> Proceed to Step 3 (CLI detection)

**Note:** MCP tool names are LOW confidence. Use try-catch pattern - attempt the call, handle "unknown tool" gracefully.

### Step 3: Check CLI Availability (MCP unavailable)

Check if `gt` CLI is installed and accessible:

```bash
gt --version >/dev/null 2>&1 && echo "gt-available" || echo "gt-unavailable"
```

**Routing:**
- "gt-available" -> Use CLI backend
- "gt-unavailable" -> Use git fallback (with warning)

### Step 4: Set Session Backend

Store the detected backend for the session:

| Detection Result | Backend | User Notification |
|------------------|---------|-------------------|
| MCP available | `mcp` | None (silent) |
| CLI available (MCP failed) | `cli` | None (silent fallback) |
| Git only (config says git) | `git` | None (user's choice) |
| Git fallback (graphite config but unavailable) | `git` | Warn once per session |

**Warning message (git fallback only):**
> Graphite tools not available. Using git-only mode. Stacking features will not work.

</backend_detection_protocol>

<session_state>
## Session State Management

### Lazy Detection
- Do NOT detect backend at session start
- Detect on first Graphite operation
- Avoids overhead when operations not needed

### Sticky Per Session
- Once backend is determined, stick with it for the session
- No re-detection after first operation
- Provides consistent behavior within session

### Re-detect Each New Session
- Each new agent spawn starts fresh
- Config may have changed
- MCP may have become available/unavailable
- Do NOT persist backend choice to disk

### Passing State Between Agents (Optional)
If orchestrating multiple agents, backend can be passed via prompt context:
```markdown
Session backend: mcp
```
This avoids redundant detection but is optional.

</session_state>

<operation_mappings>
## Operation Mappings

GSD operations mapped to each backend. Use the detected backend's command.

### Core Operations Table

| Operation | MCP | CLI | Git |
|-----------|-----|-----|-----|
| Create branch | `mcp__graphite__create_branch` | `gt create "{name}" -m "{msg}"` | `git checkout -b "{name}"` |
| Commit (new) | `mcp__graphite__modify` | `gt modify -cam "{msg}"` | `git add -A && git commit -m "{msg}"` |
| Commit (amend) | `mcp__graphite__modify` (amend flag) | `gt modify -am "{msg}"` | `git commit --amend -m "{msg}"` |
| Push/Submit | `mcp__graphite__submit` | `gt submit --stack` | `git push -u origin HEAD` |
| Sync | `mcp__graphite__sync` | `gt sync` | `git pull --rebase` |
| Status | `mcp__graphite__status` | `gt log --stack` | `git status` |
| Stack log | `mcp__graphite__log` | `gt log` | `git log --oneline` |

**Important:** MCP tool names are LOW confidence - not publicly documented in detail. Agents should use try-catch pattern and fall back to CLI if MCP call fails.

### Create Branch

**MCP Backend:**
```
mcp__graphite__create_branch
  name: "{username}/{feature-name}"
  commit_message: "{message}"
```

**CLI Backend:**
```bash
gt create "{username}/{feature-name}" -m "{message}"
```

**Git Backend (fallback):**
```bash
git checkout -b "{username}/{feature-name}"
git commit --allow-empty -m "{message}"
```

Note: Git fallback loses Graphite stacking benefits.

### Commit Changes

**New Commit:**

| Backend | Command |
|---------|---------|
| MCP | `mcp__graphite__modify` with `commit: true`, `message: "{msg}"` |
| CLI | `gt modify -cam "{msg}"` |
| Git | `git add -A && git commit -m "{msg}"` |

**Amend Existing:**

| Backend | Command |
|---------|---------|
| MCP | `mcp__graphite__modify` with `amend: true`, `message: "{msg}"` |
| CLI | `gt modify -am "{msg}"` |
| Git | `git add -A && git commit --amend -m "{msg}"` |

### Push/Submit

**MCP Backend:**
```
mcp__graphite__submit
  stack: true
```

**CLI Backend:**
```bash
gt submit --stack
```

**Git Backend:**
```bash
git push -u origin HEAD
```

Note: Git push does not create PRs or maintain stack relationships.

### Sync with Remote

**MCP Backend:**
```
mcp__graphite__sync
```

**CLI Backend:**
```bash
gt sync
```

**Git Backend:**
```bash
git pull --rebase
```

### View Status/Stack

**MCP Backend:**
```
mcp__graphite__status  # Current status
mcp__graphite__log     # Stack log
```

**CLI Backend:**
```bash
gt log --stack  # Current branch stack
gt log          # Full stack visualization
```

**Git Backend:**
```bash
git status      # Working tree status
git log --oneline -10  # Recent commits
```

</operation_mappings>

<fallback_pattern>
## Fallback Pattern

Template for agents to follow when executing Graphite operations:

### Execution Flow

```
For each Graphite operation:

1. Check session backend (from detection protocol)
   - If not yet determined, run detection

2. Execute operation for that backend
   - MCP: Call mcp__graphite__* tool
   - CLI: Run gt command
   - Git: Run git command

3. If operation fails:
   a. If MCP failed -> retry with CLI silently
   b. If CLI failed -> retry with git (if operation is safe)
   c. If all fail -> report error to user

4. Continue with result
```

### Safe Fallback Operations

These operations can safely fall back through all backends:
- Create branch
- Commit changes
- View status/log
- Sync/pull

### Caution Fallback Operations

These operations have different semantics per backend:
- Submit/push: Git push doesn't create PRs
- Stack operations: Git has no stack concept

Warn user when falling back on these.

### Agent Implementation Example

```markdown
## Creating a Branch

Backend: {session_backend}

If MCP:
  Call mcp__graphite__create_branch(name="{branch}", message="{message}")
  If tool not found: set backend to CLI and retry

If CLI:
  Run: gt create "{branch}" -m "{message}"
  If command fails: set backend to git, warn user, retry

If Git:
  Run: git checkout -b "{branch}" && git add -A && git commit --allow-empty -m "{message}"
  If fails: report error
```

</fallback_pattern>

<notification_policy>
## Notification Policy

Based on user decisions from project context.

### Silent Fallback

| Fallback Path | Notification |
|---------------|--------------|
| MCP -> CLI | None (silent) |
| Within same tier | None (retry is internal) |
| User chose git mode | None (their choice) |

### Warn Once Per Session

| Scenario | Notification |
|----------|--------------|
| Graphite config but falling back to git-only | Warn once |
| Graphite features unavailable | Warn once |

**Warning format:**
> Graphite tools not available. Using git-only mode. Stacking features will not work.

### Never Notify

- On successful operations
- On every fallback occurrence (only first time)
- Internal retry attempts

### Error Reporting

Only report to user when:
- All backends failed
- Operation cannot complete
- User action required

</notification_policy>

<anti_patterns>
## Anti-Patterns to Avoid

### Don't: Check Backend Every Operation
**Wrong:**
```markdown
For each operation:
1. Detect backend
2. Execute
```

**Right:**
```markdown
On first operation:
1. Detect backend (lazy)
2. Store for session

For each operation:
1. Use stored backend
2. Execute
```

### Don't: Persist Backend Across Sessions
**Wrong:**
```bash
echo "mcp" > .planning/.backend-cache
```

**Right:**
Re-detect each session. Config may change, MCP availability may change.

### Don't: Hardcode MCP Tool Names in Multiple Places
**Wrong:** Scatter `mcp__graphite__create_branch` throughout various prompts.

**Right:** Reference this document. Single source of truth for tool names.

### Don't: Notify on Every Fallback
**Wrong:**
```
[Info] Falling back to CLI...
[Info] Falling back to CLI...
[Info] Falling back to CLI...
```

**Right:** Warn once per session when downgrading to git-only. Silent otherwise.

### Don't: Use Lowest Common Denominator
**Wrong:** Only use git commands since they always work.

**Right:** Use full Graphite features when available. Expose MCP/CLI capabilities.

### Don't: Ignore Config Authority
**Wrong:** Always try MCP first regardless of config.

**Right:** If config says "git", use git-only. No Graphite detection needed.

</anti_patterns>

<config_reading>
## Config Reading Reference

### Read Version Control Settings

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

### Expected Config Structure

```json
{
  "versionControl": {
    "type": "graphite",
    "username": "developer-name"
  }
}
```

### Config Values

| Field | Values | Meaning |
|-------|--------|---------|
| `type` | `"graphite"` | Use Graphite stack workflow (MCP/CLI) |
| `type` | `"git"` | Use standard git workflow only |
| `username` | string | Username prefix for branch names |

</config_reading>

<graphite_cli_reference>
## Graphite CLI Quick Reference

### Key Commands for GSD Operations

| Command | Purpose | Flags |
|---------|---------|-------|
| `gt create [name] -m [msg]` | Create stacked branch | `-m` message |
| `gt modify -cam [msg]` | Stage all + new commit | `-c` commit, `-a` all, `-m` message |
| `gt modify -am [msg]` | Stage all + amend | `-a` all, `-m` message (no `-c` = amend) |
| `gt submit --stack` | Push entire stack | `--stack` all branches |
| `gt sync` | Sync with remote | |
| `gt log` | Show full stack | |
| `gt log --stack` | Show current branch stack | `--stack` current only |

### Common Flag Combinations

| Flags | Meaning |
|-------|---------|
| `-cam` | Commit (new), All files, Message |
| `-am` | Amend, All files, Message |
| `-m` | Message only (use with create) |
| `--stack` | Operate on entire stack |
| `--ai` | AI-generate PR description |

### Graphite MCP Server

- **Minimum version:** gt CLI 1.6.7+
- **Setup command:** `claude mcp add graphite gt mcp`
- **Status:** Beta

</graphite_cli_reference>

<requirements_coverage>
## Requirements Coverage

This reference document addresses the following MCP abstraction layer requirements:

| Requirement | Coverage |
|-------------|----------|
| **MCP-01:** MCP operation mappings | Operation Mappings section - MCP column in all tables |
| **MCP-02:** CLI fallback operations | Operation Mappings section - CLI column in all tables |
| **MCP-03:** Git fallback operations | Operation Mappings section - Git column in all tables |
| **MCP-04:** Detection protocol | Backend Detection Protocol section - 4-step flow |
| **MCP-05:** Notification policy | Notification Policy section - silent/warn rules |

### Traceability

- **Detection protocol:** Steps 1-4 cover config authority, MCP check, CLI check, backend selection
- **Operation mappings:** Create, Commit, Push, Sync, Status all mapped to MCP/CLI/Git
- **Fallback chain:** Pattern documented with safe/caution operation guidance
- **Notification:** Silent fallback except git-only downgrade warning
- **Anti-patterns:** Common mistakes documented with correct alternatives

</requirements_coverage>

<error_handling>
## Error Handling

### Retry Transient Errors

Retry once automatically for:
- Network timeouts
- Temporary MCP connection issues
- Transient git lock errors

### Error Recovery Flow

```
1. Operation fails
2. If retryable error: retry once
3. If still fails or non-retryable:
   a. If MCP: try CLI
   b. If CLI: try Git (with warning)
   c. If Git: report error to user
```

### Optional Verbose Mode

For troubleshooting, agents can enable verbose output:
```bash
# Show detection steps
echo "[DEBUG] Config type: $VC_TYPE"
echo "[DEBUG] Backend: $SESSION_BACKEND"
```

Use only when diagnosing issues, not in normal operation.

### When All Backends Fail

Report to user with:
- What operation was attempted
- Which backends were tried
- The final error message
- Suggested resolution (if known)

</error_handling>
