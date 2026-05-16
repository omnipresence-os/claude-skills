---
name: list-projects
description: Lists all projects in the member's synapse fork at custom/projects/. Trigger when the user says any of "list my projects", "show all projects", "what projects do I have", "show me my projects", "all my clients", "list clients". Walks custom/projects/ and prints a clean human-readable list with project name, one-line description, active strategies count, last-modified date. Stars the currently active project. Read-only — never modifies anything.
---

# List Projects — Show All Projects in the Member's Fork

Read-only listing of every project under `custom/projects/`. Marks the active one.

## How to talk to the user

Don't show shell commands. Plain English explanation of what was found.

## Execute these steps

### Step 1: Resolve synapse fork path

Read `~/.claude/skills/.omnipresence-path`. If missing, search common locations. If still missing, tell the user: *"Omnipresence isn't set up yet. Run `getting started` first."* STOP.

### Step 2: Read the active project

Read `~/.claude/skills/.omnipresence-active-project`. If file exists, capture as `active_slug`. If missing or empty, set `active_slug = None`.

### Step 3: Walk the projects directory

Look at `<synapse-path>/custom/projects/`.

- **If the directory doesn't exist OR is empty**, tell the user: *"You don't have any projects yet. Run `Create a new project for X` to get started."* STOP.

- **Otherwise**, list every subdirectory. For each subdirectory `<slug>/`:
  - Read its `README.md` (silently). Extract the first H1 (display name) and the first blockquote (one-line description).
  - Count `.md` files in `<slug>/strategies/` (excluding `.gitkeep`).
  - Get last-modified date of the folder (or its most recently modified file inside).
  - Mark this entry with `★` if `<slug> == active_slug`.

### Step 4: Render the list

Tell the user:

```
Your projects (<N> total):

  ★ <slug>          <display_name>
                    <one_line_description>
                    Strategies: <strategies_count>
                    Last activity: <YYYY-MM-DD>

    <slug>          <display_name>
                    <one_line_description>
                    Strategies: <strategies_count>
                    Last activity: <YYYY-MM-DD>

  ...

Active project: <active_slug or "none set">

Switch projects with `Switch to project <slug>.`
Create a new one with `Create a new project for X.`
```

If no active project is set, omit the star and the active-project line gets "(none set — say `Switch to project <slug>` to pick one)".

### Stop here.

## What this skill MUST NOT do

- Do NOT modify any files.
- Do NOT auto-pick an active project just because the user listed.
- Do NOT show raw filesystem listings — translate to the human-readable format above.
