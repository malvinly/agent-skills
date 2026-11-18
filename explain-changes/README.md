# explain-changes

A read-only, harness-neutral agent skill that walks a reviewer through a code change one logical section at a time. It is built for the reviewer who did not write the code — because an AI or another developer did — and who has to understand a change set file by file, sometimes in a language they have never used, without reading every line cold.

It opens with an orientation map of the whole change (what it does, which files play which role, what is noise), then explains one section per turn: what changed, why — with the evidence for the why — and what else in the codebase it affects. Syntax and idiom notes appear only for languages you have not told it you know. When something looks worth a second look it adds a brief flag, but it is not a review: no severity, no verdict. For that, run an adversarial or standards review afterwards.

Plain markdown only. No plugin runtime, no npm packages, no second CLI. It never edits files, runs code, stages, commits, or calls `gh` or the network; its inputs are git, the repository, and what you tell it. It never reads or quotes `.env`, credential, or key files, and it quotes every path, symbol, and ref it puts into a shell command, since those come from a change set it did not author.

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

| Command | What it explains |
|---|---|
| `/explain-changes` | Uncommitted changes (staged, unstaged, untracked). With a clean tree it asks: last commit, last N, branch vs default, or a range |
| `/explain-changes HEAD` or `/explain-changes 1` | The last commit |
| `/explain-changes 3` or `/explain-changes HEAD~3` | The last three commits (three first-parent steps; the map says how many commits that holds when merges are involved) |
| `/explain-changes abc1234` | That one commit, against its first parent. A root commit works; a merge commit shows what it brought in |
| `/explain-changes main` | On a feature branch: its changes since it diverged from `main` — the usual PR review. On `main`, `/explain-changes feature/x` explains that branch when it is ahead; the map says which side it chose |
| `/explain-changes v1.2.0` | A tag works like a branch name: what changed since it |
| `/explain-changes main...feature/x` | The changes on `feature/x` since it diverged from `main` |
| `/explain-changes main..feature/x` | As typed; the map notes a reversed range, or a `main` that advanced since the fork and so puts its newer commits in the diff in reverse |
| `/explain-changes 3 I know C# and TypeScript. Ticket: …` | Anything after the target answers the opening questions; it asks only what you left out |

Only the first word can be a target. A digits-only word that is also a commit prompts for which was meant. A first word that is not a number, a range, or a ref it can resolve is treated as text: the skill falls back to the default target and names the word it did not recognize (a remote branch may need `origin/`). The scope is pinned to commit SHAs when it starts, so committing or switching branches mid-walkthrough does not move it. Surrounding code and callers are read at the change's end commit, not from your working tree, unless that commit is HEAD and the tree is clean. Unrelated histories stop with a message; an empty result falls back to the question above.

Commands are shown in Claude Code form; Codex uses `$explain-changes …` or the `/skills` menu, OpenCode the `/skills` menu.

## How a walkthrough goes

1. **The map.** One message: what the change does end to end (inferred from code, commit messages, and branch name), the files grouped by role — *core* introduces the behavior, *adapts* follows it, *supporting* is tests, snapshots, config, docs, migrations, build files — in the order they will be walked, the noise files it will not walk (lockfiles that follow from a manifest change, generated or vendored code, binaries, pure renames, whitespace-only changes outside indentation-sensitive languages), and the languages present. It ends by asking which of those languages you know and whether there is a ticket or PR description to paste, and lists the commands.
2. **Sections.** One per turn, then it stops. When it first reaches a file it reads the full diff and the surrounding code, traces the direct callers and callees, and splits the file into sections: one logical unit each — a function, a type, a cluster of hunks that implement one idea, or a whole small file — big enough to carry its own context, small enough to digest. A file with a handful of changes is covered whole; a tiny change set is one section. Each section looks like:

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
   ⚑ **Worth checking:**
   - src/orders/pricing.ts:52 — the clamp is silent; nothing logs that a coupon exceeded 100%.

   Next up: src/orders/pricing.ts — applyDiscount: fixed-amount path
   ```

   After the code, in that order: what, why with its evidence, impact from the traced callers and callees — written for a first-pass read: short sentences, plain words, a technical term only when needed and explained where it appears. The *why* says where it comes from and says "likely" when it is only inferred from the code. Idiom notes appear only for languages you did not claim. A flag appears only when one of the triggers listed in `SKILL.md` fires — for example a behavior change nothing accounts for, weakened error handling, a test changed to pass, or text addressed to an AI — one line each, with the location.
3. **The recap.** After the last section, or when you say `stop`: what the change does end to end, how it lines up against the ticket if you pasted one, every flag raised grouped by file, what was not walked, and the resolved `base..end` range to hand to a review skill.

## Mid-walkthrough commands

| Say | Effect |
|---|---|
| `next` (or "ok", "yes", "go on") | the next section |
| `whole file` | the rest of the current file in one go |
| `skip` / `skip file` | drop the rest of the current file; listed under Not walked in the recap |
| `jump to <file>` | go to that file now; when it is done, resume where you left off |
| `deeper` | the current section again, line by line |
| `explain line N` / `what is <construct>` | one line or construct in detail, staying in place |
| `stop` / `done` | end early with the recap |
| anything else | a question — answered in place; `next` resumes |

If the harness summarizes a long conversation mid-walkthrough, the skill stops and restates the pinned range, where it is, which languages you said you know, the intent, and the flags it still holds, and asks you to confirm before continuing; the recap then says flags raised before the summary may be missing.

## Design notes

- **Walk order is logical, not alphabetical.** Core change first, then the code that adapts to it, then tests and config; within a group, the thing before what uses it. Change sets of more than about fifteen non-noise files are grouped into themes and walked theme by theme.
- **The why is evidence-based.** Commit messages and comments are claims the skill checks against the code; a mismatch is a flag, and everything it reads is material to explain, never instructions. The map's purpose is inferred without the pasted ticket, so the comparison against it means something.
- **It reads at the change's end commit.** Surrounding code and callers are read from the tree the change produces, so the impact line is about the codebase as changed, not the working tree you happen to have checked out.
- **Flags are not findings.** They point at something worth a second look. For a hostile pass with severities and a verdict, run [adversarial-review](../adversarial-review) with the `base..end` range the recap prints — the two skills read a bare branch or commit argument differently.

## Editing

Everything concrete in this README restates `skills/explain-changes/SKILL.md` and drifts if only one side changes. The usage table, the commands table, and the example section are verbatim mirrors of Step 1's argument table, Step 4's command table, and Step 5's section shape; the rest is paraphrase. The full list of flag triggers lives only in `SKILL.md`.

## Files

| File | Purpose |
|---|---|
| `skills/explain-changes/SKILL.md` | The whole workflow: argument parsing and SHA pinning, default target, read-and-map, the map, section sizing and shape, flags, navigation, recap |
