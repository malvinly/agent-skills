# orchestrator

A lightweight, harness-neutral agent skill that implements an already-decided plan by dispatching implementer subagents while the main session stays in charge as orchestrator, reviewer, and final approver. It needs a harness that can dispatch subagents; without one it says so and stops.

It exists for one situation: you've refined an idea into a solid multi-step plan (e.g. via a planning or interview skill) and now want it executed with discipline — each task done by a fresh subagent with a self-contained prompt, every diff independently reviewed in the main session, and, by default, one clean commit at the end.

## Installation

Copy `skills/orchestrator/` (the folder that contains `SKILL.md`) into your harness's skills directory, so that `SKILL.md` sits at the top of the installed folder:

    .claude/skills/orchestrator/      # per project: Claude Code, GitHub Copilot, OpenCode
    .agents/skills/orchestrator/      # per project: OpenAI Codex CLI, GitHub Copilot, OpenCode
    ~/.claude/skills/orchestrator/            # global: Claude Code, OpenCode
    ~/.copilot/skills/orchestrator/           # global: GitHub Copilot CLI
    ~/.agents/skills/orchestrator/            # global: Codex CLI, Copilot, OpenCode
    ~/.config/opencode/skills/orchestrator/   # global: OpenCode

Copilot also reads a repo-local `.github/skills/`. The installed folder:

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

Invoke it as `/orchestrator` in Claude Code and Copilot, as `$orchestrator` or from the `/skills` menu in Codex, and from the `/skills` menu in OpenCode. Claude Code and VS Code Copilot honor `disable-model-invocation: true` and make it manual-invoke only; harnesses that ignore the flag rely on the description, which also tells the model never to auto-trigger. OpenCode can expose a `/orchestrator` command if you add `metadata: { opencode/slash: "true" }` to the frontmatter.

## Usage

Invoke it once a plan exists:

- **Plan in conversation** (the typical flow after a planning or interview skill): just run `/orchestrator`. The skill synthesizes the plan into `work-plan.md` at the repo root and shows it to you for confirmation before doing anything.
- **Plan already on disk:** run `/orchestrator path/to/plan.md`. Your file becomes the plan file: progress checkboxes and discoveries are written into it, and it is never staged, committed, or deleted.

Before any work is dispatched, the skill will:

1. **Gate on size** — if the work is trivial (single file, one obvious change), it recommends skipping orchestration and stops unless you insist.
2. **Ask which models to use** for implementer subagents, proposing a per-task split of the default: a fast, cheaper tier for easy/mechanical tasks, the strongest available tier for anything else (Sonnet and Opus on Claude Code). Skipped if the harness cannot set a model per subagent.
3. **Confirm git policy** — it states your current branch and the default of one commit at the end, and requires a clean working tree (only the plan file may be dirty). Override here if you want per-task commits or a new branch. On Codex, the default sandbox keeps `.git` read-only, so git writes need the git directory made writable or an approval each.

Then it works through the plan sequentially (parallel only when tasks are provably independent), one fresh subagent per task. Each subagent reports back with a status code (`DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, `BLOCKED`), and the main session reviews every result itself: reading the real diff, re-running the verification command, checking spec compliance before code quality. Nothing is taken on the subagent's word, and no reviewer subagents are ever spawned.

Approved work is staged after each review, so every task is reviewed as an isolated diff and only staged files are ever committed (the plan file never is). After all tasks pass review, it reviews the whole cumulative diff, runs the plan's full verification command, makes the single commit, and — if it created `work-plan.md` — deletes it.

## Resuming an interrupted run

If a run aborts or blocks, the plan file is left in place with completed tasks checked off; together with the staged and committed work it is the resume state. Re-invoke with `/orchestrator work-plan.md` (or your own plan's path) to pick up where it stopped.

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
