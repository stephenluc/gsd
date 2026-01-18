---
name: gsd:progress
description: Check project progress, show context, and route to next action (execute or plan)
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
  - SlashCommand
---

<objective>
Check project progress, summarize recent work and what's ahead, then intelligently route to the next action - either executing an existing plan or creating the next one.

Provides situational awareness before continuing work.
</objective>


<process>

<step name="verify">
**Verify planning structure exists:**

If no `.planning/` directory:

```
No planning structure found.

Run /gsd:new-project to start a new project.
```

Exit.

If missing STATE.md: suggest `/gsd:new-project`.

**If ROADMAP.md missing but PROJECT.md exists:**

This means a milestone was completed and archived. Go to **Route F** (between milestones).

If missing both ROADMAP.md and PROJECT.md: suggest `/gsd:new-project`.
</step>

<step name="load">
**Load full project context:**

- Read `.planning/STATE.md` for living memory (position, decisions, issues)
- Read `.planning/ROADMAP.md` for phase structure and objectives
- Read `.planning/PROJECT.md` for current state (What This Is, Core Value, Requirements)

**Detect version control mode:**

```bash
VC_TYPE=$(cat .planning/config.json 2>/dev/null | grep -o '"type"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4)
```

Track `VC_TYPE` for conditional display later (graphite vs git).
</step>

<step name="recent">
**Gather recent work context:**

- Find the 2-3 most recent SUMMARY.md files
- Extract from each: what was accomplished, key decisions, any issues logged
- This shows "what we've been working on"
  </step>

<step name="position">
**Parse current position:**

- From STATE.md: current phase, plan number, status
- Calculate: total plans, completed plans, remaining plans
- Note any blockers or concerns
- Check for CONTEXT.md: For phases without PLAN.md files, check if `{phase}-CONTEXT.md` exists in phase directory
- Count pending todos: `ls .planning/todos/pending/*.md 2>/dev/null | wc -l`
- Check for active debug sessions: `ls .planning/debug/*.md 2>/dev/null | grep -v resolved | wc -l`
</step>

<step name="display_stack">
**Build stack visualization (Graphite mode only):**

Only execute this step if `VC_TYPE` is "graphite". Skip for "git" or unset.

**1. Build stack tree data:**

a) Parse all phases from ROADMAP.md:
```bash
# Extract phase numbers, names, dependencies, and status
grep -E "^### Phase [0-9]+:" .planning/ROADMAP.md
grep -E "^\*\*Status:\*\*" .planning/ROADMAP.md
grep -E "^\*\*Depends on:\*\*" .planning/ROADMAP.md
```

For each phase, extract:
- Phase number (N)
- Phase name
- Dependencies (list of phase numbers, or empty for independent)
- Status (Complete/Pending)

b) Read branch mappings from STATE.md:
```bash
# Extract phase-to-branch mappings from "## Branch Mappings" section
grep -E "^\| [0-9]+ \|" .planning/STATE.md
```

c) Get current context:
```bash
# Current branch
CURRENT_BRANCH=$(git branch --show-current)

# Trunk name
TRUNK=$(gt trunk 2>/dev/null || echo "main")
```

**2. Build tree structure:**

Using the dependency graph:
- Root = trunk branch (main/master)
- Children of trunk = phases with "Depends on: None"
- Children of phase N = phases where highest dependency is N

**3. Render ASCII tree:**

Use Unicode box-drawing characters:

```
main
├── P1: stephen/config-foundation (Configuration Foundation) ✓
│   └── P2: stephen/mcp-layer (MCP Abstraction Layer) ✓
│       └── P3: stephen/branch-mgmt (Basic Branch Management) ✓
│           ├── P4: stephen/stacking (Intelligent Branch Stacking) ✓
│           │   └── P5: stephen/stack-viz (Stack Visualization) ▶ ◀ current
│           └── P6: (planned) Basic Commit Operations ○
```

**Tree characters:**
- `├── ` branch with siblings below
- `└── ` last branch at this level
- `│   ` vertical connector for nested items
- `    ` (4 spaces) indent without connector

**Status indicators:**
- `✓` for Complete phases
- `▶` for In Progress (has branch but not complete)
- `○` for Pending (no branch yet)

**Current branch highlighting:**
- Add ` ◀ current` marker at end of line for the branch matching `git branch --show-current`

