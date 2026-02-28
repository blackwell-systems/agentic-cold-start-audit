# Sandbox Setup — Isolation Modes and Agent Access Patterns

This document explains how to choose an isolation mode for CLI tool UX auditing
and how to set up each mode.

---

## Mode Selection

The audit's core invariant: **the audit agent must not modify production state.**

Choose the isolation mechanism based on your tool's blast radius:

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

A sandbox Dockerfile typically uses a **multi-stage build**:

```
Stage 1 (builder)
  - Compiles the tool's binaries for the target platform

Stage 2 (runtime)
  - Installs the tool's dependencies (package managers, runtimes, etc.)
  - Installs a realistic set of packages/data for the audit
  - Copies compiled binaries from the builder stage
  - Runs as a non-root user if the tool or its dependencies require it
```

**Use real dependencies, not mocks.** Mocking produces predictable, sanitized output
that doesn't surface real edge cases. Real package managers have real dependency graphs,
real metadata, and real edge cases your tool must handle correctly.

### Example skeleton

```dockerfile
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/mytool ./cmd/mytool

FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl git sudo && rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash appuser && echo "appuser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
USER appuser
# Install tool dependencies here
COPY --from=builder --chown=appuser:appuser /out/mytool /usr/local/bin/mytool
CMD ["/bin/bash"]
```

### Build and run

```bash
docker build -t <image-name> -f <path/to/Dockerfile> .
docker run -d --name <container-name> --rm <image-name> sleep 3600
docker ps --filter name=<container-name>
```

### Cleanup

```bash
docker rm -f <container-name>   # stop and remove container
docker rmi <image-name>         # remove image
```

---

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

Repeat the `mktemp -d` setup — each round gets a fresh empty state, equivalent to a new user
running the tool for the first time.

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
