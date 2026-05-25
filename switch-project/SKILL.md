---
name: switch-project
description: Sets the active project for this chat (per-chat scope, default) or as the member's global default across all chats (opt-in). Trigger when the user says any of "switch to project X", "set active project to X", "work on X", "I'm working on X today", "use project X", "make X the active project". For the global-default variant trigger on "set X as my default project", "make X my default project", "set default project to X", "this should be my default project". The chat-scope (default) is meant for the common agency workflow of running parallel chats per client — each chat's active project is independent. The global default applies whenever a chat doesn't have its own session-active set; single-client members use it as their permanent setting. Handles fuzzy matching when the user's input doesn't exactly match a folder name. Auto-picks the single project when there's only one and no active is set.
---

# Switch Project — Set Active Project (Per-Chat by Default, Globally by Opt-In)

Two scopes, one skill. The default scope is **per-chat** so parallel chats (e.g., "Spellbook writer" + "Kyper writer" running simultaneously) don't trample each other. The **global** scope is for members who only work on one project — set it once during onboarding, every chat picks it up.

## How to talk to the user

Plain English. No shell commands shown. ALWAYS surface which scope (chat / global) you wrote to at the end so the member knows.

## Execute these steps

### Step 1: Resolve synapse fork path

Read `~/.claude/skills/.omnipresence-path`. If missing, search common locations. If still missing, tell the user: *"Omnipresence isn't set up yet. Run `getting started` first."* STOP.

### Step 2: Detect scope (chat vs global)

Default scope is **chat** — overridden by explicit signals in the prompt:

- *"set as my default project"*, *"make X my default"*, *"set default to X"*, *"globally"*, *"--global"*, *"for all chats"* → **global** scope.
- Everything else (default phrasing "switch to X", "work on X", etc.) → **chat** scope.

If the prompt is ambiguous AND the member has more than one project, ask once:

> *"Set X as the active project for **just this chat** (so other chats stay on their own active projects), or as your **global default** (becomes the active project for any chat that doesn't override)?"*

Single-project members default to global silently (no ambiguity to resolve).

### Step 3: Extract the target slug

Parse the user's prompt for the project name. Apply kebab-case normalization (same as `new-project`): lowercase, hyphens for spaces, strip non-alphanumeric except hyphens.

Examples:
- *"Switch to project AcmeCorp"* → `acmecorp`
- *"Set Teal HQ as my default"* → `teal-hq`
- *"Work on my fitness blog"* → `my-fitness-blog`

### Step 4: Validate the target matches a project folder

Look at `<synapse-path>/custom/projects/`. Check if `<target-slug>/` exists.

- **Exact match:** proceed to Step 5.
- **No exact match:** apply fuzzy matching (substring or Levenshtein ≤ 3). Behavior identical to the previous version of this skill — see below for the cases.
  - **One close match:** ask: *"I don't see `<target>`, but `<close-match>` is close. Did you mean that?"* — Yes uses the close match.
  - **Multiple close matches:** list them and ask the member to pick.
  - **No matches and no projects exist:** tell the member to run `Create a new project for X` first. STOP.
  - **No close matches but projects exist:** list all projects, ask which one.

### Step 5: Single-project auto-pick (edge case)

If the user's intent is ambiguous (e.g., just "switch project" with no name) AND only one project folder exists, auto-pick it silently. Don't ask. Use **chat** scope by default; if they meant global they can re-issue with "set as default."

### Step 6: Write the resolved scope

**For chat scope (default):**

Emit a clearly-formatted marker line in your response output that `project-resolver` will scan for on subsequent skill calls in this chat:

```
[OMNI_SESSION_ACTIVE = <resolved-slug>]
```

The marker MUST be on its own line, with the exact bracket + token format above. project-resolver scans the conversation history for the most recent such line.

For belt-and-suspenders robustness against conversation compaction, ALSO write the slug to a session-scoped backup file if the agent harness exposes a session ID. On Claude Code, a reasonable path is `<claude-temp-dir>/omni-session-active` where `<claude-temp-dir>` is the per-session temp directory the harness creates (varies by Claude Code version; if you can't reliably get it, skip — the conversation marker is the primary mechanism).

DO NOT write to `~/.claude/skills/.omnipresence-active-project` in chat scope. That's the global pointer; chat-scope writes leave it alone.

**For global scope:**

Write `<resolved-slug>` to `~/.claude/skills/.omnipresence-active-project` (Mac/Linux) or `%USERPROFILE%\.claude\skills\.omnipresence-active-project` (Windows). Overwrite any existing value.

ALSO emit the chat-session marker, so the chat you ran this command IN reflects the change immediately (otherwise the chat would still use the OLD chat-session value, if any, until a new chat is opened):

```
[OMNI_SESSION_ACTIVE = <resolved-slug>]
```

### Step 7: Report success

Read the project's `README.md` to get the display name.

**For chat scope:**

```
✅ Switched to project '<display_name>' for this chat.

  Slug: <slug>
  Folder: custom/projects/<slug>/
  Strategies in progress: <count>
  Scope: chat-session (this chat only — your global default is unchanged)

Subsequent prompts in THIS chat will source from this project. Other chats stay on their own active projects. To make this your default across all chats, say: "Set <display_name> as my default project."
```

**For global scope:**

```
✅ Set '<display_name>' as your global default project.

  Slug: <slug>
  Folder: custom/projects/<slug>/
  Strategies in progress: <count>
  Scope: global (default for every chat that doesn't override)

This is also active in THIS chat. Any chat that explicitly sets its own active (via "switch to X") will override this default for that chat only.
```

If the project has no strategies yet, omit the "Strategies in progress" line and replace with: *"This project doesn't have any strategies yet — say `Create a new strategy for this project` to add one."*

### Stop here.

## What this skill MUST NOT do

- Do NOT create a new project if the target doesn't match. Always ask first.
- Do NOT modify any project files; just the active-project pointer (global) and/or the chat conversation marker.
- Do NOT write to the global pointer file in chat scope. Members frequently want to "try" a project for one chat without changing their default — global writes should be explicit.
- Do NOT omit the `[OMNI_SESSION_ACTIVE = <slug>]` marker. project-resolver depends on it; skipping it means downstream skills in this chat won't see the switch.
- Do NOT show raw filesystem output (write paths, hex dumps, etc.).