**Format each phase line as:**
`P{N}: {branch-name-or-planned} ({Phase Name}) {status}{current-marker}`

Where `{branch-name-or-planned}` is:
- The actual branch name from STATE.md Branch Mappings if exists
- `(planned)` if no branch mapping exists yet

**4. Handle collapsed view for large stacks (>8 phases):**

If total phases > 8, show a windowed view:
- Show trunk
- Show 2 phases before current
- Show current phase
- Show 2 phases after current
- Show ellipsis markers for collapsed sections

```
main
├── ... (3 earlier phases)
├── P4: stephen/stacking (Intelligent Branch Stacking) ✓
│   └── P5: stephen/stack-viz (Stack Visualization) ▶ ◀ current
└── ... (3 more phases)

[8 total branches - expand with --full]
```

**5. Store rendered tree for report step.**
</step>

<step name="report">
**Present rich status report:**

**Conditional display based on VC_TYPE:**

**If VC_TYPE is "graphite":** Show stack visualization INSTEAD OF phase table.

```
# [Project Name]

**Progress:** [████████░░] 8/10 plans complete

## Recent Work
- [Phase X, Plan Y]: [what was accomplished - 1 line]
- [Phase X, Plan Z]: [what was accomplished - 1 line]

## Current Stack

main
├── P1: stephen/config-foundation (Configuration Foundation) ✓
│   └── P2: stephen/mcp-layer (MCP Abstraction Layer) ✓
│       └── P3: stephen/branch-mgmt (Basic Branch Management) ✓
│           ├── P4: stephen/stacking (Intelligent Branch Stacking) ✓
│           │   └── P5: stephen/stack-viz (Stack Visualization) ▶ ◀ current
│           └── P6: (planned) Basic Commit Operations ○
│               └── P7: (planned) Advanced Commit Operations ○
│                   └── P8: (planned) Safety Layer ○

## Key Decisions Made
- [decision 1 from STATE.md]
- [decision 2]

## Blockers/Concerns
- [any blockers or concerns from STATE.md]

## Pending Todos
- [count] pending — /gsd:check-todos to review

## Active Debug Sessions
- [count] active — /gsd:debug to continue
(Only show this section if count > 0)

## What's Next
[Next phase/plan objective from ROADMAP]
```

**If VC_TYPE is "git" or unset:** Show standard phase table (existing behavior).

```
# [Project Name]

**Progress:** [████████░░] 8/10 plans complete

## Recent Work
- [Phase X, Plan Y]: [what was accomplished - 1 line]
- [Phase X, Plan Z]: [what was accomplished - 1 line]

## Current Position
Phase [N] of [total]: [phase-name]
Plan [M] of [phase-total]: [status]
CONTEXT: [✓ if CONTEXT.md exists | - if not]

## Key Decisions Made
- [decision 1 from STATE.md]
- [decision 2]

## Blockers/Concerns
- [any blockers or concerns from STATE.md]

## Pending Todos
- [count] pending — /gsd:check-todos to review

## Active Debug Sessions
- [count] active — /gsd:debug to continue
(Only show this section if count > 0)

## What's Next
[Next phase/plan objective from ROADMAP]
```

**Key difference:** In Graphite mode, "## Current Stack" with ASCII tree replaces "## Current Position" with text.

</step>

<step name="route">
**Determine next action based on verified counts.**

**Step 1: Count plans, summaries, and issues in current phase**

List files in the current phase directory:

```bash
ls -1 .planning/phases/[current-phase-dir]/*-PLAN.md 2>/dev/null | wc -l
ls -1 .planning/phases/[current-phase-dir]/*-SUMMARY.md 2>/dev/null | wc -l
ls -1 .planning/phases/[current-phase-dir]/*-UAT.md 2>/dev/null | wc -l
```

State: "This phase has {X} plans, {Y} summaries."

**Step 1.5: Check for unaddressed UAT gaps**

Check for UAT.md files with status "diagnosed" (has gaps needing fixes).

```bash
# Check for diagnosed UAT with gaps
grep -l "status: diagnosed" .planning/phases/[current-phase-dir]/*-UAT.md 2>/dev/null
```

Track:
- `uat_with_gaps`: UAT.md files with status "diagnosed" (gaps need fixing)

**Step 2: Route based on counts**

