# Slack PM Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a separate `add-slack-pm` skill that gives the container agent Bash CLI tools for Slack Lists and Canvas, enabling it to act as a project manager.

**Architecture:** New NanoClaw skill (`add-slack-pm`) that depends on the existing `slack` skill. Two Bash scripts (`slack-lists`, `slack-canvas`) are installed into the container image and exposed via an agent-facing SKILL.md, following the `agent-browser` pattern. The Dockerfile is modified to install `jq` and copy the scripts. No orchestrator changes.

**Tech Stack:** Bash (`curl` + `jq`), NanoClaw skills engine, Docker

---

### Task 1: Create skill scaffold — manifest and SKILL.md

**Files:**
- Create: `.claude/skills/add-slack-pm/manifest.yaml`
- Create: `.claude/skills/add-slack-pm/SKILL.md`

**Step 1: Create the manifest**

```yaml
# .claude/skills/add-slack-pm/manifest.yaml
skill: slack-pm
version: 1.0.0
description: "Slack Lists & Canvas PM tools for the container agent"
core_version: 0.1.0
adds:
  - container/skills/slack-pm/SKILL.md
  - container/bin/slack-lists
  - container/bin/slack-canvas
modifies:
  - container/Dockerfile
structured:
  env_additions: []
conflicts: []
depends:
  - slack
test: "./container/build.sh"
```

**Step 2: Create the skill SKILL.md (setup instructions)**

This is the skill that the human runs via `/add-slack-pm`. It guides setup — adding OAuth scopes, rebuilding the container, verifying. Model it after `.claude/skills/add-slack/SKILL.md`.

Contents:
- Phase 1: Pre-flight — check `slack` is applied, check if `slack-pm` already applied
- Phase 2: Apply code — `npx tsx scripts/apply-skill.ts .claude/skills/add-slack-pm`
- Phase 3: Setup — guide user to add 4 OAuth scopes (`lists:read`, `lists:write`, `canvases:read`, `canvases:write`), reinstall app, sync token
- Phase 4: Rebuild — `./container/build.sh` to install the CLI scripts into the image
- Phase 5: Verify — test from a running container that `slack-lists` and `slack-canvas` are accessible

**Step 3: Commit**

```bash
git add .claude/skills/add-slack-pm/manifest.yaml .claude/skills/add-slack-pm/SKILL.md
git commit -m "feat(slack-pm): add skill scaffold with manifest and setup instructions"
```

---

### Task 2: Write the `slack-lists` CLI script

**Files:**
- Create: `.claude/skills/add-slack-pm/add/container/bin/slack-lists`

**Step 1: Write the script**

A Bash script that wraps `curl` calls to the Slack API for list operations. It reads `SLACK_BOT_TOKEN` from the environment.

Subcommands to implement:
- `create <name>` — `POST /api/slackLists.create` with optional `--description`, `--todo`, `--schema`
- `update <list_id>` — `POST /api/slackLists.update` with optional `--name`, `--description`
- `items list <list_id>` — `GET /api/slackLists.items.list` with optional `--limit`, `--offset`
- `items info <list_id> <item_id>` — `GET /api/slackLists.items.info`
- `items create <list_id>` — `POST /api/slackLists.items.create` with `--fields` JSON, optional `--parent`
- `items update <list_id> <item_id>` — `POST /api/slackLists.items.update` with `--updates` JSON
- `items delete <list_id> <item_id>` — `POST /api/slackLists.items.delete`
- `items delete-multiple <list_id>` — `POST /api/slackLists.items.deleteMultiple` with `--ids`
- `download <list_id>` — `POST /api/slackLists.download.start` then poll `GET /api/slackLists.download.get`
- `access set <list_id>` — `POST /api/slackLists.access.set` with `--channels`, `--level`
- `access remove <list_id>` — `POST /api/slackLists.access.delete` with `--channels`

Pattern for each subcommand:
```bash
slack_api() {
  local method="$1" endpoint="$2" shift 2
  local response
  response=$(curl -s -X "$method" "https://slack.com/api/$endpoint" \
    -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    -H "Content-Type: application/json; charset=utf-8" \
    "${@}")
  local ok=$(echo "$response" | jq -r '.ok')
  if [ "$ok" != "true" ]; then
    local err=$(echo "$response" | jq -r '.error // "unknown_error"')
    echo "Error: $err" >&2
    echo "$response" | jq . >&2
    exit 1
  fi
  echo "$response" | jq .
}
```

The script must:
- Check `SLACK_BOT_TOKEN` is set, error if not
- Use `set -euo pipefail`
- Print usage on no arguments or `--help`
- Be executable (`chmod +x`)

