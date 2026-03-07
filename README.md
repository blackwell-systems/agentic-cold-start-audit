# agentic-cold-start-audit

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
![Version](https://img.shields.io/badge/version-1.1.0-blue)
[![Agent Skills](badge-agentskills.svg)](https://agentskills.io)

A two-agent protocol for discovering cold-start UX friction in CLI tools. A filler agent reads your project's help output and populates a structured audit prompt. An audit agent executes every subcommand as a simulated new user in a sandboxed environment, producing a severity-tiered findings report with exact reproduction steps. Human review is preserved: the skill waits for your confirmation before launching agents. Includes a Claude Code `/cold-start-audit` skill.

> [!NOTE]
> Follows the [Agent Skills](https://agentskills.io) open standard — compatible with Claude Code, Cursor, GitHub Copilot, and other Agent Skills-compatible tools.

## Installation

### Prerequisites
- [Claude Code](https://code.anthropic.com) (CLI or IDE extension)
- Docker (for container mode only - skip if using local/worktree modes)

### Install the Skill
```bash
# Copy skill to Claude Code skills directory
mkdir -p ~/.claude/skills
cp -r prompts/cold-start-audit-skill ~/.claude/skills/cold-start-audit

# Restart Claude Code (skills load at session start)
# Verify: Type /cold-start-audit and see the skill appear
```

### Optional: Install Custom Agent Types
For enhanced observability and tool enforcement:
```bash
mkdir -p ~/.claude/agents
ln -sf $(pwd)/prompts/agents/filler.md ~/.claude/agents/filler.md
ln -sf $(pwd)/prompts/agents/audit.md ~/.claude/agents/audit.md
# Then restart Claude Code
```

## Core Concepts

**Cold-Start Audit:** Simulating a brand-new user who has never seen your tool, following only help text, and reporting every friction point encountered.

**Two-Agent Pattern:**
1. **Filler Agent** - Reads `--help` output from your tool, discovers subcommands/flags, and generates a customized audit prompt with exact commands to test. Runs once per audit round (or reuse previous).
2. **Audit Agent** - Executes the filled prompt in a fresh session with zero knowledge, runs 30+ commands in sandbox, reports findings. This is the "new user" simulation.

**Why two agents?** The audit agent MUST have zero context (simulating a new user). If it read help text itself, it would know what to expect. The filler agent handles the "setup" knowledge separately.

**Sandbox Isolation:** All commands run through an isolation layer (Docker container, env vars, or temp directory copy) so destructive operations are safe.

**Pure Discovery:** The audit agent has no memory of previous rounds. It discovers friction organically, not from a checklist.

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

## Choosing a Sandbox Mode

Not sure which mode to use? Run:
```bash
/cold-start-audit mode mytool
```

This analyzes your tool's `--help` output and asks:
1. **Q1:** Does the tool have destructive operations? (remove, delete, uninstall, etc.)
   → If yes: **container mode** (strongest isolation)
2. **Q2:** Does it write to self-managed state? (databases, config files it manages)
   → If yes, check Q3
3. **Q3:** Can state be redirected via environment variables?
   → If yes: **local mode** (env var redirection)
   → If no: **container mode**

If the tool only operates on files in the current directory: **worktree mode** (temp directory copy)

## Quickstart

### First Time Setup

1. **Discover the right mode:**
```bash
/cold-start-audit mode mytool
```

2. **Create a Dockerfile** (container mode only):

Container mode requires an **audit-ready Dockerfile** (not your production Dockerfile). This special Dockerfile:
- Installs the tool at `/usr/local/bin/<tool-name>`
- Creates a non-root user with sudo access
- Simulates a realistic user environment

You have three options:

- **Generate automatically** with [dockerfile-sandbox-gen](https://github.com/blackwell-systems/dockerfile-sandbox-gen):
  ```bash
  /dockerfile-sandbox-gen mytool
  ```

- **Write manually** at `Dockerfile.sandbox` in your project root

- **Use custom path** with `--dockerfile PATH` when building the container

See [sandbox-setup.md](docs/sandbox-setup.md#dockerfile-pattern) for the full Dockerfile contract requirements.

3. **Run the audit:**

**Container mode** (destructive ops, package managers, system writes):
```bash
# Step 1: Build a Docker container with your tool installed
# This auto-names the container "mytool-r1" (round 1)
/cold-start-audit container build mytool

# Step 2: Run the full audit using that container
# The agent will execute 30+ commands inside mytool-r1 and write findings
/cold-start-audit run mytool --mode container mytool-r1
```

**Local mode** (tool writes only to its own DB or config):
```bash
/cold-start-audit run mytool --mode local --env MYTOOL_DB=/tmp/audit-$$/db.sqlite3
```

**Worktree mode** (tool reads/writes files in current directory):
```bash
/cold-start-audit run mytool --mode worktree --dir /path/to/project
```

The report lands in `docs/cold-start-audit.md`.

### Verify Your First Audit

After running `/cold-start-audit run`:

1. **Check agent output** - You'll see the filler agent discover subcommands, then the audit agent launch in background
2. **Monitor progress** - Look for "Running command X/30..." style output
3. **Expected duration** - 5-10 minutes for typical CLI tool with 5-10 subcommands
4. **Check results** - `ls docs/` should show:
   - `cold-start-audit-prompt.md` - The filled audit prompt
   - `cold-start-audit.md` - The findings report with severity tiers

**If files exist but are empty:** Check permissions (see [Permissions](#permissions) section).

### Subsequent Rounds

After fixing issues from the first audit, running the next round is one command:

```bash
/cold-start-audit container build mytool  # Auto-detects r1, creates r2
/cold-start-audit run mytool --mode container mytool-r2
```

The skill:
- Auto-increments round numbers
- Checks if container is already running (reuses if so)
- Validates all prerequisites before starting
- Offers to reuse previous audit prompt (fast) or regenerate (thorough)

### Workflow Comparison

**Before (manual):**
```bash
# Figure out next round number manually
docker images | grep mytool-r

# Build and start container
docker build -t mytool-r8 -f Dockerfile.sandbox .
docker run -d --name mytool-r8 --rm mytool-r8 sleep 3600

# Hope permissions are configured correctly
# (silent failures if not)

# Run audit
/cold-start-audit run mytool --mode container mytool-r8

# Later: remember to clean up old containers
docker rm -f mytool-r1 mytool-r2 mytool-r3...
docker rmi mytool-r1 mytool-r2 mytool-r3...
```

**After (automated):**
```bash
# One command builds & starts with auto-incrementing
/cold-start-audit container build mytool

# Preflight validates everything, shows fix steps if needed
/cold-start-audit run mytool --mode container mytool-r8

# One command cleanup
/cold-start-audit container cleanup mytool --keep-latest 2
```

## Commands

```
/cold-start-audit mode <tool-name>
  Recommend a sandbox isolation mode. Reads --help output and answers
  three diagnostic questions. Does not write any files.

/cold-start-audit preflight <tool-name> --mode <mode> [mode-args]
  Validate all prerequisites before running an audit. Checks tool installation,
  permissions, Docker status, and provides actionable fix steps if any fail.
  Auto-runs before setup and run commands.

/cold-start-audit container build <tool-name> [--dockerfile PATH]
  Auto-increment round number, check for reusable containers, build image, and
  start container. Detects brewprune-r7 → creates brewprune-r8 automatically.
  Offers to reuse if container already running or image exists.

/cold-start-audit container list <tool-name>
  Show all images and containers for a tool with their status (running/stopped).

/cold-start-audit container cleanup <tool-name> [--keep-latest N]
  Remove old containers and images, keeping the latest N rounds (default: 2).
  Prompts for confirmation before deleting anything.

/cold-start-audit setup <tool-name> --mode <mode> [mode-args]
  Run the filler agent only: discover tool metadata and write the filled
  audit prompt to docs/cold-start-audit-prompt.md. Use when you want to
  review the prompt before running the full audit.

/cold-start-audit run <tool-name> --mode <mode> [mode-args]
  Run the full audit (filler + audit agent). Automatically runs preflight checks.
  Checks for existing prompt and offers to reuse or regenerate. Writes findings
  to docs/cold-start-audit.md.

/cold-start-audit report
  Read and summarize the findings from docs/cold-start-audit.md, grouped by severity.

/cold-start-audit init <container-name>
  Scaffold .claude/settings.json with scoped permissions for a named container.
```

## How It Works

1. **Choose a sandbox** — container, local env var, or worktree isolation, depending on the tool's blast radius. Run `/cold-start-audit mode <tool-name>` if unsure.

2. **Validate prerequisites** — preflight checks run automatically before setup/run:
   - Tool installed (`<tool-name> --help` succeeds)
   - Permissions configured (`~/.claude/settings.json` has Bash allowed)
   - Docker running (container mode only)
   - Container exists and running (container mode only)
   - If any check fails, get actionable fix steps immediately

3. **Filler agent** — runs `--help` on every subcommand, inspects the sandbox environment, and populates the prompt template with real commands and the correct exec prefix. Validates the filled prompt for unfilled placeholders.

4. **Audit agent** — executes the filled prompt inside the sandbox as a new user, noting every output, error, and friction point. Runs with zero knowledge of previous findings.

5. **Structured report** — findings grouped by area with severity tiers and exact reproduction steps.

For subsequent audits, the skill checks for an existing prompt and offers to reuse it (fast — only updates container name and date) or regenerate from scratch (when tool structure changed). Reusing the prompt doesn't change what the agent discovers — it still runs with zero context — it just skips re-reading help text.

## Preflight Validation Example

When running a container mode audit, preflight checks all prerequisites:

```
✓ Preflight checks (7/7 passed)

  ✓ Tool installed: brewprune --help succeeds
  ✓ Bash permissions configured in ~/.claude/settings.json
  ✓ Session restarted since last settings change
  ✓ No broken hooks found
  ✓ Docker is running
  ✓ Container brewprune-r8 exists and is running
  ✓ Tool installed in container

All prerequisites met. Ready to run audit.
```

If checks fail, you get actionable fix steps:

```
✗ Preflight checks failed (5/7 passed)

Failures:
  ✗ Docker not running
    Fix: Start Docker Desktop or run: sudo systemctl start docker

  ✗ Container brewprune-r8 not found
    Fix: /cold-start-audit container build brewprune
    Or manually: docker run -d --name brewprune-r8 brewprune-r8 sleep 3600

Run these fixes, then retry.
```

## Container Lifecycle Management

The `container build` command handles the full lifecycle automatically:

1. **Auto-detects round number** by finding highest existing (e.g., brewprune-r7)
2. **Checks for reusable containers** — if brewprune-r8 is already running, offers to reuse
3. **Checks for reusable images** — if image exists but container stopped, offers to just start it
4. **Builds new image** only when needed (source changed since last audit)
5. **Starts container** with correct name and keeps it running

**Managing multiple rounds:**
```bash
/cold-start-audit container list brewprune

Images for brewprune:
REPOSITORY      TAG     SIZE    CREATED
brewprune-r6    latest  155MB   5 days ago
brewprune-r7    latest  156MB   2 days ago
brewprune-r8    latest  157MB   3 hours ago

Containers for brewprune:
NAMES           STATUS                  IMAGE
brewprune-r6    Exited (0) 5 days ago   brewprune-r6
brewprune-r7    Exited (0) 2 days ago   brewprune-r7
brewprune-r8    Up 3 hours              brewprune-r8
```

**Cleanup old rounds:**
```bash
/cold-start-audit container cleanup brewprune --keep-latest 2

Found 8 rounds. Keeping latest 2 (r7, r8).

Rounds to remove:
  - brewprune-r1
  - brewprune-r2
  - brewprune-r3
  - brewprune-r4
  - brewprune-r5
  - brewprune-r6

Continue? [y/N]: y

✓ Removed container: brewprune-r1
✓ Removed image: brewprune-r1
...
✓ Cleanup complete
```

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

**Symptom:** Agents launch but produce no output, or audit completes with empty files.
**Cause:** Background agents (agents that run without blocking your session) need explicit permission to use Bash, Read, and Write tools.

### Quick Fix (Recommended)

Add to `~/.claude/settings.json` (in your home directory):

```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```

Then **restart Claude Code** (settings load at session start, not hot-reloaded).

To verify: Try `/cold-start-audit preflight <tool>` - it will check permissions.

> **Common mistake:** `allow_bash: true` is invalid syntax and silently ignored. Use `"allow": ["Bash"]` instead.

### Advanced: Project-Level Scoped Permissions

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
| [`prompts/cold-start-audit-skill/`](prompts/cold-start-audit-skill/SKILL.md) | Portable Claude Code `/cold-start-audit` skill |

## Composing with Scout-and-Wave

Audit findings are structured input for parallel fixes. Once you have a severity-tiered findings report, hand it to [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave):

```bash
/saw scout "Fix UX findings from cold-start audit"
/saw wave
```

The scout runs a pre-implementation check to filter already-fixed items, assigns remaining findings to parallel agents with disjoint file ownership, and executes in waves. The [agentic-workflows](https://github.com/blackwell-systems/agentic-workflows) repo documents the full Audit→Fix→Rediscover cycle — six complete iterations discovering 118 total findings in [brewprune](https://github.com/blackwell-systems/brewprune). Each audit discovers organically; fixed issues that regress are simply rediscovered.

## License

MIT
