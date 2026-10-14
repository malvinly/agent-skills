---
name: orchestrator
description: >
  Implement an already-decided plan by chunking it into tasks, dispatching implementer subagents, and reviewing every diff in the main session before accepting. For medium-to-large multi-step coding work only (e.g. a plan produced by a planning or interview skill), and only when the user explicitly invokes this skill by name — never auto-trigger. Not for single-file or trivial changes.
disable-model-invocation: true
---

# Orchestrator

**Usage:** invoke `/orchestrator` once a plan exists — either in conversation (e.g. the output of a planning or interview skill) or as a file (`/orchestrator docs/plan.md`). The main session orchestrates, reviews, and approves; fresh subagents implement.

This skill is harness-neutral: it runs on any coding agent that supports the Agent Skills (`SKILL.md`) standard. Where it names a capability (subagents, model tiers, asking the user), use your harness's equivalent; Claude Code specifics are given as examples. It requires subagents — if your harness cannot dispatch any, say so and stop rather than implementing the plan yourself under this skill's name. Supporting files live in `references/` and `assets/` next to this SKILL.md; read each when the step that needs it is reached.

## Role split

- **Main session (you):** orchestrator, reviewer, final approver. Never spawn a reviewer subagent — review means *you* reading the diff and re-running verification. Subagents can be wrong about their own success.
- **Implementer subagents:** one fresh subagent per task. They start with no context — they are not given the plan file, the conversation, or earlier subagents' reports — so every dispatch prompt must be self-contained.

## Workflow

### 1. Startup gate

Announce that orchestrator is active. Size up the work first: if it is trivially small (a single file, one obvious change), say so, recommend doing it directly without orchestration overhead, and stop — unless the user insists.

### 2. Plan file

- **User pointed at an existing plan file:** use that file. Never delete it, unless it is this orchestrator's own synthesized `work-plan.md` being resumed (step 7).
- **Plan lives in conversation:** synthesize it into `work-plan.md` at the repo root using `assets/work-plan-template.md`, show it to the user, and get confirmation before dispatching anything.

Whichever it is, that file is **the plan file** for the rest of this workflow: checkbox ticks, discoveries, and gap-filling are written into it, it is never staged or committed, and it is exempt from the file-scope check. Every task must carry every field in the template, and the plan header must state a **full verification** command for the whole change (run in step 6) and the **base commit** (`git rev-parse HEAD` when the run first starts — written once and never refreshed on resume; `none` if the repo has no commits yet). Fill any gap from the conversation or ask the user before dispatching anything. Change descriptions must be self-contained because the subagent sees only its dispatch prompt.

### 3. Model selection — ask and wait

Before dispatching anything, ask the user which models to use for implementer subagents. Present the default — **a fast, cheaper tier for easy/mechanical tasks, the strongest available tier for anything else** (Claude Code: Sonnet and Opus) — as a proposed per-task assignment so the user can see which task gets which model. Wait for confirmation or an override. The main session stays on its current model. This may be combined with step 4 into one prompt. If your harness cannot set a model per subagent, say so and skip this step.

### 4. Branch and commit confirmation

State the current branch (`git branch --show-current`) so the user can catch a mistake, and state the default git policy: **work on the current branch, one commit at the end after final review.** Proceed with the default unless the user overrides (e.g. "commit per task", "use a new branch"). Never enforce anything mechanically — this is a confirmation, not a gate.

Before the first dispatch, check the tree: `git status --porcelain` should show nothing except the plan file and, on a resumed run, the staged files of tasks already ticked in it. Review isolation (step 5) and the final commit (step 6) both assume that everything staged was approved by this run or the run it resumes. If anything else is dirty, list it and ask the user to commit or stash it first. The user may waive this; then record the pre-existing dirty paths under Discoveries in the plan file, exempt them from the file-scope check, and never stage them. On Codex, note that the default sandbox keeps `.git` read-only, so staging and committing need the git directory made writable or an approval per git write.

### 5. Task loop

**Sequential is the rule.** Parallel dispatch is a narrow exception, allowed only when tasks are provably independent — disjoint write scopes — and low-risk. If independence is uncertain, run sequentially. Never parallelize speculatively. When you do dispatch a batch in parallel, wait until every task in it has reported before reviewing any of them — verification runs against the shared working tree — and, under a per-task commit policy, commit the batch together once all of it is approved.

For each task:

1. **Dispatch** a fresh implementer subagent on the chosen model (Claude Code: the Agent tool with `model` set; elsewhere: your harness's subagent feature) using `references/implementer-prompt.md`. Include the full task text and any accumulated discoveries from earlier tasks.
2. **Read the status code** that opens the subagent's report and respond per the status protocol below. A report with no status code is treated as `DONE_WITH_CONCERNS`.
3. **Review in the main session** — read `references/review-checklist.md` and follow it: read the actual unstaged diff (earlier approved tasks are already staged, so it shows only this task's work), re-run the verification command yourself, check spec compliance first, then code quality.
4. **On problems**, dispatch a follow-up subagent — same template and status protocol, with the specific fix feedback as the change description — then re-review.
5. **On approval**, stage everything you approved for the task — its listed files plus any extra change the user accepted during review, renames and deletions included — with `git add <paths>` (never the plan file), so approved work is separated from the next task's diff. Then tick the task's checkboxes in the plan file and record any reusable discoveries (codebase quirks, patterns, gotchas) to inject into later dispatch prompts.

### Status protocol

Every implementer report must begin with one of these codes:

| Code | Meaning | Controller response |
|---|---|---|
| `DONE` | Implemented; verification passed | Review as normal — verify independently, never take the report on trust |
| `DONE_WITH_CONCERNS` | Implemented, but flagging something | Review with the flagged concern front of mind |
| `NEEDS_CONTEXT` | Cannot proceed without more information | Gather what is missing (investigate in the main session if needed), record it under Discoveries in the plan file, then re-dispatch a fresh subagent with that context added. A second `NEEDS_CONTEXT` on the same task after context was supplied → treat as `BLOCKED` |
| `BLOCKED` | Cannot complete the task | Split the task, upgrade the model, or stop and ask the user. Never re-dispatch unchanged |

### 6. Final review

After all tasks are approved: review the whole cumulative diff — `git diff --cached` under the default policy, `git diff <base commit>..HEAD` under a per-task commit policy (with base `none`, diff from git's empty tree: `git diff 4b825dc642cb6eb9a060e54bf8d69288fbee4904..HEAD`) — and run the plan's full verification command. Before committing, confirm `git status --short` shows no unstaged or untracked entries other than the plan file (staged entries are the work about to be committed): anything else means the commit would not match what you just verified, so stop and resolve it. Then commit with `git commit -m "<summary>"` — never `git commit -a` or `git add -A`, which would sweep the plan file into the commit. Under a per-task commit policy, only the commit step moves: commit the staged files the same way after each task's approval; the final cumulative review and full verification still run.

### 7. Cleanup

Delete the plan file only after step 6 has completed (including, under the default policy, the commit), and only if it is `work-plan.md` at the repo root with a `Source:` header that says it was synthesized — that is how a resumed run recognizes its own file. A plan supplied at any other path is never deleted. On abort or block, leave it in place — together with the staged and committed work it is the resume state — and tell the user to resume by re-invoking `/orchestrator <plan file>` (completed tasks stay checked off and their files stay staged).

### Stop and ask the user whenever

- a task is still `BLOCKED` after one adjusted retry;
- the plan has a gap or contradiction;
- verification fails repeatedly;
- a diff contains changes outside the task's file scope (or, in a parallel batch, outside its siblings' scopes too).