**Step 2: Test the script locally (syntax check)**

```bash
bash -n .claude/skills/add-slack-pm/add/container/bin/slack-lists
```

Expected: no output (clean syntax).

**Step 3: Commit**

```bash
git add .claude/skills/add-slack-pm/add/container/bin/slack-lists
git commit -m "feat(slack-pm): add slack-lists CLI script"
```

---

### Task 3: Write the `slack-canvas` CLI script

**Files:**
- Create: `.claude/skills/add-slack-pm/add/container/bin/slack-canvas`

**Step 1: Write the script**

Same pattern as `slack-lists`, wrapping Canvas API calls.

Subcommands:
- `create <title>` — `POST /api/canvases.create` with `--markdown`, optional `--channel`
- `channel-create <channel_id>` — `POST /api/conversations.canvases.create` with optional `--title`, `--markdown`
- `edit <canvas_id>` — `POST /api/canvases.edit` with `--markdown`. The `changes` array uses `operation: "replace"` or `"insert_after"` with `document_content` of type `markdown`.
- `delete <canvas_id>` — `POST /api/canvases.delete`
- `sections <canvas_id>` — `POST /api/canvases.sections.lookup` with `--criteria` JSON
- `access set <canvas_id>` — `POST /api/canvases.access.set` with `--channels`, `--level`
- `access remove <canvas_id>` — `POST /api/canvases.access.delete` with `--channels`

Reuse the same `slack_api` helper pattern. The `--markdown` content may be multi-line, so read from stdin if value is `-`:
```bash
# Allow: slack-canvas create "Title" --markdown "# Hello"
# Or:   echo "# Hello" | slack-canvas create "Title" --markdown -
```

**Step 2: Test the script locally (syntax check)**

```bash
bash -n .claude/skills/add-slack-pm/add/container/bin/slack-canvas
```

Expected: no output (clean syntax).

**Step 3: Commit**

```bash
git add .claude/skills/add-slack-pm/add/container/bin/slack-canvas
git commit -m "feat(slack-pm): add slack-canvas CLI script"
```

---

### Task 4: Write the Dockerfile modification

**Files:**
- Create: `.claude/skills/add-slack-pm/modify/container/Dockerfile`
- Create: `.claude/skills/add-slack-pm/modify/container/Dockerfile.intent.md`

**Step 1: Create the modified Dockerfile**

Copy the current `container/Dockerfile` and add two changes:

1. Add `jq` to the `apt-get install` list (line 7-26 block):
   ```
   jq \
   ```

2. After the `COPY agent-runner/ ./` line (line 45), add:
   ```dockerfile
   # Copy Slack PM CLI scripts
   COPY bin/ /usr/local/bin/
   ```

This works because the build context is `container/`, and the `adds` step puts the scripts at `container/bin/slack-lists` and `container/bin/slack-canvas`.

**Step 2: Create the intent file**

The intent file explains what changed and why, so merge conflicts can be resolved:

```markdown
# Dockerfile Modification Intent

## Changes
1. Added `jq` to apt-get install block — needed by slack-lists and slack-canvas CLI scripts to parse JSON responses
2. Added `COPY bin/ /usr/local/bin/` after agent-runner copy — installs the CLI scripts into PATH

## Invariants
- jq must be in the apt-get block (not a separate RUN layer, to keep the image small)
- COPY bin/ must come after WORKDIR /app but before the entrypoint setup
- The bin/ scripts must be executable (chmod +x is done at authoring time, preserved by COPY)
```

**Step 3: Commit**

```bash
git add .claude/skills/add-slack-pm/modify/container/Dockerfile .claude/skills/add-slack-pm/modify/container/Dockerfile.intent.md
git commit -m "feat(slack-pm): add Dockerfile modification for jq and CLI scripts"
```

---

### Task 5: Write the agent-facing SKILL.md

**Files:**
- Create: `.claude/skills/add-slack-pm/add/container/skills/slack-pm/SKILL.md`

**Step 1: Write the skill**

This is the SKILL.md the container agent sees. Follow the `agent-browser` SKILL.md pattern at `container/skills/agent-browser/SKILL.md`.

Structure:
```yaml
---
name: slack-pm
description: Project management via Slack Lists and Canvas. Create task boards, track work items, write specs and meeting notes. Use proactively when managing projects, not just when explicitly asked.
allowed-tools: Bash(slack-lists:*, slack-canvas:*)
---
```

Body sections:

