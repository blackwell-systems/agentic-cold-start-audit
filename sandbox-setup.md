# Sandbox Setup — Isolation Modes and Agent Access Patterns

This document explains how to choose an isolation mode for CLI tool UX auditing
and how to set up each mode.

---

## Mode Selection

The audit's core invariant: **the audit agent must not modify production state.**

Choose the isolation mechanism based on your tool's blast radius:

> **Optional: Enhanced protection with Claude Code sandboxing**
> Enable Claude Code's built-in sandboxing via `/sandbox` for additional OS-level isolation. This complements the protocol's exec prefix isolation and provides defense-in-depth against prompt injection or malicious dependencies. Not required — the protocol's isolation modes are sufficient on their own. See [Claude Code Sandboxing Docs](https://code.claude.com/docs/en/sandboxing) for details.

| Mode | When to use | Setup required |
|---|---|---|
| **Container** | Tool has destructive ops (remove, delete, system writes, package managers) | Docker running, Dockerfile, image build |
| **Local** | Tool writes only to self-managed state (own DB, config file pointed to by env var) | Temp directory, env var |
| **Worktree** | Tool reads/writes files in the current directory | Temp copy of the working directory |

**If in doubt, use `container`** — it's the most conservative isolation and correct for any tool.

Run `/cold-start-audit mode <tool-name>` to get an automated recommendation before choosing.

---

---

## Container Mode

The UX audit simulates a **new user** on a fresh machine. Running commands against the
developer's own environment would:

- Produce different output depending on what the dev has installed
- Risk modifying real data or state
- Not reflect what a first-time user would see (no prior setup, no history)

A Docker container gives a clean, reproducible environment every time. Use this mode
for any tool with destructive operations (remove, delete, system writes, package managers).

### Why a Container?

---

## Container Design

Container mode expects Dockerfiles following the **audit-ready Dockerfile convention**.

### Contract Requirements

Your Dockerfile must provide:

1. **Multi-stage build** (recommended, not required)
   - Keeps runtime image small
   - Separates build dependencies from runtime

2. **Non-root user with sudo**
   - User must not be root
   - Passwordless sudo configured
   - Tests realistic permissions and surfaces bugs

3. **Tool at standard path**
   - Binary: `/usr/local/bin/<tool-name>`
   - Must be executable and owned by container user

4. **Clean base environment**
   - Ubuntu 22.04 or Debian-based
   - Basic utilities: `curl`, `git`, `sudo`
   - No unnecessary build deps in final image

5. **Interactive shell**
   - `CMD ["/bin/bash"]` for agent access
   - No ENTRYPOINT that blocks shell access

### How to Create

