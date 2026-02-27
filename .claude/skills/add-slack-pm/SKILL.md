---
name: add-slack-pm
description: Add Slack Lists & Canvas PM capabilities to the container agent. Depends on add-slack being applied first. Gives the agent tools to create task boards, track work items, write specs, and monitor human changes.
---

# Add Slack PM (Lists & Canvas)

This skill adds project management capabilities to the container agent via Slack's Lists API and Canvas API.

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `slack-pm` is in `applied_skills`, skip to Phase 3 (Setup). The code changes are already in place.

### Check dependency

Read `.nanoclaw/state.yaml`. If `slack` is NOT in `applied_skills`, stop and tell the user to run `/add-slack` first.

## Phase 2: Apply Code Changes

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

### Apply the skill

```bash
npx tsx scripts/apply-skill.ts .claude/skills/add-slack-pm
```

This deterministically:
- Adds `container/skills/slack-pm/SKILL.md` (agent-facing PM skill with workflows and command reference)
- Adds `container/bin/slack-lists` (Bash CLI wrapper for Slack Lists API)
- Adds `container/bin/slack-canvas` (Bash CLI wrapper for Slack Canvas API)
- Three-way merges Dockerfile changes (adds `jq` dependency, copies bin/ scripts into image)
- Records the application in `.nanoclaw/state.yaml`

If the apply reports merge conflicts, read the intent file:
- `modify/container/Dockerfile.intent.md` — what changed and invariants for the Dockerfile

### Rebuild container

```bash
./container/build.sh
```

### Validate

```bash
npm test
npm run build
```

## Phase 3: Setup

### Add OAuth Scopes

The user needs to add 4 new OAuth scopes to their Slack app:

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and select your app
2. Go to **OAuth & Permissions** > **Scopes** > **Bot Token Scopes**
3. Add these scopes:

| Scope | Why it's needed |
|-------|----------------|
| `lists:read` | Read list items and download lists |
| `lists:write` | Create, update, and delete lists and list items |
| `canvases:read` | Look up canvas sections |
| `canvases:write` | Create, edit, and delete canvases |

4. **Reinstall the app** to your workspace — scope changes require reinstallation
5. Copy the new Bot Token (it changes on reinstall) and update `.env`
6. Sync: `mkdir -p data/env && cp .env data/env/env`
7. Restart: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw`

Wait for the user to confirm they've added the scopes and reinstalled.

## Phase 4: Verify

### Test from container

Tell the user:

> Send a message to your bot asking it to create a test list. For example:
> "Create a test list called 'PM Test' with columns for Task and Status"
>
> The bot should successfully create the list. Then ask it to delete the test list.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### "missing_scope" errors

If the agent logs `missing_scope` errors for lists or canvases:
1. Go back to **OAuth & Permissions** and add the missing scope
2. **Reinstall the app** to your workspace
3. Copy the new Bot Token and update `.env`
4. Sync: `mkdir -p data/env && cp .env data/env/env`
5. Restart: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw`

### CLI scripts not found in container

1. Verify the scripts exist: `ls container/bin/`
2. Rebuild the container: `./container/build.sh`
3. Verify in container: `docker run --rm nanoclaw-agent which slack-lists`

### "not_allowed_token_type" errors

The Lists and Canvas APIs require bot tokens (xoxb-). If you see this error, verify your SLACK_BOT_TOKEN starts with `xoxb-`.

## After Setup

The container agent now has access to:
- **Slack Lists** — Create project boards, track tasks, manage items with typed columns
- **Slack Canvas** — Create specs, meeting notes, status reports as rich documents
- **Polling** — Agent can set up scheduled reads to detect human-made changes to lists

These tools are available to ALL groups. Each group's agent can use them independently.

## Known Limitations

- **No Canvas change awareness** — Slack's Canvas API doesn't expose modification timestamps. The agent can poll Lists for changes but cannot detect Canvas edits made by humans.
- **Lists schema is fixed at creation** — Columns can't be added/removed after a list is created.
- **Canvas edit is full-replace** — The agent must read canvas content before editing, as edits replace sections.
- **Rate limits** — Slack APIs are rate-limited (~20-50 req/min). Avoid bulk operations.
