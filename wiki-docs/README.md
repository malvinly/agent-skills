# wiki-docs

A documentation skill for **Claude Code**, **GitHub Copilot CLI**, **OpenAI Codex CLI**, and **OpenCode**. Point it at a repository and it writes a navigable documentation to `.wiki-docs/`, then keeps it up to date as the code changes.

This skill was inspired by [OpenWiki](https://github.com/langchain-ai/openwiki/). I needed something I could use directly inside AI coding tools, without having to run it through a separate CLI. It was built for my own use case specifically, so it reflects how I like documentation to be generated and maintained.

## What it does

- **Generates** a docs wiki from scratch. It builds a `quickstart.md` entrypoint plus focused section pages by inspecting source, config, entrypoints, tests, and git history. On non-trivial repos, quickstart includes a task-routing table (change intent → page → source entrypoints → tests → validation command) and pages get grounded Mermaid diagrams where flows, lifecycles, or data models warrant them.
- **Checks its own coverage on larger repos.** Init adds a skeleton-critic pass before writing (an independent read-only subagent audits the planned structure; without subagent support it degrades to a structured self-review). With subagents it also runs a QA loop after writing: questions generated from source alone are verified against the wiki alone, and missing answers get repaired.
- **Maintains** it surgically. Later runs compare what changed since the previous run and edit only the affected pages, and an update can be a no-op when nothing relevant has changed. If drift since the last run looks structural (new services, reorganized layout), it recommends a re-init instead of patching an obsolete structure.
- **Grounds** every claim in real source, docs, or git evidence, and never invents files or behavior.
- **Makes docs discoverable** by maintaining a marker-managed reference section in the repository's top-level `AGENTS.md` and `CLAUDE.md` that points at `.wiki-docs/quickstart.md` as optional just-in-time context.
- **Respects secrets** and never reads `.env` or credential files.

## Install

All four tools read the same skill folder — only the location they look in differs. Copy the skill folder at `skills/wiki-docs/` in this repo (SKILL.md plus its `references/` directory) so that SKILL.md sits at the top of the installed folder — not just the SKILL.md file, and not this repo's outer `wiki-docs/` folder.

Per project:

    .claude/skills/wiki-docs/      # Claude Code, GitHub Copilot CLI, OpenCode
    .agents/skills/wiki-docs/      # OpenAI Codex CLI, GitHub Copilot CLI, OpenCode

Globally (copy the folder into each tool's home):

    ~/.claude/skills/wiki-docs/            # Claude Code, OpenCode
    ~/.copilot/skills/wiki-docs/           # GitHub Copilot CLI
    ~/.agents/skills/wiki-docs/            # OpenAI Codex CLI, GitHub Copilot CLI, OpenCode
    ~/.config/opencode/skills/wiki-docs/   # OpenCode

Only Claude Code and Codex need distinct homes: `.claude/skills/` covers Claude Code (also read by Copilot and OpenCode) and `.agents/skills/` covers Codex (also read by Copilot and OpenCode), so at most two copies cover all four tools.

## Usage

    /wiki-docs               # auto-detect: create if no docs yet, else update
    /wiki-docs init          # force a from-scratch generation
    /wiki-docs update        # force a surgical update
    /wiki-docs init quick    # skip the deep-init steps (critic + QA) regardless of repo size
    /wiki-docs init max      # force the deep-init steps even on a small repo

By default, init picks its own thoroughness: tiny repos get a fast standard run; larger repos get the skeleton-critic pass and QA verification loop when the harness supports subagents. `quick` and `max` override that in either direction.

The `/wiki-docs` command works in Claude Code and GitHub Copilot CLI. In Codex, invoke it from the `/skills` menu, with a `$wiki-docs` mention, or in natural language; OpenCode picks it up from the `/skills` menu or by matching the skill description.

You can also ask in natural language in any of the four tools, for example "generate the wiki docs for this repo."

## How it works

1. **Gather evidence.** It reads `.wiki-docs/.last-update.json` if present and collects git evidence (`git status`, `git log <lastHead>..HEAD`, `git diff`), falling back to timestamp-based history when the recorded commit no longer exists (squash/rebase). It falls back to filesystem inspection when the repository is not under git.
2. **Gate on drift (updates).** If the changes since the last run look structural — not just content churn — it stops and recommends a re-init rather than patching pages onto an obsolete structure.
3. **Plan.** It decides the page list and the source evidence behind each page. Deep init runs an independent critic over the planned skeleton first.
4. **Write.** It creates or updates pages under `.wiki-docs/`, keeping each concept in one canonical home, only after meeting an evidence gate (entrypoints, implementations, callers, and tests actually inspected). Deep init then verifies the result with source-blind/wiki-blind QA subagents.
5. **Record.** It writes `.wiki-docs/.last-update.json` (timestamp, git head, and completion status) only when the docs actually change, so the next run knows its baseline.

## Output

    .wiki-docs/
      quickstart.md          # entrypoint: overview, task-routing table, backlog, links to every section
      <section>/...          # focused pages: architecture, workflows, domain, ops, and more
      .last-update.json      # run metadata (timestamp, git head, status) that powers surgical updates

## OpenWiki sync baseline

This skill is periodically compared against upstream OpenWiki to adopt improvements that fit its constraints (pure agent skill, no binaries, no CI). Last synced against:

- **Version:** 0.3.0
- **Commit:** `a86d0ba` — "fix: fingerprint innermost cause and chain-walk origin tag (#589)"
- **Date:** 2026-08-04

To re-run the comparison, use the prompt in [SYNC.md](SYNC.md) and update this section afterward.
