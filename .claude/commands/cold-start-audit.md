Cold-Start UX Audit: AI agents simulate new users in containerized sandboxes.

## Arguments

- `setup <container-name> <tool-name>`: Run the filler agent to discover tool metadata from the running container and produce a filled audit prompt. Write the filled prompt to `docs/cold-start-audit-prompt.md`.
- `run <container-name> <tool-name>`: Run the full audit. If `docs/cold-start-audit-prompt.md` exists, use it. Otherwise, run the filler agent first to generate it. Launch the audit agent as a background Task agent. Write the findings report to `docs/cold-start-audit.md`.
- `report`: Read and summarize the findings from `docs/cold-start-audit.md`, grouped by severity.
- `init <container-name>`: Scaffold `.claude/settings.json` with scoped permissions for the container.

## Before Running

Verify these prerequisites. If any fail, guide the user through fixing them.

1. **Docker is running** - `docker ps` should succeed
2. **Sandbox container exists and is running** - `docker ps --filter name=<container-name>`
3. **Tool is installed inside the container** - `docker exec <container-name> <tool-name> --help`
4. **Bash permissions allow docker exec** - check `.claude/settings.json` (see Permissions below)

If the container doesn't exist, help the user create one. See the Container Setup section.

## Permissions

Background agents cannot prompt for tool approval. Without an explicit `allow` rule, every Bash call is denied silently.

### Project-level (recommended, scoped)

Create `.claude/settings.json` in the project repo:

```json
{
  "permissions": {
    "allow": ["Bash(docker exec <container-name>*)"]
  }
}
```

### User-level (broader, needed for background agents)

Ensure `~/.claude/settings.json` includes:

```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```

Common mistake: `allow_bash: true` is not valid and is silently ignored.

After editing settings, restart the Claude Code session. Settings do not hot-reload.

## Container Setup

The audit simulates a new user on a fresh machine. A Docker container gives a clean, reproducible environment.

### Dockerfile pattern (multi-stage build)

```dockerfile
# Stage 1: build the tool
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/mytool ./cmd/mytool

# Stage 2: clean runtime environment
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl git sudo && rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash appuser && echo "appuser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
USER appuser
# Install tool dependencies here
COPY --from=builder --chown=appuser:appuser /out/mytool /usr/local/bin/mytool
CMD ["/bin/bash"]
```

Adapt the builder stage and dependencies for the tool's language and requirements. Use real dependencies, not mocks - real package managers surface real edge cases.

### Build and run

```bash
docker build -t <image-name> -f Dockerfile.sandbox .
docker run -d --name <container-name> --rm <image-name> sleep 3600
docker ps --filter name=<container-name>
```

### Cleanup

```bash
docker rm -f <container-name>
docker rmi <image-name>
```

## Filler Agent

The filler agent discovers tool metadata from the running container and produces a filled audit prompt.

When running the filler agent, follow these steps:

### Step 1 - Discover the tool

Run via `docker exec <container-name>`:

1. `<tool-name> --help` - get top-level help and subcommand list
2. `<tool-name> --version` - confirm the version
3. For each subcommand: `<tool-name> <subcommand> --help` - get flags and usage

### Step 2 - Discover the environment

Adapt to the container's package manager:
- apt (Debian/Ubuntu): `dpkg --get-selections | grep -v deinstall | head -30`
- apk (Alpine): `apk list --installed | head -30`
- Homebrew: `brew list --formula`
- No package manager: `ls /usr/local/bin/ | head -30`

Also run:
1. `which <tool-name>` - confirm on PATH
2. `echo $PATH` - confirm PATH setup

### Step 3 - Construct audit areas

Build a numbered list of areas with exact commands (not placeholders):

1. **Discovery** - `--help`, `--version`, `<subcommand> --help` for each
2. **Setup / onboarding** - first-run commands, initialization, status checks
3. **Core feature** - primary command with all flag combinations
4. **Data / tracking** - if the tool collects data: start background processes, generate activity, wait, query
5. **Explanation / detail** - per-item drill-down with valid and invalid inputs
6. **Diagnostics** - doctor/health/debug subcommands with all flags
7. **Destructive / write operations** - remove/delete/reset, always with --dry-run first
8. **Edge cases** - no args, unknown subcommand (`blorp`), invalid flags, invalid enum values
9. **Output review** - table alignment, colors, header/footer clarity, terminology consistency

### Step 4 - Write the filled prompt

Produce a self-contained audit prompt with all variables substituted. Write to `docs/cold-start-audit-prompt.md`. The filled prompt should follow this structure:

```
You are performing a UX audit of <tool-name> - a tool that <description from help>.
You are acting as a **new user** encountering this tool for the first time.

You have access to a Docker container called `<container-name>` with <tool-name> installed
and the following packages available: <installed packages>.

Run all commands using: `docker exec <container-name> <command>`

## Audit Areas

<filled audit areas with exact commands>

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
- Write the complete report to docs/cold-start-audit.md using the Write tool

IMPORTANT: Run ALL commands via `docker exec <container-name> <command>`.
Do not run <tool-name> directly on the host.
```

## Launching the Audit Agent

The audit agent MUST run in a fresh context with zero knowledge of the project. Launch it as a background Task agent:

- subagent_type: general-purpose
- run_in_background: true
- prompt: the filled audit prompt from `docs/cold-start-audit-prompt.md`

The agent will run 30+ commands against the container, observe behavior at each step, and write the findings report to `docs/cold-start-audit.md`.

## Severity Tiers

| Tier | Meaning | Action |
|------|---------|--------|
| UX-critical | Broken, misleading, or blocks the user | Fix before next release |
| UX-improvement | Confusing but functional | Prioritize for next sprint |
| UX-polish | Minor friction or inconsistency | Batch into a cleanup PR |

The audit agent must run ALL commands via `docker exec <container-name>` - never run the tool directly on the host.
