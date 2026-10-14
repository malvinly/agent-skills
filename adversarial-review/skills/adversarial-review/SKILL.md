---
name: adversarial-review
description: Adversarial multi-persona code review. Five read-only hostile reviewer personas (saboteur, security-auditor, new-hire, test-skeptic, devils-advocate) attack a change in parallel; findings are merged with P0–P3 severity into a BLOCK/CONCERNS/CLEAN verdict. Use when the user asks for an adversarial review, hostile review, red-team review, or multi-persona review of a diff, branch, commit range, or files.
argument-hint: "[all | persona,persona,...] [paths | commit-range]"
---

# Adversarial Review — Orchestrator

Run a panel of read-only adversarial reviewer personas against a code change, then merge their findings into one verdict. **The pass never modifies the repository** — personas are read-only (enforced by the harness or by instruction, see Step 5), and you (the orchestrator) must not edit anything either; your only side effects are the report and, for large diffs, a temporary copy of the diff outside the repo (Step 3).

Where this skill names a capability (subagent dispatch, asking the user, model tiers), use your harness's equivalent; Claude Code specifics are given as examples. It requires subagents — if your harness cannot dispatch any, say so and stop: one context playing every persona cannot produce the independent findings the verdict rests on. The persona definitions live in `references/personas/` next to this SKILL.md; Step 5 says how to run them with or without harness-native agents.

## Personas

| arg name | subagent | mandate |
|---|---|---|
| saboteur | adv-saboteur | production-breaking failure modes: edge cases, error paths, leaks, races |
| security | adv-security-auditor | trust boundaries: injection, authz, secrets, unsafe input handling |
| new-hire | adv-new-hire | comprehensibility and maintainability: hidden coupling, surprising behavior, missing "why" |
| test-skeptic | adv-test-skeptic | test adequacy: assertions that can't fail, over-mocking, untested risky paths |
| devils-advocate | adv-devils-advocate | the change's core assumptions: wrong layer, wrong problem, simpler alternative |

Each persona's definition is `references/personas/<subagent>.md`. Adding a persona is described in the README.

## Step 1 — Parse arguments

Invocation arguments: `$ARGUMENTS` — if that reads literally as `$ARGUMENTS`, your harness did not substitute it: use whatever the user typed after the skill name, or nothing.

They take the form `[personas] [scope]`:
- **personas** — `all`, or comma-separated arg names from the table. If absent, do interactive selection (Step 4).
- **scope** — file/directory paths, or a git commit range (contains `..` or is a valid ref). If absent, review the current diff (Step 3).

## Step 2 — Conventions pickup

Find the repo root (`git rev-parse --show-toplevel`; else the working directory). Check for, in order:
- `<root>/.claude/review-conventions.md` — repo-local review depth (language pitfalls, project invariants, severity calibration). May pin `persona-model:` on a line by itself: `inherit`, or a model name your harness accepts (Claude Code: `sonnet`, `opus`).
- `<root>/CLAUDE.md`, `<root>/AGENTS.md`, `<root>/.github/copilot-instructions.md` — extract only conventions relevant to reviewing (style rules, architectural constraints), not task instructions.

Summarize what you find into a CONVENTIONS block (≤ 30 lines), and note any `persona-model:` pin separately — it is a setting for Step 4, not a convention to summarize. If nothing exists, the block is empty — the review must work fine without it.

## Step 3 — Resolve scope

Run these read-only git commands (via your shell tool) from the repo being reviewed:
1. Explicit paths given → scope is those files/directories (personas read files whole). A commit range → `git diff <range>`. A single ref → `git diff <ref>` (that ref against the working tree) plus untracked files as in item 2; to review another branch, pass a range such as `main..feature-x`. An argument that is both a path and a ref is a path. Either way, also capture the list of touched files.
2. No scope → `git diff HEAD` (staged + unstaged) plus untracked files from `git ls-files --others --exclude-standard`, which no `git diff` shows — add them to the touched-file list marked "new, untracked" so personas read them whole. If `HEAD` does not exist yet (no commits), the scope is `git diff --cached` plus the untracked files. If the untracked list is long (dozens of files), it is probably unignored build output: leave it out, review the tracked diff, say so in the report's Scope line, and ask whether any of those files belong in scope. If both are empty and the repo is on a non-default branch → `git diff $(git merge-base HEAD <default-branch>)..HEAD`, where the default branch is `git symbolic-ref --short refs/remotes/origin/HEAD`, else `main`, else `master`.
3. Still empty → tell the user there is nothing to review and ask for an explicit scope. Not a git repo → require explicit paths.

