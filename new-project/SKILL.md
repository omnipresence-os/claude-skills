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
- `custom/projects/<slug>/writing-samples/` (with a README explaining what to drop in)
- `custom/projects/<slug>/strategies/` (with `.gitkeep`)
- `custom/projects/<slug>/workers/` (with `.gitkeep` — populated by `create-worker`)

**Files (templates below):**

Four of these files ship as TBD placeholders so that future agents can detect they're empty and offer to fill them. Each placeholder has a `<!-- TBD-PLACEHOLDER: <slot> -->` marker at the top — agents grep for this string to know whether a slot has been filled. Detection plus interrupt-and-fill lives in the `fill-project-gap` skill; here we just write the stubs.

#### `custom/projects/<slug>/README.md`

```markdown
# <display_name>

> <one_line_description>

**Slug:** `<slug>`
**Created:** <YYYY-MM-DD>
**Active strategies:** 0 (run `Create a new strategy for this project` to add one)

## What's in this folder

Core files (filled in over time — every project needs these eventually):
- `project-config.md` — the 8-field structured config (or TODO if deferred)
- `style-guide.md` — writing voice, tone, brand rules **(starts as TBD placeholder)**
- `glossary.md` — preferred terminology, central-entity names **(starts as TBD placeholder)**
- `banned-phrases.md` — never-say list **(starts as TBD placeholder)**
- `editor-rules.md` — editorial publish gates **(starts as TBD placeholder)**
- `notes.md` — operator running log

Collections:
- `writing-samples/` — voice samples (drop in over time as you accumulate them)
- `strategies/` — multi-day/week implementation plans for this project
- `workers/` — saved worker specs (populated by `create-worker`)
- `outputs/` — generated drafts, audits, reports (populated as work runs)

Project-specific files (create only when you have content for them):
- `bios.md` — for personal-brand projects
- `messaging.md` — for surface-copy projects (homepage, LinkedIn, etc.)
- `image-style-guide.md` / `image-policy.md` — for projects with explicit visual rules
- `prompt-overrides/` — per-pipeline-stage system prompt overrides

## Filling in the TBD placeholders

The four placeholder files above are intentionally short stubs. When an agent loads this project and notices a placeholder, it'll offer to walk you through filling it (~2-3 min each). Or trigger it explicitly:

- `Fill in my style guide`
- `Fill in my glossary`
- `Fill in my banned phrases`
- `Fill in my editor rules`

You don't have to fill them all up front. The system works fine with partial content — placeholders just surface as "heads up" notes when relevant.

## Common prompts

- `What's the brand voice for this project?` — surfaces style-guide.md
- `Create a new strategy for X` — adds a strategy to strategies/
- `Create a worker for X` — sets up a reusable specialist for this project
- `List strategies for this project` — shows what's running
- `Continue strategy X` — resume a strategy from where you left off
```

#### `custom/projects/<slug>/style-guide.md`

```markdown
# Style Guide — <display_name>

<!-- TBD-PLACEHOLDER: style-guide. Run `Fill in my style guide` or paste your voice rules below. -->

> What good content for this slot looks like: a short, opinionated description of voice + tone, followed by a hard list of rules (never-do-X, always-do-Y) and 2-3 example paragraphs that exemplify the voice. Two pages max. Specific beats comprehensive.

## Voice

