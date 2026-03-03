# Cold-Start Audit Specification

Formal specification for the agentic-cold-start-audit protocol. Defines participant roles, invariants, preconditions, and correctness guarantees. Last updated 2026-03-02.

## Participants

**Skill (Orchestrator):** The synchronous agent running in the user's own Claude Code session. Interprets the slash command, checks preconditions, runs the mode advisory if requested, and launches the Filler and Audit agents. Human confirmation is preserved at the launch step — the skill does not advance to the audit agent without user review of the filled prompt.

**Filler Agent:** An asynchronous agent launched by the skill. Reads the tool's help output (host-side, no sandbox needed), discovers all subcommands and flags, and populates the audit prompt template with real commands and the correct sandbox exec prefix. Produces `docs/cold-start-audit-prompt.md`. Does not execute commands inside the sandbox.

**Audit Agent:** An asynchronous agent launched by the skill after the filled prompt is ready. Executes every subcommand and flag documented in the filled prompt, inside the configured sandbox, as a simulated new user. Records every output, error, and friction point. Produces `docs/cold-start-audit.md`.

## Execution Model

```
User invokes /cold-start-audit run <tool> --mode <mode> [args]
        │
        ▼
Skill (Orchestrator) — synchronous
  1. Check preconditions (tool installed, sandbox reachable, permissions set)
  2. Check for existing prompt → offer reuse or regenerate
        │
        ▼
Filler Agent — asynchronous, host-side
  3. Read tool --help and all subcommand --help output
  4. Populate prompt template → write docs/cold-start-audit-prompt.md
        │
        ▼
Skill — resume, show prompt to user, wait for confirmation
        │
        ▼
Audit Agent — asynchronous, sandbox-isolated
  5. Execute every command in the filled prompt through the isolation layer
  6. Record outputs, errors, friction → write docs/cold-start-audit.md
        │
        ▼
Skill — report summary
```

## Invariants

### I1 — Sandbox Isolation

The audit agent must not modify production state. Every command executed by the audit agent goes through the configured isolation layer before reaching the host:

- **Container mode:** all commands prefixed with `docker exec <container-name>`
- **Local mode:** tool's state directories redirected to a temp path via env vars before launch
- **Worktree mode:** tool operates inside a fresh copy of the source directory

Violation: audit agent executing commands directly against the host filesystem or database without isolation. The skill enforces this by constructing the exec prefix before launching the audit agent and embedding it in the filled prompt — the agent cannot bypass it without ignoring its own instructions.

### I2 — Filler Agent is Read-Only on the Sandbox

The filler agent reads help output from the host (not the sandbox) and does not execute commands inside the sandbox. It never invokes the exec prefix.

Violation: filler agent running tool commands with `docker exec` or inside the temp directory.

### I3 — Findings Must Be Reproducible

Every finding in the report must include exact reproduction steps: the specific command that triggered the issue, the output or error observed, and the expected behavior. Vague findings ("output is confusing") without a reproduction path are not valid findings.

### I4 — Mode Selection Must Be Explicit

The audit agent is launched with a specific, named sandbox mode. "Use whatever works" is not a mode. The skill enforces this by requiring `--mode container`, `--mode local`, or `--mode worktree` on every `run` and `setup` invocation. The `mode` subcommand exists to derive this value when the user is unsure; the derived recommendation is shown to the user before any agents launch.

## Mode Selection Algorithm

The `mode` subcommand answers three diagnostic questions to recommend an isolation mode:

**Q1: Does the tool have destructive operations?**
Check `--help` for commands named `remove`, `delete`, `clean`, `uninstall`, `prune`, `drop`, `destroy`, or similar. Also check for system-level operations: package installation, file system writes outside the tool's own config dir, process management.

- If yes → **container mode** (strongest isolation, prevents all host-side side effects)

**Q2: Does the tool write only to self-managed state?**
Check whether the tool's writes can be redirected via environment variable — look for `--db`, `--data-dir`, `--config`, or similar flags, or env var documentation in `--help`.

- If yes (and Q1 is no) → **local mode** (no Docker needed, redirect via env var)

**Q3: Does the tool operate on the current directory or file tree?**
Check for commands that read, write, or analyze files in the working directory: `init`, `add`, `index`, `analyze`, `check`.

- If yes (and Q1-Q2 are no) → **worktree mode** (copy the directory, test inside the copy)

If none of the above — recommend local mode with a note that the tool may be stateless (no isolation needed, but local mode is safe by default).

## Preconditions

These must hold before launching agents:

1. **Tool is installed and runnable** — `<tool-name> --help` must succeed on the host.
2. **Bash permissions are configured** — `~/.claude/settings.json` must include `"Bash"` in the allow list. See [README.md#permissions](README.md#permissions).
3. **Claude Code session was restarted after last settings change** — permissions are loaded at session start.
4. **Sandbox is reachable (mode-specific):**
   - Container mode: Docker is running (`docker ps` succeeds) and the named container exists.
   - Local mode: the temp directory path is writable.
   - Worktree mode: the source directory path exists and is readable.

If any precondition fails, the skill reports the failure and stops. It does not attempt to fix preconditions automatically.

## Correctness Guarantee

When all preconditions hold and all invariants are maintained, the protocol provides this guarantee:

**If the audit agent completes without violating I1, every finding in the report was produced by commands that ran inside the sandbox.** The host's production state was not modified. The findings reflect the tool's behavior from a new user's perspective, not from within a broken or tainted environment.

This is not a guarantee of finding completeness — the audit covers the commands documented in `--help`. Commands with undocumented behavior or interactive prompts may be missed. **Coverage gaps are acceptable.** The goal is discovering salient friction that new users actually trip over by following help text, not exhaustive validation of all possible behaviors.

## Prompt Reuse Policy

On subsequent audits of the same tool:

- **Reuse** (recommended for iteration rounds): If `docs/cold-start-audit-prompt.md` exists and the tool's command structure hasn't changed since it was generated, update only the container name and date. Skips re-running the filler agent. The audit agent still operates with zero context — reuse only skips the help text discovery phase, not the audit execution.
- **Regenerate**: If new subcommands have been added, flags have changed, or this is the first audit, run the full filler agent.

The skill checks for an existing prompt and presents the option; the user decides. This preserves human review at the prompt stage.

**Important:** Prompt reuse is a performance optimization, not a change in methodology. The audit agent never has knowledge of previous findings — it discovers friction organically by executing commands and observing output.
