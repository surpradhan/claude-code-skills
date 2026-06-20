# /pr-loop

> A Claude Code skill that drives work items through a full multi-agent PR pipeline - autonomously.

**pre-flight → branch → conflict check → implement → gates → PR → 2 independent reviews → fix every nit → re-review → CI → human merge**

---

## What it does

`/pr-loop` takes one or more issues or work items and drives each one through your project's complete PR pipeline using three independent AI contexts:

| Context | Role |
|---|---|
| **Author** | Reads project rules, branches, implements, runs gates, opens PR |
| **Reviewer × 2** | Independent security/correctness/perf review + empirical gate validation |
| **Merger** *(opt-in)* | Re-verifies everything, squash-merges, cleans up |

The context that wrote the code never reviews or merges it. Same principle as human PR etiquette - enforced by design.

---

## Quick start

```bash
# Install: drop pr-loop.md into your Claude Code skills directory
# Then in any project:

/pr-loop "fix the auth token expiry bug"
/pr-loop 42
/pr-loop "refactor payment module #88" "add retry logic #91"
```

Chain multiple items - each gets its own PR, processed in order. Item N+1 doesn't start until item N merges.

**Wrap in `/loop` to run hands-free across a long queue:**
```bash
/loop /pr-loop 42 57 91
```
The loop waits for review and merge subagents to finish before advancing - no polling. A fallback heartbeat (1200–1800 s) wakes it if a notification is missed. The loop ends automatically once all items are merged or handed off.

---

## How it works

### 0. Learn the project first
Before writing a single line, `/pr-loop` reads your project's contribution rules:

- `CONTRIBUTING.md` — commit format, gate commands
- `CLAUDE.md` / `AGENTS.md` — agent-specific guidance
- `README` — anything else

No hardcoded assumptions. Adapts to each project's conventions.

### 1. Pre-flight & ground
If the issue is ambiguous about scope, acceptance criteria, or approach, `/pr-loop` stops and asks for clarification before writing any code. Once clear, it re-reads the full issue and the code paths it touches.

### 2. Branch & conflict check
Creates a properly named branch in a fresh git worktree, then verifies the branch merges cleanly with the default branch (`git merge --no-commit --no-ff`) before writing a single line. Surfaces conflicts to the user rather than auto-resolving.

### 3. Implement
- Implements the change, matching surrounding code style
- For large changes (>3 files or a design decision): opens a draft PR early for directional feedback, re-checks for conflicts before marking it ready
- Spins off follow-up issues for out-of-scope items (no scope creep)

### 4. Local gates
Runs exactly what your project requires - tests, linter, type-check, build. All green before pushing.

### 5. PR
- Commit format from your project rules
- PR body references the issue (`Closes #n`)
- Title per the project's commit format

### 6. Two independent reviews (parallel)
Two separate subagents with no shared context with the author:

**`pr-code-reviewer`** - structured review covering:
- Security vulnerabilities
- Correctness and edge cases
- Performance implications
- Maintainability and readability

**`pr-validator`** - empirical review:
- Actually runs the gates
- Probes the change's documented claims
- Verifies behaviour matches the PR description

### 7. Fix *everything*
Every finding gets addressed - including nits. Nothing is silently dropped. If a reviewer marks something "no change needed," the author either makes the improvement or records the deliberate decision in code/PR comments.

### 8. Re-review loop
Both reviewers run again on the updated PR. Loops until:
- ✅ Zero actionable items
- ✅ Zero new findings  
- ✅ Both approve

**Cap: 3 rounds.** If items still surface after round 3, the loop pauses and asks you.

### 9. CI
Waits for CI to go green. Retries a failing job once, inspects logs before assuming flakiness, surfaces persistent or environmental failures to the user rather than retrying indefinitely.

### 10. Merge
**Default (safe):** stops at hand-off once CI is green and both reviews are clean. Reports PR link and status. A human merges.

**`autonomous-merge` (opt-in):** only activates if you explicitly request it AND your project permits agent merges. A separate merger agent re-verifies everything before squash-merging.

---

## Merge authority

```
/pr-loop 42                          # → stops at hand-off (default)
/pr-loop 42 autonomous-merge         # → agent merges if project permits
```

The `autonomous-merge` token requires both:
1. Explicit user request
2. Project documentation permitting agent merges

Without both conditions, it always hands off to a human.

---

## Project configuration

Define a `## PR-loop` section in your `CONTRIBUTING.md` to control:

```markdown
## PR-loop

- **Commit format**: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`
- **Gate commands**: `npm test && npm run lint`
- **Merge authority**: human-only
- **Branch convention**: `<type>/<issue-number>-<slug>`
```

If none of this is documented, `/pr-loop` asks you to confirm before starting.

---

## Safety valves

The loop pauses and asks you when:

- 🔴 Review findings still surface after 3 rounds
- 🔴 Design ambiguity the issue/project rules don't settle
- 🔴 A `Request Changes` it can't confidently resolve
- 🔴 CI fails and it can't fix it
- 🔴 Merge agent refuses, or project forbids the requested merge

You're always in control.

---

## Three-context separation

```
         ┌─────────────────┐
         │  AUTHOR (you)   │  ← writes code, opens PR
         └────────┬────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
 ┌─────────────┐    ┌──────────────────┐
 │ pr-code-    │    │  pr-validator    │
 │ reviewer    │    │  (runs gates,    │
 │             │    │  probes claims)  │
 └──────┬──────┘    └────────┬─────────┘
        └──────────┬─────────┘
                   │ (opt-in)
                   ▼
          ┌─────────────────┐
          │  pr-merger      │  ← separate context, never the author
          └─────────────────┘
```

The author context never reviews or merges. Always.

---

## Installation

1. Clone this repo or copy `pr-loop.md` into your Claude Code skills directory:
   ```
   ~/.claude/skills/pr-loop.md
   ```

2. In any project with Claude Code, type:
   ```
   /pr-loop <issue-number or description>
   ```

3. Watch your PR ship itself.

---

## Requirements

- [Claude Code](https://claude.ai/code) (CLI)
- `gh` CLI authenticated to a GitHub remote
- `git` with worktree support

---

## License

MIT
