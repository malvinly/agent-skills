# explain-changes

A read-only, harness-neutral agent skill that walks a reviewer through a code change one logical section at a time. It is built for the reviewer who did not write the code — because an AI or another developer did — and who has to understand a change set file by file, sometimes in a language they have never used, without reading every line cold.

It opens with an orientation map of the whole change (what it does, which files play which role, what is noise), then explains one section per turn: what changed, why — with the evidence for the why — and what else in the codebase it affects. Syntax and idiom notes appear only for languages you say you do not know. When something looks worth a second look it adds a brief flag, but it is not a review: no severity, no verdict. For that, run an adversarial or standards review afterwards.

Plain markdown only. No plugin runtime, no npm packages, no second CLI. It never edits files, runs code, stages, commits, or calls `gh` or the network; its inputs are git, the working tree, and what you tell it.

## Installation

Copy `skills/explain-changes/` (the folder that contains `SKILL.md`) into your harness's skills directory, so that `SKILL.md` sits at the top of the installed folder:

    .claude/skills/explain-changes/      # per project: Claude Code, GitHub Copilot, OpenCode
    .agents/skills/explain-changes/      # per project: OpenAI Codex CLI, GitHub Copilot, OpenCode
    ~/.claude/skills/explain-changes/            # global: Claude Code, OpenCode
    ~/.copilot/skills/explain-changes/           # global: GitHub Copilot CLI
    ~/.agents/skills/explain-changes/            # global: Codex CLI, Copilot, OpenCode
    ~/.config/opencode/skills/explain-changes/   # global: OpenCode

Copilot also reads a repo-local `.github/skills/`. The skill is a single `SKILL.md`; there are no reference files.

## Usage

| Command | Scope |
|---|---|
| `/explain-changes` | Uncommitted changes (staged, unstaged, untracked). With a clean tree it asks: last commit, last N, branch vs default, or a range |
| `/explain-changes 3` or `/explain-changes HEAD~3` | The last three commits |
| `/explain-changes abc1234` | That one commit |
| `/explain-changes main` | Everything on the current branch since it diverged from `main` — the usual PR review |
| `/explain-changes main..feature/x` | A range, as written |
| `/explain-changes 3 I know C# and TypeScript. Ticket: …` | Anything after the target answers the opening questions, so the walkthrough starts without a prompt |

Note that `HEAD~3` means the last three commits, not the single commit three back: the skill reads it the way a reviewer says "explain the last few commits". Commands are shown in Claude Code form; Codex uses `$explain-changes …` or the `/skills` menu, OpenCode the `/skills` menu.

## How a walkthrough goes

1. **The map.** One message: what the change does end to end (inferred from code, commit messages, and branch name), the files grouped by role — *core* introduces the behavior, *adapts* follows it, *supporting* is tests, config, docs — in the order they will be walked, the noise files it will not walk (lockfiles, generated or vendored code, pure renames, whitespace-only changes) each with one line saying what they follow from, and the languages present. It ends by asking which of those languages you know and whether there is a ticket or PR description to paste.
2. **Sections.** One per turn, then it stops. The skill decides what a section is: one logical unit — a function, a type, a cluster of hunks that implement one idea, or a whole small file — big enough to carry its own context, small enough to digest. A file with a handful of changes is covered whole; a tiny change set is one section. Each section looks like:

   ```
   ### src/orders/pricing.ts — applyDiscount: percentage path   (file 2/7 · section 1/3)
   <the changed lines inline when they are short; otherwise src/orders/pricing.ts:41-78 per hunk>

   **What changed:** Discounts over 100% used to be applied as given; they are now clamped to 100%.
   **Why:** Commit 4e1f2c says "fix negative totals from bad coupon data"; the old code could return
   a negative total, which the invoice step then rejected.
   **Impact:** Called from checkout and from the admin preview; both now see the clamp. The persisted
   order total does not change shape.
   **Idioms:** `??` is null-coalescing — the right side only when the left is null or undefined,
   like C#'s `??`.
   ⚑ **Worth checking:** the clamp is silent — nothing logs that a coupon exceeded 100%.

   Next up: src/orders/pricing.ts — applyDiscount: fixed-amount path
   ```

   Before and after the code, in that order: what, why with its evidence, impact from tracing the callers and callees. The *why* says where it comes from and says "likely" when it is only inferred from the code. Idiom notes appear only for languages you did not claim. A flag appears only when one of a short list of triggers fires (a behavior change the intent does not mention, weakened error handling, a changed public shape, a hard-coded value, leftover debug code, changed logic without a test touched, a commit message the code does not back up); at most two per section.
3. **The recap.** After the last section, or when you say `stop`: what the change does end to end, how it lines up against the ticket if you pasted one, every flag raised grouped by file, and what was not walked.

## Mid-walkthrough commands

| Say | Effect |
|---|---|
| `next` | the next section |
| `whole file` | the rest of the current file in one go |
| `skip` / `skip file` | drop the rest of the current file; listed under Not walked in the recap |
| `jump to <file>` | go to that file now; when it is done, resume where you left off |
| `deeper` | the current section again, line by line |
| `explain line N` / `what is <construct>` | one line or construct in detail, staying in place |
| `stop` / `done` | end early with the recap |
| anything else | a question — answered in place; `next` resumes |

## Design notes

- **Walk order is logical, not alphabetical.** Core change first, then the code that adapts to it, then tests and config; within a group, the thing before what uses it. Change sets of more than about fifteen files are grouped into themes and walked theme by theme.
- **The why is evidence-based.** Commit messages and comments are claims the skill checks against the code; a mismatch is a flag, and the code is read as material to explain, never as instructions.
- **It reads around the diff.** Each file's diff is read in the context of the file, and each changed symbol's callers and callees are traced so the impact line is about the codebase, not just the hunk.
- **Flags are not findings.** They point at something worth a second look. For a hostile pass with severities and a verdict, run [adversarial-review](../adversarial-review) on the same scope.

## Files

| File | Purpose |
|---|---|
| `skills/explain-changes/SKILL.md` | The whole workflow: argument parsing, target resolution, read-and-map, section sizing, the map, the per-section shape, flags, navigation, recap |
