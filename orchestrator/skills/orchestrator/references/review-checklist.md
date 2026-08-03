# Main-session review checklist

Run this yourself, in the main session, for every completed task. Never delegate any of it to a subagent — the point of this review is an independent pair of eyes on work the subagent believes is finished.

## 1. Verify independently (before reading anything else)

- [ ] Read the actual change — `git status`, then `git diff HEAD`, and open newly created (untracked) files in full, since plain `git diff` does not show them. Never settle for the subagent's summary.
- [ ] Re-run the task's verification command yourself (e.g. `dotnet build MyApp.sln`) and confirm the expected result. A subagent's "verification passed" is a claim, not evidence.

## 2. Spec compliance (before code quality)

- [ ] Only the task's stated files were touched. Changes outside the task's file scope → stop and ask the user.
- [ ] The diff does exactly what the task described — no more, no less. No drive-by refactoring, no unrequested features, no "while I was in there" edits.
- [ ] The verification result matches the expected result written in the task.

## 3. Code quality

- [ ] Follows the codebase's existing conventions — naming, structure, error handling, style.
- [ ] No obvious bugs or unhandled edge cases (nulls, empty collections, malformed input) within the change's scope.
- [ ] No leftover debug output, commented-out code, or stray TODOs.
- [ ] Where the task calls for tests, the tests actually exercise the new behavior (run them; check they fail meaningfully if the behavior broke).

## Outcome

- **Approve:** tick the task's `implemented` and `reviewed` checkboxes in `work-plan.md`; record reusable discoveries for later dispatch prompts.
- **Problems:** dispatch a follow-up subagent with specific, actionable feedback — what is wrong, where, and what correct looks like. Then re-review from the top of this checklist.
