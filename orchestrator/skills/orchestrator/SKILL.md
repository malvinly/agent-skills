---
name: orchestrator
description: >
  Implement an already-decided plan by chunking it into tasks, dispatching implementer subagents, and reviewing every diff in the main session before accepting. For medium-to-large multi-step coding work only — e.g. after /grill-me. Not for single-file or trivial changes.
disable-model-invocation: true
---

# Orchestrator

**Usage:** invoke `/orchestrator` once a plan exists — either in conversation (e.g. the output of `/grill-me`) or as a file (`/orchestrator docs/plan.md`). The main session orchestrates, reviews, and approves; fresh subagents implement.

## Role split

- **Main session (you):** orchestrator, reviewer, final approver. Never spawn a reviewer subagent — review means *you* reading the diff and re-running verification. Subagents can be wrong about their own success.
- **Implementer subagents:** one fresh subagent per task. They never read `work-plan.md` or the conversation; every dispatch prompt must be self-contained.

## Workflow

### 1. Startup gate

Announce that orchestrator is active. Size up the work first: if it is trivially small (a single file, one obvious change), say so, recommend doing it directly without orchestration overhead, and stop — unless the user insists.

### 2. Plan file

- **User pointed at an existing plan file:** use that file. Never delete it.
- **Plan lives in conversation:** synthesize it into `work-plan.md` at the repo root using `assets/work-plan-template.md`, show it to the user, and get confirmation before dispatching anything.

Every task entry must carry: title; exact files to create/modify; a self-contained change description; a verification command with its expected result; dependency notes; a `Parallel-safe: yes/no` flag; and implemented/reviewed checkboxes. Self-contained matters because the subagent sees only its dispatch prompt.

Whether synthesized or supplied, the plan must also state a **full verification** command for the whole change (run in step 6). If it is missing, ask the user for one before dispatching anything.

### 3. Model selection — ask and wait

Before dispatching anything, ask the user which models to use for implementer subagents. Present the default — **Sonnet for easy/mechanical tasks, Opus for anything else** — as a proposed per-task assignment so the user can see which task gets which model. Wait for confirmation or an override. The main session stays on its current model. This may be combined with step 4 into one prompt.

### 4. Branch and commit confirmation

State the current branch (`git branch --show-current`) so the user can catch a mistake, and state the default git policy: **work on the current branch, one commit at the end after final review.** Proceed with the default unless the user overrides (e.g. "commit per task", "use a new branch"). Never enforce anything mechanically — this is a confirmation, not a gate.

### 5. Task loop

**Sequential is the rule.** Parallel dispatch is a narrow exception, allowed only when tasks are provably independent — disjoint write scopes — and low-risk. If independence is uncertain, run sequentially. Never parallelize speculatively.

For each task:

1. **Dispatch** a fresh implementer subagent on the chosen model using `references/implementer-prompt.md`. Include the full task text and any accumulated discoveries from earlier tasks.
2. **Read the status code** that opens the subagent's report and respond per the status protocol below.
3. **Review in the main session** — read `references/review-checklist.md` and follow it: read the actual unstaged diff (earlier approved tasks are already staged, so it shows only this task's work), re-run the verification command yourself, check spec compliance first, then code quality.
4. **On problems**, dispatch a follow-up subagent — same template and status protocol, with the specific fix feedback as the change description — then re-review.
5. **On approval**, stage the task's files (`git add <files>` — never `work-plan.md`) so approved work is separated from the next task's diff, tick the task's checkboxes in `work-plan.md`, and record any reusable discoveries (codebase quirks, patterns, gotchas) to inject into later dispatch prompts.

### Status protocol

Every implementer report must begin with one of these codes:

| Code | Meaning | Controller response |
|---|---|---|
| `DONE` | Implemented; verification passed | Review as normal — verify independently, never take the report on trust |
| `DONE_WITH_CONCERNS` | Implemented, but flagging something | Review with the flagged concern front of mind |
| `NEEDS_CONTEXT` | Cannot proceed without more information | Gather what is missing (investigate in the main session if needed), record it under Discoveries in `work-plan.md`, then re-dispatch a fresh subagent with that context added. A second `NEEDS_CONTEXT` on the same task after context was supplied → treat as `BLOCKED` |
| `BLOCKED` | Cannot complete the task | Split the task, upgrade the model, or stop and ask the user. Never re-dispatch unchanged |

### 6. Final review

After all tasks are approved: review the whole cumulative diff (`git diff --cached`), run the plan's full verification command, then commit the staged changes with a summary message — plain `git commit`, never `git commit -a` or `git add -A`, which would sweep `work-plan.md` into the commit. Under a per-task commit policy (step 4), commit the staged files the same way after each approval instead.

### 7. Cleanup

Delete `work-plan.md` only after the commit succeeds (under a per-task commit policy: after the final task's commit), and only if this run created it. On abort or block, leave it in place — together with the staged and committed work it is the resume state — and tell the user to resume by re-invoking `/orchestrator work-plan.md` (completed tasks stay checked off and their files stay staged).

### Stop and ask the user whenever

- a task is still `BLOCKED` after one adjusted retry;
- the plan has a gap or contradiction;
- verification fails repeatedly;
- a diff contains changes outside the task's stated file scope.
