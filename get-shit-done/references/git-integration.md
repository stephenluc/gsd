<overview>
Git integration for GSD framework.
</overview>

<core_principle>

**Commit outcomes, not process.**

The git log should read like a changelog of what shipped, not a diary of planning activity.
</core_principle>

<commit_points>

| Event                   | Commit? | Why                                              |
| ----------------------- | ------- | ------------------------------------------------ |
| BRIEF + ROADMAP created | NO      | Planning process, not outcome                    |
| PLAN.md created         | NO      | Planning process                                 |
| RESEARCH.md created     | NO      | Planning process                                 |
| DISCOVERY.md created    | NO      | Planning process                                 |
| **Task completed**      | YES     | Atomic unit of work (1 commit per task)         |
| **Plan completed**      | NO      | Planning process (code already committed)        |
| Handoff created         | NO      | Planning process                                 |

</commit_points>

<git_check>

```bash
[ -d .git ] && echo "GIT_EXISTS" || echo "NO_GIT"
```

If NO_GIT: Run `git init` silently. GSD projects always get their own repo.
</git_check>

<commit_formats>


<format name="task-completion">
## Task Completion (During Plan Execution)

Each task gets its own commit immediately after completion.

```
{type}({phase}-{plan}): {task-name}

- [Key change 1]
- [Key change 2]
- [Key change 3]
```

**Commit types:**
- `feat` - New feature/functionality
- `fix` - Bug fix
- `test` - Test-only (TDD RED phase)
- `refactor` - Code cleanup (TDD REFACTOR phase)
- `perf` - Performance improvement
- `chore` - Dependencies, config, tooling

**Examples:**

```bash
# Standard task
git add src/api/auth.ts src/types/user.ts
git commit -m "feat(08-02): create user registration endpoint

- POST /auth/register validates email and password
- Checks for duplicate users
- Returns JWT token on success
"

# TDD task - RED phase
git add src/__tests__/jwt.test.ts
git commit -m "test(07-02): add failing test for JWT generation

- Tests token contains user ID claim
- Tests token expires in 1 hour
- Tests signature verification
"

# TDD task - GREEN phase
git add src/utils/jwt.ts
git commit -m "feat(07-02): implement JWT generation

- Uses jose library for signing
- Includes user ID and expiry claims
- Signs with HS256 algorithm
"
```

</format>


</commit_formats>

<example_log>

**Old approach (per-plan commits):**
```
a7f2d1 feat(checkout): Stripe payments with webhook verification
3e9c4b feat(products): catalog with search, filters, and pagination
8a1b2c feat(auth): JWT with refresh rotation using jose
5c3d7e feat(foundation): Next.js 15 + Prisma + Tailwind scaffold
2f4a8d docs: initialize ecommerce-app (5 phases)
```

**New approach (per-task commits, code only):**
```
# Phase 04 - Checkout
4d5e6f feat(04-01): add webhook signature verification
7g8h9i feat(04-01): implement payment session creation
0j1k2l feat(04-01): create checkout page component

# Phase 03 - Products
6p7q8r feat(03-02): add pagination controls
9s0t1u feat(03-02): implement search and filters
2v3w4x feat(03-01): create product catalog schema

# Phase 02 - Auth
8b9c0d feat(02-02): implement refresh token rotation
1e2f3g test(02-02): add failing test for token refresh
7k8l9m feat(02-01): add JWT generation and validation
0n1o2p chore(02-01): install jose library

# Phase 01 - Foundation
6t7u8v feat(01-01): configure Tailwind and globals
9w0x1y feat(01-01): set up Prisma with database
2z3a4b feat(01-01): create Next.js 15 project
```

Each task gets exactly one commit. Clean, granular, bisectable. No planning artifacts in git history.

</example_log>

<anti_patterns>

**Never commit .planning/ files:**
- PROJECT.md, ROADMAP.md, REQUIREMENTS.md
- PLAN.md, RESEARCH.md, DISCOVERY.md
- SUMMARY.md, STATE.md
- VERIFICATION.md, MILESTONE-AUDIT.md
- codebase/*.md
- todos/*
- .continue-here.md

**Only commit working code:**
- Each task completion (feat/fix/test/refactor/chore)

**Key principle:** Commit working code only. Planning artifacts stay local, never in git history.

</anti_patterns>

<commit_strategy_rationale>

## Why Per-Task Commits?

**Context engineering for AI:**
- Git history becomes primary context source for future Claude sessions
- `git log --grep="{phase}-{plan}"` shows all work for a plan
- `git diff <hash>^..<hash>` shows exact changes per task
- Less reliance on parsing SUMMARY.md = more context for actual work

**Failure recovery:**
- Task 1 committed ✅, Task 2 failed ❌
- Claude in next session: sees task 1 complete, can retry task 2
- Can `git reset --hard` to last successful task

**Debugging:**
- `git bisect` finds exact failing task, not just failing plan
- `git blame` traces line to specific task context
- Each commit is independently revertable

**Observability:**
- Solo developer + Claude workflow benefits from granular attribution
- Atomic commits are git best practice
- "Commit noise" irrelevant when consumer is Claude, not humans

</commit_strategy_rationale>
