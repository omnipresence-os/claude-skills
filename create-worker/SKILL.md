---
name: create-worker
description: Stands up a project-scoped Worker — a reusable specialist agent profile (content writer, link builder, technical SEO auditor, etc.) saved as a markdown spec under custom/projects/<active>/workers/. Walks the user through a preset-or-custom interview, captures the role's mission, auto-load list (methodologies / processes / skills / project files), project-specific quirks, propose-then-execute plan, optional automation hook, and a live test run. Trigger when the user says any of "create a worker", "create a new worker", "set up a worker", "make a content writer for X", "build me a link builder agent", "new worker for X", "add a specialist for X", "automate X for project Y", "I keep doing this manually — bake it in", "turn this into a repeatable agent", "create a routine for X", "set up an agent for X", "make a worker that does X". This is the ONE canonical worker setup flow — replaces the manual 12-prompt warmup sequence (start fresh agent → load project → load methodologies → load skills → set quirks → propose plan → run → guide → schedule → test) with a single guided wizard. Eight presets available (content-writer, content-refresher, link-builder, technical-seo-auditor, topical-map-builder, aeo-optimizer, programmatic-seo-operator, brand-voice-extractor) plus a custom path. Always runs a live test before claiming success. Optional automation hook (Hermes / OpenClaw cron) captures intent for follow-up wiring. Idempotent — safe to re-run to add another worker, refresh an existing one, or update its auto-load list. Chains to push-changes at the end.
---

# Create Worker — Project-Scoped Specialist Setup

Walks the user through standing up a reusable worker (content writer, link builder, technical SEO auditor, etc.) that future chats can load with one command instead of 12 manual prompts.

## How to talk to the user during this skill

Plain English. ONE question at a time. Wait for each answer before asking the next. Don't show shell commands. Don't paste raw frontmatter. Surface the auto-load list and the propose-then-execute plan as plain bullet lists, not as YAML.

After the worker spec file is written, **always do a live test run before claiming success** (unless the user explicitly skipped it). Don't claim it works without evidence — that's the rule connect-google-cloud follows for the same reason.

✅ Good: *"For a content writer I'd auto-load these methodologies: retrieval-readiness-writing, answer-engine-optimization, intro-pattern-selection… want me to add or remove anything?"*

❌ Bad: *"Writing `auto_load: [methodologies/execution/retrieval-readiness-writing.md, ...]` to YAML frontmatter…"*

Never show: raw frontmatter, file-write commands, MCP tool calls, or the full content of every auto-loaded file.

## Prerequisites

- An active project exists (resolved via [project-resolver](../project-resolver/SKILL.md)). If none, route the user to `new-project` or `switch-project` first and STOP.
- Synapse fork path cached at `~/.claude/skills/.omnipresence-path`.

## Execute these steps in order

### Step 0 — Check existing workers for this project

Silently check `<synapse-path>/custom/projects/<active-slug>/workers/`. If the folder doesn't exist, that's fine — create it in Step 9.

Branch on what exists:

- **No `workers/` folder OR empty folder:** new setup. Continue to Step 1.
- **One or more workers already defined:** ask, in a single AskUserQuestion (max 4 options): *"You already have workers set up for this project (`<slug-1>`, `<slug-2>`, …). What do you want to do?"*

  | Option | When to pick |
  |---|---|
  | Add a new worker | Standing up a fresh specialist alongside the existing ones |
  | Refresh an existing worker | Update an existing worker's auto-load list or process |
  | Test an existing worker | Just want a live test run, not a new spec |
  | Full re-setup of one | Throw away and rebuild from the wizard (warns before overwrite) |

  Branch: (Add) continue Step 1, (Refresh) jump to Step 7 with the chosen worker pre-loaded, (Test) jump to Step 10, (Full re-setup) continue Step 1 and warn it will overwrite.

### Step 1 — Resolve active project

Follow the [project-resolver](../project-resolver/SKILL.md) protocol: chat-session marker first, then global default.

- **Project resolves and folder exists:** use it. Continue.
- **No project AND zero projects exist:** tell the user *"You need a project before creating a worker. Run `Create a new project for X` first."* STOP.
- **No project AND multiple projects exist:** ask *"Which project is this worker for? Your options: `<list>`."* Once they pick, emit `[OMNI_SESSION_ACTIVE = <slug>]` so the rest of this chat uses that project.

