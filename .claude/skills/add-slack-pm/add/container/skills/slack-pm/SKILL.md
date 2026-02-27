---
name: slack-pm
description: Project management via Slack Lists and Canvas. Create task boards, track work items, write specs and meeting notes. Use proactively when managing projects, not just when explicitly asked.
allowed-tools: Bash(slack-lists:*, slack-canvas:*)
---

# Project Management with Slack Lists & Canvas

## Quick start

```bash
slack-lists create "Sprint 42" --schema '[{"name":"Task","type":"text"},{"name":"Status","type":"select"},{"name":"Assignee","type":"user"},{"name":"Due Date","type":"date"},{"name":"Priority","type":"select"}]'
slack-lists items create <list_id> --fields '[{"name":"Task","value":"Build login page"},{"name":"Status","value":"To Do"},{"name":"Priority","value":"High"}]'
slack-canvas create "Project Brief" --markdown - <<< "# Project Brief\n\n## Goals\n..."
slack-lists items list <list_id>
```

## PM Workflows

### New project kickoff

1. Create a task board list with typed columns:

```bash
slack-lists create "[ProjectName] Board" --schema '[
  {"name":"Task","type":"text"},
  {"name":"Status","type":"select"},
  {"name":"Assignee","type":"user"},
  {"name":"Due Date","type":"date"},
  {"name":"Priority","type":"select"},
  {"name":"Story Points","type":"number"}
]'
```

2. Create a project brief canvas:

```bash
slack-canvas create "[ProjectName] Brief" --markdown - << 'BRIEF'
# [ProjectName] Brief

## Problem
What problem are we solving?

## Goals
- Goal 1
- Goal 2

## Non-goals
- Explicitly out of scope

## Approach
High-level technical approach.

## Milestones
| Milestone | Target Date | Status |
|-----------|-------------|--------|
| M1: ...   | YYYY-MM-DD  | Not started |

## Open questions
- Question 1
BRIEF
```

3. Optionally share both to a channel:

```bash
slack-lists access set <list_id> --channels C123 --level write
slack-canvas access set <canvas_id> --channels C123 --level write
```

### Task tracking

Add items:

```bash
slack-lists items create <list_id> --fields '[
  {"name":"Task","value":"Implement auth"},
  {"name":"Status","value":"To Do"},
  {"name":"Assignee","value":"U123ABC"},
  {"name":"Priority","value":"High"},
  {"name":"Story Points","value":"5"}
]'
```

Create subtasks under a parent item:

```bash
slack-lists items create <list_id> --fields '[
  {"name":"Task","value":"Write unit tests for auth"},
  {"name":"Status","value":"To Do"}
]' --parent <parent_item_id>
```

Update status:

```bash
slack-lists items update <list_id> <item_id> --updates '{"Status":"In Progress","Assignee":"U123ABC"}'
```

Bulk delete completed items:

```bash
slack-lists items delete-multiple <list_id> --ids "item1,item2,item3"
```

### Meeting notes

Create a canvas from a template:

```bash
slack-canvas create "Team Sync - 2026-02-26" --markdown - << 'NOTES'
# Team Sync - 2026-02-26

## Attendees
- @person1, @person2

## Agenda
1. Topic A
2. Topic B

## Decisions
- Decision 1: ...

## Action Items
- [ ] @person1: Do X by YYYY-MM-DD
- [ ] @person2: Do Y by YYYY-MM-DD

## Notes
...
NOTES
```

To create a canvas pinned to a specific channel:

```bash
slack-canvas create "Team Sync - 2026-02-26" --markdown "..." --channel C123
```

Or create a channel canvas (one per channel, appears as a tab):

```bash
slack-canvas channel-create C123 --title "Team Sync" --markdown "..."
```

### Status reports

Read current list state, then summarize into a canvas:

```bash
# Get all items
slack-lists items list <list_id> > /tmp/items.json

# Count by status
echo "To Do: $(jq '[.items[] | select(.fields.Status == "To Do")] | length' /tmp/items.json)"
echo "In Progress: $(jq '[.items[] | select(.fields.Status == "In Progress")] | length' /tmp/items.json)"
echo "Done: $(jq '[.items[] | select(.fields.Status == "Done")] | length' /tmp/items.json)"

# Write report canvas
slack-canvas create "Status Report - 2026-02-26" --markdown - << REPORT
# Status Report - $(date +%Y-%m-%d)

## Summary
- To Do: X
- In Progress: Y
- Done: Z
- Completion: Z/(X+Y+Z) = ...%

## Highlights
- Completed: ...
- Blocked: ...

## Next Week
- ...
REPORT
```

## Polling for human changes

Set up a scheduled task to detect when humans modify list items outside of agent conversations. Do this once per list, on first use -- not on every interaction.

### Setup polling

Use the `schedule_task` MCP tool to create a recurring interval task:

```
schedule_task(
  prompt: "Poll Slack list <list_id> for changes.
    1. Run: slack-lists items list <list_id> > /tmp/current.json
    2. Compare against .slack-pm/snapshots/<list_id>.json (if it exists)
    3. Use jq to diff: added items, removed items, changed fields
    4. If changes found, append a summary to .slack-pm/changes.md with timestamp
    5. Copy /tmp/current.json to .slack-pm/snapshots/<list_id>.json
    6. Do NOT send a message unless there are changes. If there are changes, send a brief summary.",
  schedule_type: "interval",
  schedule_value: "1800000",
  context_mode: "isolated"
)
```