1. **Quick start** — 3-4 one-liners showing common operations:
   ```bash
   slack-lists create "Sprint Board" --todo --schema '[{"name":"Task","type":"text"},{"name":"Status","type":"select"},{"name":"Assignee","type":"user"},{"name":"Due","type":"date"},{"name":"Priority","type":"select"}]'
   slack-lists items create L123 --fields '[{"name":"Task","value":"Implement auth"},{"name":"Status","value":"To Do"}]'
   slack-canvas create "Project Brief" --markdown "# Project X\n\n## Goals\n..." --channel C123
   ```

2. **PM Workflows** — opinionated patterns:
   - **New project kickoff**: Create list + canvas, link them by naming convention (`[ProjectName] Board` / `[ProjectName] Brief`)
   - **Task tracking**: Add items, update status, subtasks via `--parent`
   - **Meeting notes**: Canvas template with date, attendees, decisions, action items
   - **Status reports**: Read list items with `slack-lists items list`, summarize into canvas

3. **Polling for changes** — how to use `schedule_task` to set up a recurring poll:
   - Every 30 minutes, run `slack-lists items list <list_id>` and compare against `.slack-pm/snapshots/<list_id>.json`
   - If items changed, write a summary to `.slack-pm/changes.md` which the agent reads on next conversation turn
   - Teach the agent to set this up on first use of a list

4. **Schema templates** — pre-built JSON for common list types:
   - Sprint board: Task, Status (To Do/In Progress/Done/Blocked), Assignee, Due Date, Priority (P0-P3), Story Points
   - Bug tracker: Title, Severity, Reporter, Status, Steps to Reproduce
   - Feature backlog: Feature, Description, Priority, Effort, Status

5. **Command reference** — full docs for both `slack-lists` and `slack-canvas` (mirror the CLI help)

**Step 2: Commit**

```bash
git add .claude/skills/add-slack-pm/add/container/skills/slack-pm/SKILL.md
git commit -m "feat(slack-pm): add agent-facing PM skill with workflows and templates"
```

---

### Task 6: Update SLACK_SETUP.md with new OAuth scopes

**Files:**
- Modify: `.claude/skills/add-slack/SLACK_SETUP.md`

**Step 1: Add the new scopes to the setup guide**

In Step 4 (Set Bot Permissions), add a new section after the existing scopes table:

```markdown
### Additional scopes for Slack PM (Lists & Canvas)

If you've applied the `add-slack-pm` skill, also add these scopes:

| Scope | Why it's needed |
|-------|----------------|
| `lists:read` | Read list items and download lists |
| `lists:write` | Create, update, and delete lists and list items |
| `canvases:read` | Look up canvas sections |
| `canvases:write` | Create, edit, and delete canvases |

After adding scopes, you must **reinstall the app** to your workspace.
```

**Step 2: Commit**

```bash
git add .claude/skills/add-slack/SLACK_SETUP.md
git commit -m "docs(slack): add Lists & Canvas OAuth scopes to setup guide"
```

---

### Task 7: Test the skill application

**Step 1: Ensure skills system is initialized**

```bash
ls .nanoclaw/state.yaml || npx tsx scripts/apply-skill.ts --init
```

**Step 2: Verify the `slack` skill is applied**

```bash
cat .nanoclaw/state.yaml | grep slack
```

If `slack` is not in `applied_skills`, the `add-slack-pm` skill will fail the dependency check. In that case, apply `add-slack` first or temporarily remove the `depends` check for testing.

**Step 3: Apply the skill**

```bash
npx tsx scripts/apply-skill.ts .claude/skills/add-slack-pm
```

Expected: success, with the Dockerfile merged and new files copied.

**Step 4: Verify files are in place**

```bash
ls -la container/bin/slack-lists container/bin/slack-canvas
ls container/skills/slack-pm/SKILL.md
head -5 container/Dockerfile | grep jq  # verify jq was added
grep "COPY bin/" container/Dockerfile    # verify bin copy was added
```

**Step 5: Build the container**

```bash
./container/build.sh
```

Expected: successful build with `jq` installed and scripts in `/usr/local/bin/`.

**Step 6: Verify scripts are accessible inside container**

```bash
echo '{}' | docker run -i --rm nanoclaw-agent bash -c "which slack-lists && which slack-canvas && slack-lists --help"
```

Expected: paths to both scripts shown, usage help printed.

**Step 7: Commit any fixes**

If any changes were needed during testing, commit them.

---

### Task 8: Final review and cleanup

**Step 1: Review all added files**

```bash
git diff main --stat
```

Verify the changeset is clean and contains only the expected files.

**Step 2: Run the existing test suite**

```bash
npm test
npm run build
```

Expected: all existing tests pass, build is clean. (The new skill doesn't modify any TypeScript files, so existing tests should be unaffected.)

**Step 3: Final commit if needed**

```bash
git log --oneline main..HEAD
```

Verify all commits are clean and well-described.
