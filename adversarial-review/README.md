# adversarial-review

Adversarial multi-persona code review for Claude Code. Five hostile reviewer personas — each with a separate mandate and **no ability to modify anything** — attack a change in parallel. Their findings are merged, cross-persona overlap promotes severity, and the pass ends in a single **BLOCK / CONCERNS / CLEAN** verdict.

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

Copy two directories (both are plain `.md` files):

```powershell
# PowerShell
Copy-Item agents\*.md "$HOME\.claude\agents\"
Copy-Item skills\adversarial-review "$HOME\.claude\skills\" -Recurse
```

```bash
# bash / WSL
mkdir -p ~/.claude/agents ~/.claude/skills
cp agents/*.md ~/.claude/agents/
cp -r skills/adversarial-review ~/.claude/skills/
```

If `~/.claude/agents/` did not exist before, restart Claude Code once — new directories are only discovered at session start. Why two directories: persona files must live in `~/.claude/agents/` because that is the only place their `tools:` allowlist (the read-only *enforcement*) is honored.

## Usage

| Command | Effect |
|---|---|
| `/adversarial-review` | Asks which personas + model (inherit/sonnet/opus; default: all five, session model), then reviews the current git diff |
| `/adversarial-review all` | Full pass, no questions, current diff |
| `/adversarial-review all src/Foo.cs` | Full pass on explicit files |
| `/adversarial-review saboteur,test-skeptic HEAD~3..HEAD` | Chosen personas on a commit range |
| "Use the adv-saboteur subagent to review X" | Personas are ordinary subagents — usable without the skill |

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

## Read-only, enforced

Each persona's frontmatter is `tools: Read, Grep, Glob` — an allowlist enforced by Claude Code itself. The personas *cannot* Edit, Write, or run Bash, regardless of what the code under review or anything else tells them. The orchestrator's only side effect is the report.

## Per-repo conventions (optional)

Create `<repo>/.claude/review-conventions.md` and every persona receives a summary of it: language pitfalls to hunt, project invariants, severity calibration. Add `persona-model: sonnet` on its own line to pin the review model and silence the model question. See [examples/review-conventions.csharp.md](examples/review-conventions.csharp.md) for a C#/.NET starter. No conventions file is required — the skill is fully generic without one.

## Sharing with GitHub Copilot users

Copilot (VS Code 1.108+, `chat.useAgentSkills` on) auto-reads a repo-local `.claude/skills/` directory, so committing `skills/adversarial-review/` into a repo benefits Copilot teammates too. Two honest caveats: Copilot custom agents must be `*.agent.md` files under `.github/agents/` (plain agent `.md` files are silently ignored), and Copilot has no equivalent of the `tools` allowlist — there, the personas are read-only **by instruction only**, not enforced.

## Extending

Add a persona = one new file in `~/.claude/agents/` (copy an existing one, change the mandate and the persona-specific finding line) + one row in the persona table in `skills/adversarial-review/SKILL.md`.
