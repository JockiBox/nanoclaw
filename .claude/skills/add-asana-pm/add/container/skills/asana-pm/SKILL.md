---
name: asana-pm
description: Project management via Asana. Create tasks, track work, query status, complete items. Asana is the source of truth — all reads and writes go directly to Asana. Use proactively when managing tasks, not just when explicitly asked.
allowed-tools: Bash(asana:*)
---

# Project Management with Asana

Asana is the *source of truth* for all tasks. Do not maintain a separate task list — every read queries Asana, every write goes to Asana.

## Quick start

```bash
# Create a task
asana tasks create "Fix auth bug on staging" --assignee sarah@jockibox.com --due 2026-03-07

# List open tasks
asana tasks list

# Complete a task
asana tasks update <gid> --completed true

# Search for tasks
asana tasks search "auth bug"
```

## Core PM Workflows

### Capturing a task from chat

When someone mentions work to be done, create it in Asana immediately:

```bash
asana tasks create "fix the auth bug on staging" \
  --assignee sarah@jockibox.com \
  --notes "Reported by Jock via Telegram"
```

Then confirm to the user with the task name. Include the Asana link if available from the response (`permalink_url`).

### Showing open tasks

```bash
asana tasks list --completed false
```

Format the output for Telegram:
```
*Open tasks:*
• Fix auth bug on staging _(Sarah, due Mar 7)_
• Update onboarding flow _(unassigned)_
• Deploy v2.3 to prod _(Jock, due Mar 5)_ 🔴 overdue

3 open tasks
```

### Completing a task

Find the task first, then mark it done:

```bash
# Search by name
asana tasks search "auth bug" --completed false

# Mark complete (use the gid from search results)
asana tasks update <gid> --completed true
```

### Viewing task details

```bash
asana tasks get <gid>
```

Format key fields: name, assignee, due date, notes, section, and Asana URL.

### Updating tasks

```bash
# Reassign
asana tasks update <gid> --assignee jock@jockibox.com

# Change due date
asana tasks update <gid> --due 2026-03-14

# Update description
asana tasks update <gid> --notes "Updated: also fix the password reset flow"
```

### What's due today / this week / overdue

Use the `tasks due` shortcut for common date queries:

```bash
# What needs attention today?
asana tasks due today

# What's coming this week?
asana tasks due week

# What's overdue?
asana tasks due overdue

# What's due tomorrow?
asana tasks due tomorrow

# Scoped to a person
asana tasks due today --assignee sarah@jockibox.com
```

For custom date ranges, use search with date filters:

```bash
# Tasks due on a specific date
asana tasks search --due-on 2026-03-15 --completed false

# Tasks due before end of month
asana tasks search --due-before 2026-03-31 --completed false

# Tasks completed this week (for standup)
asana tasks search --completed-after 2026-02-24 --completed true
```

### Status summary

```bash
# Get counts for a standup
asana tasks due overdue > /tmp/overdue.json
asana tasks list --completed false > /tmp/open.json
asana tasks search --completed-after "$(date -d '-7 days' +%Y-%m-%d 2>/dev/null || date -v-7d +%Y-%m-%d)" --completed true > /tmp/done_week.json

echo "Open: $(jq '.data | length' /tmp/open.json)"
echo "Overdue: $(jq '.data | length' /tmp/overdue.json)"
echo "Done this week: $(jq '.data | length' /tmp/done_week.json)"
```

Format as a standup-style summary:
```
*Status update:*
• 5 open tasks (1 overdue)
• 3 completed this week
• Blocked: none

*Overdue:*
• Deploy v2.3 to prod _(Jock, was due Mar 1)_

*Due today:*
• Fix auth bug on staging _(Sarah)_
• Review PR #42 _(Jock)_
```

### Moving due dates

When someone says "push the auth bug to next Friday" or "move the deploy to March 14":

```bash
# Find the task
asana tasks search "auth bug" --completed false

# Update the due date
asana tasks update <gid> --due 2026-03-14
```

Confirm with the old and new date:
```
Moved! *"fix auth bug"* due date changed from Mar 7 → Mar 14
```

## Team Meetings

### Monday Sync

Triggered by "monday sync", "weekly sync", "kick off the week" — or auto-posted Monday mornings via scheduled task.

Workflow:
1. Gather data from Asana
2. Post the summary
3. Open the floor for discussion

```bash
# 1. Overdue from last week
asana tasks due overdue > /tmp/overdue.json

# 2. What's due this week
asana tasks due week > /tmp/this_week.json

# 3. What got done last week
last_monday=$(date -d 'last monday' +%Y-%m-%d 2>/dev/null || date -v-7d +%Y-%m-%d)
asana tasks search --completed-after "$last_monday" --completed true > /tmp/done_last_week.json

# 4. Unassigned tasks
asana tasks list --completed false > /tmp/all_open.json
# Filter unassigned with jq:
jq '[.data[] | select(.assignee == null)]' /tmp/all_open.json > /tmp/unassigned.json
```

Format the Monday sync message:
```
*Monday Sync* 🗓️

*Carried over (overdue):*
• Deploy v2.3 to prod _(Jock, was due Mar 1)_

*This week:*
• Fix auth bug on staging _(Sarah, due Tue)_
• Review onboarding copy _(Jock, due Thu)_
• Ship billing fix _(unassigned, due Fri)_

*Completed last week:* ✅
• Set up CI pipeline _(Sarah)_
• Write API docs _(Jock)_

*Needs an owner:*
• Ship billing fix
• Update privacy policy

Anything to add or flag?
```

