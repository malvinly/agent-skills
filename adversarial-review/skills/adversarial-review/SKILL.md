---
name: adversarial-review
description: Adversarial multi-persona code review. Five read-only hostile reviewer personas (saboteur, security-auditor, new-hire, test-skeptic, devils-advocate) attack a change in parallel; findings are merged with P0–P3 severity into a BLOCK/CONCERNS/CLEAN verdict. Use when the user asks for an adversarial review, hostile review, red-team review, or multi-persona review of a diff, branch, commit range, or files.
argument-hint: "[all | persona,persona,...] [paths | commit-range]"
---

# Adversarial Review — Orchestrator

Run a panel of read-only adversarial reviewer personas against a code change, then merge their findings into one verdict. **The pass never modifies the repository** — personas have no write or execute tools, and you (the orchestrator) must not edit anything either; your only side effect is the report.

## Personas

| arg name | subagent | mandate |
|---|---|---|
| saboteur | adv-saboteur | production-breaking failure modes: edge cases, error paths, leaks, races |
| security | adv-security-auditor | trust boundaries: injection, authz, secrets, unsafe input handling |
| new-hire | adv-new-hire | comprehensibility and maintainability: hidden coupling, surprising behavior, missing "why" |
| test-skeptic | adv-test-skeptic | test adequacy: assertions that can't fail, over-mocking, untested risky paths |
| devils-advocate | adv-devils-advocate | the change's core assumptions: wrong layer, wrong problem, simpler alternative |

To add a persona: create the agent file in `~/.claude/agents/` (copy an existing one's structure) and add a row here.

## Step 1 — Parse arguments

Invocation arguments: `$ARGUMENTS`

They take the form `[personas] [scope]`:
- **personas** — `all`, or comma-separated arg names from the table. If absent, do interactive selection (Step 2).
- **scope** — file/directory paths, or a git commit range (contains `..` or is a valid ref). If absent, review the current diff (Step 3).

## Step 2 — Interactive selection (only when no persona argument given)

Read the conventions file first (Step 4) — if it pins `persona-model:`, skip question 2. Ask ONE AskUserQuestion containing:

1. **"Which personas should review this change?"** — options: "All five (Recommended)"; "Correctness trio — saboteur, test-skeptic, devils-advocate"; "Security only". Any other subset can be typed via Other (accept persona arg names, comma- or space-separated).
2. **"Which model should the personas run on?"** — options: "Inherit session model (Recommended)"; "Sonnet — faster and cheaper, five parallel reviewers add up"; "Haiku — smoke-test speed, shallowest findings".

## Step 3 — Resolve scope

Run these read-only git commands (Bash) from the repo being reviewed:
1. Explicit paths given → scope is those files/directories (personas read files whole). A commit range → `git diff <range>` plus the list of touched files.
2. No scope → `git diff HEAD` (staged + unstaged). If empty and the repo is on a non-default branch → `git diff $(git merge-base HEAD <default-branch>)..HEAD`.
3. Still empty → tell the user there is nothing to review and ask for an explicit scope. Not a git repo → require explicit paths.

Capture the diff text and the touched-file list. If the diff exceeds ~400 lines, pass only the file list plus a one-line note of what changed per file (from `git diff --stat`).

## Step 4 — Conventions pickup

Find the repo root (`git rev-parse --show-toplevel`; else the working directory). Check for, in order:
- `<root>/.claude/review-conventions.md` — repo-local review depth (language pitfalls, project invariants, severity calibration). May pin `persona-model: <sonnet|haiku|inherit>` on a line by itself.
- `<root>/CLAUDE.md` — extract only conventions relevant to reviewing (style rules, architectural constraints), not task instructions.

Summarize what you find into a CONVENTIONS block (≤ 30 lines). If nothing exists, the block is empty — the review must work fine without it.

## Step 5 — Dispatch personas in parallel

Dispatch every selected persona as a subagent via the Agent tool — **all calls in a single message so they run concurrently**, `subagent_type` from the table. Pass `model` on each call only if a non-inherit model was chosen. Personas must never invoke each other; only you dispatch. Each persona gets this task prompt:

```
Adversarial-review pass. Review ONLY against your persona's mandate.

SCOPE (review this change):
<diff text, or file list with per-file change notes>
Repo root: <path>. Read the full files for context — the diff alone is not enough.
Cite repo-relative paths in findings.

CONVENTIONS (repo-specific; may be empty):
<conventions block>

Follow your persona instructions exactly. Return ONLY your output contract
(FINDINGS / SUMMARY / OUT OF SCOPE), no preamble.
```

If a persona returns nothing or errors, report that persona as FAILED in the final report — never silently drop it or invent its findings.

## Step 6 — Merge

1. Collect all findings. Group findings that point at the same code (same file and overlapping lines) or the same root cause.
2. **Severity promotion:** when 2+ personas independently flag the same group as a developed FINDING, promote the group's highest severity one level (P2→P1, P1→P0; P0 stays) and record which personas flagged it. Independent overlap is signal. OUT OF SCOPE one-liners corroborate (mention them) but never trigger promotion — the observing persona didn't investigate.
3. Deduplicate within a group: keep the sharpest description, credit all personas.
4. Sort: severity, then epistemic weight (confirmed > suspected > speculative).

## Step 7 — Verdict and report

- **BLOCK** — any P0, or any finding promoted to P1+ by multi-persona overlap.
- **CONCERNS** — any P1 or P2, none blocking.
- **CLEAN** — at most P3s.

Report to chat in this shape:

```
# Adversarial Review — VERDICT: <BLOCK|CONCERNS|CLEAN>
Scope: <what was reviewed> | Personas: <run/selected, plus any FAILED> | Model: <choice>
<one-paragraph justification of the verdict>

## Blocking
<P0s and promoted findings, full detail, worst first — or "none">

## Advisory
<remaining findings, one line each: [Px][label] file:line — title (persona)>

## Persona summaries
- saboteur: <its SUMMARY, condensed to 1–2 sentences>
- ...
```

Findings are the personas' claims, verified only as far as their epistemic labels state — preserve those labels; do not upgrade `suspected` to fact in the report. Offer to write the report to a markdown file only if the user asks.
