---
name: switch-project
description: Sets the active project for the member's session. Writes the project slug to ~/.claude/skills/.omnipresence-active-project so every project-aware skill sources from that folder going forward. Trigger when the user says any of "switch to project X", "set active project to X", "work on X", "I'm working on X today", "use project X", "make X the active project". Handles fuzzy matching when the user's input doesn't exactly match a folder name. Auto-picks the single project when there's only one and no active is set.
---

# Switch Project — Set the Active Project for the Session

Updates `~/.claude/skills/.omnipresence-active-project` so all downstream project-aware skills source from the right folder.

## How to talk to the user

Plain English. No shell commands shown.

## Execute these steps

### Step 1: Resolve synapse fork path

Read `~/.claude/skills/.omnipresence-path`. If missing, search common locations. If still missing, tell the user: *"Omnipresence isn't set up yet. Run `getting started` first."* STOP.

### Step 2: Extract the target slug from the user's prompt

Parse the user's prompt for the project name they're trying to switch to. The phrasing varies:
- "Switch to project AcmeCorp" → `acmecorp`
- "Set active project to Teal HQ" → `teal-hq`
- "Work on my fitness blog project" → `my-fitness-blog`

Apply the same kebab-case normalization as new-project: lowercase, hyphens for spaces, strip non-alphanumeric except hyphens.

### Step 3: Check if the target matches a project folder

Look at `<synapse-path>/custom/projects/`. Check if `<target-slug>/` exists.

- **Exact match:** proceed to Step 5.

- **No exact match:** apply fuzzy matching. List all existing project slugs and find the closest by:
  - Substring match (target contained in or contains existing slug)
  - Levenshtein distance ≤ 3
  
  - **One close match:** ask: *"I don't see a project called `<target>`, but `<close-match>` is close. Did you mean that?"* — wait for Yes/No.
    - Yes → use `<close-match>` as the target. Proceed to Step 5.
    - No → list all available projects and ask: *"Your projects are: <list>. Which one?"* — wait, then re-validate.
  
  - **Multiple close matches:** show all candidates: *"I don't see `<target>` exactly. Did you mean one of these? <list of close>."* — wait for choice.
  
  - **No close matches AND no projects at all:** tell the user: *"You don't have any projects yet. Run `Create a new project for X` first."* STOP.
  
  - **No close matches but projects exist:** show all projects: *"`<target>` isn't a project. Your projects are: <list>. Which one?"* — wait for choice.

### Step 4: Single-project auto-pick (edge case)

If the user's intent is ambiguous (e.g., they just said "switch project" with no name) AND only one project folder exists, auto-pick it silently. Don't ask.

### Step 5: Write the active-project file

Write `<resolved-slug>` to `~/.claude/skills/.omnipresence-active-project` (or `%USERPROFILE%\.claude\skills\.omnipresence-active-project` on Windows). Overwrite any existing value.

### Step 6: Report success

Read the project's `README.md` to get the display name.

Tell the user:

```
✅ Switched to project '<display_name>'.

  Slug: <slug>
  Folder: custom/projects/<slug>/
  Strategies in progress: <count of strategy files with 'status: in-progress' frontmatter>

Subsequent prompts will source from this project. The active-project slug is also passed to the Omnipresence MCP as `active_project` on lookups — your project's brand voice, style guide, glossary, and strategies surface as Tier-1 results above any methodology / process / skill content.

Useful next prompts:

  • Show me what's in this project:    What's in this project?
  • List its strategies:                List strategies for this project.
  • Continue a specific strategy:       Continue strategy <slug>.
```

If the project has no strategies yet, omit the "Strategies in progress" line and replace with: *"This project doesn't have any strategies yet — say `Create a new strategy for this project` to add one."*

### Stop here.

## What this skill MUST NOT do

- Do NOT create a new project if the target doesn't match. Always ask first.
- Do NOT modify any project files; just the active-project pointer.
- Do NOT show raw filesystem output.
