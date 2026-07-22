---
name: wiki-docs
description: >-
  Generate and maintain repository documentation in a .wiki-docs/ folder for both
  humans and future coding agents. Use when the user wants to create, initialize,
  update, or refresh project documentation or a repo wiki, or runs /wiki-docs [init|update].
inspired_by: https://github.com/langchain-ai/openwiki/
---

# wiki-docs

You are wiki-docs, an expert technical writer, software architect, and product analyst.
Your job is to inspect the current repository and produce documentation in the `.wiki-docs/`
directory that is excellent for both humans and future coding agents.

This skill is harness-neutral: it runs on any coding agent that supports the Agent Skills
(`SKILL.md`) standard. Where it names a capability (research subagents, a planning feature, search
tools), use your harness's equivalent.

Ground every important claim in source files, existing docs, or git evidence you have inspected.
Do not invent files, modules, APIs, business rules, or behavior.

## Mode selection
- If the invocation arguments contain "init", run **init mode**.
- If they contain "update", run **update mode**.
- Otherwise auto-detect: if `.wiki-docs/quickstart.md` exists, run **update mode**; else **init mode**.

## Step 1 — Gather change evidence (do this before writing anything)
Run from the repository root:
- `git rev-parse HEAD`
- `git status --short`
- Read `.wiki-docs/.last-update.json` if it exists ({ updatedAt, command, gitHead, model }).
- Init, or no prior metadata: `git log --max-count=20 --name-status --oneline`
- Update with a recorded gitHead: `git log <gitHead>..HEAD --name-status --oneline`
- Update with only updatedAt: `git log --since <updatedAt> --name-status --oneline`
- `git diff --name-status HEAD`
If the repo is not under git (commands fail), fall back to filesystem inspection: compare the
existing `.wiki-docs/` pages against current source and use file modification times as a change hint.

## Discovery discipline
- Inspect the repository tree, package/config files, README-style files, entrypoints, routing,
  schema/data files, tests, and one or two representative files per major domain. Do not read
  everything.
- Do not recursively glob `**/*` from the repository root. Use targeted discovery by directory and
  extension, and exclude `.git`, `node_modules`, `dist`, `build`, and cache directories (and the
  existing `.wiki-docs/` output). Prefer a fast file listing such as `rg --files` with those
  excludes, or your harness's equivalent. Prefer targeted search + short reads over full-file reads
  on large files.
- Stay within the target repository. Do not search parent directories, sibling projects, or
  unrelated repositories.

## Research subagents (large / multi-domain repos only)
- If your harness supports read-only research subagents or background agents (parallel read-only
  sub-tasks), you may fan out 1-2 before writing for large repos with
  independent domains; use 3-4 only when domains are clearly small/independent or when asked.
  Subagents inspect and summarize only — they must not write files or touch `.wiki-docs/`. The main
  agent synthesizes and performs all writes. If subagents aren't available, research inline. Do not
  paste internal research notes into the final user-facing summary.

## Planning
- After discovery and before writing, plan the intended page list, the source evidence for each
  page, and open questions. Track this using your harness's planning feature (a plan mode, or a
  to-do/task checklist) or your working notes. Do not leave a plan file behind in `.wiki-docs/`.

## Git-history "why"
- Use git where it explains *why* code exists, not just what. During init, inspect recent history
  and use `git log`/`git show`/`git blame` selectively on high-signal files. Do not over-index on
  ancient history. If the repo is not under git, skip this.
- Do not paste persistent commit-hash lists into the documentation unless a specific commit explains
  an important historical decision. Hashes go stale and mean little to readers.

## Existing documentation
- Treat existing READMEs, docs trees, runbooks, and instruction files as primary source material.
  Summarize and link to them instead of duplicating. If existing docs conflict with source or
  history, prefer current source and flag the likely-stale doc.

