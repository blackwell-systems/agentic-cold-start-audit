Cold-Start UX Audit: AI agents simulate new users in sandboxed environments.

## Arguments

- `mode <tool-name>`: Analyze the tool's help output and recommend the appropriate sandbox isolation mode. Runs `<tool-name> --help` and each subcommand's `--help`, then answers three questions to produce a mode recommendation (container / local / worktree) with rationale. Does not write any files.
- `preflight <tool-name> --mode <mode> [mode-args]`: Validate all prerequisites before running an audit. Checks tool installation, permissions, Docker status (container mode), and provides actionable fix steps if any checks fail. Auto-run before `setup` and `run`.
- `container build <tool-name> [--dockerfile PATH]`: Auto-increment round number, check for reusable containers/images, build new image if needed, and start container. Detects highest existing round (e.g., brewprune-r7) and creates next (brewprune-r8). Offers to reuse if container already running.
- `container list <tool-name>`: Show all images and containers for a tool with their status (running/stopped/dangling).
- `container cleanup <tool-name> [--keep-latest N]`: Remove old containers and images, optionally keeping the latest N rounds. Default: keep latest 2.
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
- **Reuse** (recommended for iteration rounds): Tool structure is stable, no new commands or flags. Only container name and date need updating. The agent still discovers organically — reuse just skips re-reading help text.
- **Regenerate**: New subcommands added, flags changed, or first audit of the tool.

**Implementation:**
- For reuse: Use Edit tool to update container name throughout and metadata date
- For regenerate: Run the full filler agent as before

This prevents wasteful parallel execution of filler agent when audit agent doesn't need it.

## Preflight Validation (`preflight <tool-name> --mode <mode> [args]`)

Validate all prerequisites before launching agents. Automatically run before `setup` and `run` commands. Fail early with actionable fix steps.

### All Modes - Common Checks

**1. Tool installation:**
```bash
<tool-name> --help
```
- **✓ Pass:** Command succeeds, help text displayed
- **✗ Fail:** Command not found or errors
  - **Fix:** Install the tool first. Check installation docs.

**2. Bash permissions:**
```bash
# Check ~/.claude/settings.json for:
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```
- **✓ Pass:** "Bash" in allow list
- **✗ Fail:** Missing or using invalid format (`allow_bash: true`)
  - **Fix:**
    ```bash
    # Edit ~/.claude/settings.json and add:
    {
      "permissions": {
        "allow": ["Bash", "Read", "Write"]
      }
    }
    # Then restart Claude Code session (settings don't hot-reload)
    ```

**3. Session restart check:**
- **Warn:** "If you edited settings recently, restart Claude Code now. Settings are loaded at session start only."

**4. Hooks check:**
```bash
# Parse ~/.claude/settings.json for PreToolUse hooks on Bash
# For each hook script referenced, check:
ls <hook-script-path>
```
- **✓ Pass:** All hook scripts exist
- **✗ Fail:** Hook script missing
  - **Fix:** Remove hook reference from settings or create stub script

### Container Mode - Additional Checks

**5. Docker running:**
```bash
docker ps
```
- **✓ Pass:** Command succeeds, shows container list
- **✗ Fail:** Cannot connect to Docker daemon
  - **Fix:**
    ```bash
    # macOS/Linux:
    # Check if Docker Desktop is running, or start docker service
    sudo systemctl start docker  # Linux
    # macOS: Open Docker Desktop app
    ```

