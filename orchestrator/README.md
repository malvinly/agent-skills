# orchestrator

A lightweight, harness-neutral agent skill that implements an already-decided plan by dispatching implementer subagents while the main session stays in charge as orchestrator, reviewer, and final approver. It needs a harness that can dispatch subagents; without one it says so and stops.

It exists for one situation: you've refined an idea into a solid multi-step plan (e.g. via a planning or interview skill) and now want it executed with discipline — each task done by a fresh subagent with a self-contained prompt, every diff independently reviewed in the main session, and one clean commit at the end.

## Installation

Copy `skills/orchestrator/` (the folder that contains `SKILL.md`) into your harness's skills directory — `~/.claude/skills/` for Claude Code, `~/.copilot/skills/` for GitHub Copilot, or a repo-local `.github/skills/` or `.claude/skills/` to share it with teammates:

```
~/.claude/skills/
  orchestrator/
    SKILL.md
    references/
      implementer-prompt.md
      review-checklist.md
    assets/
      work-plan-template.md
```

`/orchestrator` then appears as an invokable skill. In Claude Code it is manual-invoke only (`disable-model-invocation: true`); other harnesses ignore that flag, so the description itself also tells the model never to auto-trigger.

## Usage

Invoke it once a plan exists:

- **Plan in conversation** (the typical flow after a planning or interview skill): just run `/orchestrator`. The skill synthesizes the plan into `work-plan.md` at the repo root and shows it to you for confirmation before doing anything.
- **Plan already on disk:** run `/orchestrator path/to/plan.md`. Your file is used as-is and never deleted.

Before any work is dispatched, the skill will:

1. **Gate on size** — if the work is trivial (single file, one obvious change), it recommends skipping orchestration and stops unless you insist.
2. **Ask which models to use** for implementer subagents, proposing a per-task split of the default: a fast, cheaper tier for easy/mechanical tasks, the strongest available tier for anything else (Sonnet and Opus on Claude Code). Skipped if the harness cannot set a model per subagent.
3. **Confirm git policy** — it states your current branch and the default of one commit at the end. Override here if you want per-task commits or a new branch.

Then it works through the plan sequentially (parallel only when tasks are provably independent), one fresh subagent per task. Each subagent reports back with a status code (`DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, `BLOCKED`), and the main session reviews every result itself: reading the real diff, re-running the verification command, checking spec compliance before code quality. Nothing is taken on the subagent's word, and no reviewer subagents are ever spawned.

Approved work is staged after each review, so every task is reviewed as an isolated diff and only staged files are ever committed (`work-plan.md` never is). After all tasks pass review, it reviews the whole cumulative diff, runs the plan's full verification command, makes the single commit, and — if it created `work-plan.md` — deletes it.

## Resuming an interrupted run

If a run aborts or blocks, `work-plan.md` is left in place with completed tasks checked off; together with the staged and committed work it is the resume state. Re-invoke with `/orchestrator work-plan.md` to pick up where it stopped.

## When not to use it

Single-file or trivial changes — the orchestration overhead isn't worth it, and the skill will tell you so. Just ask the agent to make the change directly.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | The workflow: startup gate, plan persistence, model/branch confirmation, task loop, status protocol, final review, cleanup |
| `references/implementer-prompt.md` | Dispatch template for implementer subagents — self-contained task prompt plus status-reporting rules |
| `references/review-checklist.md` | Main-session review checklist: independent verification, spec compliance, then code quality |
| `assets/work-plan-template.md` | Schema for `work-plan.md`, with a filled-in C# example (CSV export feature) |

The skill is language-agnostic; the examples use C#/.NET (`dotnet build` / `dotnet test`) but nothing in the logic assumes it.
