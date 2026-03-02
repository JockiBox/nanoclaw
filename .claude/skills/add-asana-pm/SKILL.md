---
name: add-asana-pm
description: Add Asana PM capabilities to the container agent. Gives the agent tools to create tasks, query projects, update status, and report on progress — all via Asana as the source of truth.
---

# Add Asana PM

This skill adds project management capabilities to the container agent via the Asana REST API.

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `asana-pm` is in `applied_skills`, skip to Phase 3 (Setup). The code changes are already in place.

## Phase 2: Apply Code Changes

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

### Apply the skill

```bash
npx tsx scripts/apply-skill.ts .claude/skills/add-asana-pm
```

This deterministically:
- Adds `container/skills/asana-pm/SKILL.md` (agent-facing PM skill with workflows and command reference)
- Adds `container/bin/asana` (Bash CLI wrapper for Asana REST API)
- Three-way merges Dockerfile changes (copies bin/ scripts into image)
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

### Get Asana Personal Access Token

1. Go to [app.asana.com/0/my-apps](https://app.asana.com/0/my-apps) in Asana
2. Click **Create new token**
3. Give it a name like "JockiTasker"
4. Copy the token

### Get Workspace and Project IDs

The user needs to provide:
- **Workspace ID**: Found in the URL when viewing Asana (the numeric ID)
- **Default Project ID**: The Asana project where WhatsApp tasks should land

If they don't know the IDs, they can provide names and we'll look them up after setting the token.

### Configure .env

Add these to `.env`:

```
ASANA_ACCESS_TOKEN=<their token>
ASANA_WORKSPACE_ID=<workspace gid>
ASANA_DEFAULT_PROJECT_ID=<project gid>
```

### Sync env to container

```bash
mkdir -p data/env && cp .env data/env/env
```

### Restart

```bash
# macOS
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Linux
systemctl --user restart nanoclaw
```

Wait for the user to confirm they've set up the token.

## Phase 4: Verify

### Test from container

Tell the user:

> Send a message to your bot asking it to list Asana projects. For example:
> "@JockiTasker what projects do we have in Asana?"
>
> Then try creating a test task:
> "@JockiTasker add task: test asana integration"
>
> Verify the task appears in Asana, then mark it done:
> "@JockiTasker done with the test task"

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### "ASANA_ACCESS_TOKEN is not set"

1. Check that `.env` has `ASANA_ACCESS_TOKEN=...`
2. Sync: `mkdir -p data/env && cp .env data/env/env`
3. Restart the service

### "401 Unauthorized" from Asana

1. Token may have expired — generate a new one at [app.asana.com/0/my-apps](https://app.asana.com/0/my-apps)
2. Update `.env` and sync: `cp .env data/env/env`
3. Restart

### "403 Forbidden" or "Not a member"

The token's user doesn't have access to the specified project. Check that the project GID is correct and the user is a member.

### CLI script not found in container

1. Verify the script exists: `ls container/bin/asana`
2. Rebuild the container: `./container/build.sh`
3. Verify: `docker run --rm nanoclaw-agent which asana`

## Phase 5: Schedule Team Meetings

Ask the user if they want to set up recurring meeting summaries.

Tell them:

> JockiTasker can auto-post meeting summaries on a schedule:
> - *Monday Sync* (Monday 9am) — overdue tasks, this week's plan, last week's wins, unassigned items
> - *Friday Review* (Friday 4pm) — what got done, what's carrying over, what's coming next week
>
> Want me to set these up? You can also trigger them manually anytime by saying "monday sync" or "friday review".

If they say yes, have the bot send a test message to the registered group, then set up the scheduled tasks:

```
@JockiTasker set up monday sync for 9am and friday review for 4pm
```

The agent will use `schedule_task` with cron expressions `0 9 * * 1` and `0 16 * * 5`.

The times can be adjusted later by pausing and recreating the tasks.

## After Setup

The container agent now has access to:
- **Asana Tasks** — Create, query, update, complete tasks in any project
- **Asana Projects** — List and inspect projects, sections, team members
- **Search** — Find tasks by name, assignee, or status

These tools are available to ALL groups. Each group's agent can use them independently.