**Surface the resolved project in the first output line:**

```
Creating worker for project=<slug> (resolved from <chat-session | global-default>).
```

### Step 2 — Pick preset or custom (AskUserQuestion, two panels)

There are 9 total options (8 presets + Custom). Split across two AskUserQuestion calls.

**Panel 1 — "What kind of worker?":**

| Option | When to pick |
|---|---|
| Content writer | Writes blog posts, pillar pages, landing copy from briefs or topics |
| Content refresher | Audits and rewrites existing pages to fix decay, AEO gaps, or rank slippage |
| Link builder | Discovers prospects, drafts outreach, triages replies, tracks placements |
| Technical SEO auditor | Crawls a site (Screaming Frog + GSC), surfaces issues, recommends fixes |

If the user picks one of these four, jump to Step 3 with `preset_type = <chosen>`.

If the user picks "Other / Show me more," ask Panel 2.

**Panel 2 — "More worker types:":**

| Option | When to pick |
|---|---|
| Topical map builder | Generates and validates topical maps for a new lane or niche |
| AEO / AIO optimizer | Targets answer-engine retrieval (ChatGPT, Perplexity, AIO panel) specifically |
| Programmatic SEO operator | Spins up large sets of templated pages (locations, comparisons, alternatives) |
| Brand voice extractor | Reads writing samples + brand pages to extract a style guide and writing-samples corpus |

If still none match, fall back to a Custom path: ask *"Describe the worker's role in one sentence — I'll build a custom auto-load list using the methodology MCP."* and set `mode = custom`. Otherwise set `mode = preset`.

### Step 3 — Capture worker display name + derive slug

From the user's original prompt, try to extract a display name (e.g., "content writer for jonathan-boshoff" → display: "Content Writer"). If absent, ask: *"What should I call this worker? (e.g., 'Pillar Page Writer', 'GSC Auditor')"*

Derive a kebab-case slug. Show: *"I'll save this as `<slug>.md`. Press Enter or paste an alternative."*

**Slug collision check:** if `<synapse-path>/custom/projects/<active-slug>/workers/<slug>.md` exists, tell the user *"A worker with that slug already exists. Pick a different slug, or say 'refresh' to update the existing one."* If `refresh`, jump to Step 7 with the existing file loaded.

### Step 4 — Capture mission (one sentence)

Ask: *"In one sentence, what should this worker do every time it's called?"*

Examples to nudge if they're stuck: "Write retrieval-ready blog posts from a topic brief." "Audit and refresh a single existing page." "Run weekly link-prospect outreach for our SaaS niche."

Store as `mission`.

### Step 5 — Build the auto-load list

**If `mode = preset`:** look up the hardcoded bundle for `preset_type` in the **Preset auto-load bundles** section below. Show the user the proposed list as plain bullets grouped by type:

```
For a <preset_type>, I'd auto-load these on every session:

Methodologies (read first — these are the "why"):
  • <file 1>
  • <file 2>
  • ...

Processes (the playbooks this worker runs):
  • <file 1>
  • <file 2>

Core skills (tools it invokes):
  • <file 1>
  • <file 2>

Project files (per-client context):
  • project-config.md
  • style-guide.md
  • writing-samples/ (all files)

Want me to add or remove anything?
```

If user says *"good"* / *"yes"* / *"looks right"*, proceed. If they ask to add/remove, edit the list inline and re-confirm.

**If `mode = custom`:** use the MCP `lookup` tool (Omni MCP) with the mission as the query to surface methodology / process / skill candidates that match. Propose ~5–10 best matches as bullets, ask: *"Here's what I'd auto-load for a worker with that mission. Add anything? Remove anything?"* Iterate until confirmed.

**Anti-hallucination guard:** before writing the spec, verify every file path in the proposed list exists in the local fork. If any are missing, flag them inline: `<path> (missing — run sync-omnipresence)`. Don't silently drop them.

Store final list as `auto_load`.

### Step 6 — Capture project-specific quirks (the curve balls)

