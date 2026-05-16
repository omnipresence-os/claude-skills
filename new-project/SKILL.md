---
name: new-project
description: Creates a new project folder under custom/projects/<slug>/ in the member's synapse fork — brand/, strategies/, README.md, project-config.md, notes.md. Trigger when the user says any of "create a new project", "set up a new client", "add a new project called X", "new project for X", "start a new client", "onboard a new project". Implements the new-project-setup process from synapse. Optionally chains to project-config-generation if the member wants the structured 8-field config. Sets the new project as active. Idempotent on re-run when the project already exists (it redirects to switch-project instead of clobbering).
---

# New Project — Stand Up a Member's Per-Client Folder

This skill creates a new per-project folder in the member's synapse fork. ONE canonical flow. Follow the steps exactly.

## How to talk to the user during this skill

**Critical UX rule:** do NOT show the user shell commands or terminal output. Run silently and explain in plain English. The user is not a coder.

✅ Good: *"Creating the project folder structure... done. I added the default files (style guide starter, README, strategies folder)."*
❌ Bad: *"Running `mkdir -p custom/projects/acmecorp/brand`..."*

Only show: plain-English progress, the final folder path, and the next-step prompts.

## Prerequisites

- The member has run `getting-started` (synapse fork exists, path cached at `~/.claude/skills/.omnipresence-path`).

## Execute these steps in order

### Step 1: Resolve the synapse fork path

Read `~/.claude/skills/.omnipresence-path` (or `%USERPROFILE%\.claude\skills\.omnipresence-path` on Windows).

- **If found and valid** (contains `core/`, `custom/`, `package.json`), use it.
- **If not found**, search `~/Documents/omnipresence/synapse`, `~/synapse`, `~/dev/synapse`, `~/Code/synapse`.
- **If still not found**, tell the user: *"Omnipresence isn't set up yet on this machine. Run `getting started` first, then come back to create your project."* STOP.

### Step 2: Get the project name and derive a slug

From the user's prompt (e.g., "Create a new project for AcmeCorp"), extract the project display name.

If the prompt didn't include a name, ask: *"What's the project called?"*

Derive a kebab-case slug:
- Lowercase
- Spaces → hyphens
- Strip non-alphanumeric except hyphens
- Collapse double hyphens
- Trim leading/trailing hyphens

Example: "AcmeCorp Marketing" → `acmecorp-marketing`. "Teal HQ" → `teal-hq`.

Show the user: *"I'll use `<slug>` as the folder name. Press Enter to accept, or paste a different slug."*

If the user pastes an alternative, use that (lightly validate it matches kebab-case rules; if not, fix it).

### Step 3: Check for slug collision

Check if `<synapse-path>/custom/projects/<slug>/` already exists.

- **If yes**, tell the user: *"A project with slug `<slug>` already exists. To work on it, say `Switch to project <slug>`. If you meant to create a NEW project that's similar, pick a different slug."* STOP.
- **If no**, proceed.

### Step 4: Capture one-line description

Ask: *"In one sentence, what does this project do? (Example: 'B2B SaaS for HR teams, helps with employee onboarding.')"*

