---
description: Drive one or more work items through a full multi-agent PR pipeline (pre-flight → branch → conflict check → implement → gates → PR → 2 independent reviews → fix every nit → re-review → CI → human merge, or opt-in autonomous merge). Requires GitHub remote and the `gh` CLI.
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

1. **Pre-flight.** If the issue is ambiguous about scope, acceptance criteria, or
   approach, STOP and ask the user to clarify before writing any code.
2. **Ground.** First, verify GitHub connectivity: run `gh auth status` and `gh repo view`. If either fails, STOP immediately and report: "GitHub CLI is not authenticated or this repository is not hosted on GitHub. /pr-loop requires the gh CLI with GitHub access." Do not proceed. Then re-read the issue/spec (`gh issue view <n>` if it's an issue) and the actual code paths it touches before writing anything.
3. **Branch.** Create a fresh git worktree on a meaningfully-named branch off the
   default branch (per the project's convention). Never push an auto-generated
   slug branch name. Capture and record the worktree path immediately after creating it. You will need this path when passing arguments to the review subagents in step 8. Do not reconstruct it by convention later — record the exact path returned or used.
4. **Conflict check.** Verify the fresh branch merges cleanly before writing any code:
   ```
   git fetch origin
   git merge --no-commit --no-ff origin/<default-branch>
   git merge --abort 2>/dev/null || true   # no-op if already up to date
   ```
   If conflicts exist, STOP and surface them to the user; do not attempt to
   auto-resolve non-trivial conflicts.
5. **Implement** the change, matching surrounding code and the issue's scope.
   Resist scope creep; spin off a follow-up issue for out-of-scope items.
   **For large changes** (touching >3 files or requiring a design decision): push
   an early draft PR (`gh pr create --draft`) to get directional feedback before
   full implementation. Re-run the conflict check before marking the draft ready.
6. **Local gates.** Run exactly what the project requires (e.g. tests + linter,
   plus a build step if relevant files changed). All green before pushing.
7. **Push & open a PR.** Use the project's commit format.
   Reference the issue using the keyword format the project uses (discovered in §0 — common formats include `Closes #n`, `Fixes #n`, `Resolves #n`). If the project uses a non-GitHub issue tracker (Jira, Linear), use the format documented in `CONTRIBUTING.md`. Title per the project's commit format.
8. **Two independent reviews.** Spawn TWO separate review subagents in parallel
   (separate contexts from you, the author):
   - **pr-code-reviewer** — structured security/correctness/perf/maintainability review;
   - **pr-validator** — PR review **and** actually runs the gates + empirically
     probes the change's claims.
   Pass arguments as comma-separated values: `<PR number>,<worktree path>`. (`pr-code-reviewer`,
   `pr-validator`, and `pr-merger` are Claude Code subagent types; if they aren't
   available in your environment, spawn general-purpose agents with the same
   instructions.)
9. **Fix every finding — including nits.** Nothing is ignored. For a finding a
   reviewer marks "no change needed," either make the clean improvement or record
   the deliberate decision in code/PR; do not silently drop it.
10. **Re-review.** Re-run BOTH review agents on the updated PR. The exit condition for the review loop is both subagents returning APPROVE (from pr-code-reviewer) and VALIDATED or VALIDATED_WITH_PREEXISTING_FAILURES (from pr-validator). A verdict of APPROVE may still contain MINOR or NIT findings — these should be addressed but do not prevent advancing. The re-review loop continues only if either subagent returns REQUEST_CHANGES or FAILED. **Cap: 3 rounds.**
    If items still surface after 3 rounds, PAUSE and ask the user.

11. **CI.** Wait for CI to go green. Poll CI status with `gh pr checks <n>` every 60 seconds. After 30 minutes of polling with no green result, STOP and surface to the user: "CI has not completed after 30 minutes. Check the CI system directly." If CI fails, retry the failing job once. If it fails again, inspect the logs before assuming flakiness; fix real failures. If a job fails a second time and the failure looks environmental (infra error, unrelated test, network timeout), STOP and surface it to the user rather than retrying indefinitely.

    Once CI is green **and** both reviews are clean (from step 10), post their approvals to GitHub: run `gh pr review <n> --approve --body 'pr-code-reviewer: APPROVE'` and `gh pr review <n> --approve --body 'pr-validator: <actual verdict>'` — use the exact verdict string returned by the validator (`VALIDATED` or `VALIDATED_WITH_PREEXISTING_FAILURES`); if the verdict was `VALIDATED_WITH_PREEXISTING_FAILURES`, surface the pre-existing failures to the user before proceeding. Note: these reviews are posted under the orchestrator's GitHub identity and serve as audit markers only — they will not satisfy branch protection rules that require non-author approvers. Then, when invoking pr-merger, append `REVIEWS_CERTIFIED` to the arguments string.
12. **Merge step — depends on merge authority (see §2).**
13. **Advance.** Update the local default branch (`git pull --ff-only`), confirm the
    merge landed and the issue closed, remove any stray worktrees. Search for
    changelog files (`find . -maxdepth 3 -iname 'CHANGELOG*' -o -iname 'CHANGES*'`)
    or any path the project's CONTRIBUTING/README designates for release notes;
    update if one exists. Post a one-line status, then move to the next item.

## 2. Merge authority

**Default (safe, conservative): stop at hand-off.** Once both reviews are clean and
CI is green, STOP at: *"PR open, CI green, both reviews clean — awaiting human
review/merge."* Do not merge. Report the PR link and gate status. This is the
default unless the project's rules AND the user both permit otherwise.

**Opt-in: `autonomous-merge`.** Only if BOTH (a) the user explicitly requested
autonomous merge (e.g. the `autonomous-merge` token in `$ARGUMENTS`, or a clear
instruction) AND (b) the project permits agent merge — spawn a SEPARATE
**pr-merger** agent (a third context). Pass arguments as comma-separated values: `<PR number>,<branch name>,<worktree path>,REVIEWS_CERTIFIED` (the `REVIEWS_CERTIFIED` signal must only be appended after CI is green, both review agents have returned a clean verdict, and their approvals have been posted to GitHub as described in step 11). It independently re-verifies the gate (CI green, mergeable CLEAN, both reviews clean), squash-merges using the project's mechanics, cleans up the branch/worktree, and reports the squash SHA. If it refuses or the gate fails, STOP and surface it. The authoring context never merges its own PR.

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
and wake the loop on completion, so use a long fallback heartbeat (1200–1800 s)
rather than polling. End the loop (no further wakeup) once every item in
`$ARGUMENTS` is merged or handed off.
