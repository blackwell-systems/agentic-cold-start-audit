# ai-cold-start-audit

Cold-start UX auditing for CLI tools. AI agents simulate new users in containerized sandboxes, surfacing friction you can't see yourself.

A filler agent reads your project's help output and subcommands to populate a structured prompt template. An audit agent executes every subcommand as a simulated new user inside a Docker container, producing a severity-tiered findings report with reproduction steps.

## Why

You can't UX-test your own tool — you know too much. AI agents have zero prior context, making them ideal cold-start testers. Run them in a disposable container with full access and they'll find the friction you can't see.

## How

1. Build and start a sandbox container with your tool installed
2. A **filler agent** runs `--help` on every subcommand, inspects the environment, and fills the prompt template
3. An **audit agent** executes the filled prompt inside the container as a new user
4. Findings are written as a structured report grouped by area with severity tiers

See [`workflow.md`](workflow.md) for the full repeatable process including prerequisites, permissions setup, and launch instructions.

## Files

| File | Purpose |
|------|---------|
| [`workflow.md`](workflow.md) | Repeatable process — prerequisites, permissions, launch steps, triage |
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
