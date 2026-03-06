# CLAUDE.md — AI Agent Instructions for agentic-cold-start-audit

## Project Overview

This repository defines a protocol for discovering UX friction in CLI tools using AI agents in sandboxed environments. The core methodology: simulate a new user with zero prior knowledge, run commands in isolation, and report every friction point encountered.

**Not a test framework.** This is pure discovery, not regression testing. Each audit is organic — the agent follows help text, tries obvious commands, and reports what breaks. No memory between rounds, no checklists of known issues.

## Documentation Structure

Read these in order to understand the system:

1. **README.md** — Protocol overview, use cases, quickstart, permissions
2. **SPEC.md** — Formal specification: participants, invariants, preconditions, correctness guarantees
3. **workflow.md** — Repeatable process: prerequisites, launch steps, triage
4. **sandbox-setup.md** — Isolation modes: container/local/worktree setup and access patterns
5. **CHANGELOG.md** — Pattern evolution timeline with real-world lessons

## Core Principles

### 1. Pure Discovery, Not Regression Testing

**Hard constraint:** The audit agent MUST have zero knowledge of previous findings. It discovers friction by following help text and trying obvious commands, not by verifying a checklist.

If old issues regress, they surface naturally through rediscovery (agent hits the same friction a new user would). This is the methodology's strength: no bias from knowing what "should" work.

**Never add:**
- Regression verification sections to prompts
- Checklists of known issues to validate
- Comparisons to previous audit rounds
- "Verify the following fixed behaviors" sections

### 2. Sandbox Isolation (Invariant I1)

The audit agent MUST NOT modify production state. Every command goes through an isolation layer:

- **Container mode:** `docker exec <container-name>` prefix
- **Local mode:** env vars redirect state to temp dir
- **Worktree mode:** `cd <temp-dir> &&` prefix operates on a copy

