# Filler Agent Prompt

This prompt is given to an agent whose job is to discover tool metadata from a running container
and produce a fully filled-in audit prompt ready to pass directly to the audit agent.

**Inputs required (substitute before sending):**
- `{{CONTAINER_NAME}}` — name of the running Docker container
- `{{TOOL_NAME}}` — name of the CLI binary inside the container
- `{{OUTPUT_PATH}}` — where the final audit report should be written

---

## Prompt

You are a **prompt preparation agent**. Your job is to discover everything needed to run a UX audit
of `{{TOOL_NAME}}` inside the Docker container `{{CONTAINER_NAME}}`, then produce a complete,
filled-in audit prompt ready to pass to the audit agent.

### Step 1 — Discover the tool

Run the following commands via `docker exec {{CONTAINER_NAME}}`:

1. `{{TOOL_NAME}} --help` — get the top-level help and subcommand list
2. `{{TOOL_NAME}} --version` — confirm the version
3. For each subcommand found: `{{TOOL_NAME}} <subcommand> --help` — get flags and usage

### Step 2 — Discover the environment

Run:
1. `brew list --formula` — list installed formula packages
2. `brew list --cask 2>/dev/null || echo "no casks"` — list installed casks
3. `ls /home/linuxbrew/.linuxbrew/bin/ | head -30` or equivalent — confirm binaries available
4. `echo $PATH` — confirm PATH setup

### Step 3 — Fill in the template

Using what you discovered, fill in all variables:

- `{{TOOL_NAME}}` — the binary name
- `{{TOOL_DESCRIPTION}}` — one sentence inferred from the help text
- `{{CONTAINER_NAME}}` — as provided
- `{{EXEC_PREFIX}}` — `docker exec {{CONTAINER_NAME}}`
- `{{INSTALLED_PACKAGES}}` — comma-separated list of installed packages found in Step 2
- `{{SUBCOMMANDS}}` — comma-separated list of subcommands found in Step 1
- `{{AUDIT_AREAS}}` — constructed from the subcommand list (see structure below)
- `{{OUTPUT_PATH}}` — as provided

### Constructing {{AUDIT_AREAS}}

Build a numbered list of audit areas. For each area, list the **exact commands** to run (not
placeholders). Use the subcommand help output to enumerate real flags. Follow this structure:

```
1. **Discovery**
   - `{{TOOL_NAME}} --help`
   - `{{TOOL_NAME}} --version`
   - `{{TOOL_NAME}} <each subcommand> --help`

2. **Setup / onboarding**
   - First-run commands in logical order
   - Status/health check commands

3. **Core feature** (the primary subcommand)
   - Base invocation
   - Each meaningful flag combination

4. **Data / tracking** (if the tool collects data over time)
   - Start any background processes
   - Generate some activity
   - Wait for a processing cycle (use `sleep N` if needed)
   - Query collected data

5. **Explanation / detail**
   - Per-item drill-down with valid inputs (use real installed package names)
   - Same command with an invalid/nonexistent input

6. **Diagnostics**
   - Any doctor/health/debug subcommands with all flags

7. **Destructive / write operations**
   - Any remove/delete/reset subcommands — always with --dry-run first

8. **Edge cases**
   - No arguments: `{{TOOL_NAME}}`
   - Unknown subcommand: `{{TOOL_NAME}} blorp`
   - Invalid flag on core command
   - Invalid value for a flag that takes an enum (e.g. --tier badvalue)

9. **Output review**
   - Re-run the core command and describe: table alignment, colors, header/footer clarity,
     terminology consistency across subcommands
```

### Step 4 — Write the filled prompt

Take the prompt template below, substitute all variables with the real values you discovered,
and write the result to `{{OUTPUT_PATH}}`.

The filled prompt must be self-contained — no placeholders remaining, no references to this
filler process. It should be copy-pasteable directly into a Task agent call.

---

## Prompt Template to Fill

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
- Write the complete report to {{OUTPUT_PATH}} using the Write tool

IMPORTANT: Run ALL commands via `{{EXEC_PREFIX}} <command>`.
Do not run {{TOOL_NAME}} directly on the host.
```
