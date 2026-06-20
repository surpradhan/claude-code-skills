---
description: Drive one or more work items through a full multi-agent PR pipeline (pre-flight → branch → implement → conflict check → gates → PR → 2 independent reviews → fix every nit → re-review → CI → human merge, or opt-in autonomous merge). Requires GitHub remote and the `gh` CLI.
argument-hint: <issue # / task description> [, <next item> …] [autonomous-merge]
---

# /pr-loop

**Prerequisites:** `git` with worktree support, `gh` CLI installed and authenticated to a GitHub remote.

Take the work item(s) in `$ARGUMENTS` and drive each one, in order, through this
project's PR pipeline. One PR per item; advance to the next item only after the
current one is merged (or handed off for merge).

## 0. Learn this project's rules first (REQUIRED — adapts the skill to project conventions)

Before doing anything, discover the **project's contribution rules** and obey
them. Read, in order, whichever exist: a `## PR-loop` / "PR-loop contract"
section in `CONTRIBUTING.md`; then `CONTRIBUTING.md` generally; then `CLAUDE.md`
/ `AGENTS.md`; then `README`. Extract the project-specific values this command
deliberately does **not** hardcode:

- **commit format** (type-prefix set);
- **local gates** to run before pushing (test command, linter, type-check, build);
- **required CI checks** / what "mergeable" means here;
- **branch & worktree convention**, and any **merge-mechanics** quirks (e.g. how the
  remote branch must be deleted, hooks that block delete refspecs);
- **merge authority** — whether agent merge is permitted at all, or a human must
  approve and merge.

If the project documents none of this, ask the user to confirm: commit format,
the exact gate commands, and whether autonomous merge is allowed. **Do not
guess project policy.**

## 1. Per-item pipeline

For each work item:

1. **Ground.** Re-read the issue/spec (`gh issue view <n>` if it's an issue) and
   the actual code paths it touches before writing anything.
   **Pre-flight check:** if the issue is ambiguous about scope, acceptance criteria,
   or approach, STOP and ask the user to clarify before writing any code. A 3-word
   issue title is not enough to act on.
2. **Branch.** Create a fresh git worktree on a meaningfully-named branch off the
   default branch (per the project's convention). Never push an auto-generated
   slug branch name.
3. **Implement** the change, matching surrounding code and the issue's scope.
   Resist scope creep; spin off a follow-up issue for out-of-scope items.
   **For large changes** (touching >3 files or requiring a design decision): push
   an early draft PR (`gh pr create --draft`) to get directional feedback before
   full implementation. Mark it ready when implementation is complete.
4. **Conflict check.** Before and after implementation, check for merge conflicts
   (`git merge-base --fork-point <default-branch>` or attempt a dry merge). If
   conflicts exist, STOP and surface them to the user; do not attempt to
   auto-resolve non-trivial conflicts.
5. **Local gates.** Run exactly what the project requires (e.g. tests + linter,
   plus a build step if relevant files changed). All green before pushing.
6. **Push & open a PR.** Use the project's commit format.
   Reference the issue (`Closes #n`) in the PR body. Title per the project's commit format.
7. **Two independent reviews.** Spawn TWO separate review subagents in parallel
   (separate contexts from you, the author):
   - **pr-code-reviewer** — structured security/correctness/perf/maintainability review;
   - **pr-validator** — PR review **and** actually runs the gates + empirically
     probes the change's claims.
   Give each the PR number and the branch worktree path. (`pr-code-reviewer`,
   `pr-validator`, and `pr-merger` are Claude Code subagent types; if they aren't
   available in your environment, spawn general-purpose agents with the same
   instructions.)
8. **Fix every finding — including nits.** Nothing is ignored. For a finding a
   reviewer marks "no change needed," either make the clean improvement or record
   the deliberate decision in code/PR; do not silently drop it.
9. **Re-review.** Re-run BOTH review agents on the updated PR until both return a
   clean approve with zero actionable items and no new findings. **Cap: 3 rounds.**
   If items still surface after 3 rounds, PAUSE and ask the user.
10. **CI.** Wait for CI to go green. If CI fails, retry the failing job once. If it
    fails again, inspect the logs before assuming flakiness; fix real failures. If
    a job fails a second time and the failure looks environmental (infra error,
    unrelated test, network timeout), STOP and surface it to the user rather than
    retrying indefinitely.
11. **Merge step — depends on merge authority (see §2).**
12. **Advance.** Update the local default branch (`git pull --ff-only`), confirm the
    merge landed and the issue closed, remove any stray worktrees. Check for and
    update project changelog files: look for `CHANGELOG.md`, `docs/changelog*`, or
    any file the project's CONTRIBUTING/README designates for release notes. Post a
    one-line status, then move to the next item.

## 2. Merge authority

**Default (safe, conservative): stop at hand-off.** Once both reviews are clean and
CI is green, STOP at: *"PR open, CI green, both reviews clean — awaiting human
review/merge."* Do not merge. Report the PR link and gate status. This is the
default unless the project's rules AND the user both permit otherwise.

**Opt-in: `autonomous-merge`.** Only if BOTH (a) the user explicitly requested
autonomous merge (e.g. the `autonomous-merge` token in `$ARGUMENTS`, or a clear
instruction) AND (b) the project permits agent merge — spawn a SEPARATE
**pr-merger** agent (a third context). It independently re-verifies the gate (CI
green, mergeable CLEAN, both reviews clean), squash-merges using the project's
mechanics, cleans up the branch/worktree, and reports the squash SHA. If it
refuses or the gate fails, STOP and surface it. The authoring context never merges
its own PR.

## 3. Three-context separation (always)

author (this loop) → two **review** agents → one **merge** agent (opt-in only).
The context that wrote the code never reviews or merges it.

## 4. Safety valves — PAUSE and ask the user on

- review still surfacing actionable items after 3 rounds;
- design ambiguity or a decision the issue/project rules don't settle;
- a `Request Changes` you can't confidently resolve;
- CI red you can't fix;
- a merge agent refusal, or the project forbidding a merge the user asked for.

## 5. Pacing (when invoked via `/loop`)

If wrapped in `/loop`, self-pace: the review/merge subagents are harness-tracked
and wake the loop on completion, so use a long fallback heartbeat rather than
polling (consult the `/loop` harness docs for the current recommended value).
End the loop (no further wakeup) once every item in `$ARGUMENTS` is merged or
handed off.
