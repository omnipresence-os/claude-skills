---
name: project-resolver
description: Resolves which Omnipresence project a command should target — checking chat-session state first (set per-chat by switch-project), falling back to the global default (~/.claude/skills/.omnipresence-active-project), then surfacing "no project set" if neither exists. Called internally by every project-aware skill (new-strategy, continue-strategy, list-strategies, project-info, etc.) — not invoked directly by members. Trigger if you see internal invocations like "use project-resolver to find the active project" or "resolve project via project-resolver". This is the canonical resolution chain for per-chat active project, replacing direct reads of the global pointer file by project-aware skills.
---

# Project Resolver — Per-Chat Active Project Resolution

This is a HELPER skill called by other project-aware skills. Members don't invoke it directly. Each project-aware skill (`new-strategy`, `list-strategies`, `continue-strategy`, `project-info`, etc.) calls this skill at its start to figure out which project's folder to source from.

## The resolution chain

Check sources in this order. Stop at the first that yields a project slug:

### 1. Chat-session active (per-chat scope)

Look back through the current chat's conversation history for the most recent line matching the marker:

```
[OMNI_SESSION_ACTIVE = <slug>]
```

This marker is emitted by `switch-project` when the user switches projects within a chat (default behavior). It's chat-scoped — different chats have different histories, so different chats have different session-actives. Naturally isolates parallel agency workflows ("Spellbook writer" chat ≠ "Kyper writer" chat).

If you find a marker, return:
```json
{ "active_project": "<slug-from-marker>", "source": "chat-session" }
```

**On compaction:** if the conversation got compacted and the marker disappeared, this lookup misses. Falls through to source #2 (global default). Member can re-issue `switch-project` to restore the marker.

### 2. Global default (per-machine fallback)

Read `~/.claude/skills/.omnipresence-active-project` (Mac/Linux) or `%USERPROFILE%\.claude\skills\.omnipresence-active-project` (Windows). The file contains a single project slug on one line.

If present and non-empty, return:
```json
{ "active_project": "<slug-from-file>", "source": "global-default" }
```

This is the member's default project for any chat that doesn't override. Single-client members set this once during onboarding and never think about it. Agency owners typically set this to their most-used / personal-brand project and override per-chat for client work.

### 3. No project set

If neither source yields a slug, return:
```json
{ "active_project": null, "source": "none" }
```

The calling skill decides how to handle this. Most should ask the member: *"No active project for this chat — which one do you mean? (Or run `switch to project X` to set it for this chat / `set my default project to X` to set it globally.)"*

## Inputs

None required. The skill scans the chat history and the global pointer file on its own.

Optional input (rarely used):
- `prefer` — one of `chat-session` (only check chat marker, ignore global), `global-default` (only check global, ignore chat marker), `auto` (default — the full chain above). Use `prefer: global-default` when the calling skill explicitly wants the member's default regardless of chat state (e.g., the `list-projects` skill showing which project is starred as the default).

## Outputs

```json
{
  "active_project": "<slug>" | null,
  "source": "chat-session" | "global-default" | "none"
}
```

The source field is informational — calling skills can use it to surface the routing decision in their output (e.g., *"Drafting brief for project=spellbook (from chat-session)…"*) so members can verify the agent picked the right project.

## When in doubt: surface the resolution to the operator

Every project-aware skill should display the resolved project + source in its first line of output, like:

```
Drafting brief for project=spellbook (resolved from chat-session).
```

This makes wrong inferences immediately visible — if the marker got compacted and we fell back to the wrong global default, the operator sees it on the first line and can fix it.

## What this skill does NOT do

- Does NOT write any state. It's read-only. State changes happen via `switch-project`.
- Does NOT make assumptions about which project the user "probably" meant. If chat-session is empty and global-default is empty, it surfaces "none" and lets the caller ask.
- Does NOT support project name (display-name) lookup. Only slugs. Callers that want display-name resolution should chain to `switch-project`'s fuzzy-matching logic.

## See also

- [switch-project](../switch-project/SKILL.md) — writes the chat-session marker (and optionally the global default).
- [list-projects](../list-projects/SKILL.md) — uses this skill to mark which project is "active" in its listing.
- [new-strategy](../new-strategy/SKILL.md), [continue-strategy](../continue-strategy/SKILL.md), [list-strategies](../list-strategies/SKILL.md), [project-info](../project-info/SKILL.md) — all call this skill to resolve the project before sourcing from `custom/projects/<slug>/`.
