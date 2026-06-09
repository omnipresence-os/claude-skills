---
name: reload-worker
description: Re-warms a Claude Code chat session after context compaction (or any time the worker's auto-load corpus has been lost from context). Re-reads the active worker's spec file and re-loads every methodology / process / skill / project file in its auto-load block. Trigger when the user says any of "pick up where we left off", "rehydrate", "you compacted — restore my worker", "warm me back up", "reload the worker", "re-warm the session", "context compacted, restore me", "you lost context, reload", "the conversation just compacted", "I think you compacted", "rehydrate context", "re-load the worker", "context refresh". Also trigger silently when scanning a conversation summary post-compaction and detecting an `[OMNI_ACTIVE_WORKER = <slug>]` marker without the corresponding methodology files in current context. This skill is the SURVIVAL mechanism for the post-compaction context-loss problem — Claude Code's compaction hooks cannot re-inject file contents, so the only honest fix is a re-load triggered by the agent or user when compaction is detected. Idempotent — safe to call repeatedly. Pairs with load-worker (initial warmup) and create-worker (spec authoring).
---

# Reload Worker — Post-Compaction Context Recovery

When a Claude Code conversation compacts, the summary preserves the *fact* a worker was loaded but DROPS the contents of the methodology / process / skill files that were Read into context. This skill restores them.

## When this should fire

- **User-triggered:** anything like "pick up where we left off", "rehydrate", "you compacted — restore my worker", "warm me back up", "reload the worker".
- **Agent-triggered (silent):** if you (Claude) detect that the conversation has just been compacted AND a worker marker like `[OMNI_ACTIVE_WORKER = <slug>]` appears in the summary AND you cannot quote specific lines from the worker's auto-load files, you should invoke this skill BEFORE answering the user's next prompt. Tell the user *"Context just compacted — re-warming the `<slug>` worker so I don't drift."*

This is rule zero of [CLAUDE.md](../../Documents/Omnipresence/omnipresence-os/CLAUDE.md)'s **Compact Instructions** section: never trust post-compaction memory of methodology — always re-read.

## How to talk to the user during this skill

Fast. Plain English. Compact. Don't enumerate every file — surface counts. The user's annoyance budget for "the AI lost context" is low; don't dwell.

## Execute these steps in order

### Step 1 — Identify the active worker

Resolve the active worker in this order:

1. **Explicit slug in the user's prompt** (e.g., "reload content-writer", "rehydrate the link-builder") — use that slug directly.
2. **Chat-session marker:** scan the conversation (including any summary block) for `[OMNI_ACTIVE_WORKER = <slug>]`. Use the most recent match.
3. **Conversation summary mentions a worker:** if the summary says "acting as the content-writer worker" or similar, infer the slug.
4. **Project-level fallback:** if no worker marker exists but a project marker (`[OMNI_SESSION_ACTIVE = <slug>]` or global default) is set AND the project has exactly one worker, use that one.
5. **Cannot resolve:** ask the user *"Which worker was loaded? Your options for project `<slug>`: `<list of available workers>`."* If the user says "no worker, just project context," skip to Step 3 (re-load project files only).

### Step 2 — Resolve the worker spec path

Path: `<synapse-path>/custom/projects/<active-project-slug>/workers/<worker-slug>.md`

If missing, tell the user *"No worker spec found at `<path>`. Run `create-worker` to set one up, or pick from the existing workers: `<list>`."* STOP.

### Step 3 — Fan-out re-read every file in the auto-load block

This step is identical to [`load-worker`](../load-worker/SKILL.md) Step 3 — in a SINGLE message, call Read in parallel for every file listed in the worker spec's auto-load block. Apply local-first resolution (`custom/` → `overrides/` → `core/`). Expand glob patterns (`writing-samples/**`). Skill directory entries resolve to their `SKILL.md`.

Skip silently for files that are already verifiably in context (rare post-compaction, but possible if compaction was partial). If unsure, re-read — re-reads are cheap, drift is expensive.

### Step 4 — Surface the recovery confirmation

Tell the user, exactly:

```
🔄 Worker re-warmed: <display_name> for project '<active-project-slug>'.

  • Re-loaded: <N> methodologies, <M> processes, <K> skills, <P> project files
  • Mission: <mission from spec>
  • Autonomy: <Propose-then-execute | Autonomous>
  • Missing files: <list, if any — or 'none'>

Back to it. <pick one based on the conversation context:>
  - "Continuing the task — <one-line summary of what you were doing>."
  - "Ready. What's next?"
```

If there was an in-flight task at the time of compaction (visible in the summary), name it explicitly so the user doesn't have to repeat themselves.

### Step 5 — Resume

If an in-flight task is identifiable, resume it now, applying the freshly-loaded methodology. If unclear what was in flight, wait for the user's next prompt.

### Stop here.

## Edge cases

**The conversation didn't actually compact, the user just said "rehydrate" because they want a fresh state.** Treat it the same — re-load is idempotent. The cost is a few seconds of file reads.

**The worker spec itself has changed since the original load (someone edited it mid-session).** Surface the change in Step 4: *"Worker spec was updated since the original warmup — re-loaded the new version."* Don't compare contents, just re-load.

**The auto-load list references files that no longer exist locally (member is on a stale fork).** Same as load-worker — surface missing files in the confirmation block, suggest `sync-omnipresence`.

**Two workers were loaded in the original session (user kept switching).** Use the most recent marker. If unsure, ask.

**The user invoked this skill but there's no compaction marker and no `[OMNI_ACTIVE_WORKER]` and no project active.** Tell the user *"Nothing to reload — no active worker or project. If you meant to load a worker, try `Load worker <slug>` instead."* STOP.

**Claude Code's compaction is partial and some files survived.** Re-read anyway. Verifying every file is in context costs more than just re-reading.

## What this skill MUST NOT do

- Do NOT modify the worker spec file.
- Do NOT modify any other file. This skill is pure context warming.
- Do NOT chain to `push-changes` (read-only operation).
- Do NOT echo full file contents in the chat — counts only.
- Do NOT silently skip the user-visible confirmation in Step 4. The user needs to know the re-warm completed before continuing the task.
- Do NOT touch the global active-project pointer.
- Do NOT prompt the user with "are you sure?" — reload-worker is cheap and the cost of NOT reloading is methodology drift.
- Do NOT invoke `skill-creator` to "create" this skill or any other Omnipresence skill if a skill appears missing. The Omnipresence skill set is canonical, vendored from `omnipresence-os/claude-skills`, and installed via `getting-started` Step 8.5 / refreshed via `sync-omnipresence`. Missing skills mean a broken install — surface that and route to `getting-started`, never hand-write.

## Why this exists

Claude Code's compaction summarizes the conversation but does not preserve the contents of files Read into context. The `PreCompact` hook can only block compaction, not modify the summarization prompt. The `PostCompact` hook is purely advisory and cannot re-inject content.

So the only honest way to survive compaction for a worker that depends on a methodology corpus is: **re-read the files on detection**. This skill is that re-read, made invokable both manually (user types "pick up where we left off") and silently (Claude detects compaction + worker marker, fires this skill before answering).

Without this skill, a worker post-compaction has the *name* of its role but not the *content* of its methodology. It will then freestyle — apply training-data approximations of Omni methodology rather than the actual methodology — and produce output that drifts from the framework. The cost of one round-trip re-read is ~10 seconds; the cost of freestyling is days of bad output.

## Skill family

- [`create-worker`](../create-worker/SKILL.md) — wizard for authoring a worker spec
- [`load-worker`](../load-worker/SKILL.md) — initial session warmup from a spec
- [`reload-worker`](../reload-worker/SKILL.md) — this skill; post-compaction re-warmup from the same spec

create-worker writes once. load-worker warms once. reload-worker warms again and again over the conversation's lifetime, whenever compaction has eaten the methodology corpus.
