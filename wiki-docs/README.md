# wiki-docs

A documentation skill for **Claude Code** and **GitHub Copilot CLI**. Point it at a repository and it writes a navigable documentation to `.wiki-docs/`, then keeps it up to date as the code changes.

This skill was inspired by [OpenWiki](https://github.com/langchain-ai/openwiki/). I needed something I could use directly inside AI coding tools, without having to run it through a separate CLI. It was built for my own use case specifically, so it reflects how I like documentation to be generated and maintained.

## What it does

- **Generates** a docs wiki from scratch. It builds a `quickstart.md` entrypoint plus focused section pages by inspecting source, config, entrypoints, tests, and git history.
- **Maintains** it surgically. Later runs compare what changed since the previous run and edit only the affected pages, and an update can be a no-op when nothing relevant has changed.
- **Grounds** every claim in real source, docs, or git evidence, and never invents files or behavior.
- **Makes docs discoverable** by adding a reference section to the repository's top-level `AGENTS.md` and `CLAUDE.md` that points at `.wiki-docs/quickstart.md`.
- **Respects secrets** and never reads `.env` or credential files.

## Install

One `SKILL.md` works for both tools.

Per project (one copy, read by both tools):

    .claude/skills/wiki-docs/SKILL.md

Globally (copy the folder into each tool's home):

    ~/.claude/skills/wiki-docs/     # Claude Code
    ~/.copilot/skills/wiki-docs/    # GitHub Copilot CLI

## Usage

    /wiki-docs           # auto-detect: create if no docs yet, else update
    /wiki-docs init      # force a from-scratch generation
    /wiki-docs update    # force a surgical update

You can also ask in natural language, for example "generate the wiki docs for this repo."

## How it works

1. **Gather evidence.** It reads `.wiki-docs/.last-update.json` if present and collects git evidence (`git status`, `git log <lastHead>..HEAD`, `git diff`). It falls back to filesystem inspection when the repository is not under git.
2. **Plan.** It decides the page list and the source evidence behind each page.
3. **Write.** It creates or updates pages under `.wiki-docs/`, keeping each concept in one canonical home.
4. **Record.** It writes `.wiki-docs/.last-update.json` (timestamp and git head) only when the docs actually change, so the next run knows its baseline.

## Output

    .wiki-docs/
      quickstart.md          # entrypoint: overview and links to every section
      <section>/...          # focused pages: architecture, workflows, domain, ops, and more
      .last-update.json      # run metadata (timestamp and git head) that powers surgical updates