**6. Container exists and is running:**
```bash
docker ps --filter name=<container-name> --format '{{.Names}}'
```
- **✓ Pass:** Container name appears in output
- **✗ Fail:** No output (container not running)
  - **Check if container exists but is stopped:**
    ```bash
    docker ps -a --filter name=<container-name> --format '{{.Names}}'
    ```
  - **Fix (container exists):**
    ```bash
    docker start <container-name>
    ```
  - **Fix (container doesn't exist):**
    ```bash
    # Use the container build command:
    /cold-start-audit container build <tool-name>

    # Or manually:
    docker run -d --name <container-name> <image-name> sleep 3600
    ```

**7. Tool installed in container:**
```bash
docker exec <container-name> <tool-name> --help
```
- **✓ Pass:** Help text displayed
- **✗ Fail:** Command not found in container
  - **Fix:** Rebuild image with tool installed. Check Dockerfile.sandbox.

### Local Mode - Additional Checks

**8. Temp directory writable:**
```bash
# Extract temp path from env var args
TEMP_DIR=$(echo "<env args>" | grep -oP '/tmp/[^"]+' | head -1 | xargs dirname)
mkdir -p $TEMP_DIR && touch $TEMP_DIR/.test && rm $TEMP_DIR/.test
```
- **✓ Pass:** Directory created and writable
- **✗ Fail:** Permission denied
  - **Fix:**
    ```bash
    # Use a path you have write access to:
    mkdir -p /tmp/audit-$$
    # Then use: --env MYTOOL_DB=/tmp/audit-$$/db
    ```

**9. Tool respects env var:**
```bash
env <KEY=VALUE> <tool-name> --help
```
- **✓ Pass:** Command succeeds with env var set
- **✗ Fail:** Tool errors or ignores env var
  - **Fix:** Check tool docs for correct env var name. Tool may not support env var isolation.

### Worktree Mode - Additional Checks

**10. Source directory exists:**
```bash
ls <source-path>
```
- **✓ Pass:** Directory exists and readable
- **✗ Fail:** No such file or directory
  - **Fix:** Check the path. Use absolute path or correct relative path.

### Output Format

When all checks pass:
```
✓ Preflight checks passed (10/10)

All prerequisites met. Ready to run audit.

Next: /cold-start-audit run <tool-name> --mode <mode> [args]
```

When checks fail:
```
✗ Preflight checks failed (7/10 passed)

Failures:
  ✗ Docker not running
    Fix: Start Docker Desktop or run: sudo systemctl start docker

  ✗ Container brewprune-r8 not found
    Fix: /cold-start-audit container build brewprune
    Or manually: docker run -d --name brewprune-r8 brewprune-r8 sleep 3600

  ✗ Tool not installed in container
    Fix: Rebuild image. Check Dockerfile.sandbox includes tool installation.

Run these fixes, then retry: /cold-start-audit preflight <tool-name> --mode <mode> [args]
```

**Auto-integration:** `setup` and `run` commands automatically run preflight first. If preflight fails, stop and show fix steps. User must resolve before continuing.

## Container Lifecycle Management

### `container build <tool-name> [--dockerfile PATH]`

Automate the container build workflow: auto-increment round number, check for reusable containers, build and start.

**Step 1: Auto-detect round number**
```bash
# Find highest existing round
HIGHEST=$(docker images --format '{{.Repository}}' | grep "^${TOOL_NAME}-r" | sed 's/.*-r//' | sort -n | tail -1)

if [ -z "$HIGHEST" ]; then
  NEXT_ROUND=1
else
  NEXT_ROUND=$((HIGHEST + 1))
fi

CONTAINER_NAME="${TOOL_NAME}-r${NEXT_ROUND}"
IMAGE_NAME="${TOOL_NAME}-r${NEXT_ROUND}"
```

**Step 2: Check for reusable container**
```bash
RUNNING=$(docker ps --filter name=^${CONTAINER_NAME}$ --format '{{.Names}}')

if [ -n "$RUNNING" ]; then
  echo "✓ Container $CONTAINER_NAME already running"
  echo "Reusing existing container. No build needed."
  echo ""
  echo "Next: /cold-start-audit run $TOOL_NAME --mode container $CONTAINER_NAME"
  exit 0
fi
```

**Step 3: Check for existing image (source unchanged)**
```bash
IMAGE_EXISTS=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep "^${IMAGE_NAME}:")

if [ -n "$IMAGE_EXISTS" ]; then
  echo "? Image $IMAGE_NAME exists. Options:"
  echo "  1. Reuse existing image (fast, use if source code unchanged)"
  echo "  2. Rebuild image (use if source code changed since last audit)"

  # Ask user or default to reuse
  read -p "Choice [1/2]: " CHOICE

  if [ "$CHOICE" = "1" ]; then
    echo "Starting container from existing image..."
    docker run -d --name $CONTAINER_NAME --rm $IMAGE_NAME sleep 3600
    echo "✓ Container $CONTAINER_NAME started"
    exit 0
  fi
fi
```

**Step 4: Build new image**
```bash
# Find Dockerfile
DOCKERFILE_PATH="${DOCKERFILE_ARG:-Dockerfile.sandbox}"
if [ ! -f "$DOCKERFILE_PATH" ]; then
  DOCKERFILE_PATH="docker/Dockerfile.sandbox"
fi

if [ ! -f "$DOCKERFILE_PATH" ]; then
  echo "✗ Error: Dockerfile not found at Dockerfile.sandbox or docker/Dockerfile.sandbox"
  echo ""
  echo "Container mode requires an audit-ready Dockerfile with:"
  echo "  - Multi-stage build (builder + runtime)"
  echo "  - Non-root user with sudo access"
  echo "  - Tool at /usr/local/bin/<tool-name>"
  echo ""
  echo "Options:"
  echo "  1. Generate: /dockerfile-sandbox-gen <tool-name>"
  echo "  2. Manual: Create Dockerfile.sandbox (see sandbox-setup.md for template)"
  echo "  3. Custom: Use --dockerfile PATH to specify a different file"
  echo ""
  echo "Contract details:"
  echo "  https://github.com/blackwell-systems/agentic-cold-start-audit/blob/main/sandbox-setup.md#dockerfile-pattern"
  exit 1
fi

echo "Building image $IMAGE_NAME from $DOCKERFILE_PATH..."
docker build -t $IMAGE_NAME -f $DOCKERFILE_PATH .

if [ $? -ne 0 ]; then
  echo "✗ Build failed"
  exit 1
fi

echo "✓ Image built: $IMAGE_NAME"
```

**Step 5: Start container**
```bash
echo "Starting container..."
docker run -d --name $CONTAINER_NAME --rm $IMAGE_NAME sleep 3600

if [ $? -ne 0 ]; then
  echo "✗ Failed to start container"
  exit 1
fi

echo "✓ Container started: $CONTAINER_NAME"
echo ""
echo "Next: /cold-start-audit run $TOOL_NAME --mode container $CONTAINER_NAME"
```

**Output example:**
```
Auto-detected round: brewprune-r8 (highest existing: brewprune-r7)

Building image brewprune-r8 from Dockerfile.sandbox...
[docker build output...]
✓ Image built: brewprune-r8

Starting container...
✓ Container started: brewprune-r8

Next: /cold-start-audit run brewprune --mode container brewprune-r8
```

### `container list <tool-name>`

Show all images and containers for a tool with status.

**Implementation:**
```bash
TOOL_NAME="$1"

echo "Images for $TOOL_NAME:"
echo ""
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}' | grep "^${TOOL_NAME}-r"

echo ""
echo "Containers for $TOOL_NAME:"
echo ""
docker ps -a --filter name="^${TOOL_NAME}-r" --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
```

**Output example:**
```
Images for brewprune:

REPOSITORY      TAG     SIZE    CREATED
brewprune-r7    latest  156MB   2 days ago
brewprune-r8    latest  157MB   5 hours ago

Containers for brewprune:

NAMES           STATUS                  IMAGE
brewprune-r7    Exited (0) 2 days ago   brewprune-r7
brewprune-r8    Up 5 hours              brewprune-r8
```

### `container cleanup <tool-name> [--keep-latest N]`

Remove old containers and images, keeping the latest N rounds.

**Implementation:**
```bash
TOOL_NAME="$1"
KEEP_LATEST="${KEEP_LATEST:-2}"  # Default: keep 2 most recent

# Get all rounds sorted
ALL_ROUNDS=$(docker images --format '{{.Repository}}' | grep "^${TOOL_NAME}-r" | sed 's/.*-r//' | sort -n)
TOTAL_ROUNDS=$(echo "$ALL_ROUNDS" | wc -l)

if [ "$TOTAL_ROUNDS" -le "$KEEP_LATEST" ]; then
  echo "✓ Only $TOTAL_ROUNDS rounds exist. No cleanup needed (keeping latest $KEEP_LATEST)."
  exit 0
fi

ROUNDS_TO_REMOVE=$(echo "$ALL_ROUNDS" | head -n $((TOTAL_ROUNDS - KEEP_LATEST)))

echo "Found $TOTAL_ROUNDS rounds. Keeping latest $KEEP_LATEST."
echo ""
echo "Rounds to remove:"
for ROUND in $ROUNDS_TO_REMOVE; do
  echo "  - ${TOOL_NAME}-r${ROUND}"
done
echo ""
read -p "Continue? [y/N]: " CONFIRM

if [ "$CONFIRM" != "y" ]; then
  echo "Cancelled."
  exit 0
fi

for ROUND in $ROUNDS_TO_REMOVE; do
  NAME="${TOOL_NAME}-r${ROUND}"

  # Remove container (if exists)
  docker rm -f $NAME 2>/dev/null && echo "✓ Removed container: $NAME" || echo "  (no container)"

  # Remove image
  docker rmi $NAME 2>/dev/null && echo "✓ Removed image: $NAME" || echo "✗ Failed to remove image: $NAME"
done

echo ""
echo "✓ Cleanup complete"
```

**Output example:**
```
Found 8 rounds. Keeping latest 2.

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
  (no container)
✓ Removed image: brewprune-r2
...

✓ Cleanup complete
```

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

**Auto-increment logic** — use the `container build` command which handles this automatically, or manually:

```bash
# Find the highest existing round number for this tool
docker images --format '{{.Repository}}' | grep '^{tool}-r' | sort -V | tail -1
# e.g. "brewprune-r7" → next round is brewprune-r8
```

If no existing images are found, start at r1.

### Container Reuse

The `container build` command handles reuse automatically. It checks if a container is already running and offers to reuse it instead of rebuilding.

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

For the initial audit round, use the automated command:

```bash
/cold-start-audit container build <tool-name>
```

Or manually:

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

Use the automated workflow:

```bash
/cold-start-audit container build <tool-name>
# Auto-detects highest round, offers reuse if unchanged, builds if needed
```

**Key principle:** The Dockerfile.sandbox is stable infrastructure. Only the image (built binary) and container (running instance) change between rounds.

### When to update the Dockerfile

Only modify Dockerfile.sandbox when:
- Runtime dependencies change (new packages needed in the container)
- Tool installation method changes (different build flags, new subcommands to copy)
- Environment setup changes (different user permissions, PATH modifications)

Do NOT recreate the Dockerfile just because the tool's source code changed — rebuilding the image is sufficient.

### Cleanup

Use the automated command:

```bash
/cold-start-audit container cleanup <tool-name> --keep-latest 2
```

Or manually:

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

Suggested commands:
  /cold-start-audit container build <tool-name>  # (if container mode)
  /cold-start-audit run <tool-name> --mode <mode> [mode-specific args]
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
# Cold-Start UX Audit Prompt

**Metadata:**
- Audit Date: <YYYY-MM-DD>
- Tool: <tool-name>
- Tool Version: <version from --version>
- Sandbox mode: container | local | worktree
- Sandbox: <container-name | env vars | temp dir path>
- Exec prefix: `<exec-prefix>`

---

You are performing a UX audit of <tool-name> — a tool that <description from help>.
You are acting as a **new user** encountering this tool for the first time.

Sandbox: <sandbox context>

Run all commands using: `<exec-prefix> <command>`

## Audit Areas

<filled audit areas with exact commands>

Run ALL commands listed. Do not skip areas.
Note exact output, errors, exit codes, and behavior at each step.
Describe color usage (e.g. "package names appear in bold white, tier labels in green/yellow/red").

You are not trying to find every possible issue — you are discovering what friction a new user would naturally encounter by following the help text and trying obvious commands.

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

**CRITICAL - Do NOT add regression verification sections:**

The prompt template above is complete. DO NOT append:
- Regression verification sections
- Checklists of known issues to validate
- "Verify the following fixed behaviors" sections
- Comparisons to previous audit rounds
- Any content that gives the audit agent prior knowledge of what issues to look for

Every audit must be pure discovery with zero context. If old issues regressed, they surface through organic rediscovery (agent encounters the same friction) not through explicit verification checklists. Adding regression sections violates the methodology's core principle: the agent simulates a new user with no prior knowledge.

### Step 5 — Validate the filled prompt

Before returning, check for unfilled placeholders:

```bash
grep -o '{{[^}]*}}' docs/cold-start-audit-prompt.md
```

If any placeholders remain, report them as errors and do NOT launch the audit agent. Common issues:
- `{{TOOL_NAME}}` — forgot to substitute tool name
- `{{EXEC_PREFIX}}` — forgot to substitute docker exec or env prefix
- `{{AUDIT_AREAS}}` — didn't construct the command list

**Fix:** Re-run filler agent or manually edit the prompt file.

## Agent Type Preference

Use custom agent types when launching agents. These provide:
- Tool-level enforcement (filler cannot execute in sandbox, audit cannot bypass isolation)
- Behavioral instructions in the type definition
- Better observability in claudewatch metrics (separate success rates per agent type)

**Filler agent:** Use `subagent_type: filler` when spawning the metadata discovery agent.
**Audit agent:** Use `subagent_type: audit` when spawning the sandbox execution agent.

If these custom types fail to load, automatically fall back to `subagent_type: general-purpose` with the full prompt content from `prompts/agents/filler.md` or `prompts/agents/audit.md`.

Custom agent types are **optional** — users can install them for enhanced observability:
```bash
mkdir -p ~/.claude/agents
ln -sf $(pwd)/prompts/agents/filler.md ~/.claude/agents/filler.md
ln -sf $(pwd)/prompts/agents/audit.md ~/.claude/agents/audit.md
```

## Launching the Filler Agent

The filler agent discovers tool metadata and produces the filled audit prompt. Launch as a background Task agent:

- subagent_type: filler (falls back to general-purpose if not installed)
- run_in_background: true
- prompt: Instructions with {{TOOL_NAME}}, {{SANDBOX_MODE}}, {{EXEC_PREFIX}}, {{SANDBOX_CONTEXT}}, {{OUTPUT_PATH}} substituted

## Launching the Audit Agent

The audit agent MUST run in a fresh context with zero knowledge of the project. Launch it as a background Task agent:

- subagent_type: audit (falls back to general-purpose if not installed)
- run_in_background: true
- prompt: the filled audit prompt content (read from `docs/cold-start-audit-prompt.md` or pass directly as text)

The agent will run 30+ commands in the sandbox, observe behavior at each step, and write the findings report to `docs/cold-start-audit.md`.

**IMPORTANT - Sequencing:**

When regenerating the prompt (not reusing):
1. **Run preflight first** — auto-validates prerequisites
2. Launch filler agent OR run filler steps directly
3. **Wait for completion** - filler must finish writing the prompt file
4. **Validate prompt** - check for unfilled placeholders
5. **Then** launch audit agent with that prompt content

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

The audit agent must run ALL commands via the exec prefix - never bypass the sandbox.
