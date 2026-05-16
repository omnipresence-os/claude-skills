---
name: new-strategy
description: Creates a new strategy document under the active project's strategies/ folder. A strategy is a multi-day/multi-week implementation plan — phased, checkable, resumable. Trigger when the user says any of "create a new strategy", "plan an initiative", "build me a strategy for X", "I need a strategy for X", "create a roadmap", "plan a multi-week project for X". Asks 5-7 scoping questions (context, success criteria, time horizon, phases, per-phase steps, autonomy mode, notes), drafts the strategy following the convention shape, saves to custom/projects/<active>/strategies/<slug>.md. Implements the strategy-creation process from synapse.
---

# New Strategy — Scoping Interview + Draft

Walks the user through a 5-7 question interview, then drafts and saves the strategy doc.

## How to talk to the user

Plain English. ONE question at a time. Wait for each answer before asking the next. Don't show shell commands.

## Execute these steps

### Step 1: Resolve active project

Read `~/.claude/skills/.omnipresence-active-project`.

- **If missing** AND `<synapse-path>/custom/projects/` has exactly one folder: auto-pick it.
- **If missing** AND zero projects: tell the user: *"You need a project before creating a strategy. Run `Create a new project for X` first."* STOP.
- **If missing** AND multiple projects: ask *"Which project is this strategy for? Your options: <list>."* — wait for response, save selection as active.
- **If present:** verify the folder exists. If not, clear the file and treat as zero-project case.

### Step 2: Extract strategy name + derive slug

From the user's prompt, extract the strategy name. If not given, ask: *"What should I call this strategy? (e.g., 'Q3 Content Pivot', 'New Niche Launch')"*

Derive a kebab-case slug. Show: *"I'll use `<slug>` as the file name. Press Enter or paste an alternative."*

### Step 3: Check for collision

Check `<synapse-path>/custom/projects/<active-slug>/strategies/<strategy-slug>.md`.

- **Exists:** tell the user: *"A strategy with that name already exists. Use `Continue strategy <slug>` to resume, or pick a different name."* STOP.
- **Doesn't exist:** proceed.

### Step 4: Scoping interview — ask one question at a time

Walk through these 7 questions in order. Wait for an answer before asking the next.

**Q1 — Context:** *"What's happening that's prompting this strategy? What's changed for the project recently?"*

**Q2 — Success criteria:** *"What does success look like? Be specific — measurable where possible. (e.g., 'rank top 5 for our 10 money keywords by end of Q3' is better than 'rank better')."*

**Q3 — Time horizon:** *"Roughly how long do you expect this to take? Weeks? Months?"*

**Q4 — Phases:** *"Walk me through the rough phases. What's week 1? Week 2-3? Week 4+? Just the headline of each phase, one per line."*

**Q5 — Per-phase steps:** for each phase the user listed, ask: *"What are the 2-5 concrete steps in phase '<phase name>'?"*

**Q6 — Autonomy:** *"When I continue this strategy in future sessions, do you want me to execute steps autonomously (faster but I just go) or pause for confirmation each step (slower, you approve each move)?"* Options: *"Autonomous"* / *"Step-by-step"*.

**Q7 — Anything else:** *"Anything else I should remember about this strategy? Anything that didn't fit above?"*

**Sanity check before drafting:** if the user's scope sounds like a single task (one or two steps total, completable in under a day), gently push back: *"That sounds like a single task, not a multi-week strategy. Want me to just do it now instead?"* — if they confirm it's still a strategy, proceed.

### Step 5: Draft the strategy file

Write `<synapse-path>/custom/projects/<active-slug>/strategies/<strategy-slug>.md` with this structure:

```markdown
---
id: <strategy-slug>
type: strategy
project: <active-slug>
created: <YYYY-MM-DD>
status: in-progress
autonomous: <true | false>
target: <one-sentence from Q2, polished>
---

# <Strategy Display Name>

## Context

<Q1 answer, polished into a paragraph>

## Success criteria

<Q2 answer, expanded into a bulleted list when possible>

- Criterion 1
- Criterion 2
- Criterion 3

## Phases

### Phase 1: <Phase 1 name from Q4> (week 1)

- [ ] <Step 1 from Q5 for Phase 1>
- [ ] <Step 2>
- [ ] <Step 3>

### Phase 2: <Phase 2 name> (week 2-3)

- [ ] <Step 1 from Q5 for Phase 2>
- [ ] <Step 2>

### Phase 3: <Phase 3 name> (week 4+)

- [ ] <Step 1 from Q5 for Phase 3>
- [ ] <Step 2>

## Decisions log

<!-- continue-strategy appends here when a decision is made during execution. Format: YYYY-MM-DD: <decision>. -->

## Notes

<Q7 answer, free-form>
```

Light polish: clean up the user's raw text into prose, but preserve their phrasing where it captures intent precisely. Don't over-rewrite.

### Step 6: Update the project README

Read `<synapse-path>/custom/projects/<active-slug>/README.md`. Find the line `**Active strategies:** N` and update the count. Optionally append a `## Strategies` section listing the strategy. Save.

### Step 7: Report success

Tell the user, exactly:

```
✅ Strategy '<strategy display name>' created.

  File: custom/projects/<active-slug>/strategies/<strategy-slug>.md

Phases captured:
  • Phase 1: <name> — <N> steps
  • Phase 2: <name> — <N> steps
  • Phase 3: <name> — <N> steps

Total: <total> steps across <P> phases.
Mode: <Autonomous | Step-by-step with your confirmation>

Next prompts:

  • Start work:        Continue strategy <strategy-slug>.
  • Review the plan:   Show me strategy <strategy-slug>.
  • Save to GitHub:    Push my synapse changes.
```

### Stop here.

## What this skill MUST NOT do

- Do NOT skip questions. The 7-question interview is the strategy's foundation.
- Do NOT batch multiple questions into one ask.
- Do NOT auto-decide autonomous vs step-by-step (Q6 is the user's call).
- Do NOT save the file before all 7 questions are answered.
- Do NOT modify any file outside the active project's strategies/ folder + the project README.
