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
- **Worker not found AND no close fuzzy match:** tell the user *"No worker named `<slug>` for project `<active-slug>`, and no core default exists for that slug either. Create one with `Create a worker for <role>`, or pick from: `<list of available across all tiers>`."* STOP.

**Four-tier resolution (first hit wins).** Walk the worker spec path in this order:

1. `<synapse-path>/custom/projects/<active-slug>/workers/<worker-slug>.md` — project-scoped (always wins)
2. `<synapse-path>/custom/workers/<worker-slug>.md` — member-global (across all projects)
3. `<synapse-path>/overrides/workers/<worker-slug>.md` — member's override of a core default
4. `<synapse-path>/core/workers/<worker-slug>.md` — shipped baseline

Capture which tier resolved into a `loaded_from` variable (`project | member-global | override | core-default`). Surface this in the Step 4 confirmation block so the user knows whether they're running a project-customized worker or the shipped baseline.

When listing "available workers" for the disambiguation / not-found error, list across ALL four tiers (project workers + member-global workers + core defaults), deduplicated by slug, with the resolving tier shown.

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

**If `loaded_from === "core-default"` OR `loaded_from === "override"` (tiers 3-4), substitute `<project-slug>` placeholders** in the spec's auto-load block with the active project slug BEFORE fanning out reads. Core/override specs use `<project-slug>` as a literal placeholder in path references (e.g., `custom/projects/<project-slug>/project-config.md`); the active project slug from Step 1 replaces every occurrence. Project-tier specs (tiers 1-2) have hardcoded slugs — no substitution needed.

In a SINGLE message, call the Read tool in parallel for every file listed in the spec's auto-load block. Resolve paths:

- **Methodologies / processes / skills:** relative to synapse fork root. Apply local-first resolution: `custom/` → `overrides/` → `core/` (Omni MCP resolution order).
- **Skills entries** that are directory paths (e.g., `core/skills/generation/outline-generator`) → read `<path>/SKILL.md`.
- **Project files:** relative to `<synapse-path>/custom/projects/<active-slug>/`.
- **Glob patterns** (e.g., `writing-samples/**`) → expand first, then read each match.

Silently absorb the content into context. If a file is missing, note it for Step 4 but DON'T fail — the worker might still be useful with degraded context.

**Detect TBD placeholders.** For every project file in the auto-load list (style-guide.md, glossary.md, banned-phrases.md, editor-rules.md, project-config.md), check whether the file contains a `TBD-PLACEHOLDER:` marker in its first 10 lines. Any file with that marker is a stub, not real content — collect the list of placeholder files for the warmup confirmation in Step 4.

### Step 4 — Confirm warmup

Tell the user, exactly:

```
✅ Loaded as <display_name> for project '<active-slug>'.

  • Source: <project | member-global | override | core-default>
  • Mission: <mission>
  • Loaded: <N> methodologies, <M> processes, <K> skills, <P> project files
  • Quirks: <one-line summary of the quirks section, or 'none'>
  • Autonomy: <Propose-then-execute | Autonomous (no plan-approval gate)>
  • Missing files: <list, if any — or 'none'>
  • TBD placeholders: <list of files with TBD-PLACEHOLDER markers, or 'none'>

Ready. What do you want me to do?
```

If `Source: core-default`, append after the block: *"This is the shipped baseline. To customize for this project (add brand-specific quirks, change the auto-load list, flip autonomy), say `Create a worker for <slug>` — it forks the core spec into `custom/projects/<active-slug>/workers/<slug>.md` where you can edit freely without losing the upstream version."*

If `TBD placeholders` is non-empty, append a one-line suggestion AFTER the "Ready" line: *"Heads up: `<file>` is still a TBD placeholder, so I'll be guessing on `<what that file would have specified>`. Want to fill it now (~2-3 min) before we start, or push through and risk drift?"* Use the `AskUserQuestion` tool with two options:

| Option | When to pick |
|---|---|
| Fill it now | Take 2-3 minutes to set up the slot — the rest of this task will use the filled content |
| Skip and continue | Run the task with defaults / project-config alone — I'll flag if I have to guess |

If "Fill it now," invoke `fill-project-gap` (Mode C — silent agent-fired interrupt) with the placeholder slot. Resume Step 5 when it returns. If "Skip and continue," proceed to Step 5 with a noted gap.

If the user's original prompt already included a task (e.g., *"load content-writer and draft a post on programmatic SEO"*), capture it but DO NOT skip the placeholder check — the placeholder warning fires before the captured task starts. After the placeholder question is answered (fill or skip), continue to Step 5 with the captured task.

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
- Do NOT invoke `skill-creator` to "create" this skill or any other Omnipresence skill if a skill appears missing. The Omnipresence skill set is canonical, vendored from `omnipresence-os/claude-skills`, and installed via `getting-started` Step 8.5 / refreshed via `sync-omnipresence`. Missing skills mean a broken install — surface that and route to `getting-started`, never hand-write.

## Why this shape

Local-first resolution (`custom/` → `overrides/` → `core/`) mirrors the Omni MCP's tier order, so a member who overrode a core methodology in their fork gets their override loaded, not the upstream version. Fan-out reads in parallel are why warmup completes in ~10 seconds instead of ~60 — the auto-load list often has 20+ files.

The compact warmup confirmation (counts, not file paths) mirrors `connect-google-cloud`'s structured success block — proves the work was done without flooding chat with raw paths. The mission + autonomy + quirks summary is what lets the user verify the right worker was loaded before they kick off the actual task.

Pairs with [create-worker](../create-worker/SKILL.md) — that skill writes the spec, this skill reads and warms it. The two are symmetric inverses: create-worker is high-ceremony (interview + test run), load-worker is fast (read + confirm + run).
