# UX Audit Agent Prompt Template

Use this template to generate an audit prompt for any CLI tool running in a sandboxed container.
Replace all `{{VARIABLE}}` placeholders before passing to the agent.

---

## Variables

| Variable | Description | Example |
|---|---|---|
| `{{TOOL_NAME}}` | CLI tool being audited | `brewprune` |
| `{{TOOL_DESCRIPTION}}` | One-sentence description | `finds and removes unused Homebrew packages` |
| `{{CONTAINER_NAME}}` | Docker container name | `bp-sandbox` |
| `{{EXEC_PREFIX}}` | Command prefix to exec into container | `docker exec bp-sandbox` |
| `{{INSTALLED_PACKAGES}}` | Packages available in the environment | `jq, ripgrep, fd, bat, git, curl, tmux` |
| `{{SUBCOMMANDS}}` | Subcommands to audit | `scan, status, unused, watch, stats, explain, doctor, remove` |
| `{{AUDIT_AREAS}}` | Ordered list of areas with specific commands | see below |
| `{{OUTPUT_PATH}}` | Where to write the final report | `/path/to/docs/cold-start-audit.md` |

---

## Filled Prompt (copy and substitute variables)

```
You are performing a UX audit of {{TOOL_NAME}} — a tool that {{TOOL_DESCRIPTION}}.
You are acting as a **new user** encountering this tool for the first time.

You have access to a Docker container called `{{CONTAINER_NAME}}` with {{TOOL_NAME}} installed
and the following packages available: {{INSTALLED_PACKAGES}}.

Run all commands using: `{{EXEC_PREFIX}} <command>`

## Audit Areas

{{AUDIT_AREAS}}

Run ALL commands. Do not skip areas.
Note exact output, errors, exit codes, and behavior at each step.
Describe color usage (e.g. "package names appear in bold white, tier labels in green/yellow/red").

## Findings Format

For each issue found, use:

### [AREA] Finding Title
- **Severity**: UX-critical / UX-improvement / UX-polish
- **What happens**: What the user actually sees
- **Expected**: What better behavior looks like
- **Repro**: Exact command(s)

Severity guide:
- **UX-critical**: Broken, misleading, or completely missing behavior that blocks the user
- **UX-improvement**: Confusing or unhelpful behavior that a user would notice and dislike
- **UX-polish**: Minor friction, inconsistency, or missed opportunity for clarity

## Report

- Group findings by area
- Include a summary table at the top: total count by severity
- Write the complete report to `{{OUTPUT_PATH}}` using the Write tool

IMPORTANT: Run ALL commands via `{{EXEC_PREFIX}} <command>`.
Do not run {{TOOL_NAME}} directly on the host.
```

---

## Audit Areas Template

Use this structure for `{{AUDIT_AREAS}}`. Customize commands per tool.

```
1. **Discovery** — `{{TOOL_NAME}} --help`, `{{TOOL_NAME}} --version`,
   `{{TOOL_NAME}} <subcommand> --help` for each subcommand in: {{SUBCOMMANDS}}

2. **Setup / onboarding** — First-run commands, initialization, status checks

3. **Core feature** — Primary command with all flag combinations

4. **Data/tracking** — Commands that depend on collected data over time

5. **Explanation / detail** — Per-item drill-down commands, with valid and invalid inputs

6. **Diagnostics** — Health check, doctor, debug commands

7. **Destructive / write operations** — Remove, delete, reset — always with --dry-run first

8. **Edge cases** — Empty args, invalid flags, unknown subcommands, nonexistent inputs

9. **Output review** — Assess table alignment, color usage, header/footer clarity,
   consistency of terminology across subcommands
```
