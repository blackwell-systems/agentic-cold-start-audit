# Background Agent UX Audit Workflow

A repeatable process for running an AI agent inside a sandboxed environment to audit CLI tool UX as a new user.

---

## Prerequisites

### 1. User-level permissions (one-time setup)

Background agents cannot prompt for tool approval. Without an explicit `allow` rule, every `Bash` call is denied and the agent silently fails.

Ensure `~/.claude/settings.json` uses the correct permissions format:

```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write", "mcp__*"]
  }
}
```

> **Common mistake:** `allow_bash: true` is not a valid Claude Code permission format and is silently ignored.

After editing `~/.claude/settings.json`, **restart your Claude Code session** before launching background agents. Settings are loaded at session start and do not hot-reload.

### 2. Project-level permissions (optional, scoped)

If you want to restrict background agent Bash access to specific commands only (e.g. just docker exec on a named container), add `.claude/settings.json` to the project repo:

```json
{
  "permissions": {
    "allow": ["Bash(docker exec <container-name>*)"]
  }
}
```

> **Note:** Project-level permissions require user approval before they are trusted — they cannot bootstrap themselves. Use user-level permissions for background agents that need to work without interaction.

### 3. Hooks

Broken hooks block all Bash calls for background agents. Before launching agents, verify any `PreToolUse` hooks on `Bash` actually exist on disk:

```bash
ls ~/claude/hooks/  # check referenced hook scripts exist
```

Remove or stub out any missing hooks in `~/.claude/settings.json`.

---

## The Workflow

### Step 1 — Build and start the sandbox

Build a Docker image with the tool installed in a realistic user environment.
See `sandbox-setup.md` for container design options and Dockerfile patterns.

```bash
docker build -t <image-name> -f <path/to/Dockerfile> .
docker run -d --name <container-name> --rm <image-name> sleep 3600
```

Verify the container is running:

```bash
docker ps --filter name=<container-name>
```

### Step 2 — Generate the audit prompt

Use the filler agent in `filler-agent-prompt.md` to auto-discover tool metadata and produce
a fully filled prompt. Provide it two inputs:

- `CONTAINER_NAME` — the running container name
- `TOOL_NAME` — the CLI binary to audit
- `OUTPUT_PATH` — where to write the final report

Or fill `prompt-template.md` manually if you prefer.

### Step 3 — Launch the background audit agent

Pass the filled prompt to a background Task agent:

```
Task tool:
  subagent_type: general-purpose
  run_in_background: true
  prompt: <filled audit prompt>
```

The agent will:
1. Run 30+ commands against the container
2. Observe output, errors, and behavior at each step
3. Compile findings in structured format
4. Write the final report to the specified output path

### Step 4 — Review findings

Triage by severity:

| Severity | Meaning | Action |
|---|---|---|
| UX-critical | Broken behavior or misleading output | Fix before next release |
| UX-improvement | Confusing but functional | Prioritize for next sprint |
| UX-polish | Minor friction | Batch into a cleanup PR |

---

## Checklist Before Launching Agent

- [ ] `~/.claude/settings.json` has `"allow": ["Bash", "Read", "Write", "mcp__*"]`
- [ ] Session was restarted after last settings change
- [ ] All `PreToolUse` hook scripts exist on disk (or hooks are removed)
- [ ] Sandbox container is running (`docker ps --filter name=...`)
- [ ] Audit prompt has been filled with tool-specific commands

---

## Files in This Folder

| File | Purpose |
|---|---|
| `workflow.md` | This file — the repeatable process |
| `sandbox-setup.md` | Container design, build steps, bind-mount vs docker exec |
| `prompt-template.md` | Generalized audit prompt with variable table |
| `filler-agent-prompt.md` | Agent that discovers tool metadata and fills the template |
| `cold-start-audit-skill.md` | Portable Claude Code `/cold-start-audit` skill |
