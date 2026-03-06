# agentic-cold-start-audit

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
![Version](https://img.shields.io/badge/version-1.0.0-blue)

A two-agent protocol for discovering cold-start UX friction in CLI tools. A filler agent reads your project's help output and populates a structured audit prompt. An audit agent executes every subcommand as a simulated new user in a sandboxed environment, producing a severity-tiered findings report with exact reproduction steps. Human review is preserved: the skill waits for your confirmation before launching agents. Includes a Claude Code `/cold-start-audit` skill.

## Why

You can't UX-test your own tool — you know too much. Every mental model you've built, every shortcut you've learned, every implicit assumption about how the tool is used shapes your testing. You'll test the paths that work. You won't test the ones you never tried.

AI agents have zero prior context. They follow the help text, try the obvious commands, and report exactly what breaks — because they have no workarounds to fall back on. That naive perspective is the one you need.

The problem is safety. An agent testing your production system can delete data, corrupt state, or trigger side effects. Every audit here runs inside an isolated sandbox — container, redirected env vars, or a fresh directory copy. The audit agent can fail safely, and you see every failure.

The result is a severity-tiered findings report: what's broken, what's confusing, what's rough polish. Some findings are blockers. Others are sprint items. The [`/cold-start-audit report`](#commands) command triages them.

## When to Use It

Cold-start auditing is most useful when:

1. **Your tool ships to users who don't know you** — Every new user is a cold start. If your tool is public or widely used, discovering friction before release saves support load.
2. **You iterate on UX frequently** — Run the audit after each improvement cycle. The agent's fresh perspective surfaces whatever friction exists now, whether it's a new issue or an old one that regressed. Each audit is pure discovery with zero memory of previous rounds.
3. **Your tool has subcommands, flags, or modes** — The more surface area, the more opportunities for friction. A single-command tool with 2 flags may not benefit; a CLI with 10 subcommands will.
4. **Your help text is the primary documentation** — If users rely on `--help` chains and examples, cold-start auditing validates that the help text actually teaches the tool.

**This is pure discovery, not regression testing.** The agent has zero knowledge of previous rounds. It discovers friction organically by following help text and trying obvious commands. If a fixed issue breaks again, it surfaces naturally through rediscovery — the agent hits the same friction a new user would. That's the methodology's strength: you don't maintain test checklists, you just run the agent and see what a fresh user actually encounters.

It is less useful for:

- **Internal tools with a small team that knows the author** — Training cost exceeds discovery value.
- **Tools requiring non-reproducible external setup** — APIs, live databases, or third-party accounts that can't be mocked in a sandbox reduce coverage.

## Discovering the Right Sandbox Mode

Not sure which isolation mode fits your tool? Ask the skill before anything else:

```bash
/cold-start-audit mode mytool
```

The skill reads your tool's `--help` output and answers three diagnostic questions:

1. **Does the tool have destructive operations?** (remove, delete, system writes)
2. **Does the tool write only to self-managed state?** (its own DB, config file, redirectable via env var)
3. **Does the tool operate on the current directory or file tree?**

It emits a mode recommendation with rationale and a ready-to-run command. You don't need to understand the isolation mechanisms to get started — the skill handles the decision.

## Quickstart

After running `mode` to get your command, the full audit is one line:

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

See [`sandbox-setup.md`](sandbox-setup.md) for Dockerfile patterns, Docker setup, and mode guidance.

## Commands

```
/cold-start-audit mode <tool-name>
  Recommend a sandbox isolation mode. Reads --help output and answers
  three diagnostic questions. Does not write any files.

/cold-start-audit setup <tool-name> --mode <mode> [mode-args]
  Run the filler agent only: discover tool metadata and write the filled
  audit prompt to docs/cold-start-audit-prompt.md. Use when you want to
  review the prompt before running the full audit.

/cold-start-audit run <tool-name> --mode <mode> [mode-args]
  Run the full audit (filler + audit agent). Checks for an existing prompt
  and offers to reuse or regenerate. Writes findings to docs/cold-start-audit.md.

/cold-start-audit report
  Read and summarize the findings from docs/cold-start-audit.md, grouped by severity.

/cold-start-audit init <container-name>
  Scaffold .claude/settings.json with scoped Bash permissions for a named container.
```

## How It Works

1. **Choose a sandbox** — container, local env var, or worktree isolation, depending on the tool's blast radius. Run `/cold-start-audit mode <tool-name>` if unsure.
2. **Filler agent** — runs `--help` on every subcommand, inspects the sandbox environment, and populates the prompt template with real commands and the correct exec prefix.
3. **Audit agent** — executes the filled prompt inside the sandbox as a new user, noting every output, error, and friction point.
4. **Structured report** — findings grouped by area with severity tiers and exact reproduction steps.

For subsequent audits, the skill checks for an existing prompt and offers to reuse it (fast — only updates container name and date) or regenerate from scratch (when tool structure changed). Reusing the prompt doesn't change what the agent discovers — it still runs with zero context — it just skips re-reading help text.

## Installation

### Custom Agent Types (Optional)

For enhanced observability and tool enforcement, install custom agent types:

```bash
# From the agentic-cold-start-audit repository:
mkdir -p ~/.claude/agents
ln -sf $(pwd)/prompts/agents/filler.md ~/.claude/agents/filler.md
ln -sf $(pwd)/prompts/agents/audit.md ~/.claude/agents/audit.md
```

**Benefits:**
- **Tool enforcement** — Filler agent cannot execute sandbox commands; Audit agent cannot bypass isolation
- **Claudewatch visibility** — Separate success rates per agent type in metrics
- **Behavioral instructions** — Carried in the type definition for consistency

**Graceful fallback:** The skill automatically falls back to `general-purpose` agents if custom types are not installed. Installation is recommended but not required.

After installation, restart your Claude Code session (settings are not hot-reloaded).

## Permissions

Background agents cannot prompt for tool approval. Without an explicit `allow` rule, every `Bash` call is denied and the agent silently fails.

### User-level (one-time setup)

Add to `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```

> **Common mistake:** `allow_bash: true` is not a valid Claude Code permission format and is silently ignored.

After editing, restart your Claude Code session. Settings are not hot-reloaded.

### Project-level (optional, scoped)

For tighter control, add `.claude/settings.json` to the project repo:

```json
{
  "permissions": {
    "allow": ["Bash(docker exec <container-name>*)"]
  }
}
```

> **Note:** Project-level permissions require user approval before they are trusted. Use user-level permissions for background agents that need to work without interaction.

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

## Severity Tiers

| Tier | Meaning | Examples | Action |
|------|---------|----------|--------|
| **UX-critical** | Broken, misleading, or completely blocks the user | Flag advertised in help doesn't exist; wrong error message; subcommand crashes | Fix before next release |
| **UX-improvement** | Confusing or unhelpful; user can work around it | Error message doesn't explain the fix; help text is vague; output is ambiguous | Prioritize for next sprint |
| **UX-polish** | Minor friction, edge case, or consistency issue | Inconsistent flag ordering; minor output inconsistency; formatting | Batch into a cleanup PR |

## Files

| File | Purpose |
|------|---------|
| [`SPEC.md`](SPEC.md) | Formal specification: execution model, invariants, preconditions, correctness guarantees |
| [`workflow.md`](workflow.md) | Repeatable process — prerequisites, permissions, launch steps, triage |
| [`sandbox-setup.md`](sandbox-setup.md) | Container design, Dockerfile patterns, docker exec vs bind mount |
| [`prompts/prompt-template.md`](prompts/prompt-template.md) | Audit prompt template with variable table and audit areas structure |
| [`prompts/filler-agent-prompt.md`](prompts/filler-agent-prompt.md) | Agent that discovers tool metadata and fills the template |
| [`prompts/cold-start-audit-skill.md`](prompts/cold-start-audit-skill.md) | Portable Claude Code `/cold-start-audit` skill |

## Composing with Scout-and-Wave

Audit findings are structured input for parallel fixes. Once you have a severity-tiered findings report, hand it to [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave):

```bash
/saw scout "Fix UX findings from cold-start audit"
/saw wave
```

The scout runs a pre-implementation check to filter already-fixed items, assigns remaining findings to parallel agents with disjoint file ownership, and executes in waves. The [agentic-workflows](https://github.com/blackwell-systems/agentic-workflows) repo documents the full Audit→Fix→Rediscover cycle — six complete iterations discovering 118 total findings in [brewprune](https://github.com/blackwell-systems/brewprune). Each audit discovers organically; fixed issues that regress are simply rediscovered.

## License

MIT
