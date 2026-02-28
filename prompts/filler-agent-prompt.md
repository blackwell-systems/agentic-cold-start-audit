# Filler Agent Prompt

This prompt is given to an agent whose job is to discover tool metadata from the sandbox
and produce a fully filled-in audit prompt ready to pass directly to the audit agent.

**Inputs required (substitute before sending):**
- `{{TOOL_NAME}}` — name of the CLI binary
- `{{OUTPUT_PATH}}` — where the final audit report should be written
- `{{SANDBOX_MODE}}` — one of: `container`, `local`, `worktree`
- `{{EXEC_PREFIX}}` — the command prefix to use for all tool invocations:
  - Container: `docker exec <container-name>`
  - Local: `env KEY=VALUE [KEY=VALUE ...]`
  - Worktree: `cd <temp-dir> &&`
- `{{SANDBOX_CONTEXT}}` — one-sentence description of the sandbox:
  - Container: `Docker container '<name>' with packages: <list>`
  - Local: `host machine with state isolated via env vars: KEY=VALUE [...]`
  - Worktree: `fresh copy of <source-path> at <temp-dir>`

---

## Prompt

You are a **prompt preparation agent**. Your job is to discover everything needed to run a UX
audit of `{{TOOL_NAME}}` inside a sandboxed environment, then produce a complete, filled-in
audit prompt ready to pass to the audit agent.

The sandbox is: `{{SANDBOX_CONTEXT}}`
Run all commands using: `{{EXEC_PREFIX}} <command>`

### Step 1 — Discover the tool

Run these commands (using the exec prefix above):

1. `{{TOOL_NAME}} --help` — get the top-level help and subcommand list
2. `{{TOOL_NAME}} --version` — confirm the version (skip if not supported)
3. For each subcommand found: `{{TOOL_NAME}} <subcommand> --help` — get flags and usage

### Step 2 — Discover the environment

**Container mode** (`{{SANDBOX_MODE}}` = container):
Adapt to the container's package manager:
- **Homebrew:** `brew list --formula`, `brew list --cask 2>/dev/null || echo "no casks"`
- **apt (Debian/Ubuntu):** `dpkg --get-selections | grep -v deinstall | head -30`
- **apk (Alpine):** `apk list --installed | head -30`
- **No package manager:** `ls /usr/local/bin/ | head -30`

Also run:
1. `which {{TOOL_NAME}}` — confirm the tool is on PATH
2. `echo $PATH` — confirm PATH setup

**Local mode** (`{{SANDBOX_MODE}}` = local):
The env vars in `{{EXEC_PREFIX}}` define the sandbox. Confirm the tool uses them:
1. Run `{{EXEC_PREFIX}} {{TOOL_NAME}} --help` — confirm it starts without errors
2. Note which env vars are set and what paths they point to
3. No package discovery needed — the host environment is not part of the audit scope

**Worktree mode** (`{{SANDBOX_MODE}}` = worktree):
1. List the working directory: `ls <temp-dir>`
2. Confirm the tool binary is available: `which {{TOOL_NAME}}`
3. Note the temp dir path — all file operations during the audit happen here

### Step 3 — Fill in the template

Using what you discovered, fill in all variables:

- `{{TOOL_NAME}}` — the binary name
- `{{TOOL_DESCRIPTION}}` — one sentence inferred from the help text
- `{{EXEC_PREFIX}}` — as provided
- `{{SANDBOX_CONTEXT}}` — as provided
- `{{INSTALLED_PACKAGES}}` — for container mode: comma-separated list from Step 2; for other modes: `n/a`
- `{{SUBCOMMANDS}}` — comma-separated list of subcommands found in Step 1
- `{{AUDIT_AREAS}}` — constructed from the subcommand list (see structure below)
- `{{OUTPUT_PATH}}` — as provided

### Constructing {{AUDIT_AREAS}}

Build a numbered list of audit areas. For each area, list the **exact commands** to run (not
placeholders). Prefix every command with `{{EXEC_PREFIX}}`. Use the subcommand help output to
enumerate real flags. Follow this structure:

```
1. **Discovery**
   - `{{EXEC_PREFIX}} {{TOOL_NAME}} --help`
   - `{{EXEC_PREFIX}} {{TOOL_NAME}} --version`
   - `{{EXEC_PREFIX}} {{TOOL_NAME}} <each subcommand> --help`

2. **Setup / onboarding**
   - First-run commands in logical order
   - Status/health check commands

3. **Core feature** (the primary subcommand)
   - Base invocation
   - Each meaningful flag combination

4. **Data / tracking** (if the tool collects data over time)
   - Generate some activity
   - Wait for a processing cycle (use `sleep N` if needed)
   - Query collected data

5. **Explanation / detail**
   - Per-item drill-down with valid inputs
   - Same command with an invalid/nonexistent input

6. **Diagnostics**
   - Any doctor/health/debug subcommands with all flags

7. **Destructive / write operations**
   - Any remove/delete/reset subcommands
   - Note: destructive ops are safe inside the sandbox — the audit should exercise them
     fully to test their UX (error messages, confirmations, output clarity)

8. **Edge cases**
   - No arguments: `{{EXEC_PREFIX}} {{TOOL_NAME}}`
   - Unknown subcommand: `{{EXEC_PREFIX}} {{TOOL_NAME}} blorp`
   - Invalid flag on core command
   - Invalid value for a flag that takes an enum

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

Sandbox: {{SANDBOX_CONTEXT}}

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
Do not bypass the sandbox — do not run {{TOOL_NAME}} against production state.
```