Always end with "Anything to add or flag?" to invite team input.

### Friday Review

Triggered by "friday review", "week in review", "wrap up the week" — or auto-posted Friday afternoons via scheduled task.

Workflow:
1. Gather what got done
2. What's carrying over
3. What's coming next week

```bash
# 1. Completed this week
this_monday=$(date -d 'last monday' +%Y-%m-%d 2>/dev/null || date -v-mon +%Y-%m-%d)
asana tasks search --completed-after "$this_monday" --completed true > /tmp/done_this_week.json

# 2. Still open (carrying over)
asana tasks list --completed false > /tmp/still_open.json

# 3. Overdue
asana tasks due overdue > /tmp/overdue.json

# 4. Due next week
next_monday=$(date -d 'next monday' +%Y-%m-%d 2>/dev/null || date -v+mon +%Y-%m-%d)
next_friday=$(date -d 'next friday' +%Y-%m-%d 2>/dev/null || date -v+fri +%Y-%m-%d)
asana tasks search --due-after "$next_monday" --due-before "$next_friday" --completed false > /tmp/next_week.json
```

Format the Friday review message:
```
*Friday Review* 🎉

*Done this week:* ✅
• Fix auth bug on staging _(Sarah)_
• Set up monitoring alerts _(Jock)_
• Review onboarding copy _(Jock)_

3 tasks completed!

*Carrying over:*
• Deploy v2.3 to prod _(Jock, due was Mon — overdue)_
• Ship billing fix _(unassigned)_

*Coming next week:*
• Launch beta signup page _(Sarah, due Tue)_
• Database migration _(Jock, due Thu)_

Anything to add or flag for next week?
```

Always end with "Anything to add or flag for next week?"

### Setting up scheduled meetings

Use the `schedule_task` MCP tool to auto-post these summaries. Set up once during initial configuration:

```
# Monday sync — every Monday at 9:00 AM
schedule_task(
  prompt: "Run the Monday Sync workflow. Query Asana for overdue tasks, this week's tasks, last week's completions, and unassigned items. Post the formatted summary to the group. End with 'Anything to add or flag?'",
  schedule_type: "cron",
  schedule_value: "0 9 * * 1",
  context_mode: "group"
)

# Friday review — every Friday at 4:00 PM
schedule_task(
  prompt: "Run the Friday Review workflow. Query Asana for this week's completions, carrying-over tasks, overdue items, and next week's upcoming tasks. Post the formatted summary to the group. End with 'Anything to add or flag for next week?'",
  schedule_type: "cron",
  schedule_value: "0 16 * * 5",
  context_mode: "group"
)
```

These can also be triggered manually anytime — someone just says "monday sync" or "friday review" and you run the same workflow on demand.

---

### Listing sections (workflow stages)

```bash
asana sections list <project_gid>
```

Use sections to understand workflow stages (e.g., "To Do", "In Progress", "Done").

### Adding tasks to sections

```bash
asana tasks create "New feature" --section <section_gid>
```

## User Mapping

Map Telegram sender names to Asana users. Check the group's CLAUDE.md for the mapping table. If a sender isn't in the mapping, create the task unassigned and mention who requested it in the notes.

To look up Asana users:

```bash
asana users list
```

## Task Identification

Use Asana GIDs internally. When talking to users, refer to tasks by name. If ambiguous, include enough context to identify: "the auth bug task (assigned to Sarah)".

When the user says "done with #3" or "close the auth task", search Asana to find the matching task, then update it.

## Error Handling

If Asana API fails:
1. Tell the user directly: "Couldn't reach Asana right now"
2. Do NOT queue or cache locally — Asana is the only source of truth
3. Suggest they try again in a moment

If a task isn't found:
1. Search with broader terms
2. Check if it was already completed
3. Ask the user for clarification

## Response Style

Keep confirmations short:
- Creating: `Got it! Created *"fix auth bug"* in Asana (assigned to Sarah, due Mar 7)`
- Completing: `Done! ✅ Marked *"fix auth bug"* as complete`
- Listing: Show task name, assignee first name, due date. Skip GIDs and noise.

## Command Reference

```bash
# Tasks — CRUD
asana tasks list [--project <gid>] [--assignee <email>] [--completed <bool>] [--section <gid>] [--limit <N>]
asana tasks create <name> [--project <gid>] [--assignee <email>] [--due <date>] [--notes "..."] [--section <gid>]
asana tasks update <gid> [--name "..."] [--assignee <email>] [--due <date>] [--notes "..."] [--completed <bool>]
asana tasks get <gid> [--fields <csv>]

# Tasks — Search with date filters
asana tasks search [<query>] [--project <gid>] [--assignee <email>] [--completed <bool>] [--limit <N>]
  [--due-on <date>] [--due-before <date>] [--due-after <date>] [--completed-after <date>]

# Tasks — Due date shortcuts
asana tasks due today      # What's due today
asana tasks due tomorrow   # What's due tomorrow
asana tasks due week       # Due in the next 7 days
asana tasks due overdue    # Past due and still open
  [--project <gid>] [--assignee <email>]

# Projects
asana projects list [--team <gid>] [--archived <bool>]
asana projects get <gid> [--sections]

# Sections
asana sections list <project_gid>

# Users
asana users list
asana users me
```