| Condition | Meaning | Action |
|-----------|---------|--------|
| uat_with_gaps > 0 | UAT gaps need fix plans | Go to **Route E** |
| summaries < plans | Unexecuted plans exist | Go to **Route A** |
| summaries = plans AND plans > 0 | Phase complete | Go to Step 3 |
| plans = 0 | Phase not yet planned | Go to **Route B** |

---

**Route A: Unexecuted plan exists**

Find the first PLAN.md without matching SUMMARY.md.
Read its `<objective>` section.

```
---

## ▶ Next Up

**{phase}-{plan}: [Plan Name]** — [objective summary from PLAN.md]

`/gsd:execute-phase {phase}`

<sub>`/clear` first → fresh context window</sub>

---
```

---

**Route B: Phase needs planning**

Check if `{phase}-CONTEXT.md` exists in phase directory.

**If CONTEXT.md exists:**

```
---

## ▶ Next Up

**Phase {N}: {Name}** — {Goal from ROADMAP.md}
<sub>✓ Context gathered, ready to plan</sub>

`/gsd:plan-phase {phase-number}`

<sub>`/clear` first → fresh context window</sub>

---
```

**If CONTEXT.md does NOT exist:**

```
---

## ▶ Next Up

**Phase {N}: {Name}** — {Goal from ROADMAP.md}

`/gsd:discuss-phase {phase}` — gather context and clarify approach

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- `/gsd:plan-phase {phase}` — skip discussion, plan directly
- `/gsd:list-phase-assumptions {phase}` — see Claude's assumptions

---
```

---

**Route E: UAT gaps need fix plans**

UAT.md exists with gaps (diagnosed issues). User needs to plan fixes.

```
---

## ⚠ UAT Gaps Found

**{phase}-UAT.md** has {N} gaps requiring fixes.

`/gsd:plan-phase {phase} --gaps`

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- `/gsd:execute-phase {phase}` — execute phase plans
- `/gsd:verify-work {phase}` — run more UAT testing

---
```

---

**Step 3: Check milestone status (only when phase complete)**

Read ROADMAP.md and identify:
1. Current phase number
2. All phase numbers in the current milestone section

Count total phases and identify the highest phase number.

State: "Current phase is {X}. Milestone has {N} phases (highest: {Y})."

**Route based on milestone status:**

| Condition | Meaning | Action |
|-----------|---------|--------|
| current phase < highest phase | More phases remain | Go to **Route C** |
| current phase = highest phase | Milestone complete | Go to **Route D** |

---

**Route C: Phase complete, more phases remain**

Read ROADMAP.md to get the next phase's name and goal.

```
---

## ✓ Phase {Z} Complete

## ▶ Next Up

**Phase {Z+1}: {Name}** — {Goal from ROADMAP.md}

`/gsd:discuss-phase {Z+1}` — gather context and clarify approach

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- `/gsd:plan-phase {Z+1}` — skip discussion, plan directly
- `/gsd:verify-work {Z}` — user acceptance test before continuing

---
```

---

**Route D: Milestone complete**

```
---

## 🎉 Milestone Complete

All {N} phases finished!

## ▶ Next Up

**Complete Milestone** — archive and prepare for next

`/gsd:complete-milestone`

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- `/gsd:verify-work` — user acceptance test before completing milestone

---
```

---

**Route F: Between milestones (ROADMAP.md missing, PROJECT.md exists)**

A milestone was completed and archived. Ready to start the next milestone cycle.

Read MILESTONES.md to find the last completed milestone version.

```
---

## ✓ Milestone v{X.Y} Complete

Ready to plan the next milestone.

## ▶ Next Up

**Start Next Milestone** — questioning → research → requirements → roadmap

`/gsd:new-milestone`

<sub>`/clear` first → fresh context window</sub>

---
```

</step>

<step name="edge_cases">
**Handle edge cases:**

- Phase complete but next phase not planned → offer `/gsd:plan-phase [next]`
- All work complete → offer milestone completion
- Blockers present → highlight before offering to continue
- Handoff file exists → mention it, offer `/gsd:resume-work`
  </step>

</process>

<success_criteria>

- [ ] Rich context provided (recent work, decisions, issues)
- [ ] Current position clear with visual progress
- [ ] What's next clearly explained
- [ ] Smart routing: /gsd:execute-phase if plans exist, /gsd:plan-phase if not
- [ ] User confirms before any action
- [ ] Seamless handoff to appropriate gsd command
      </success_criteria>
