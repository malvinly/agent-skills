# SYNC.md — re-compare wiki-docs against upstream OpenWiki

This is a reusable prompt. Paste it (or point your agent at this file) to re-run the upstream comparison that keeps this skill current.

---

## Prompt

Compare my wiki-docs skill against the current OpenWiki source and determine what is worth adopting, adapting, or ignoring. Research first and report back before changing anything.

All paths below are relative to the directory containing this SYNC.md file: the skill lives at `skills/wiki-docs/` (SKILL.md plus its `references/` files) and the baseline lives in the `README.md` next to this file. (In the agent-skills monorepo that directory is the `wiki-docs/` subfolder, not the repo root.)

### Standing constraints

These are permanent; do not re-litigate them:

- I cannot run unauthorized binaries or CI at work. This must stay a **pure agent skill** — a skill folder invocable via `/wiki-docs` in Claude Code, and equivalently in GitHub Copilot CLI, OpenAI Codex CLI, and OpenCode. Supporting markdown/reference files inside the skill folder are fine; executables, scheduled workflows, and separate CLIs are not.
- The skill must remain **harness-neutral** and lightweight. Only behaviors expressible as instructions qualify. OpenWiki's deterministic post-run code is permanently out of scope: link and mermaid validators, index.md generation, front-matter repair, translation middleware, telemetry, connectors/ingestion, the visualizer, chat mode, and `personal` mode. Skip churn in those areas without deep reading — but do adopt *instruction-level* ideas they imply (e.g., a validator's rules can become a self-check instruction).

### Procedure

1. Read the **OpenWiki sync baseline** section of the `README.md` next to this file for the version and commit of the last comparison.
2. Ensure a current OpenWiki clone is available locally — ask me where my clone lives. If it is stale or missing, ask me before pulling or cloning (https://github.com/langchain-ai/openwiki).
3. Review what changed upstream since the baseline: `git log <baseline-commit>..HEAD --oneline`, `CHANGELOG.md`, and the high-signal files —
   - `src/agent/prompts/code.ts` (the core init/update/chat prompts — the closest equivalent of this skill's SKILL.md),
   - `src/agent/prompt.ts` (template placeholders and shared instruction blocks),
   - `src/agent/*.ts` subagent definitions (e.g., skeleton critic, QA question finder/verifier),
   - `skills/*/SKILL.md` (their bundled skills, e.g., mermaid-diagrams).
4. Classify each meaningful upstream change as **adopt** (instruction-expressible, fits the constraints), **adapt** (good idea, needs harness-neutral rework), or **ignore** (code-only or out of scope), with a one-line reason each. Also flag anything in my skill that upstream has since reversed or contradicted.
5. Present the findings as a table and wait for my selection before editing anything.
6. After implementing the selected changes, run an **adversarial review of the resulting diff using the adversarial-review skill with Opus as the reviewer model**. Address P0/P1 findings before finishing.
7. Update the **OpenWiki sync baseline** section of `README.md` to the new upstream commit, version, and date.
