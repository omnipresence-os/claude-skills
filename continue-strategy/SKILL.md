---
name: continue-strategy
description: Resumes a strategy from where it was last left off. Reads the strategy markdown, parses checkboxes to find the first unchecked step, executes or assists with that step (autonomous or step-by-step per the strategy's frontmatter), checks off the box on completion, appends to Decisions log if relevant, reports the next pending step. When all steps done, updates status to done. Trigger when the user says any of "continue strategy X", "resume strategy X", "what's next on strategy X", "work on strategy X", "next step for strategy X". Designed to be called repeatedly across many sessions over weeks. Implements the strategy-execution process from synapse.
---

# Continue Strategy — Resume + Move One Step Forward

Reads a strategy doc, finds the next unchecked step, does it (or assists), checks it off, reports what's next.

## How to talk to the user

Plain English. Show progress markers. Don't show shell commands or raw markdown unless the user explicitly asks to see the file.

## Execute these steps

### Step 1: Resolve active project + target strategy

Read `~/.claude/skills/.omnipresence-active-project`. If missing, ask which project. Resolve `<active-slug>`.

From the user's prompt, extract the target strategy slug:
- "Continue strategy q3-content-pivot" → `q3-content-pivot`
- "Resume Q3 Content Pivot" → derive `q3-content-pivot`
- "Continue strategy" (no name) → if exactly one in-progress strategy for active project, auto-pick. Otherwise ask which one.

Verify `<synapse-path>/custom/projects/<active-slug>/strategies/<strategy-slug>.md` exists.
- **If not:** apply fuzzy matching to existing strategy files. If close match, ask. If no close, redirect to `list-strategies`.

### Step 2: Read the strategy file + check status

Parse frontmatter: `id`, `status`, `autonomous`, `project`, `target`.

- **If `status: done`:** tell the user: *"This strategy is already done. Want me to surface what got accomplished (`Show me strategy <slug>`), or create a follow-on strategy?"* STOP.
- **If `status: paused`:** ask: *"This strategy is paused. Resume it now? (Yes / No)"*. If Yes, update frontmatter to `status: in-progress` and continue. If No, STOP.
- **If `status: in-progress`:** proceed.

### Step 3: Find the first unchecked step

Parse the body. Walk in document order, looking for the first `- [ ] <text>` line. Capture:
- The step text
- The phase header (`### Phase N: <name>`) it sits under
- The line number (for updating later)

**If no unchecked items remain:** all steps complete.
- Update frontmatter `status: done`.
- Append to Notes section: `- Completed: <YYYY-MM-DD>`.
- Update the project's README.md strategies list to mark this one as done.
- Tell the user, exactly:

```
🎉 Strategy '<display name>' is COMPLETE.

All <N> steps done across <P> phases.

Final summary:
  <recap from Decisions log + checked steps in 2-3 sentences>

Status: done. File: custom/projects/<active-slug>/strategies/<strategy-slug>.md

Next prompts:

  • Save to GitHub:                  Push my synapse changes.
  • Review what got done:            Show me strategy <strategy-slug>.
  • Create a follow-on strategy:     Create a new strategy for <display project name>.
```

STOP.

### Step 4: Branch on autonomy mode

- **If `autonomous: true`:** proceed silently to Step 5.
- **If `autonomous: false`:** show the user the upcoming step:

```
Next step on '<strategy display name>':

  📍 <phase header>
  → <step text>

Should I do this now? (Yes / Edit / Skip)
```

Wait for response.

- **Yes:** proceed to Step 5.
- **Edit:** ask what to change. Rewrite the checkbox line in the file. Loop back to Step 3 to re-find the next unchecked.
- **Skip:** move to the next unchecked item in the file (preserve current as unchecked). Repeat Step 4 for the new step. If user skips ≥3 times in a row, suggest: *"Want to review the strategy file directly? Something might be off."*

### Step 5: Execute or assist the step

The step text is a natural-language instruction. Route it like any member prompt:

- "Run a SERP audit for our money keywords" → call the appropriate audit skill (use MCP lookup if needed to find it).
- "Generate a refresh report for /blog/foo" → call refresh-report-generator.
- "Write 3 outline drafts for new pillar content" → call outline-generator 3 times.
- "Email Jonathan about the timeline" → not an Omni capability. Tell the user: *"This step is a human task. Reply 'done' to check it off when you've done it."*

For steps that complete in this session (under ~5 min), execute, surface results, then continue.

For steps that span hours/days ("Wait for backlinks from outreach to land"), tell the user: *"This is a longer-running step. I'll leave it as in-progress and check on it next time. Want me to note specific completion criteria?"* — don't check the box yet.

### Step 6: Check off the box + log decisions

When the step completes successfully:

1. Change `- [ ]` to `- [x]` on the captured line number.
2. If the step produced a decision (e.g., "decided to target 'blue widgets' instead of 'widget reviews'"), append to the `## Decisions log` section: `- <YYYY-MM-DD>: <decision>`.
3. Save the file.

### Step 7: Update project README progress

Read `<synapse-path>/custom/projects/<active-slug>/README.md`. Find the strategy entry and update its progress display (e.g., `<slug> — 5/15 steps`). Save.

### Step 8: Check for phase completion

If checking this box completed all checkboxes under the current `### Phase N: <name>` header, surface to the user:

```
📈 Phase '<phase name>' complete. Moving to '<next phase name>'.
```

### Step 9: Report next pending step

Tell the user, exactly:

```
✅ Step done: <step text>

Strategy: <strategy display name>
Progress: <checked>/<total> steps (<percent>%) — <phases-done>/<total-phases> phases

Next pending step:
  📍 <next phase header>
  → <next step text>

Run `Continue strategy <slug>` again when you're ready for it. (Or `Push my synapse changes` to back up the progress to GitHub now.)
```

### Stop here.

## What this skill MUST NOT do

- Do NOT check off a box for a step that didn't actually complete. If execution failed, leave it unchecked and report the failure.
- Do NOT execute a step autonomously when `autonomous: false`. Always ask at Step 4.
- Do NOT skip the Decisions log when a real decision was made during execution.
- Do NOT modify frontmatter status until all checkboxes are checked (then set `done`).
- Do NOT force-execute multiple steps in one call. One step per invocation, max.
