---
name: load-worker
description: Warms up the current chat session as a project-scoped worker by reading the worker spec file from custom/projects/<active>/workers/<slug>.md and loading every methodology / process / skill / project file listed in its auto-load block before doing any task work. Trigger when the user says any of "load worker X", "load the X worker", "be the X worker", "act as the content writer", "warm up worker X", "I want to use the X worker", "as the X worker, do Y", "X worker please do Y", "use worker X for this". This is the ONE canonical way to invoke a saved worker — replaces the manual "read project info, read methodology A, read methodology B, read skill C..." prompt sequence with a single command. Reads the spec, fans out reads in parallel, surfaces a compact "loaded as <worker> for <project>: ready" confirmation before doing anything else, then routes the user's actual ask through the worker's propose-then-execute process plan. Idempotent — calling it twice in the same chat just re-confirms the active worker. Pairs with create-worker (which writes the spec).
---

# Load Worker — Session Warmup as a Project Specialist

Reads a saved worker spec from the active project, fan-out-loads every file in its auto-load block in parallel, surfaces a compact warmup confirmation, then runs the user's ask through the worker's propose-then-execute process.

## How to talk to the user during this skill

Plain English. Fast. Do NOT enumerate every file you're reading — surface a compact summary (e.g. *"Loaded: 8 methodologies, 2 processes, 6 skills, project config + style guide + 4 writing samples"*). Don't show shell commands. Don't paste raw spec frontmatter.

## Execute these steps in order

### Step 1 — Resolve active project + target worker

Follow [project-resolver](../project-resolver/SKILL.md). If no project, route the user to `switch-project` / `new-project` and STOP.

Extract worker slug from the user's prompt:

- *"Load worker content-writer"* → slug = `content-writer`
- *"Be the link builder"* → slug = `link-builder` (apply fuzzy match)
- *"<slug> please <task>"* → slug = `<slug>`, capture the task for Step 5

**Disambiguation:**

- **Slug matches exactly one worker file:** use it.
- **Slug ambiguous (fuzzy match returns multiple):** list them: *"Which worker? Your options: `<bullet list of slugs + missions>`."*
- **Only one worker exists in the project AND no slug given:** auto-pick it.
- **Worker not found AND no close fuzzy match:** tell the user *"No worker named `<slug>` for project `<active-slug>`. Create one with `Create a worker for <role>`, or pick from: `<list>`."* STOP.

Verify `<synapse-path>/custom/projects/<active-slug>/workers/<worker-slug>.md` exists before continuing.

### Step 2 — Read the worker spec

Read the spec file. Parse frontmatter and body sections. Extract:

- `mission` (frontmatter)
- `autonomous` (frontmatter — defaults to `false` if missing)
- `display_name` (frontmatter)
- `Auto-load on session start` block (methodologies / processes / skills / project files)
- `Project-specific quirks` (free-form prose)
- `End-to-end process` (the propose-then-execute plan)
- `Automation` (yaml block — optional)

### Step 3 — Fan-out read every file in auto_load

In a SINGLE message, call the Read tool in parallel for every file listed in the spec's auto-load block. Resolve paths:

- **Methodologies / processes / skills:** relative to synapse fork root. Apply local-first resolution: `custom/` → `overrides/` → `core/` (Omni MCP resolution order).
- **Skills entries** that are directory paths (e.g., `core/skills/generation/outline-generator`) → read `<path>/SKILL.md`.
- **Project files:** relative to `<synapse-path>/custom/projects/<active-slug>/`.
- **Glob patterns** (e.g., `writing-samples/**`) → expand first, then read each match.

Silently absorb the content into context. If a file is missing, note it for Step 4 but DON'T fail — the worker might still be useful with degraded context.

### Step 4 — Confirm warmup

Tell the user, exactly:

```
✅ Loaded as <display_name> for project '<active-slug>'.

  • Mission: <mission>
  • Loaded: <N> methodologies, <M> processes, <K> skills, <P> project files
  • Quirks: <one-line summary of the quirks section, or 'none'>
  • Autonomy: <Propose-then-execute | Autonomous (no plan-approval gate)>
  • Missing files: <list, if any — or 'none'>

Ready. What do you want me to do?
```

If the user's original prompt already included a task (e.g., *"load content-writer and draft a post on programmatic SEO"*), skip the trailing question and continue to Step 5 with the captured task.

If `Missing files` is non-empty, append: *"If those files matter, run `Sync omnipresence` to pull the latest synapse upstream."*

Optionally emit `[OMNI_ACTIVE_WORKER = <worker-slug>]` as a chat-session marker so follow-up prompts in the same chat can infer which worker is active without re-loading.

### Step 5 — Execute the worker's propose-then-execute process

Apply the spec's `End-to-end process` to the user's task.

- **If `autonomous: false` (default):** follow the propose-then-execute pattern — draft the plan, surface it to the user as plain bullets, wait for "Yes" / "Edit" before executing.
- **If `autonomous: true`:** execute directly, but still surface progress checkpoints at each major step.

Respect the worker's `Project-specific quirks` throughout. If a quirk says "drafts go to Drive folder X, not WordPress," route the output there.

### Stop here.

## Edge cases

**The user gave a fuzzy slug that matches one worker.** Confirm inline before loading: *"Loading `<matched-slug>` — that's the closest match to `<user-said>`. Right one?"* If they say no, list options.

**The worker spec references files that don't exist locally.** Don't fail. Load what's available, surface the missing list in Step 4, suggest `sync-omnipresence`.

**The user said "load worker X" but didn't give a task.** Step 4 already asks *"Ready. What do you want me to do?"* — wait for the next prompt.

**The user invoked the worker via shortcut (`<slug> please <task>`).** Step 1 captures the slug, Step 5 receives the task. No extra Q&A needed.

**The worker spec's `Auto-load` block is empty or malformed.** Surface to the user: *"The worker spec for `<slug>` has no auto-load block. Run `Create a worker` with refresh mode to rebuild it."* STOP.

**Two workers in the project share a fuzzy-matched name.** Always list them and ask — never guess.

## What this skill MUST NOT do

- Do NOT modify the worker spec file.
- Do NOT modify any other file unless the worker's `End-to-end process` explicitly does so.
- Do NOT echo the full content of every auto-loaded file (surface counts only).
- Do NOT silently skip missing auto-load files — report them in Step 4.
- Do NOT load a worker from a project other than the active one without an explicit slug match.
- Do NOT chain to `push-changes` (this skill is read-only context-warming).
- Do NOT touch the global active-project pointer.
- Do NOT re-load if the same worker is already loaded in this chat session — just re-confirm and run the new task.

## Why this shape

Local-first resolution (`custom/` → `overrides/` → `core/`) mirrors the Omni MCP's tier order, so a member who overrode a core methodology in their fork gets their override loaded, not the upstream version. Fan-out reads in parallel are why warmup completes in ~10 seconds instead of ~60 — the auto-load list often has 20+ files.

The compact warmup confirmation (counts, not file paths) mirrors `connect-google-cloud`'s structured success block — proves the work was done without flooding chat with raw paths. The mission + autonomy + quirks summary is what lets the user verify the right worker was loaded before they kick off the actual task.

Pairs with [create-worker](../create-worker/SKILL.md) — that skill writes the spec, this skill reads and warms it. The two are symmetric inverses: create-worker is high-ceremony (interview + test run), load-worker is fast (read + confirm + run).