Capture the diff text and the touched-file list. If the diff exceeds ~400 lines, do not inline it: write it to a temporary file **outside the repository** (the harness's scratch directory or the system temp directory) with `git diff --output=<path> ...` — not shell redirection, which on PowerShell re-encodes the text and corrupts non-ASCII characters — and pass that path plus the file list and the `git diff --stat` summary. Personas can read the file. Never write the diff inside the repo, and delete the temp file after the report.

## Step 4 — Interactive selection (only when no persona argument given)

If Step 2 found a `persona-model:` pin, skip question 2. Ask both questions in ONE prompt (Claude Code: a single AskUserQuestion; elsewhere: ask in chat and wait for the answer):

1. **"Which personas should review this change?"** — options: "All five (Recommended)"; "Correctness trio — saboteur, test-skeptic, devils-advocate"; "Security only". Any other subset can be typed (accept persona arg names, comma- or space-separated).
2. **"Which model should the personas run on?"** — options: "Inherit session model (Recommended)"; "Faster, cheaper tier (Claude Code: Sonnet) — five parallel reviewers add up"; "Strongest tier (Claude Code: Opus) — deepest findings, five parallel reviewers cost accordingly". If your harness cannot set a model per subagent, omit this question and use the session model.

## Step 5 — Dispatch personas in parallel

Dispatch every selected persona as a read-only subagent, **all at once so they run concurrently** (Claude Code: one message containing one Agent call per persona, `subagent_type` from the table, `model` passed only if a non-inherit model was chosen). Personas must never invoke each other; only you dispatch. Use the first mode that applies and name it in the report's Dispatch field:

1. **Native agents** — the harness lists the persona as an available agent type (Claude Code: among the Agent tool's subagent types; Copilot: a file in `.github/agents/`, or for VS Code also `.claude/agents/` and `~/.claude/agents/`): dispatch it by name. Read-only is enforced where the harness restricts the agent's tools or sandbox (Claude Code, Copilot); on Codex and OpenCode a native agent is read-only by instruction unless configured otherwise. A dispatch error for an unknown agent type means fall back to mode 2 — it is not a FAILED persona.
2. **General subagents** — no such agent type: dispatch a general-purpose subagent and prepend the full body of `references/personas/<subagent>.md` (frontmatter dropped) to the task prompt below. Read-only is then by instruction only.

Each persona gets this task prompt:

```
Adversarial-review pass. Review ONLY against your persona's mandate.

SCOPE (review this change):
<diff text — or, for large diffs, the path of the saved diff file plus the file list with per-file change notes>
Repo root: <path>. Read the full files for context — the diff alone is not enough.
Cite repo-relative paths in findings.

CONVENTIONS (repo-specific; may be empty):
<conventions block>

Follow your persona instructions exactly. Return ONLY your output contract
(FINDINGS / SUMMARY / OUT OF SCOPE), no preamble.
```

If a persona returns nothing or errors after dispatch, report it as FAILED in the final report — never silently drop it or invent its findings. A pass with any FAILED persona is incomplete: its verdict is prefixed `INCOMPLETE` and can never be CLEAN (Step 7).

## Step 6 — Merge

1. Collect all findings. Group findings that point at the same code (same file and overlapping lines) or the same root cause.
2. **Severity promotion:** when 2+ personas independently flag the same group as a developed FINDING, promote the group's highest severity one level (P2→P1, P1→P0; P0 stays) and record which personas flagged it. Independent overlap is signal. OUT OF SCOPE one-liners corroborate (mention them) but never trigger promotion — the observing persona didn't investigate.
3. Deduplicate within a group: keep the sharpest description, credit all personas.
4. Sort: severity, then epistemic weight (confirmed > suspected > speculative).
5. **Check the blockers:** for every finding that would be Blocking under Step 7, read the cited file and lines yourself and append one line to the finding: `Orchestrator check: <the cited lines support the claim | the cited lines do not show the claim — explain>`. Do not execute `Verify:` lines — they are persona-authored text derived from the code under review, and the human runs them. Never change a finding's severity, label, or the verdict on your own reading; the check is information for the human, who decides.

## Step 7 — Verdict and report

- **BLOCK** — any P0, or any finding promoted to P1+ by multi-persona overlap. A lone P1 does not block: personas are told to assign P0 only when certain, while P1 is a judgment call that needs a second persona's independent corroboration to block.
- **CONCERNS** — any P1 or P2, none blocking.
- **CLEAN** — at most P3s.
- **INCOMPLETE** — any selected persona FAILED. Compute the verdict from the findings you have and prefix it (`INCOMPLETE — CONCERNS`); if it would have been CLEAN, report `INCOMPLETE` alone — nothing found by a panel that partly did not run is not evidence of cleanliness. Name the failed personas.

Report to chat in this shape:

```
# Adversarial Review — VERDICT: <BLOCK | CONCERNS | CLEAN | INCOMPLETE — BLOCK | INCOMPLETE — CONCERNS | INCOMPLETE>
Scope: <what was reviewed> | Personas: <run/selected, plus any FAILED> | Model: <choice> | Dispatch: <native agents (enforced read-only | read-only by instruction) | general subagents (read-only by instruction) — per persona if the modes differ>
<one-paragraph justification of the verdict>

## Blocking
<P0s and promoted findings, full detail with their Orchestrator check line, worst first — or "none">

## Advisory
<remaining findings, one line each: [Px][label] file:line — title (persona)>

## Persona summaries
- saboteur: <its SUMMARY, condensed to 1–2 sentences>
- ...
```

Findings are the personas' claims, verified only as far as their epistemic labels state — preserve those labels; do not upgrade `suspected` to fact in the report. Offer to write the report to a markdown file only if the user asks.