Wait for response. If the user gives a multi-sentence answer, accept it (we'll use the whole thing in the README).

### Step 5: Ask about structured config now vs later

Ask: *"Do you want me to generate the structured project config now (8 questions about your brand, topical map, voice — takes 2-3 minutes), or do you want to skip and run it later with `Generate project config for <slug>`?"*

Options: *"Now"* / *"Later"*.

- **Now:** mark `config_action = "generate-now"`.
- **Later:** mark `config_action = "deferred"`.

### Step 6: Create the folder structure

The project folder is **flat** — files live directly at the project root. No `brand/` subfolder; the slug already brands the folder. Only collections (writing-samples/, strategies/, optional prompt-overrides/) nest as subfolders.

Create these directories and files at `<synapse-path>/custom/projects/<slug>/`:

**Directories:**
- `custom/projects/<slug>/`
- `custom/projects/<slug>/writing-samples/` (with `.gitkeep` so git tracks the empty dir)
- `custom/projects/<slug>/strategies/` (with `.gitkeep`)

**Files (templates below):**

#### `custom/projects/<slug>/README.md`

```markdown
# <display_name>

> <one_line_description>

**Slug:** `<slug>`
**Created:** <YYYY-MM-DD>
**Active strategies:** 0 (run `Create a new strategy for this project` to add one)

## What's in this folder

- `project-config.md` — the 8-field structured config (or TODO if deferred)
- `style-guide.md` — writing voice, tone, brand rules
- `writing-samples/` — voice samples (add as you accumulate them)
- `strategies/` — multi-day/week implementation plans for this project
- `notes.md` — operator running log

Optional files (create when you have content for them):
- `glossary.md` — preferred terminology
- `banned-phrases.md` — never-say list
- `editor-rules.md` — editorial constraints
- `image-style-guide.md` — visual brand
- `prompt-overrides/` — per-pipeline-stage system prompt overrides

## Common prompts

- `What's the brand voice for this project?` — surfaces style-guide.md
- `Create a new strategy for X` — adds a strategy to strategies/
- `List strategies for this project` — shows what's running
- `Continue strategy X` — resume a strategy from where you left off
```

#### `custom/projects/<slug>/style-guide.md`

```markdown
# Style Guide — <display_name>

<!-- Starter style guide. Fill in or run 'Update brand voice for <slug>' to have Omni draft from your project config. -->

## Voice

(Conversational? Authoritative? Technical? Plain-language?)

## Tone

(Warm? Direct? Formal?)

## Rules

(One per line: e.g., "Never use em-dashes." / "Always cite sources inline.")

## Examples

(Two or three short paragraphs that exemplify the voice.)
```

#### `custom/projects/<slug>/notes.md`

```markdown
# Notes — <display_name>
```

#### `custom/projects/<slug>/project-config.md`

If `config_action == "deferred"`, write:

```markdown
---
project: <slug>
status: deferred
---

# Project Config — <display_name>

<!-- TODO: run `Generate project config for <slug>` to fill this in. -->
```

If `config_action == "generate-now"`, leave this file unwritten for now — Step 7 will chain to project-config-generation which writes it.

### Step 7: Optionally chain to project-config-generation

- **If `config_action == "deferred"`**, skip to Step 8.
- **If `config_action == "generate-now"`**, invoke the `project-config-generation` synapse process (look it up via the MCP lookup tool if needed, or follow the steps inline from `core/processes/client-engagement/project-config-generation.md`). The process's Step 6b will write `custom/projects/<slug>/project-config.md`.

### Step 8: Set the new project as active

Write `<slug>` to `~/.claude/skills/.omnipresence-active-project` (or `%USERPROFILE%\.claude\skills\.omnipresence-active-project` on Windows). Overwrite any existing value.

### Step 9: Report success

Tell the user, exactly:

```
✅ Project '<display_name>' created and set as active.

  Folder: custom/projects/<slug>/

Created:
  • README.md
  • style-guide.md (starter)
  • writing-samples/ (empty)
  • strategies/ (empty)
  • notes.md
  • project-config.md (<status: ready | deferred>)

Active project is now '<slug>'. Subsequent prompts will source from this folder.

Next prompts you might want:

  • Add a strategy:        Create a new strategy for <display_name>.
  • Update brand voice:    Update brand voice for <slug>.
  • Save to GitHub:        Push my synapse changes.
```

### Stop here. Do NOT propose other steps.

## What this skill MUST NOT do

- Do NOT create files outside `custom/projects/<slug>/`.
- Do NOT prompt the user with shell commands they have to type.
- Do NOT overwrite an existing project folder (Step 3 catches this).
- Do NOT skip setting active project (Step 8 — every other skill depends on it).
- Do NOT auto-run project-config-generation without the user choosing "Now" in Step 5.