**Option 1: Auto-generate**
```bash
/dockerfile-sandbox-gen <tool-name>
```
Uses [dockerfile-sandbox-gen](https://github.com/blackwell-systems/dockerfile-sandbox-gen) to create a contract-compliant Dockerfile automatically.

**Option 2: Manual creation**

Create `Dockerfile.sandbox` following the contract:

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
COPY --from=builder --chown=appuser:appuser /out/mytool /usr/local/bin/mytool
CMD ["/bin/bash"]
```

Adjust the build stage for your language (Rust, Python, Node, etc.).

**Option 3: Custom Dockerfile**

Use any Dockerfile that meets the contract requirements, specify with `--dockerfile PATH`:

```bash
/cold-start-audit container build mytool --dockerfile docker/custom.dockerfile
```

### Contract Validation

Test compliance:

```bash
docker build -t test-r1 -f Dockerfile.sandbox .
docker run --rm test-r1 bash -c "whoami && which mytool && sudo -n true && echo '✓ Contract valid'"
```

Expected output:
```
appuser
/usr/local/bin/mytool
✓ Contract valid
```

### Why These Requirements

- **Non-root user**: Simulates realistic user environment, not administrator
- **Sudo access**: Lets audit test elevated operations without always running as root
- **Clean base**: Real dependencies surface real edge cases (mocks hide bugs)
- **Standard path**: `/usr/local/bin` is always in PATH, conventional location


## Agent Access Patterns

There are two ways to give a Claude Code agent access to run commands inside the container.

### Pattern 1 — `docker exec` (recommended)

The agent runs on the **host** and uses `docker exec` to send commands into the container.

```
Agent on host  →  docker exec <container-name> <command>  →  Container
```

**Pros:**
- Agent has full access to host filesystem (can write reports, read project files)
- No networking required
- Agent can interleave container commands with host file reads/writes naturally
- Works with background agents

**Cons:**
- Requires Docker to be running and accessible on the host
- Agent must prefix every command with `docker exec <container-name>`

**Setup:** Just ensure the container is running. The agent prompt specifies the exec prefix.

---

### Pattern 2 — Bind mount (alternative)

Mount the host project directory into the container at `docker run` time.

```bash
docker run -d --name <container-name> --rm \
  -v /path/to/project:/project \
  <image-name> sleep 3600
```

Inside the container, `/project` mirrors the host directory in real time.

**Pros:**
- Agent can write output files from *inside* the container and they appear on the host
- Useful if the agent needs to run build tools that only exist inside the container
- No path translation needed for file writes

**Cons:**
- More complex setup — must pass volume flag at `docker run` time
- File permission issues can arise (host UID vs container UID mismatch)
- Container user must have write permission on the mounted path

**When to use:** When the container environment provides tools not available on the host
(e.g. Linux-only builds), and the agent's primary output is files on disk.

---

## Which Pattern to Use

| Situation | Use |
|---|---|
| Agent audits UX, writes report to host | Pattern 1 (`docker exec`) |
| Agent runs container-only builds, output goes to host | Pattern 2 (bind mount) |
| Agent needs both host files and container tools | Pattern 2 (bind mount) |
| Simplest possible setup | Pattern 1 (`docker exec`) |

For the UX audit workflow, **Pattern 1 is sufficient** — the agent sends commands via
`docker exec` and writes the report directly to the host filesystem via the Write tool.

---

## Colima Note

If Docker was installed via [Colima](https://github.com/abiosoft/colima) (common on macOS),
`docker compose` may not be available — the compose plugin ships separately.

Use plain `docker build` and `docker run` instead of `docker compose up`.
`docker exec`, `docker ps`, `docker rm`, and `docker rmi` all work normally.

---

## Local Mode

Use this mode when the tool writes only to self-managed state — a database, config file, or
data directory — that can be redirected to a temp location via an environment variable. No
Docker required; the audit agent runs the tool directly on the host.

**When to use:**
- Tool has a `--db`, `--config`, `--data-dir` flag, or equivalent env var
- All persistent state lives under one path the tool owns
- No destructive operations on system state (no package managers, no system files)

**Examples:**
- `commitmux` — state lives in `~/.commitmux/db.sqlite3`, redirectable via `COMMITMUX_DB`
- Any CLI tool with a `--config ~/.mytool/config.json` flag

### Setup

```bash
# Create an isolated temp directory for the audit
export AUDIT_DIR=$(mktemp -d)

# Verify the tool respects the env var
env MYTOOL_DB=$AUDIT_DIR/db <tool-name> --help

# Run the audit
/cold-start-audit run <tool-name> --mode local --env MYTOOL_DB=$AUDIT_DIR/db
```

### Exec prefix

```
env MYTOOL_DB=/tmp/audit-xxx/db
```

Every command the audit agent runs is prefixed with this, so the tool always reads/writes
the temp state, never the real one.

### Cleanup

```bash
rm -rf $AUDIT_DIR
```

### Subsequent rounds

Repeat the `mktemp -d` setup — each round gets a fresh empty state, equivalent to a new user running the tool for the first time. The agent has no memory of what it found before; it just runs commands and reports what happens.

---

## Worktree Mode

Use this mode when the tool reads or writes files in the current directory — file processors,
project scaffolders, tools that operate on a repo. The audit agent works inside a temp copy
of a directory, so the original is never modified.

**When to use:**
- Tool's primary operations are on files in `./`
- Tool is a file transformer, scaffolder, or repo-aware CLI
- No system-level state is modified (no package managers, no system config)

**Examples:**
- A code formatter that rewrites files in place
- A project scaffolding tool that generates a directory structure
- A changelog tool that reads and writes `CHANGELOG.md`

### Setup

```bash
# Copy the source directory to a temp location
export AUDIT_DIR=$(mktemp -d)
cp -r /path/to/source/. $AUDIT_DIR/

# Verify the tool works from there
cd $AUDIT_DIR && <tool-name> --help

# Run the audit
/cold-start-audit run <tool-name> --mode worktree --dir /path/to/source
```

The skill handles the `cp` step automatically when `--dir` is provided.

### Exec prefix

```
cd /tmp/audit-xxx &&
```

Every command the audit agent runs changes into the temp dir first, so file operations land
in the copy, not the original.

### Cleanup

```bash
rm -rf $AUDIT_DIR
```
