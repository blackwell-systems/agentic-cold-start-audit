Cold-Start UX Audit: AI agents simulate new users in sandboxed environments.

## Arguments

- `mode <tool-name>`: Analyze the tool's help output and recommend the appropriate sandbox isolation mode. Runs `<tool-name> --help` and each subcommand's `--help`, then answers three questions to produce a mode recommendation (container / local / worktree) with rationale. Does not write any files.
- `setup <tool-name> --mode container <container-name>`: Run the filler agent to discover tool metadata from the running container and produce a filled audit prompt. Write the filled prompt to `docs/cold-start-audit-prompt.md`.
- `setup <tool-name> --mode local --env KEY=VALUE [--env KEY=VALUE ...]`: Run the filler agent in local mode. Creates temp dir, sets env vars, discovers tool metadata from the host, produces filled prompt.
- `setup <tool-name> --mode worktree --dir <source-path>`: Run the filler agent in worktree mode. Copies `<source-path>` to a temp dir, discovers tool metadata from inside it, produces filled prompt.
- `run <tool-name> --mode <mode> [mode-args]`: Run the full audit. Checks for existing prompt and offers to reuse/adapt it or regenerate from scratch. Launch the audit agent as a background Task agent. Write the findings report to `docs/cold-start-audit.md`. Same mode args as `setup`.
- `report`: Read and summarize the findings from `docs/cold-start-audit.md`, grouped by severity.
- `init <container-name>`: Scaffold `.claude/settings.json` with scoped permissions for the container (container mode only).


## Sandbox Modes

Three modes satisfy the core invariant — **the audit agent must not modify production state** — using different isolation mechanisms:

| Mode | Argument form | When to use |
|---|---|---|
| `container` | `--mode container <container-name>` | Tool has destructive ops (remove, delete, system writes) |
| `local` | `--mode local --env KEY=VALUE [--env KEY=VALUE ...]` | Tool writes only to self-managed state (own DB or config) |
| `worktree` | `--mode worktree --dir <source-path>` | Tool reads/writes files in the current directory |

**Container mode** (default): audit agent prefixes every command with `docker exec <container-name>`. Requires Docker and a running sandbox container.

**Local mode**: audit agent sets env vars that redirect the tool's state to a temp directory. No Docker needed. Example: `--mode local --env COMMITMUX_DB=/tmp/audit-$$/db.sqlite3` isolates commitmux from the real index. The skill creates the temp directory before launching the filler agent.

**Worktree mode**: audit agent runs inside a fresh copy of a directory. Example: `--mode worktree --dir /path/to/project` gives the agent an isolated working tree with no risk to the real directory. The skill copies the source dir to a temp location before launching.

## Prompt Reuse Strategy

When running subsequent audits on the same tool:

