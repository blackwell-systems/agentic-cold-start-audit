# ai-cold-start-audit

Turn AI's lack of context into a feature. Agents cold-start your CLI in a container and report every friction point a new user would hit. Includes a Claude Code `/cold-start-audit` skill.

A filler agent reads your project's help output and subcommands to populate a structured prompt template. An audit agent executes every subcommand as a simulated new user inside a Docker container, producing a severity-tiered findings report with reproduction steps.

## Why

You can't UX-test your own tool - you know too much. AI agents have zero prior context, making them ideal cold-start testers. Run them in a disposable container with full access and they'll find the friction you can't see.

## Quickstart

```bash
# 1. Build a container with your tool installed
docker build -t my-tool-sandbox -f Dockerfile.sandbox .
docker run -d --name my-sandbox --rm my-tool-sandbox sleep 3600

# 2. Verify it works
docker exec my-sandbox mytool --help

# 3. Run the audit (using the Claude Code skill)
/cold-start-audit run my-sandbox mytool
```

The skill runs a filler agent to discover your tool's subcommands and flags, then launches an audit agent that exercises everything as a new user. The report lands in `docs/cold-start-audit.md`.

See [`sandbox-setup.md`](sandbox-setup.md) for Dockerfile patterns and container design options.

## How It Works

1. **Build a sandbox** - Docker container with your tool installed in a clean environment
2. **Filler agent** - runs `--help` on every subcommand, inspects the environment, and populates the prompt template with real commands
3. **Audit agent** - executes the filled prompt inside the container as a new user, noting every output, error, and friction point
4. **Structured report** - findings grouped by area with severity tiers and exact reproduction steps

## Usage with Claude Code

Clone this repo and the project-level skill at `.claude/commands/cold-start-audit.md` works automatically. Or copy [`prompts/cold-start-audit-skill.md`](prompts/cold-start-audit-skill.md) to `~/.claude/commands/cold-start-audit.md` for a global slash command - update the file paths to absolute paths after copying.

```bash
# Setup only: discover tool metadata and generate the filled audit prompt
/cold-start-audit setup my-sandbox mytool

# Full audit: filler + audit agent (writes report to docs/cold-start-audit.md)
/cold-start-audit run my-sandbox mytool

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

A cold-start audit of [brewprune](https://github.com/blackwell-systems/brewprune) found 21 issues:

| Severity | Count |
|----------|-------|
| UX-critical | 5 |
| UX-improvement | 9 |
| UX-polish | 7 |

Sample finding:

```
### [ANALYSIS] Score logic marks recently-used packages as "SAFE" to remove

- Severity: UX-critical
- What happens: After running jq --version via shims, jq receives score 80 (SAFE).
  The explain output says "Usage: 40/40 pts - used today" AND "Why SAFE: rarely used,
  safe to remove." These directly contradict each other.
- Expected: A package used today should score very LOW for removal confidence.
- Repro: Use shimmed packages, wait 35s, then brewprune explain jq
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

## License

MIT