The filler agent discovers metadata from the host (or container's help output) but never executes commands inside the sandbox.

### 3. Three-Tier Container Lifecycle

For container mode, understand the distinction:

- **Dockerfile** (stable infrastructure) — Created once, only updated when runtime dependencies change
- **Image** (compiled snapshot) — Rebuilt when tool source changes, tagged `{tool}-r{N}`
- **Container** (running instance) — Ephemeral execution environment, named `{tool}-r{N}`

Don't recreate the Dockerfile just because source code changed — rebuild the image.

## Architecture

### Two-Agent Pattern

1. **Filler Agent** (`filler` type) — Reads help output, discovers subcommands/flags, constructs audit areas, produces filled prompt. Enforces Invariant I2 (read-only, never executes in sandbox).
2. **Audit Agent** (`audit` type) — Executes the filled prompt in fresh context (zero knowledge), writes findings report. Enforces Invariant I1 (sandbox-scoped execution only).

The skill orchestrates both agents and enforces preconditions.

**Custom agent types are optional.** Install to `~/.claude/agents/` for:
- Tool-level enforcement (structural constraints)
- Claudewatch visibility (separate success rates)
- Graceful fallback to `general-purpose` if not installed

### Three Isolation Modes

| Mode | Exec Prefix | When to Use |
|------|------------|-------------|
| `container` | `docker exec <container-name>` | Destructive ops, system writes, package managers |
| `local` | `env KEY=VALUE [...]` | Self-managed state redirectable via env var |
| `worktree` | `cd <temp-dir> &&` | Operates on files in current directory |

The `mode` subcommand recommends a mode by answering Q1-Q3 (see SPEC.md §Mode Selection Algorithm).

### Prompt Reuse Strategy

When running subsequent audits on the same tool:

1. Check for existing `docs/cold-start-audit-prompt.md`
2. Offer user choice:
   - **Reuse** (fast) — Update only container name and date when tool structure is stable
   - **Regenerate** (thorough) — Run full filler agent when commands/flags changed

**Anti-pattern:** Launching filler agent in background + immediately launching audit agent with manually-edited old prompt = wasteful parallel execution.

Correct sequencing: filler completes → read filled prompt → launch audit agent.

## File Organization

```
prompts/
  prompt-template.md       — Generalized audit prompt with variable table
  filler-agent-prompt.md   — Agent that discovers metadata and fills template
  cold-start-audit-skill.md — Portable Claude Code skill (users install to ~/.claude/skills/)

docs/                      — Generated reports land here:
  cold-start-audit-prompt.md  (filled by filler agent)
  cold-start-audit.md         (findings report by audit agent)
```

## Key Conventions

### Container Naming

Pattern: `{tool}-r{N}` where N is the audit round number (1, 2, 3…)

**Auto-increment logic:**
```bash
docker images --format '{{.Repository}}' | grep '^{tool}-r' | sort -V | tail -1
# e.g. brewprune-r7 → next round is brewprune-r8
```

**Reuse checks:** Before building, check if container already exists and is running. Only rebuild when source changed.

### Permissions

Background agents need explicit `allow` rules. Common mistake: `allow_bash: true` is invalid.

**Valid format:**
```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```

Project-level `.claude/settings.json` can scope Bash to specific containers: `"Bash(docker exec brewprune-r*)"` covers all rounds.

### Severity Tiers

- **UX-critical** — Broken, misleading, blocks the user → fix before release
- **UX-improvement** — Confusing but functional → prioritize for sprint
- **UX-polish** — Minor friction → batch into cleanup PR

## Invariants (from SPEC.md)

**I1 — Sandbox Isolation:** Audit agent must not modify production state
**I2 — Filler is Read-Only:** Filler agent never invokes exec prefix
**I3 — Reproducible Findings:** Every finding must include exact reproduction steps
**I4 — Explicit Mode Selection:** User must specify `--mode container|local|worktree`

## Skill Commands

The `/cold-start-audit` skill (in `prompts/cold-start-audit-skill.md`) provides:

- `mode <tool-name>` — Recommend isolation mode via Q1-Q3 diagnostic
- `setup <tool-name> --mode <mode> [args]` — Run filler agent only
- `run <tool-name> --mode <mode> [args]` — Full audit (filler + audit)
- `report` — Summarize findings from `docs/cold-start-audit.md`
- `init <container-name>` — Scaffold `.claude/settings.json` with scoped permissions

## When Modifying This Repo

### Adding Features

- Maintain the pure discovery constraint — no regression testing capabilities
- Update SPEC.md if adding new invariants or modes
- Add pattern evolution notes to CHANGELOG.md if emerging from real usage
- Keep the skill self-contained (no external dependencies)

### Updating Documentation

- README.md is protocol-level (why, when, quickstart)
- SPEC.md is formal (invariants, guarantees, algorithms)
- workflow.md is operational (steps, checklists, troubleshooting)
- Don't duplicate — reference other docs when appropriate

### Testing Changes

Run a real audit on a test tool (like brewprune) to validate:
1. Filler agent correctly discovers metadata
2. Filled prompt has no placeholders
3. Audit agent runs in fresh context
4. Findings report uses correct format
5. Sandbox isolation holds (no production state modified)

## Common Pitfalls

1. **Adding regression verification to prompts** — Violates pure discovery principle
2. **Recreating Dockerfile unnecessarily** — Only rebuild image when source changes
3. **Not checking for container reuse** — Wastes build time
4. **Launching filler + audit in parallel** — When prompt already exists, just reuse
5. **Using `allow_bash: true`** — Invalid format, silently ignored

## Integration Points

- **scout-and-wave** — Parallel fix execution of audit findings
- **agentic-workflows** — Full Audit→Fix→Rediscover cycle documentation
- **claudewatch** — Session metrics (friction rate, tool errors, cost tracking)

## Questions to Ask Before Changing

1. Does this preserve the "new user with zero context" simulation?
2. Does this maintain sandbox isolation (I1)?
3. Will this work across all three modes (container/local/worktree)?
4. Is this documented in the right file (README vs SPEC vs workflow)?
5. Does the skill remain self-contained and portable?

---

**Project Status:** v1.0.0 — Production-ready, validated across 6 brewprune audit rounds

Last updated: 2026-03-06
