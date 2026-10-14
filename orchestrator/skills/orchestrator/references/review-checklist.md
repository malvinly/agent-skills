# Main-session review checklist

Run this yourself, in the main session, for every completed task. Never delegate any of it to a subagent — the point of this review is an independent pair of eyes on work the subagent believes is finished.

## 1. Verify independently (before reading anything else)

- [ ] Read the actual change — `git status`, then `git diff` (unstaged only: earlier approved tasks are already staged, so this shows just this task's work; for tasks run in parallel, narrow it with `git diff -- <task files>`), and open newly created (untracked) files in full, since `git diff` does not show them. Never settle for the subagent's summary. If the unstaged diff is empty after a `DONE` report, the subagent probably staged or stashed its work despite the prohibition (`git stash list`; `git diff --cached --stat -- <task files>`). Do not unstage anything — that would also drop earlier approved hunks in shared files. Review `git diff --cached -- <task files>` for this task instead, and record the incident as a discovery so later dispatch prompts stress the rule.
- [ ] Re-run the task's verification command yourself (e.g. `dotnet build MyApp.sln`) and confirm the expected result. A subagent's "verification passed" is a claim, not evidence.

## 2. Spec compliance (before code quality)

- [ ] Only files in scope were touched: every unstaged or untracked path in `git status` is on this task's file list, on the list of a sibling task in the same parallel batch, is the plan file, or is a pre-existing dirty path the user waived at startup. Anything else → stop and ask the user.
- [ ] The diff does exactly what the task described — no more, no less. No drive-by refactoring, no unrequested features, no "while I was in there" edits.
- [ ] The verification result matches the expected result written in the task.

## 3. Code quality

- [ ] Follows the codebase's existing conventions — naming, structure, error handling, style.
- [ ] No obvious bugs or unhandled edge cases (nulls, empty collections, malformed input) within the change's scope.
- [ ] No leftover debug output, commented-out code, or stray TODOs.
- [ ] Where the task calls for tests, the tests actually exercise the new behavior (run them; check they fail meaningfully if the behavior broke).

## Outcome

- **Approve:** stage everything approved for this task — its listed files plus any extra change the user accepted, renames and deletions included — with `git add <paths>`, never the plan file; tick the task's `implemented` and `reviewed` checkboxes in the plan file; record reusable discoveries for later dispatch prompts.
- **Problems:** dispatch a follow-up subagent with specific, actionable feedback — what is wrong, where, and what correct looks like. Then re-review from the top of this checklist.