_(Conversational? Authoritative? Technical? Plain-language? Pick one or two. Avoid "professional yet friendly" — that's filler.)_

## Tone

_(Warm? Direct? Skeptical? Dry? Same rule — specific words, not "appropriate".)_

## Rules

_(One per line. Examples: "Never use em-dashes." / "Always cite sources inline." / "Lead with the answer, not the setup.")_

## Examples

_(Two or three short paragraphs that exemplify the voice. Real writing from the brand, ideally — extracted from `writing-samples/` if available.)_
```

#### `custom/projects/<slug>/glossary.md`

```markdown
# Glossary — <display_name>

<!-- TBD-PLACEHOLDER: glossary. Run `Fill in my glossary` or paste preferred terminology below. -->

> What good content for this slot looks like: a small table of preferred names + alternatives + things-to-avoid. The central entity (the brand's main thing) is the most important entry — agents check this when deciding what to call the product / company / person.

## Preferred terms

| Concept | Preferred | Alternatives OK | Avoid |
|---|---|---|---|
| _(e.g., the central entity)_ | _(e.g., "Omnipresence")_ | _(e.g., "Omni")_ | _(e.g., "the assistant")_ |
| | | | |

## Custom terminology

_(Domain-specific terms the brand uses with a non-obvious meaning. One per entry: term + definition + example sentence.)_
```

#### `custom/projects/<slug>/banned-phrases.md`

```markdown
# Banned Phrases — <display_name>

<!-- TBD-PLACEHOLDER: banned-phrases. Run `Fill in my banned phrases` or paste your never-say list below. -->

> What good content for this slot looks like: a flat list of exact phrases / words / patterns that should never appear in content for this brand. Agents check every draft against this list. Specific beats comprehensive — "Leverage" beats "corporate-speak in general".

## Never use

_(One per line. Examples: "leverage" / "synergy" / "in today's fast-paced world" / em-dashes / oxford comma in lists / starting a sentence with "Indeed,")_

## Pattern bans

_(Sentence shapes or structural patterns to avoid. Examples: "Don't start posts with a question." / "No hedge words like 'might' or 'perhaps' in opening sentences.")_
```

#### `custom/projects/<slug>/editor-rules.md`

```markdown
# Editor Rules — <display_name>

<!-- TBD-PLACEHOLDER: editor-rules. Run `Fill in my editor rules` or paste your publish gates below. -->

> What good content for this slot looks like: the gates a piece has to pass before it ships. Each gate is a yes/no test. Drafts that fail any gate go back for revision. Specific beats comprehensive — three sharp gates beat ten fuzzy ones.

## Publish gates

_(One per line. Each is a yes/no test. Examples: "Has at least one citation to a primary source." / "Opens with a declarative answer in sentence 1." / "Passes the read-aloud test — sounds like a human, not a press release.")_

## Auto-reject conditions

_(One per line. Examples: "Any banned phrase from `banned-phrases.md` appears." / "Word count exceeds <N> words." / "Uses passive voice in the opening paragraph.")_
```

#### `custom/projects/<slug>/notes.md`

```markdown
# Notes — <display_name>

<!-- Operator running log. Append decisions, surprises, and "remember this for next time" entries with a date stamp. -->
```

#### `custom/projects/<slug>/writing-samples/README.md`

```markdown
# Writing Samples — <display_name>

Drop existing writing from the brand into this folder — one piece per file. Agents reference these when matching voice for new content.

## What to add

- Published blog posts / pages that nail the voice
- Email or LinkedIn copy the brand owner is proud of
- Spoken transcripts of the brand owner (raw voice — often the truest signal)
- A short note about WHY this sample is on-brand if it's not obvious

## What NOT to add

- Generic "look at my LinkedIn bio" stuff that's been ghostwritten or AI-generated
- Content from a different brand
- Samples where the voice is aspirational rather than current — the system mimics what's here, not what you wish was here

## File naming

Free-form, but keep it scannable: `2026-05-blog-post-on-X.md`, `linkedin-post-Y.md`, `podcast-transcript-Z.md`. The date helps when voice drifts over time.

This folder starts empty. Add one sample to start — even one good sample beats none.
```

#### `custom/projects/<slug>/project-config.md`

If `config_action == "deferred"`, write:

```markdown
---
project: <slug>
status: deferred
---

# Project Config — <display_name>

<!-- TBD-PLACEHOLDER: project-config. Run `Generate project config for <slug>` to fill this in via the structured 8-field interview. -->
```

If `config_action == "generate-now"`, leave this file unwritten for now — Step 7 will chain to project-config-generation which writes it.

### Step 7: Optionally chain to project-config-generation

- **If `config_action == "deferred"`**, skip to Step 8.
- **If `config_action == "generate-now"`**, invoke the `project-config-generation` synapse process (look it up via the MCP lookup tool if needed, or follow the steps inline from `core/processes/client-engagement/project-config-generation.md`). The process's Step 6b will write `custom/projects/<slug>/project-config.md`.

### Step 8: Set the new project as active (chat-session by default)

Emit the chat-session marker so subsequent project-aware skills in this chat use the new project:

```
[OMNI_SESSION_ACTIVE = <slug>]
```

This is chat-scoped only — it does NOT touch the global default at `~/.claude/skills/.omnipresence-active-project`. If the user only has one project (this newly-created one) AND no global default is set, ALSO write the slug to the global pointer file (one-project members benefit from having the global default be their only project). Otherwise, leave the global pointer alone — the member can opt in to making this the global default by saying *"set <project-name> as my default project."*

### Step 9: Report success

Tell the user, exactly:

```
✅ Project '<display_name>' created and set as active.

  Folder: custom/projects/<slug>/

Created:
  • README.md
  • style-guide.md       (TBD placeholder)
  • glossary.md          (TBD placeholder)
  • banned-phrases.md    (TBD placeholder)
  • editor-rules.md      (TBD placeholder)
  • notes.md
  • writing-samples/     (empty — README inside explains what to add)
  • strategies/          (empty)
  • workers/             (empty — populated by `Create a worker`)
  • project-config.md    (<status: ready | deferred>)

Active project is now '<slug>'. Subsequent prompts will source from this folder.

About the TBD placeholders: I created four stub files (style-guide, glossary, banned-phrases, editor-rules) with short explanations of what good content looks like for each slot. You don't need to fill them all now — agents will notice when they need one and offer to walk you through filling it. Or trigger it manually:

  • Fill in my style guide
  • Fill in my glossary
  • Fill in my banned phrases
  • Fill in my editor rules

Next prompts you might want:

  • Add a strategy:           Create a new strategy for <display_name>.
  • Set up a content writer:  Create a worker for content writing.
  • Fill in brand voice:      Fill in my style guide.
  • Save to GitHub:           Push my synapse changes.
```

### Stop here. Do NOT propose other steps.

## What this skill MUST NOT do

- Do NOT create files outside `custom/projects/<slug>/`.
- Do NOT prompt the user with shell commands they have to type.
- Do NOT overwrite an existing project folder (Step 3 catches this).
- Do NOT skip setting active project (Step 8 — every other skill depends on it).
- Do NOT auto-run project-config-generation without the user choosing "Now" in Step 5.
