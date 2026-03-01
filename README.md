# agentic-cold-start-audit

Turn AI's lack of context into a feature. Agents cold-start your CLI in a sandboxed environment and report every friction point a new user would hit. Includes a Claude Code `/cold-start-audit` skill.

A filler agent reads your project's help output and subcommands to populate a structured prompt template. An audit agent executes every subcommand as a simulated new user, producing a severity-tiered findings report with reproduction steps.

## Why

You can't UX-test your own tool — you know too much. AI agents have zero prior context, making them ideal cold-start testers. Run them in an isolated sandbox and they'll find the friction you can't see.

## Quickstart

Not sure which sandbox mode fits your tool? Let the skill decide:

```bash
/cold-start-audit mode mytool
# → Mode recommendation: local | container | worktree
# → Suggested command: /cold-start-audit run mytool --mode local --env MYTOOL_DB=/tmp/audit-xx/db
```

**Container mode** (destructive ops, package managers, system writes):

```bash
docker build -t my-tool-sandbox -f Dockerfile.sandbox .
docker run -d --name my-sandbox --rm my-tool-sandbox sleep 3600
/cold-start-audit run mytool --mode container my-sandbox
```

**Local mode** (tool writes only to its own DB or config, redirectable via env var):

```bash
/cold-start-audit run mytool --mode local --env MYTOOL_DB=/tmp/audit-$$/db.sqlite3
```

**Worktree mode** (tool reads/writes files in the current directory):

```bash
/cold-start-audit run mytool --mode worktree --dir /path/to/project
```

The report lands in `docs/cold-start-audit.md`.

See [`sandbox-setup.md`](sandbox-setup.md) for setup details, Dockerfile patterns, and mode guidance.

## How It Works

1. **Choose a sandbox** — container, local env var, or worktree isolation depending on the tool's blast radius. Run `/cold-start-audit mode <tool-name>` if unsure.
2. **Filler agent** — runs `--help` on every subcommand, inspects the sandbox environment, and populates the prompt template with real commands and the correct exec prefix
3. **Audit agent** — executes the filled prompt inside the sandbox as a new user, noting every output, error, and friction point
4. **Structured report** — findings grouped by area with severity tiers and exact reproduction steps

## Usage with Claude Code

Clone this repo and the project-level skill at `.claude/commands/cold-start-audit.md` works automatically. Or copy [`prompts/cold-start-audit-skill.md`](prompts/cold-start-audit-skill.md) to `~/.claude/commands/cold-start-audit.md` for a global slash command.

```bash
# Recommend isolation mode for your tool (no sandbox needed yet)
/cold-start-audit mode mytool

# Setup only: discover tool metadata and generate the filled audit prompt
/cold-start-audit setup mytool --mode container my-sandbox
/cold-start-audit setup mytool --mode local --env MYTOOL_DB=/tmp/audit-$$/db.sqlite3
/cold-start-audit setup mytool --mode worktree --dir /path/to/project

# Full audit: filler + audit agent (writes report to docs/cold-start-audit.md)
/cold-start-audit run mytool --mode container my-sandbox
/cold-start-audit run mytool --mode local --env MYTOOL_DB=/tmp/audit-$$/db.sqlite3

# Summarize: read and triage the findings
/cold-start-audit report
```

### Without the skill

You can run the audit manually without the Claude Code skill:

1. Substitute `{{CONTAINER_NAME}}`, `{{TOOL_NAME}}`, and `{{OUTPUT_PATH}}` in [`prompts/filler-agent-prompt.md`](prompts/filler-agent-prompt.md) and give it to Claude Code (or any AI agent with bash access)
2. The filler agent produces a filled audit prompt
3. Give the filled prompt to a fresh agent session - the agent runs all commands and writes the report

See [`workflow.md`](workflow.md) for the full process including permissions setup, prerequisites, and a pre-launch checklist.

## Example Output

A cold-start audit of [brewprune](https://github.com/blackwell-systems/brewprune) found 18 issues:

| Severity | Count |
|----------|-------|
| UX-critical | 5 |
| UX-improvement | 8 |
| UX-polish | 5 |

Sample finding:

```
### [DIAGNOSTICS] `doctor` output uses "Fix:" label but --fix flag does not exist

- Severity: UX-critical
- What happens: `doctor` output says "Fix: add shim directory to PATH before Homebrew"
  — the word "Fix:" implies a --fix flag. Running `brewprune doctor --fix` returns
  Error: unknown flag: --fix (exit 1).
- Expected: Either implement --fix to automate the remediation, or rename the label
  from "Fix:" to "Action needed:" to avoid implying a flag exists.
- Repro: brewprune doctor → brewprune doctor --fix
```

## Files

| File | Purpose |
|------|---------|
| [`workflow.md`](workflow.md) | Repeatable process - prerequisites, permissions, launch steps, triage |
| [`sandbox-setup.md`](sandbox-setup.md) | Container design, Dockerfile patterns, docker exec vs bind mount |
| [`prompts/prompt-template.md`](prompts/prompt-template.md) | Audit prompt template with variable table and audit areas structure |
| [`prompts/filler-agent-prompt.md`](prompts/filler-agent-prompt.md) | Agent that discovers tool metadata and fills the template |
| [`prompts/cold-start-audit-skill.md`](prompts/cold-start-audit-skill.md) | Portable Claude Code `/cold-start-audit` skill |

## Severity Tiers

| Tier | Meaning | Action |
|------|---------|--------|
| **UX-critical** | Broken, misleading, or blocks the user | Fix before next release |
| **UX-improvement** | Confusing but functional | Prioritize for next sprint |
| **UX-polish** | Minor friction or inconsistency | Batch into a cleanup PR |

## Composing with Scout-and-Wave

Audit findings are structured input for parallel fixes. Once you have a severity-tiered findings report, hand it to the [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) scout: it runs a pre-implementation check to filter already-fixed items, assigns remaining findings to parallel agents with disjoint file ownership, and executes in waves. The [agentic-workflows](https://github.com/blackwell-systems/agentic-workflows) repo documents this Audit-Fix-Verify loop and links to the reference implementation in [brewprune](https://github.com/blackwell-systems/brewprune) — six complete audit→fix cycles across 118 findings.

## License

MIT
