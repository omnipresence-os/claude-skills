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

### Step 2: Resolve the active project (via project-resolver)

Follow the [project-resolver](../project-resolver/SKILL.md) protocol. Capture both:
- `chat_active_slug` — the chat-session marker resolution (may be null if no marker exists)
- `global_default_slug` — read `~/.claude/skills/.omnipresence-active-project` directly (project-resolver does this with `prefer: global-default`)

Both can be set, with chat-active overriding global-default for the current chat. The listing should mark BOTH if they differ (e.g., chat-active gets ⭐, global-default gets 🏠) so the operator sees which chat overrides their default. If they're the same, one mark is fine.

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

⭐ Active for this chat: <chat_active_slug or "(none — using global default below)">
🏠 Global default:       <global_default_slug or "(none set)">

Switch this chat's active project: `Switch to project <slug>.`
Change your global default:        `Set <slug> as my default project.`
Create a new project:               `Create a new project for X.`
```

If neither is set, both lines show "(none)" and a hint: *"Use `Switch to project X` (this chat only) or `Set X as my default project` (across all chats)."* If chat-active and global-default are the same, only show one line (⭐): "Active (this chat + global default): <slug>".

### Stop here.

## What this skill MUST NOT do

- Do NOT modify any files.
- Do NOT auto-pick an active project just because the user listed.
- Do NOT show raw filesystem listings — translate to the human-readable format above.