**Check for existing prompt first:**
1. Look for `docs/cold-start-audit-prompt.md`
2. If found, read the metadata header to see the previous container name and date
3. Ask user: "Found existing prompt from [date] using container [name]. Options:
   - **Reuse** - Update only container name and date (fast, use when tool commands haven't changed)
   - **Regenerate** - Run filler agent to rediscover everything (use when tool structure changed)"

**When to reuse vs regenerate:**
- **Reuse** (recommended for iteration rounds): Tool structure is stable, just testing fixes from previous round. Only container name and date need updating.
- **Regenerate**: New subcommands added, flags changed, or first audit of the tool.

**Implementation:**
- For reuse: Use Edit tool to update container name throughout and metadata date
- For regenerate: Run the full filler agent as before

This prevents wasteful parallel execution of filler agent when audit agent doesn't need it.

## Before Running

Verify these prerequisites. If any fail, guide the user through fixing them.

**All modes:**
1. **Tool is installed on the host** (or in the container) — `<tool-name> --help` should succeed
2. **Bash permissions are configured** — check `.claude/settings.json` (see Permissions below)

**Container mode only:**
3. **Docker is running** — `docker ps` should succeed
4. **Sandbox container exists and is running** — `docker ps --filter name=<container-name>`
5. **Tool is installed inside the container** — `docker exec <container-name> <tool-name> --help`

If the container doesn't exist, help the user create one. See the Container Setup section in `sandbox-setup.md`.

**Local mode only:**
3. **Temp directory is created** — create it before launching: `mkdir -p $(dirname $STATE_PATH)`
4. **Tool accepts the env var** — verify with a dry run: `env KEY=VALUE <tool-name> --help`

**Worktree mode only:**
3. **Source directory exists** — `ls <source-path>` should succeed
4. **Temp copy is created** — the skill copies `<source-path>` to a temp dir before launching

## Permissions

Background agents cannot prompt for tool approval. Without an explicit `allow` rule, every Bash call is denied silently.

### Project-level (recommended, scoped)

Create `.claude/settings.json` in the project repo. Use a wildcard that covers all round numbers so it doesn't need updating each round:

```json
{
  "permissions": {
    "allow": ["Bash(docker exec <tool>-r*)"]
  }
}
```

Example for brewprune: `"Bash(docker exec brewprune-r*)"` covers r1 through r99 without ever needing to update the settings file.

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

### Container Naming Convention

Containers follow the pattern **`{tool}-r{N}`** where N is the audit round number (1, 2, 3…).

**Auto-increment logic** — before creating a new container, determine N automatically:

```bash
# Find the highest existing round number for this tool
docker images --format '{{.Repository}}' | grep '^{tool}-r' | sort -V | tail -1
# e.g. "brewprune-r7" → next round is brewprune-r8
```

If no existing images are found, start at r1.

### Container Reuse

**Before building a new image, check if a usable container already exists:**

```bash
docker ps --filter name={tool}-r{N} --format '{{.Names}}'
```

- **Container is running** → reuse it directly. Skip build and `docker run`. The binary is already in place.
- **Image exists but container is stopped** → start a new container from the existing image: `docker run -d --name {tool}-r{N} {tool}-r{N} sleep 3600`
- **No image exists** → build from scratch (see First-time setup below)

Only rebuild the image when source code has changed since the last audit round. If you just want to re-audit the same binary, reuse the existing image.

### Container Lifecycle

Understanding the distinction between these three concepts is critical:

- **Dockerfile** (template) - Infrastructure definition, created once and reused across audit rounds. Only update when runtime dependencies or environment setup changes.
- **Image** (snapshot) - Built from the Dockerfile, captures the tool's compiled binary at a point in time. Rebuild this when the tool's source code changes.
- **Container** (running instance) - Ephemeral execution environment. Create a new one for each audit round with a round-specific name following the `{tool}-r{N}` pattern.

### Dockerfile pattern (multi-stage build)

Create `Dockerfile.sandbox` once (or use `docker/Dockerfile.sandbox` if the project already has one):

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

### First-time setup

For the initial audit round:

```bash
# Create Dockerfile.sandbox (see pattern above)
# Build image with round-specific tag
docker build -t mytool-r1 -f Dockerfile.sandbox .
# Start container with round-specific name
docker run -d --name mytool-r1 --rm mytool-r1 sleep 3600
# Verify
docker ps --filter name=mytool-r1
```

### Subsequent rounds

After fixing issues from a previous round:

1. **Auto-increment** the round number (see Container Naming Convention above)
2. **Check for reuse** (see Container Reuse above) — if the binary hasn't changed, reuse the existing image
3. If source changed, rebuild:

```bash
# Determine next round number (e.g. r7 → r8)
PREV=mytool-r7
NEXT=mytool-r8
# Stop old container if running
docker rm -f $PREV 2>/dev/null || true
# Rebuild image from EXISTING Dockerfile with new round tag
docker build -t $NEXT -f docker/Dockerfile.sandbox .
# Start new container
docker run -d --name $NEXT $NEXT sleep 3600
```

**Key principle:** The Dockerfile.sandbox is stable infrastructure. Only the image (built binary) and container (running instance) change between rounds.

### When to update the Dockerfile

Only modify Dockerfile.sandbox when:
- Runtime dependencies change (new packages needed in the container)
- Tool installation method changes (different build flags, new subcommands to copy)
- Environment setup changes (different user permissions, PATH modifications)

Do NOT recreate the Dockerfile just because the tool's source code changed — rebuilding the image is sufficient.

### Cleanup

After completing an audit round:

```bash
# Remove container (happens automatically with --rm flag when container stops)
docker rm -f mytool-r1
# Optional: remove image to free disk space
docker rmi mytool-r1
```

## Mode Advisory (`mode <tool-name>`)

When the argument is `mode <tool-name>`, analyze the tool's help output and emit an isolation mode recommendation. Run the tool directly on the host (no sandbox yet):

### Step 1 — Discover commands and flags

1. `<tool-name> --help` — collect top-level subcommand list
2. For each subcommand: `<tool-name> <subcommand> --help` — collect flags

### Step 2 — Answer three diagnostic questions

**Q1: Does the tool have destructive operations?**
Look for subcommands or flags that: remove/delete/uninstall/reset/purge system state, write to package managers, modify system config, or send external requests (email, webhooks). If yes → lean toward **container**.

**Q2: Does the tool write only to self-managed state?**
Look for: `--db`, `--config`, `--data-dir`, or env vars (check `--help` text and env var mentions) that redirect all persistent state to a single path. If the tool's entire blast radius is one file or directory controlled by an env var → lean toward **local**.

**Q3: Does the tool operate on the current directory or file tree?**
Look for: commands that read/write files in `./`, git operations, file processors, or path arguments with no state isolation flag. If the tool's output is files on disk → lean toward **worktree**.

### Step 3 — Emit recommendation

```
Mode recommendation: container | local | worktree

Rationale: [2-3 sentences explaining which diagnostic questions triggered the recommendation]

Confidence: high | medium | low

If medium or low — explain what's ambiguous and how to resolve it.

Suggested command:
  /cold-start-audit setup <tool-name> --mode <mode> [mode-specific args]
```

If the tool clearly fits multiple modes (e.g., has both destructive ops AND a self-contained DB), recommend **container** — it's the most conservative isolation.

---

## Filler Agent

The filler agent discovers tool metadata and produces a filled audit prompt. The steps adapt per mode — but the output is always the same: a self-contained prompt with no placeholders.

### Step 1 — Discover the tool

**Container mode:** run via `docker exec <container-name>`
**Local mode:** run with env vars set — `env KEY=VALUE <tool-name> --help`
**Worktree mode:** run from inside the temp dir — `cd <temp-dir> && <tool-name> --help`

Commands to run (adjust exec prefix per mode):
1. `<tool-name> --help` — top-level help and subcommand list
2. `<tool-name> --version` — confirm the version
3. For each subcommand: `<tool-name> <subcommand> --help` — flags and usage

### Step 2 — Discover the environment

**Container mode:** adapt to the container's package manager:
- apt (Debian/Ubuntu): `dpkg --get-selections | grep -v deinstall | head -30`
- apk (Alpine): `apk list --installed | head -30`
- Homebrew: `brew list --formula`
- No package manager: `ls /usr/local/bin/ | head -30`

Also run: `which <tool-name>`, `echo $PATH`

**Local mode:** describe the env var isolation context — e.g., `COMMITMUX_DB=/tmp/audit-xxx/db.sqlite3`. List the env vars and their temp paths. No package discovery needed.

**Worktree mode:** describe the temp dir contents — `ls <temp-dir>`. Note the source path it was copied from.

### Step 3 — Construct audit areas

Build a numbered list of areas with exact commands (not placeholders). Use the exec prefix appropriate for the mode:

1. **Discovery** — `--help`, `--version`, `<subcommand> --help` for each
2. **Setup / onboarding** — first-run commands, initialization, status checks
3. **Core feature** — primary command with all flag combinations
4. **Data / tracking** — if the tool collects data: generate activity, wait, query
5. **Explanation / detail** — per-item drill-down with valid and invalid inputs
6. **Diagnostics** — doctor/health/debug subcommands with all flags
7. **Destructive / write operations** — remove/delete/reset (safe in sandbox — no --dry-run needed unless the tool provides it as a UX signal worth testing)
8. **Edge cases** — no args, unknown subcommand (`blorp`), invalid flags, invalid enum values
9. **Output review** — table alignment, colors, header/footer clarity, terminology consistency

### Step 4 — Write the filled prompt

Produce a self-contained audit prompt with all variables substituted. Write to `docs/cold-start-audit-prompt.md`. The filled prompt must follow this structure:

```
# Cold-Start UX Audit Report

**Metadata:**
- Audit Date: <YYYY-MM-DD>
- Tool Version: <tool-name> <version>
- Sandbox mode: container | local | worktree
- Sandbox: <container-name | env vars | temp dir path>
- Environment: <OS, or "host" for local/worktree mode>

---

You are performing a UX audit of <tool-name> — a tool that <description from help>.
You are acting as a **new user** encountering this tool for the first time.

<sandbox context — one of:>
  Container: You have access to a Docker container called `<container-name>` with <tool-name> installed and the following packages available: <installed packages>.
  Local: You are running <tool-name> on the host with state isolated to a temp directory via env vars: <KEY=VALUE list>. The tool's real data is unaffected.
  Worktree: You are working inside a fresh copy of <source-path> at <temp-dir>. The original directory is unaffected.

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

IMPORTANT: Run ALL commands via `<exec-prefix> <command>`.
Do not bypass the sandbox — do not run <tool-name> against production state.
```

## Launching the Audit Agent

The audit agent MUST run in a fresh context with zero knowledge of the project. Launch it as a background Task agent:

- subagent_type: general-purpose
- run_in_background: true
- prompt: the filled audit prompt content (read from `docs/cold-start-audit-prompt.md` or pass directly as text)

The agent will run 30+ commands in the sandbox, observe behavior at each step, and write the findings report to `docs/cold-start-audit.md`.

**IMPORTANT - Sequencing:**

When regenerating the prompt (not reusing):
1. Launch filler agent OR run filler steps directly
2. **Wait for completion** - filler must finish writing the prompt file
3. Read the completed prompt from the file
4. **Then** launch audit agent with that prompt content

**Anti-pattern to avoid:**
- ❌ Launching filler agent in background + immediately launching audit agent with manually-edited old prompt = wasteful parallel execution
- ✅ Either wait for filler to complete, OR skip filler entirely and reuse/adapt existing prompt

The audit agent receives the prompt as text in its Task invocation, so it doesn't depend on the file after launch. But launching both agents in parallel wastes compute on redundant work.

## Severity Tiers

| Tier | Meaning | Action |
|------|---------|--------|
| UX-critical | Broken, misleading, or blocks the user | Fix before next release |
| UX-improvement | Confusing but functional | Prioritize for next sprint |
| UX-polish | Minor friction or inconsistency | Batch into a cleanup PR |

The audit agent must run ALL commands via `docker exec <container-name>` - never run the tool directly on the host.
