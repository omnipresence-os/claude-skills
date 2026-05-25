---
name: project-info
description: Surfaces information about the active project — config, brand voice, style guide, strategies list, recent notes, or general overview depending on what the user asks. Trigger when the user says any of "what's the active project", "show me this project", "what's in this project", "what's the brand voice", "what's the style guide", "show me the config", "summarize this project", "what's the active project's X". Reads from custom/projects/<active>/ and surfaces the relevant file content. Read-only.
---

# Project Info — Surface Info About the Active Project

Routes the user's specific question to the right file in the active project folder.

## How to talk to the user

Plain English. Quote the relevant file content (don't just say "check the file"). When the file is long, summarize but offer to read more.

## Execute these steps

### Step 1: Resolve active project (via project-resolver)

Follow the [project-resolver](../project-resolver/SKILL.md) protocol: check the chat-session marker first (most recent `[OMNI_SESSION_ACTIVE = <slug>]` line in conversation), then the global default at `~/.claude/skills/.omnipresence-active-project`. Use whichever resolves first.

If neither resolves, tell the user: *"No active project is set for this chat, and no global default is configured. Pick one with `Switch to project X` (this chat only) or `Set X as my default project` (across all chats). Or run `List my projects` to see what you have."* STOP.

Read `~/.claude/skills/.omnipresence-path` for the synapse fork path. If missing, redirect to `getting-started`.

Verify `<synapse-path>/custom/projects/<active-slug>/` exists. If not, clear the chat-session marker (don't touch the global pointer — could be a stale chat-session reference to a now-deleted project) and tell the user the project folder is missing.

**In the output's first line, surface the resolved project + source** so the operator can verify the routing was correct:

```
Showing info for project=<slug> (resolved from <chat-session | global-default>).
```

### Step 2: Interpret the user's intent

Look at the user's prompt and route to one of these categories:

- **"What's the active project" / "show me this project" / "summarize this project"** → CATEGORY: overview
- **"What's the brand voice" / "show me the style guide" / "what's the writing style"** → CATEGORY: style-guide
- **"Show me the config" / "what's the project config" / "show project-config"** → CATEGORY: config
- **"What strategies" / "list strategies" / "show strategies"** → redirect to `list-strategies` skill
- **"What's the glossary" / "what are the banned phrases" / "show editor rules"** → CATEGORY: brand-file (with subtype)
- **"What's in my notes" / "show notes"** → CATEGORY: notes
- **Anything else specific** → try to match to a filename in `brand/` or surface a "I'm not sure what you want; here's what's in this project" overview

### Step 3a: Render the OVERVIEW

Read README.md + count strategies + read recent notes (last 3 lines).

The project folder is **flat** — files live at the project root, only collections (writing-samples/, strategies/, prompt-overrides/) are subfolders.

```
Active project: <display_name> (<slug>)

> <one_line_description>

What's here:
  • style-guide.md           — <line count from style-guide.md or "(starter — fill in)" if matches template>
  • writing-samples/         — <N> samples
  • glossary.md              — <exists | not yet>
  • banned-phrases.md        — <exists | not yet>
  • editor-rules.md          — <exists | not yet>
  • image-style-guide.md     — <exists | not yet>
  • project-config.md        — <ready | deferred>
  • strategies/              — <N> strategies (<M> in-progress)
  • notes.md                 — <N> lines

Want me to surface a specific part? Try:
  • What's the brand voice for this project?
  • Show me the project config.
  • List strategies for this project.
```

### Step 3b: Render STYLE-GUIDE

Read `<synapse-path>/custom/projects/<active-slug>/style-guide.md`. Surface the contents directly, quoted in markdown:

```
Brand voice for <display_name>:

<file contents>

(Source: custom/projects/<slug>/style-guide.md)
```

If the file is missing or contains the starter template placeholders, tell the user: *"The style guide is still a starter template — it hasn't been filled in. Want me to draft one based on the project config? Say `Update brand voice for <slug>`."*

### Step 3c: Render CONFIG

Read `<synapse-path>/custom/projects/<active-slug>/project-config.md`. Surface the parsed sections directly.

If the file shows `status: deferred` in frontmatter, tell the user: *"Project config is deferred. Generate it with `Generate project config for <slug>`."*

### Step 3d: Render BRAND-FILE (glossary, banned phrases, editor rules)

Map the user's intent to a specific file at the project root (flat — no `brand/` subfolder):
- glossary → `glossary.md`
- banned phrases → `banned-phrases.md`
- editor rules → `editor-rules.md`
- image style → `image-style-guide.md`

Read from `<synapse-path>/custom/projects/<active-slug>/<filename>.md` and surface. If missing, tell the user: *"There's no `<file>` for this project yet. Create one by adding `custom/projects/<slug>/<filename>.md` (or ask me to draft one)."*

### Step 3e: Render NOTES

Read `<synapse-path>/custom/projects/<active-slug>/notes.md`. Surface entire contents. If empty (just the header), tell the user: *"Notes file is empty. Add operator notes anytime — they're just a markdown file under `custom/projects/<slug>/notes.md`."*

### Stop here.

## What this skill MUST NOT do

- Do NOT modify any file. Read-only.
- Do NOT guess at the user's intent if the prompt is genuinely ambiguous; default to overview.
- Do NOT surface raw shell output or file paths in technical syntax — translate to human-readable.