Ask, plain prose: *"Anything project-specific this worker should know? Examples: tone the brand avoids, banned phrases, account-specific quirks (which CMS, which Drive folder, which subagent owner), or surfaces it should default to. Type 'none' if there's nothing weird."*

Store as `quirks` (free-form prose). If user says `none`, store the literal string `None.`

### Step 7 — Propose the end-to-end process

Draft a numbered process (typically 5–8 steps) that this worker would run when invoked. Base it on the auto-loaded methodologies + processes + skills. Show it as plain bullets:

```
When you say "<example invocation>", this worker will:

  1. Read project config + style guide + writing samples (auto-loaded on session start).
  2. Read the brief / topic / URL you provide.
  3. Call <skill A> to <subtask>.
  4. Call <process B> to <subtask>.
  5. Review against <methodology C>.
  6. Hand back the artifact at <path>.

Does this match how you want it to work?
```

If user says "Yes" / "Looks right", proceed. If they want changes, ask what to change, redraft, re-confirm.

**Autonomy question (inline yes/no):** *"Should this worker propose the plan before executing each time, or just go autonomously? Most workers should propose — only set autonomous if you fully trust the plan."*

- *"Propose first"* → `autonomous: false` (default)
- *"Go autonomously"* → `autonomous: true`

Store final as `process_plan` + `autonomous`.

### Step 8 — Optional automation hook

Single AskUserQuestion:

| Option | When to pick |
|---|---|
| No automation | Worker is called on-demand only |
| Hermes cron | Schedule it on Hermes (member's home server) |
| OpenClaw cron | Schedule it on the OpenClaw container |
| Decide later | Save the worker, set up cron in a follow-up session |

If Hermes or OpenClaw, ask inline: *"What cadence? (e.g., 'every Monday 9am UTC', 'daily 6am', 'every 4 hours')"* Store as `automation.platform` + `automation.cadence` + `automation.next_step` (`configure-now` or `save-for-later`).

**Do NOT configure the cron yet** — Step 11 handles it. This step only captures intent.

### Step 9 — Write the worker spec file

Write `<synapse-path>/custom/projects/<active-slug>/workers/<worker-slug>.md` using the **Worker spec schema** below. Create the `workers/` folder if it doesn't exist. Do not show the user the raw file. Tell them: *"Worker spec saved at custom/projects/`<active>`/workers/`<worker-slug>`.md."*

Also append to `<synapse-path>/custom/projects/<active-slug>/README.md` under a `## Workers` section (create the section if it doesn't exist):

```markdown
## Workers

- `<worker-slug>` — <mission> (load with `Load worker <worker-slug>`)
```

### Step 10 — Live test run

Ask inline: *"Want me to do a live test run now to confirm it works end-to-end?"*

- **Yes:** invoke the worker as a member would (via `load-worker <slug>` + a small representative task — e.g., for content-writer: "Write a 300-word draft on `<topic from project>`"; for technical-seo-auditor: "Audit `<homepage URL>`"). Walk through the process_plan. Surface the output. Confirm: *"That's the worker working. Looks right? (Yes / Iterate)"* — if Iterate, ask what to fix, edit the worker spec, re-test.
- **Skip:** continue.

Record the result in the worker spec's `Last test run` field before reporting success.

### Step 11 — Configure automation (only if Step 8 picked configure-now)

If `automation.next_step == configure-now`:

- **Hermes:** no `setup-hermes-cron` skill exists yet. For now, surface the manual instruction: *"To wire this into Hermes cron, SSH in and add this line to your crontab: `<cadence-as-cron> /path/to/hermes-run-worker.sh <worker-slug>`. Want me to walk you through?"* Save the cron expression in the worker spec's `automation.cron` field.
- **OpenClaw:** no `setup-openclaw-cron` skill exists yet. For now, surface the manual instruction: *"To schedule this on OpenClaw, add a cron entry to `openclaw.json` pointing at `<endpoint>`. Want me to walk you through?"* Save the cron expression in the worker spec's `automation.cron` field.

Tell the user: *"Cron config isn't auto-wired yet — for now, the intent is saved in the worker spec. When the setup-cron skill ships, it'll read this and configure it."*

### Step 12 — Report success

Tell the user, exactly:

```
✅ Worker '<display name>' created for project '<active-slug>'.

  Spec file: custom/projects/<active-slug>/workers/<worker-slug>.md

Confirmed:
  • Auto-load list: <N> methodologies, <M> processes, <K> skills, <P> project files
  • Test run: <Passed | Skipped>
  • Automation: <None | Hermes <cadence> | OpenClaw <cadence> | Saved for later>

From now on:

  • "Load worker <worker-slug>"          → warms up a session as this worker
  • "<worker-slug> please <task>"        → shortcut invocation

Next prompts:

  • Run the worker:           Load worker <worker-slug> and <task>.
  • Save to GitHub:           Push my synapse changes.
  • Set up another worker:    Create a worker for <other role> in this project.
```

### Stop here. Do NOT propose other steps.

---

## Preset auto-load bundles

These eight presets map onto the dominant Omni workflows. Every path below has been verified to exist in the synapse fork. All paths are relative to the synapse fork root.

### content-writer

**Mission:** "Writes retrieval-ready blog posts, pillar pages, and landing copy from a brief or topic, applying the project's voice and AEO discipline."

- **Methodologies:** `core/methodologies/execution/retrieval-readiness-writing.md`, `core/methodologies/execution/answer-engine-optimization.md`, `core/methodologies/execution/intro-pattern-selection.md`, `core/methodologies/execution/information-gain.md`, `core/methodologies/execution/citation-discipline.md`, `core/methodologies/execution/editorial-review-framework.md`, `core/methodologies/execution/section-first-content-production.md`, `core/methodologies/execution/eeat-content-sourcing.md`
- **Processes:** `core/processes/execution/content-generation.md`, `core/processes/execution/content-editing.md`
- **Core skills:** `core/skills/generation/outline-generator`, `core/skills/generation/content-section-writer`, `core/skills/generation/title-and-intro-writer`, `core/skills/generation/faq-cta-writer`, `core/skills/generation/internal-link-suggester`, `core/skills/generation/image-prompt-writer`
- **Project files:** `project-config.md`, `style-guide.md`, `writing-samples/**`, `glossary.md` (optional), `banned-phrases.md` (optional), `editor-rules.md` (optional)

### content-refresher

**Mission:** "Audits an existing page, identifies decay / AEO gaps / rank slippage causes, and rewrites the affected sections in the project's voice."

- **Methodologies:** `core/methodologies/strategy/content-refresh.md`, `core/methodologies/execution/retrieval-readiness-writing.md`, `core/methodologies/execution/answer-engine-optimization.md`, `core/methodologies/research/ai-retrieval-mechanics.md`
- **Processes:** `core/processes/execution/content-refresh.md`, `core/processes/analysis/retrieval-readiness-audit.md`
- **Core skills:** `core/skills/analysis/refresh-candidate-validator`, `core/skills/generation/refresh-section-writer`, `core/skills/generation/refresh-section-editor`, `core/skills/generation/refresh-report-generator`, `core/skills/data-access/gsc-api-reference`, `core/skills/data-access/page-scraper`
- **Project files:** `project-config.md`, `style-guide.md`, `writing-samples/**`

### link-builder

**Mission:** "Discovers link prospects, drafts on-brand outreach, classifies replies, and tracks placements end-to-end."

- **Methodologies:** `core/methodologies/execution/link-building-criteria.md`, `core/methodologies/execution/link-outreach-reply-handling.md`, `core/methodologies/execution/cold-email-deliverability.md`, `core/methodologies/execution/guest-post-content-strategy.md`, `core/methodologies/strategy/link-outreach-prospecting.md`, `core/methodologies/strategy/citation-shortlist-targeting.md`
- **Processes:** `core/processes/research/prospect-discovery.md`, `core/processes/research/citation-shortlist-discovery.md`, `core/processes/execution/outreach-campaign-run.md`, `core/processes/execution/reply-triage.md`, `core/processes/execution/placement-tracker.md`
- **Core skills:** `core/skills/analysis/link-vetter`, `core/skills/analysis/link-investment-estimator`, `core/skills/analysis/email-reply-classifier`, `core/skills/generation/outreach-message-writer`, `core/skills/data-access/contact-enricher`, `core/skills/data-access/smartlead-api`, `core/skills/data-access/prospect-discoverer-niche-relevant`, `core/skills/data-access/prospect-discoverer-broken-link`
- **Project files:** `project-config.md`, `style-guide.md`

### technical-seo-auditor

**Mission:** "Crawls a site (Screaming Frog + GSC), surfaces tech-SEO issues across architecture, schema, and metadata, and recommends prioritized fixes."

- **Methodologies:** `core/methodologies/architecture/site-architecture.md`, `core/methodologies/architecture/url-structure.md`, `core/methodologies/architecture/link-equity-management.md`, `core/methodologies/execution/schema-strategy.md`, `core/methodologies/execution/title-tag-meta-description.md`
- **Processes:** `core/processes/client-engagement/site-architecture-planning.md`
- **Core skills:** `core/skills/analysis/technical-seo-page-audit`, `core/skills/analysis/screaming-frog-crawl-compare`, `core/skills/analysis/screaming-frog-issue-extract`, `core/skills/analysis/screaming-frog-url-inspect`, `core/skills/analysis/sitemap-diff`, `core/skills/analysis/cross-link-auditor`, `core/skills/data-access/screaming-frog-crawl`, `core/skills/data-access/sitemap-discoverer`, `core/skills/data-access/gsc-api-reference`
- **Project files:** `project-config.md`

### topical-map-builder

**Mission:** "Generates and validates topical maps for a new lane or niche — user stories → queries → clusters → backlog."

- **Methodologies:** `core/methodologies/strategy/content-planning.md`, `core/methodologies/strategy/content-lane-discipline.md`, `core/methodologies/strategy/backlog-topic-generation.md`, `core/methodologies/strategy/target-market-landscape.md`, `core/methodologies/research/user-stories.md`, `core/methodologies/research/stories-to-queries.md`
- **Processes:** `core/processes/research/content-planning.md`, `core/processes/research/user-stories.md`, `core/processes/research/stories-to-queries.md`, `core/processes/analysis/lane-portfolio-audit.md`
- **Core skills:** `core/skills/generation/backlog-topic-generator`, `core/skills/generation/topic-idea-generator`, `core/skills/analysis/semantic-keyword-cluster`, `core/skills/analysis/query-deserves-page`, `core/skills/analysis/ai-visibility-topic-validator`, `core/skills/analysis/topic-final-validator`, `core/skills/data-access/knowledge-graph-lookup`, `core/skills/data-access/volume-checker`
- **Project files:** `project-config.md`, `style-guide.md`

### aeo-optimizer

**Mission:** "Targets answer-engine retrieval specifically (ChatGPT / Perplexity / AIO panels) — audits chunk-readiness, citation hooks, and verbalization patterns."

- **Methodologies:** `core/methodologies/execution/answer-engine-optimization.md`, `core/methodologies/research/ai-retrieval-mechanics.md`, `core/methodologies/research/ai-search-citation-mechanics.md`, `core/methodologies/research/answer-format-taxonomy.md`, `core/methodologies/execution/retrieval-readiness-writing.md`, `core/methodologies/execution/citation-discipline.md`
- **Processes:** `core/processes/analysis/aio-readiness-audit.md`, `core/processes/analysis/retrieval-readiness-audit.md`
- **Core skills:** `core/skills/analysis/aeo-page-audit`, `core/skills/analysis/aio-lift-analyzer`, `core/skills/analysis/ai-citation-cross-reference`, `core/skills/data-access/chunk-frequency-scorer`, `core/skills/data-access/chunkscore-relevance-scorer`, `core/skills/data-access/aio-panel-lookup`, `core/skills/generation/verbalization-generator`
- **Project files:** `project-config.md`, `style-guide.md`

### programmatic-seo-operator

**Mission:** "Spins up large sets of templated pages (locations, comparisons, alternatives) end-to-end — template → data → page → publish."

- **Methodologies:** `core/methodologies/execution/local-landing-page-anatomy.md`, `core/methodologies/execution/service-page-positioning.md`, `core/methodologies/execution/ecommerce-category-pages.md`, `core/methodologies/architecture/geographic-page-hierarchy.md`, `core/methodologies/architecture/url-structure.md`
- **Processes:** `core/processes/execution/wordpress-landing-page-create.md`
- **Core skills:** `core/skills/generation/single-location-page-writer`, `core/skills/generation/category-subfolder-suggester`, `core/skills/generation/wordpress-page-publish`, `core/skills/generation/wordpress-internal-link-injector`, `core/skills/data-access/wordpress-connection`
- **Project files:** `project-config.md`, `style-guide.md`

### brand-voice-extractor

**Mission:** "Reads writing samples + brand pages to extract / update a project's style guide, voice profile, and writing-samples corpus."

- **Methodologies:** `core/methodologies/strategy/brand-baseline-assessment.md`, `core/methodologies/strategy/project-config-foundation.md`
- **Processes:** `core/processes/client-engagement/brand-baseline-assessment.md`, `core/processes/client-engagement/profile-creation.md`
- **Core skills:** `core/skills/research/client-brand-research`, `core/skills/research/client-brand-update`, `core/skills/generation/style-guide-writer`, `core/skills/generation/style-guide-combiner`, `core/skills/generation/writing-samples-extractor`, `core/skills/generation/brand-info-assembler`, `core/skills/data-access/page-scraper`
- **Project files:** `project-config.md`, `style-guide.md`, `writing-samples/**`

---

## Worker spec schema

This is the exact markdown shape the wizard writes to `custom/projects/<active-slug>/workers/<worker-slug>.md`:

```markdown
---
worker_slug: <kebab-case-slug>
worker_type: <preset-name | custom>
project: <active-project-slug>
display_name: <Human Readable Name>
version: 1
created: <YYYY-MM-DD>
last_updated: <YYYY-MM-DD>
autonomous: <true | false>
mission: <one polished sentence from Step 4>
---

# <Display Name>

> <mission> — <preset_type or "custom"> worker for project `<project-slug>`.

## Auto-load on session start

When `load-worker <worker-slug>` is invoked, read every file in this block IN PARALLEL before responding to the user's actual ask. All paths are relative to the synapse fork root (apply local-first resolution: project → custom → overrides → core).

### Methodologies (read first — these are the "why")

- `core/methodologies/<category>/<file>.md`
- `core/methodologies/<category>/<file>.md`
- ...

### Processes (the playbooks this worker runs)

- `core/processes/<category>/<file>.md`
- `core/processes/<category>/<file>.md`
- ...

### Core skills (the tools this worker invokes)

- `core/skills/<category>/<skill-name>` (resolves to `SKILL.md` inside)
- `core/skills/<category>/<skill-name>`
- ...

### Project files (per-client context)

- `custom/projects/<project-slug>/project-config.md`
- `custom/projects/<project-slug>/style-guide.md`
- `custom/projects/<project-slug>/writing-samples/**` (glob — expand at load time)
- `custom/projects/<project-slug>/glossary.md` (optional — skip silently if missing)
- `custom/projects/<project-slug>/banned-phrases.md` (optional)
- `custom/projects/<project-slug>/editor-rules.md` (optional)

## Project-specific quirks

<Free-form prose from Step 6. Examples members might paste here:>

- Tone curve balls: we never use "leverage" or em-dashes.
- Surface preferences: drafts go to Google Drive folder `/<folder-id>/Drafts/`, not WordPress directly.
- Account quirks: WordPress login uses the `<role>` account, not admin (post-as for byline).
- Known gotchas: this client's Screaming Frog license is shared — schedule crawls outside 9-5 GMT.

(If user said "none" in Step 6, this section reads: `None.`)

## End-to-end process

When invoked with a task, follow this propose-then-execute sequence:

1. **Confirm session is warmed up.** Verify the auto-load list above is actually in your context — not just that you remember being warmed up. Pick 2–3 specific files from the auto-load block (e.g., "What does Belief 13 of `retrieval-readiness-writing.md` say?") and check whether you can quote them. If you cannot, STOP and ask the user *"Context may have compacted — I need to re-warm. Should I run `reload-worker <slug>` first?"* Compaction silently drops file contents while preserving summaries; do not trust post-compaction memory of methodology.
2. **Check for TBD placeholders.** Look at every project file in your auto-load list. If any contains `TBD-PLACEHOLDER:` in its first 10 lines, that file is a stub — the slot has not been filled. STOP before starting the task and surface it: *"Heads up: `<file>` is still a TBD placeholder, so I'll be guessing on `<what the slot represents>`. Want to fill it now (~2-3 min) using `fill-project-gap`, or push through and risk drift?"* If the user says fill, invoke `fill-project-gap` in Mode C (silent agent-fired interrupt), let it complete, then resume Step 3. If push through, note the gap and continue but flag every output where the missing slot would have changed the result.
3. **Read the source, don't freestyle.** When in doubt about anything Omni-related during this task, search and read the actual methodology / process / skill file — don't infer the framework from memory. The auto-loaded files are the source of truth. Call `lookup` (MCP) or `Read` / `Grep` (local) to verify before applying a methodology claim.
4. Read the task brief from the user. If missing required inputs (target URL, topic, brief Doc link), ask before proposing.
5. **Propose the exact plan** (which processes, in which order, which skills called when) and wait for user confirmation. Skip this ONLY if `autonomous: true`.
6. Execute the plan. Show progress checkpoints, not every tool call.
7. Hand back the artifact at `<canonical-output-path>` with a one-paragraph summary.
8. Offer follow-ups: "Iterate on this output?" / "Run again with different inputs?" / "Save to GitHub?"

(The wizard pre-fills steps 4-6 with worker-type-specific detail. For content-writer: "Call outline-generator → get user approval on outline → call content-section-writer per section → call editor-section-reviser → assemble final.")

## Automation

```yaml
platform: <none | hermes | openclaw>
cadence: <human-readable cadence, e.g., "every Monday 9am UTC" — or "n/a">
cron: <cron expression — derived from cadence, or "n/a">
status: <not-configured | configured | deferred>
inputs_for_each_run: <how the cron knows what to pass — e.g., "next item from custom/projects/<slug>/content-backlog.md", or "n/a — manual only">
output_destination: <where each run's output lands — chat, Drive folder, etc.>
```

(If `platform: none`, this section reads `Manual invocation only.`)

## Test run procedure

Representative test invocation:

```
Load worker <worker-slug> and <small representative task with concrete inputs>.
```

Expected behavior:

- Auto-load completes within ~10 seconds, surfaces confirmation block.
- Proposes a plan matching the End-to-end process above.
- Produces an artifact at the canonical output path.
- Output passes the worker's quality bar (e.g., for content-writer: matches style-guide voice, includes AEO patterns, no banned phrases).

Last test run: `<YYYY-MM-DD HH:MM>` — `<Passed | Failed: reason | Skipped>`

## Change log

<!-- create-worker appends here when refreshed. Format: YYYY-MM-DD: <what changed>. -->

- `<YYYY-MM-DD>`: Worker created (preset: `<type>`).
```

---

## What this skill MUST NOT do

- Do NOT skip the live test run unless the user explicitly skipped Step 10.
- Do NOT write the worker spec file before Steps 1–7 are answered.
- Do NOT auto-decide preset vs custom (Step 2 is the user's call).
- Do NOT modify any file outside `custom/projects/<active-slug>/workers/` and the project's `README.md`.
- Do NOT touch the global active-project pointer (`~/.claude/skills/.omnipresence-active-project`). Use chat-session marker only.
- Do NOT configure cron without user opt-in (Step 8 must say `configure-now`).
- Do NOT echo raw frontmatter or file-write commands to the user.
- Do NOT invent methodology / process / skill file paths — only reference files that exist in the synapse fork's `core/` tree. Use the MCP `lookup` tool to verify if unsure.
- Do NOT chain to `push-changes` automatically — the user opts in via the success-message next-prompts.

## Why this shape

Mirrors `new-strategy` for the wizard shape (project-scoped, interview-driven, appends to README, chains to push-changes), `connect-google-cloud` for the live-test discipline ("don't claim success without evidence"), and `new-project` for the project-folder scaffold conventions.

Workers are a peer of strategies inside a project — strategies are multi-week PLANS, workers are reusable SPECIALISTS invoked on demand. Different lifecycle, different file shape, different invocation pattern.

Path-based file references in the worker spec (not embedded copies) mirror the filesystem-first override discipline already in Omni: when upstream methodology updates and `sync-omnipresence` pulls it, every worker pointing at that path benefits automatically.