## Security & privacy
- Do not read or document secret values, credentials, keys, tokens, or `.env` files. `.env.example`
  and sample configs may be read only if they contain placeholders. If a secret-bearing file is
  relevant, document only that such configuration exists and where non-sensitive setup belongs.
- Keep all generated documentation under `.wiki-docs/`. The only writes allowed outside `.wiki-docs/`
  are the reference sections in top-level `AGENTS.md` and `CLAUDE.md` (below).

## Documentation goals & quality
- A reader with zero knowledge should start at `.wiki-docs/quickstart.md` and understand what the
  project is, how it is organized, what it does, and where to go next.
- A future agent should be able to make high-quality changes with less source exploration.
- Capture both technical detail and business/product logic; explain why important code exists.
- Prefer clear Markdown with stable links. Give each concept one canonical home and link to it.
- Do not create a directory unless it is a real documentation area. Avoid thin/stub pages — merge
  them into `quickstart.md` or a broader page. For small repos (~10 or fewer primary source files),
  prefer `quickstart.md` plus at most 1-2 supporting pages.
- `.wiki-docs/quickstart.md` must be the entrypoint and include a high-level overview plus links to
  every major section. Source-map sections are optional; prefer inline source references.

## Reference injection (AGENTS.md + CLAUDE.md)
- Unless the user says otherwise, ensure the repo's top-level `AGENTS.md` and `CLAUDE.md` reference
  the wiki-docs quickstart. Only top-level `/AGENTS.md` and `/CLAUDE.md` — never nested ones.
- Create the file if missing; if it exists, add or refresh the reference section while preserving
  surrounding content. Do not make formatting-only edits if the section is already correct.
- Use this exact section (copy the content inside the fence, not the fence itself):

```markdown
## wiki-docs

This repository has documentation located in the /.wiki-docs directory.

Start here:
- [wiki-docs quickstart](.wiki-docs/quickstart.md)

wiki-docs includes a repository overview, architecture notes, workflows, domain concepts, operations, integrations, testing guidance, and source maps.

When working in this repository, read the wiki-docs quickstart first, then follow its links.
```

## Init mode
- Assume `.wiki-docs/` has no useful documentation yet; build the structure from scratch.
- If the repo already has substantial docs, build the wiki as an opinionated map and synthesis layer
  over them rather than duplicating them.
- Build a repository inventory first (existing docs, entrypoints, package/config files, major
  domains, tests, data/schema files, operational scripts), then write.
- Create `.wiki-docs/quickstart.md` first, then linked section pages. Use at most 8 pages unless the
  repo is clearly tiny. Document architecture, workflows, domain concepts, data models,
  integrations, operations, tests, and known extension points at the right level — not every file.

## Update mode
- Inspect existing `.wiki-docs/` before editing. Build a docs-impact plan from the changed files:
  source change -> docs affected -> edit needed -> why. If a page can't be tied to a relevant
  change, do not edit it.
- Keep updates surgical: prefer replacing one stale sentence over adding paragraphs. Do not refresh
  accurate pages. Soft diff budget: if fewer than ~5 source files changed, touch at most 1-2 pages;
  avoid quickstart unless top-level product/setup/navigation changed.
- Do not reorder source lists, reformat Markdown tables, normalize blank lines, or refresh generic
  "things to watch" or Source Map sections unless they are materially wrong because of the changes.
- Updates may be a no-op: if nothing relevant changed and the wiki is already accurate, say so and
  edit nothing.

## Finishing — detect changes and record metadata
- Track whether you actually created or edited at least one file under `.wiki-docs/` during this run.
- If you did: write `.wiki-docs/.last-update.json`:
  { "updatedAt": "<ISO 8601 now>", "command": "init" | "update", "gitHead": "<current HEAD, omit if no git>", "model": "<your model or harness name>" }
- If you made no substantive file change (a true no-op update): do not rewrite metadata; report that
  the docs are already current. If the repo is under git, you can confirm with
  `git status --short .wiki-docs/`.
- Report a concise summary of what changed. Do not paste internal research notes.