### Reading changes

On each conversation turn, check for changes detected by polling:

```bash
if [ -f .slack-pm/changes.md ]; then
  cat .slack-pm/changes.md
  # Optionally clear after reading
  > .slack-pm/changes.md
fi
```

### Snapshot directory structure

```
.slack-pm/
  snapshots/
    <list_id>.json      # Last-known state of each polled list
  changes.md            # Append-only log of detected changes
```

## Schema templates

### Sprint board

```json
[
  {"name": "Task", "type": "text"},
  {"name": "Status", "type": "select"},
  {"name": "Assignee", "type": "user"},
  {"name": "Due Date", "type": "date"},
  {"name": "Priority", "type": "select"},
  {"name": "Story Points", "type": "number"}
]
```

### Bug tracker

```json
[
  {"name": "Title", "type": "text"},
  {"name": "Severity", "type": "select"},
  {"name": "Reporter", "type": "user"},
  {"name": "Status", "type": "select"},
  {"name": "Steps to Reproduce", "type": "text"}
]
```

### Feature backlog

```json
[
  {"name": "Feature", "type": "text"},
  {"name": "Description", "type": "text"},
  {"name": "Priority", "type": "select"},
  {"name": "Effort", "type": "select"},
  {"name": "Status", "type": "select"}
]
```

### Simple TODO

Use the built-in TODO list type (no custom schema needed):

```bash
slack-lists create "Quick Tasks" --todo
```

## Command reference

### slack-lists

#### List management

```bash
# Create a new list
slack-lists create <name>
slack-lists create <name> --description "Description text"
slack-lists create <name> --todo                              # TODO list (built-in schema)
slack-lists create <name> --schema '<json_array>'             # Custom columns

# Update list metadata
slack-lists update <list_id> --name "New Name"
slack-lists update <list_id> --description "New description"
slack-lists update <list_id> --name "New" --description "Both at once"
```

#### Item operations

```bash
# List all items
slack-lists items list <list_id>
slack-lists items list <list_id> --limit 50
slack-lists items list <list_id> --limit 50 --offset 100     # Pagination

# Get single item details
slack-lists items info <list_id> <item_id>

# Create an item
slack-lists items create <list_id> --fields '[{"name":"Task","value":"Do something"}]'
slack-lists items create <list_id> --fields '<json>' --parent <item_id>   # Subtask

# Update an item (object of field_name -> new_value)
slack-lists items update <list_id> <item_id> --updates '{"Status":"Done","Priority":"Low"}'

# Delete a single item
slack-lists items delete <list_id> <item_id>

# Delete multiple items at once
slack-lists items delete-multiple <list_id> --ids "item1,item2,item3"
```

**Fields JSON format** (for `items create --fields`):

```json
[
  {"name": "Task", "value": "Build feature X"},
  {"name": "Status", "value": "To Do"},
  {"name": "Assignee", "value": "U123ABC"},
  {"name": "Due Date", "value": "2026-03-15"},
  {"name": "Priority", "value": "High"},
  {"name": "Story Points", "value": "5"}
]
```

**Updates JSON format** (for `items update --updates`):

```json
{"Status": "In Progress", "Assignee": "U456DEF"}
```

#### Export

```bash
# Download list as CSV (polls until ready, prints download URL)
slack-lists download <list_id>
```

#### Access control

```bash
# Grant channel access (read or write)
slack-lists access set <list_id> --channels C123,C456 --level read
slack-lists access set <list_id> --channels C123 --level write

# Revoke channel access
slack-lists access remove <list_id> --channels C123,C456
```

### slack-canvas

#### Canvas management

```bash
# Create a standalone canvas
slack-canvas create <title> --markdown "# Content here"

# Create with stdin (preferred for long content)
slack-canvas create <title> --markdown - << 'EOF'
# Heading
Body content with **formatting**.
- List item
EOF

# Create and pin to a channel tab
slack-canvas create <title> --markdown "..." --channel C123

# Create a channel canvas (one per channel, appears as channel tab)
slack-canvas channel-create <channel_id>
slack-canvas channel-create <channel_id> --title "Title" --markdown "..."

# Edit a canvas (replaces all content)
slack-canvas edit <canvas_id> --markdown "# Updated content"

# Edit with stdin
slack-canvas edit <canvas_id> --markdown - << 'EOF'
# Updated heading
New body content.
EOF

# Delete a canvas
slack-canvas delete <canvas_id>
```

#### Content lookup

```bash
# Find sections by text content
slack-canvas sections <canvas_id> --criteria '{"contains_text":"Introduction"}'
```

#### Access control

```bash
# Grant channel access (read or write)
slack-canvas access set <canvas_id> --channels C123,C456 --level read
slack-canvas access set <canvas_id> --channels C123 --level write

# Revoke channel access
slack-canvas access remove <canvas_id> --channels C123,C456
```

#### Stdin pattern for long content

Always use `--markdown -` with a heredoc for multi-line content. This avoids shell quoting issues:

```bash
slack-canvas create "My Doc" --markdown - << 'EOF'
# Document Title

## Section 1
Content with "quotes", $variables, and special chars are safe inside heredoc.

## Section 2
| Col A | Col B |
|-------|-------|
| 1     | 2     |
EOF
```

Use single-quoted heredoc delimiter (`<< 'EOF'`) to prevent shell variable expansion. Use unquoted (`<< EOF`) when you want variable interpolation.
