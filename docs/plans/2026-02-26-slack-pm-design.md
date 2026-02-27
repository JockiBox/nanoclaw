# Slack PM Design — Lists & Canvas for Project Management

## Context

NanoClaw's Slack channel integration (`add-slack` skill) supports messaging only. This design adds project management capabilities via Slack's Lists API and Canvas API, enabling the container agent to act as a PM — creating task boards, tracking work items, writing specs, and staying aware of human-made changes.

## Decision: Separate Skill

A new `add-slack-pm` skill (not modifications to `add-slack`) because:
- Keeps the base Slack channel skill clean and upstream-mergeable
- PM features can be applied/removed independently
- Avoids fork divergence on the core messaging skill

## Decision: Bash CLI Scripts (Approach A)

Container agent tools exposed as Bash-callable CLI scripts, following the proven `agent-browser` pattern. Alternatives considered:
- **Approach B (Extend IPC MCP server)**: More secure but requires changes to both host and container IPC code. Too much complexity.
- **Approach C (Dedicated MCP server in container)**: Clean separation but adds operational complexity and is redundant with the host Slack connection.

Approach A wins because: simplest, proven pattern, no orchestrator changes, maximum agent flexibility.

## Skill Structure

```
.claude/skills/add-slack-pm/
├── SKILL.md                          # Skill instructions (setup phases)
├── manifest.yaml                     # depends: [slack]
├── add/
│   └── container/
│       ├── skills/
│       │   └── slack-pm/
│       │       └── SKILL.md          # Agent-facing skill (PM workflows + CLI reference)
│       └── bin/
│           ├── slack-lists           # Bash CLI wrapper for Lists API
│           └── slack-canvas          # Bash CLI wrapper for Canvas API
└── modify/
    └── container/
        └── Dockerfile.intent.md      # Instructions to install bin/ scripts
```

When applied:
1. `container/skills/slack-pm/SKILL.md` syncs into every group's `.claude/skills/` (existing container-runner logic)
2. `container/bin/slack-lists` and `container/bin/slack-canvas` are installed into the container image
3. Agent skill declares `allowed-tools: Bash(slack-lists:*, slack-canvas:*)` for tool permissions

## CLI Surfaces

### slack-lists

```
slack-lists create <name> [--description "..."] [--todo] [--schema '{"columns":[...]}']
slack-lists update <list_id> [--name "..."] [--description "..."]
slack-lists items list <list_id> [--limit N] [--offset N]
slack-lists items info <list_id> <item_id>
slack-lists items create <list_id> --fields '[{"name":"Task","value":"..."}]' [--parent <item_id>]
slack-lists items update <list_id> <item_id> --updates '{"Status":"Done"}'
slack-lists items delete <list_id> <item_id>
slack-lists items delete-multiple <list_id> --ids '<item_id1>,<item_id2>'
slack-lists download <list_id>
slack-lists access set <list_id> --channels C123,C456 --level <read|write>
slack-lists access remove <list_id> --channels C123
```

### slack-canvas

```
slack-canvas create <title> --markdown "# Content" [--channel <channel_id>]
slack-canvas channel-create <channel_id> [--title "..."] [--markdown "..."]
slack-canvas edit <canvas_id> --markdown "# Updated content"
slack-canvas delete <canvas_id>
slack-canvas sections <canvas_id> --criteria '{"contains_text":"..."}'
slack-canvas access set <canvas_id> --channels C123 --level <read|write>
slack-canvas access remove <canvas_id> --channels C123
```

Both scripts:
- Read `SLACK_BOT_TOKEN` from the environment (already in container via `data/env/env`)
- Output JSON responses from the Slack API
- Exit non-zero on API errors, printing the Slack error message to stderr
- Are stateless wrappers — no local state, no caching

## Agent-Facing Skill (container/skills/slack-pm/SKILL.md)

```yaml
---
name: slack-pm
description: Project management via Slack Lists and Canvas. Create task boards, track work items, write specs and meeting notes. Use proactively when managing projects.
allowed-tools: Bash(slack-lists:*, slack-canvas:*)
---
```

Contents:
1. **Quick start** — common one-liners for creating a project board and spec canvas
2. **PM Workflows** — opinionated patterns for:
   - New project kickoff: List with columns (Task, Status, Assignee, Due Date, Priority) + Canvas for project brief
   - Task tracking: Add items, update status, create subtasks via `--parent`
   - Meeting notes: Canvas with date-stamped heading, attendees, action items
   - Status reports: Read list items, summarize progress into a canvas
3. **Polling for human changes** — how to set up scheduled reads and diff against snapshots
4. **Command reference** — full CLI docs
5. **Schema templates** — pre-built JSON schemas (sprint board, bug tracker, feature backlog)

## Polling for Human Changes

Slack does not expose events for list item changes or canvas edits. The agent stays aware of human-made changes via periodic polling using existing tools:

1. **`schedule_task`** (existing IPC MCP tool) — sets up a recurring interval (e.g., every 30 min)
2. **`slack-lists items list`** (new CLI) — reads current list state
3. **File write** (existing SDK tool) — saves snapshot as JSON in the group workspace
4. On each poll: read current list, diff against snapshot, write a summary of changes to a "pending changes" file the agent picks up on its next conversation turn

No orchestrator changes needed — polling lives entirely within the agent's existing capabilities.

## OAuth Scopes

Four new scopes required (all available on bot tokens):

| Scope | Why |
|-------|-----|
| `lists:read` | Read list items, download lists |
| `lists:write` | Create/update/delete lists and items |
| `canvases:read` | Look up canvas sections |
| `canvases:write` | Create, edit, and delete canvases |

These are added to the Slack app's Bot Token Scopes. The app must be reinstalled after adding scopes (Slack requirement). `SLACK_SETUP.md` is updated with the new scopes.

## Skill Phases

| Phase | What happens |
|-------|-------------|
| 1. Pre-flight | Check `add-slack` is applied. Check if `slack-pm` already applied. |
| 2. Apply code | Install CLI scripts into container, add skill module, rebuild container. |
| 3. Setup | Guide user to add 4 OAuth scopes, reinstall app, sync token. |
| 4. Verify | Test: `slack-lists create "Test List"` from container, confirm it works, clean up. |

## Out of Scope

- No orchestrator changes (`src/index.ts`, `src/config.ts`, IPC layer)
- No new npm dependencies (scripts use `curl` + `jq`)
- No Canvas change detection (Canvas API has no modification timestamps or change history)
- No Block Kit formatting for messages about lists/canvases
- No retry logic in CLI scripts for rate limits

## Known Limitations

- **No Canvas change awareness**: Slack's Canvas API doesn't expose modification timestamps or change history. List polling works because items have discrete fields to diff; canvas content doesn't.
- **Lists schema is set at creation**: Columns can't be added/removed via API after creation (Slack limitation).
- **Canvas edit is full-replace**: `canvases.edit` replaces sections, not surgical inline edits. Agent must read sections first, modify, then write back.
- **Rate limits**: Slack APIs are rate-limited (~20-50 req/min). CLI scripts don't implement retry logic.
- **List IDs aren't human-memorable**: Agent tracks them via naming conventions and group-local state files.
