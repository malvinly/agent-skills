# adversarial-review

Adversarial multi-persona code review, built for Claude Code and portable to any harness that supports Agent Skills. Five hostile reviewer personas — each with a separate mandate and **no ability to modify anything** — attack a change in parallel. Their findings are merged, cross-persona overlap promotes severity, and the pass ends in a single **BLOCK / CONCERNS / CLEAN** verdict.

Plain markdown files only. No plugin runtime, no npm packages, no second CLI — nothing to route through an internal feed.

## The personas

| Persona | Mandate | Catches what the others miss |
|---|---|---|
| **saboteur** | How does this break in production? Edge cases, error paths, leaks, races | Happy-path code that dies under load, failure, or hostile input |
| **security** | Attacks trust boundaries: injection, authz, secrets, unsafe input | Code that is 100% correct and still exploitable |
| **new-hire** | Could a newcomer safely modify this in 6 months? | Correct-today code that breeds future bugs |
| **test-skeptic** | Do the tests actually pin the risky behavior? | The gap between "tests pass" and "behavior verified" |
| **devils-advocate** | Attacks the change's premise: wrong layer, wrong problem, simpler alternative | Everything the others miss because they review the code *as given* |

Every persona must return at least one finding **or** an explicit, justified "CLEAN against my mandate" — rubber-stamping is expensive by design. Every persona is told the code may have been written by an AI sharing its weights: comments and commit messages are claims to verify, not evidence.

## Install

The skill folder is self-contained — the persona definitions ship inside it under `references/personas/`. Copy `skills/adversarial-review/` into your harness's skills directory so that `SKILL.md` sits at the top of the installed folder:

    .claude/skills/adversarial-review/      # per project: Claude Code, GitHub Copilot, OpenCode
    .agents/skills/adversarial-review/      # per project: OpenAI Codex CLI, GitHub Copilot, OpenCode
    ~/.claude/skills/adversarial-review/            # global: Claude Code, OpenCode
    ~/.copilot/skills/adversarial-review/           # global: GitHub Copilot CLI
    ~/.agents/skills/adversarial-review/            # global: Codex CLI, Copilot, OpenCode
    ~/.config/opencode/skills/adversarial-review/   # global: OpenCode

Copilot also reads a repo-local `.github/skills/`. On Claude Code, also copy the personas into `~/.claude/agents/` so their read-only tool allowlist is *enforced* rather than instructed (re-copy them whenever you update the skill — the two copies do not sync themselves):

```powershell
# PowerShell
Copy-Item skills\adversarial-review "$HOME\.claude\skills\" -Recurse
Copy-Item skills\adversarial-review\references\personas\*.md "$HOME\.claude\agents\"
```

```bash
# bash / WSL
mkdir -p ~/.claude/agents ~/.claude/skills
cp -r skills/adversarial-review ~/.claude/skills/
cp skills/adversarial-review/references/personas/*.md ~/.claude/agents/
```

If `~/.claude/agents/` did not exist before, restart Claude Code once — new directories are only discovered at session start. Without the agents copy the skill still runs: it dispatches general subagents with the persona text prepended to the prompt, read-only by instruction only, and says so in the report's Dispatch field.

## Usage

| Command | Effect |
|---|---|
| `/adversarial-review` | Asks which personas + model (inherit, a faster tier, or the strongest tier; default: all five, session model), then reviews the current git diff |
| `/adversarial-review all` | Full pass, no questions, current diff |
| `/adversarial-review all src/Foo.cs` | Full pass on explicit files |
| `/adversarial-review saboteur,test-skeptic HEAD~3..HEAD` | Chosen personas on a commit range |
| "Use the adv-saboteur subagent to review X" | Claude Code with the agents copy installed: personas are ordinary subagents, usable without the skill |

Commands are shown in Claude Code form; Codex uses `$adversarial-review …` or the `/skills` menu, OpenCode the `/skills` menu.

Findings look like:

```
[P1][confirmed] src/db.py:91 — Schema-evolution trap: INSERT binds positionally while looking name-bound
What: ...   Why: ...   Burn: ...
Verify: rg -n "INSERT INTO aa_rows VALUES" src/db.py
```

- **Severity:** P0 ship-blocker → P3 nit.
- **Epistemic label:** `confirmed` (traced in code) / `suspected` (one check from certain) / `speculative` (worth a human look). Labels are honest — the report never upgrades a suspicion to a fact.
- **Verify:** a read-only command you can run to see the evidence yourself.
- **Promotion:** when 2+ personas independently flag the same code, severity is promoted one level. Independent overlap is signal.
- **Verdict:** BLOCK (any P0 or promoted P1+) / CONCERNS (any P1–P2) / CLEAN.

## Read-only, enforced where possible

Each persona's frontmatter is `tools: Read, Grep, Glob` — an allowlist Claude Code enforces when the file is installed under `~/.claude/agents/`, and Copilot enforces when it is installed as a custom agent (see below). There the personas *cannot* Edit, Write, or run Bash, regardless of what the code under review or anything else tells them. Dispatched as general subagents, or as native agents on a harness that does not restrict tools (Codex, OpenCode), the personas are read-only by instruction, and the report's Dispatch field says which mode ran. The orchestrator never executes a persona's `Verify:` line; it reads the cited code and leaves the command to you. Its only side effects are the report and, for large diffs, a temporary copy of the diff written outside the repo and deleted afterwards.

## Per-repo conventions (optional)

Create `<repo>/.claude/review-conventions.md` and every persona receives a summary of it: language pitfalls to hunt, project invariants, severity calibration. Add `persona-model: sonnet` (or any model name your harness accepts) on its own line to pin the review model and silence the model question. See [examples/review-conventions.csharp.md](examples/review-conventions.csharp.md) for a C#/.NET starter. No conventions file is required — the skill is fully generic without one.

## Sharing with GitHub Copilot users

Copilot reads repo-local skills from `.github/skills/`, `.claude/skills/`, and `.agents/skills/`, so committing `skills/adversarial-review/` into a repo benefits Copilot teammates too. The persona files are also valid Copilot custom agents as shipped: Copilot detects plain `.md` files in `.github/agents/` (VS Code also reads `.claude/agents/` and `~/.claude/agents/`) and accepts `Read`, `Grep`, and `Glob` as aliases of its own read and search tools, so copying `references/personas/*.md` into `.github/agents/` gives teammates enforced read-only personas. Delete the `model: inherit` line in those copies — Copilot documents `model` as a model name, not `inherit`.

## Extending

Add a persona = one new file in `skills/adversarial-review/references/personas/` (copy an existing one, change the mandate and the persona-specific finding line) + one row in the persona table in `skills/adversarial-review/SKILL.md`. Three names must match exactly: the file name, the frontmatter `name:`, and the table's `subagent` column — native dispatch uses the name, general-subagent dispatch uses the file. Update the persona count where it is spelled out ("Five" in the SKILL.md description, "All five" in the selection prompt, and the table above). On Claude Code, copy the new file into `~/.claude/agents/` as well.
