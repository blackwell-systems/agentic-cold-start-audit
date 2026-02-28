Cold-Start UX Audit: AI agents simulate new users in containerized sandboxes.

Read the workflow at `workflow.md`, the filler agent prompt at `prompts/filler-agent-prompt.md`, and the prompt template at `prompts/prompt-template.md`.

Arguments:
- `setup <container-name> <tool-name>`: Run the filler agent to discover tool metadata from the running container and produce a filled audit prompt. Write the filled prompt to `docs/cold-start-audit-prompt.md`.
- `run <container-name> <tool-name>`: Run the full audit. If `docs/cold-start-audit-prompt.md` exists, use it. Otherwise, run the filler agent first to generate it. Launch the audit agent as a background Task agent. Write the findings report to `docs/cold-start-audit.md`.
- `report`: Read and summarize the findings from `docs/cold-start-audit.md`, grouped by severity.

Before running, verify:
1. The sandbox container is running (`docker ps --filter name=<container-name>`)
2. The tool is installed inside the container (`docker exec <container-name> <tool-name> --help`)
3. Bash permissions allow `docker exec` on the container (check `.claude/settings.json`)

The audit agent must run ALL commands via `docker exec <container-name>` — never run the tool directly on the host.
