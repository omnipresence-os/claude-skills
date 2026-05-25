---
name: list-strategies
description: Lists all strategies for the active project (or all projects if requested). Shows each strategy's slug, display name, status, % complete (checked / total checkboxes), and last-modified date. Trigger when the user says any of "list strategies", "what strategies am I running", "show my strategies", "list strategies for this project", "list all strategies across projects". Read-only.
---

# List Strategies — Show All Strategies for the Active Project

Read-only listing of strategy files in the active project's `strategies/` folder.

## How to talk to the user

Plain English. Clean tabular output.

## Execute these steps

### Step 1: Resolve active project (via project-resolver)

Follow the [project-resolver](../project-resolver/SKILL.md) protocol: chat-session marker first, then global default.

- **If user explicitly asked for "all projects":** skip resolver, jump to Step 2's all-projects branch.
- **If resolver returns "none" AND zero projects exist:** tell the user: *"You don't have any projects yet. Run `Create a new project for X` first."* STOP.
- **If resolver returns "none" AND multiple projects exist:** ask *"Which project? Or say 'all' for all projects."* — wait for response. If they pick a specific one, emit `[OMNI_SESSION_ACTIVE = <slug>]` so subsequent project-aware skills in this chat use it.
- **If resolver returns a project but the folder is missing:** treat as the "none" case.

**In the output's first line, surface the resolved project + source** (when listing a single project, not when listing all):

```
Listing strategies for project=<slug> (resolved from <chat-session | global-default>).
```

Read `~/.claude/skills/.omnipresence-path`.

### Step 2: Walk strategies

**Single-project mode (default):** look at `<synapse-path>/custom/projects/<active-slug>/strategies/`.

**All-projects mode (when user said "all"):** for each `<slug>` in `custom/projects/`, look at `<slug>/strategies/`.

For each strategy file (`.md` ignoring `.gitkeep`):
- Read frontmatter: `id`, `created`, `status`, `autonomous`, `target`.
- Read body: count `- [ ]` (unchecked) and `- [x]` (checked). Total = sum.
- Get file modification time.

### Step 3: Empty case

If no strategies exist:

```
You don't have any strategies for '<active-display-name>' yet.

Create one with `Create a new strategy for this project.`
```

### Step 4: Render the list

```
Strategies for '<active-display-name>' (<N> total):

  <slug>                  <display name>
                          Status: in-progress | paused | done
                          Progress: <checked>/<total> steps (<percent>%)
                          Last activity: <YYYY-MM-DD>
                          Target: <target one-liner>

  <slug>                  ...

Resume one with `Continue strategy <slug>.`
Create a new one with `Create a new strategy for this project.`
```

**For all-projects mode:** group by project header:

```
All strategies across <N> projects:

  ── ACMECORP ──
    <slug>                <display name>
                          Status: ... Progress: ...

  ── TEALHQ ──
    <slug>                ...
```

**Sort order:** within each project, in-progress strategies first (most recent activity first), then paused, then done.

### Stop here.

## What this skill MUST NOT do

- Do NOT modify any file.
- Do NOT auto-pick one to continue.
- Do NOT show raw frontmatter — translate to human-readable.
