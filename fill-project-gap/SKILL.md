---
name: fill-project-gap
description: Fills in a TBD placeholder file in the active project (style-guide.md, glossary.md, banned-phrases.md, editor-rules.md, project-config.md) by walking the user through a short, slot-specific interview, then writing the result over the placeholder content. Trigger when the user says any of "fill in my style guide", "fill in my glossary", "fill in my banned phrases", "fill in my editor rules", "set up my brand voice", "fix my brand voice", "update my style guide", "my voice is empty", "complete my project setup", "finish project setup". ALSO trigger as an INTERRUPT when a worker or agent detects a TBD placeholder mid-task and needs to pause + offer to fill it before continuing — the user complaint "that's not my brand voice" should fire detection and route here. Interrupts pause the calling task with a 2-option AskUserQuestion ("Fill now or skip?"), fill the gap if yes (~2-3 min interview), then resume the original task at the exact point it paused. Idempotent — re-running on an already-filled file refreshes it after asking the user to confirm overwrite. Pairs with new-project (scaffolds the placeholders) and load-worker (detects + flags them at session start).
---

# Fill Project Gap — Interrupt-Then-Resume Gap Filler

Walks the user through filling in a TBD placeholder file in the active project. Designed to fire either explicitly ("fill in my style guide") or as an interrupt when another agent detects a gap mid-task ("that's not my brand voice" → agent notices `style-guide.md` is a placeholder → fires this skill → user fills it → original task resumes).

## How to talk to the user during this skill

Plain English. ONE question at a time. Wait for each answer before asking the next. Don't show shell commands. Don't paste raw frontmatter. Tight — the user is here to fill a gap, not write a manifesto. 2-3 minutes per slot, max.

**The "blame the empty file" rule:** state facts about what's missing, never accusations about what the user didn't do. *"Your style-guide.md is a TBD placeholder — want to fill it before we keep going?"* is right. *"You haven't set up your brand voice yet"* is wrong. The user is helping us help them; framing matters.

## When this skill fires

Three entry modes — handle each cleanly:

**Mode A — Explicit user request.** User typed *"fill in my style guide"* or *"set up my glossary"*. Slot is named in the prompt. Skip to Step 1.

**Mode B — Indirect dissatisfaction.** User complains about output quality in a way that implicates a missing file. Examples:
- *"That's not my brand voice"* → check `style-guide.md`
- *"You used a banned word"* → check `banned-phrases.md`
- *"What do you call the product? Use the right name"* → check `glossary.md`
- *"This wouldn't pass our editor"* → check `editor-rules.md`

In Mode B, the calling agent already did the detection (grepped for `TBD-PLACEHOLDER:` in the file) and routed here. Confirm to the user: *"Your `<slot>.md` is still a TBD placeholder — that's why I'm guessing. Want to fill it now (~2-3 min) so I stop guessing?"* If yes, continue to Step 1 with the detected slot. If no, surface what we know about the slot from context and continue the original task with that.

**Mode C — Silent agent-fired interrupt.** A worker or agent at the start of a task notices a placeholder in its auto-load list (via `load-worker`'s placeholder detection in Step 4). Before the task runs, agent fires this skill with an AskUserQuestion: *"I noticed `style-guide.md` is still a TBD placeholder. Want to fill it now (~2-3 min) using context from this chat, or skip and continue with what we have?"*

| Option | When to pick |
|---|---|
| Fill now | Take 2-3 minutes to set up the slot — the rest of this task will use it |
| Skip | Continue without it — the agent will use defaults / project-config and flag any drift |

If skip, return to the calling task and let it know the slot is still empty.

## Execute these steps in order

### Step 1 — Identify the slot

Map the user's request (or the interrupt's payload) to a slot. Supported slots:

| Slot | File | What it captures |
|---|---|---|
| `style-guide` | `style-guide.md` | Voice, tone, rules, example paragraphs |
| `glossary` | `glossary.md` | Preferred terms, central entity, custom terminology |
| `banned-phrases` | `banned-phrases.md` | Never-use list (words + pattern bans) |
| `editor-rules` | `editor-rules.md` | Publish gates + auto-reject conditions |
| `project-config` | `project-config.md` | The 8-field structured config (chains to `project-config-generation`) |

If the request is ambiguous (*"fix my project setup"*), ask: *"Which one? Style guide, glossary, banned phrases, editor rules, or project config?"*

### Step 2 — Resolve the active project + verify the file

Follow [project-resolver](../project-resolver/SKILL.md). If no project, route to `new-project` and STOP.

Read `<synapse-path>/custom/projects/<active-slug>/<file>`:

- **File doesn't exist:** create it from scratch using the slot's interview (no overwrite check needed).
- **File exists with `TBD-PLACEHOLDER:` marker:** placeholder confirmed. Walk the interview.
- **File exists WITHOUT `TBD-PLACEHOLDER:` marker:** it's already filled. Ask: *"Your `<file>` already has content. Refresh it (overwrites current content) or just show me what's there?"* If show, surface the current content and STOP. If refresh, continue with the interview.

### Step 3 — Walk the slot-specific interview

Use the appropriate interview script below. **Surface chat context first** — if the conversation history mentions voice preferences, banned words, or anything else relevant to the slot, pre-fill suggestions before asking. Don't make the user retype what they already said.

#### Interview: style-guide

Three questions, asked one at a time.

**Q1 — Voice.** *"In 5-10 words, what's the voice? (Conversational? Authoritative? Plain-language? Skeptical? Pick specific words — 'professional yet friendly' is filler.)"*

**Q2 — Top 5 rules.** *"Give me 3-5 hard rules for this brand's writing. One per line. Examples I've heard from members: 'Never use em-dashes', 'Always cite a source for claims', 'Lead sentences with the answer, not the setup', 'No hedge words like might/perhaps in opening sentences', 'Two-sentence paragraphs maximum'. What are yours?"*

**Q3 — Voice sample.** *"Paste one or two short paragraphs that nail this brand's voice — published writing, a LinkedIn post, an email you're proud of. If you don't have one handy, say 'skip' and we'll build from rules alone."*

**Pre-fill from chat context.** If the conversation has mentioned voice traits ("plain-jane", "contrarian", "no AI-slop vocab"), include them as suggested answers in Q1 / Q2.

Write to `style-guide.md`:

```markdown
# Style Guide — <display_name>

## Voice

<Q1 answer, polished into a sentence>

## Tone

<inferred from Q1 — e.g., "Direct, dry, occasionally skeptical." — or asked if Q1 left it ambiguous>

## Rules

- <Q2 rule 1>
- <Q2 rule 2>
- <Q2 rule 3>
- ...

## Examples

<Q3 paste, with light formatting>
```

#### Interview: glossary

Two questions.

**Q1 — Central entity.** *"What's the brand's central entity — the main thing it sells or is? What do you call it? (e.g., 'Omnipresence' / 'Omni' / 'the AI SEO agent'.) And what should agents NEVER call it? (e.g., not 'the assistant' / not 'the tool'.)"*

**Q2 — Other key terms.** *"Any other terms with strong preferences? Things like: product names, acronyms, domain-specific words you use differently than the industry default. One per line, or 'none' if nothing else."*

Write to `glossary.md`:

```markdown
# Glossary — <display_name>

## Preferred terms

| Concept | Preferred | Alternatives OK | Avoid |
|---|---|---|---|
| Central entity | <Q1 preferred> | <Q1 alternatives> | <Q1 avoid> |
| <other term 1> | <preferred> | <alternatives> | <avoid> |
| ...

## Custom terminology

<Q2 free-form, expanded into one entry per term with definition + example sentence>
```

#### Interview: banned-phrases

One question (it's a list-paste, not an interview).

**Q1 — The list.** *"Paste your never-use list. One word/phrase per line. If you also want to ban patterns (like 'never start a sentence with a question'), put those separately. I'll pre-suggest some common ones for you — say 'add' for any you also want, or 'skip' for the ones you don't.*

*Common bans (pick the ones you want): leverage, synergy, robust, seamless, in today's fast-paced world, indeed, furthermore, em-dashes, oxford commas in lists, starting a sentence with 'Look,' or 'Listen,'.*

*Your additions:"*

Write to `banned-phrases.md`:

```markdown
# Banned Phrases — <display_name>

## Never use

<Q1 word list>

## Pattern bans

<Q1 pattern list, if provided>
```

#### Interview: editor-rules

Two questions.

**Q1 — Publish gates.** *"What does a draft need to pass before it ships? List 3-5 hard yes/no gates. Examples: 'Cites at least one primary source.' / 'Opens with a declarative answer in sentence 1.' / 'Word count between 800-1500.' / 'Read-aloud test — sounds like a human.'"*

**Q2 — Auto-reject conditions.** *"What automatically fails a draft? Examples: 'Any banned phrase appears.' / 'Passive voice in opening paragraph.' / 'No specific examples in a how-to.' Type 'none' if nothing's auto-reject."*

Write to `editor-rules.md`:

```markdown
# Editor Rules — <display_name>

## Publish gates

- <Q1 gate 1>
- <Q1 gate 2>
- ...

## Auto-reject conditions

<Q2 list, or "None — soft review only.">
```

#### Interview: project-config

Don't reinvent. Chain to `project-config-generation` (the synapse process). Tell the user: *"Project config is the 8-field structured interview — I'll hand off to that flow now. Take 2-3 minutes, then I'll come right back to where we were."* Then invoke `project-config-generation`. Resume in Step 5 when it returns.

### Step 4 — Write the file

Overwrite the placeholder file with the filled-in content. Do NOT keep the `<!-- TBD-PLACEHOLDER: ... -->` marker — its presence is what the detection logic uses to flag the file as empty, and we just filled it.

Tell the user: *"Saved to `custom/projects/<active>/<file>`. Refreshing my context now."* Then re-Read the file into context so subsequent steps see the new content.

### Step 5 — Resume the calling task (if Mode B or C)

If this skill was invoked as an interrupt (Mode B / Mode C), return control to the calling task with a one-line handoff: *"Back to it — I had the `<original task description>` going. Continuing now with the new `<slot>` rules applied."*

If this was Mode A (explicit user request), the success message is enough — STOP.

### Step 6 — Report success (Mode A) OR pass control back (Mode B/C)

For Mode A, tell the user:

```
✅ <slot> filled in for project '<active-slug>'.

  File: custom/projects/<active-slug>/<file>

Captured:
  • <one-line summary of what's now in the file>

Next prompts:

  • Fill another:      Fill in my <next slot suggestion>.
  • Use it:            <task suggestion that benefits from the new file>.
  • Save to GitHub:    Push my synapse changes.
```

For Mode B / C, the calling task takes over with the new file applied.

### Stop here.

## Detection helper (for callers)

Other skills can detect TBD placeholders by reading the project file and grepping for `TBD-PLACEHOLDER:`. If the marker exists anywhere in the file's first 5 lines, treat the slot as empty.

A reference implementation in pseudocode (callers can inline this):

```
isPlaceholder(filePath):
    content = read(filePath, lines=1..10)
    return content contains "TBD-PLACEHOLDER:"
```

Specifically:
- `load-worker` Step 4 should run this check on every project file in the auto-load list and flag placeholders in the warmup confirmation.
- Workers mid-task should run this check before applying anything from a project file. If placeholder, fire `fill-project-gap` in Mode C.

## Edge cases

**The user said "fill in my brand voice" but `style-guide.md` already has content.** Step 2 catches this — confirm overwrite vs show. Default to "show" so we don't clobber.

**The slot's file doesn't exist at all (member is on a stale fork that predates the new-project scaffolding update).** Step 2 covers this — create the file from scratch using the same interview. No special handling.

**The user fills out style-guide but accidentally includes a banned phrase in the example paragraph.** Don't auto-correct. The user is establishing the source of truth — if they put it in, they meant it. (Optionally surface as a "heads up" after writing.)

**The user invoked this skill but there's no active project.** Step 2 routes to `new-project`. Don't try to fill an orphaned file.

**Mode C interrupt: the user picked "Fill now" but then mid-interview said "actually skip this, just keep working".** Treat as soft abort — don't write a half-filled file. Return control to the calling task with a one-line: *"Skipping `<slot>` for now — left it as a placeholder. Back to the task."*

**Two slots are empty (e.g., style-guide AND glossary) when a worker fires Mode C.** Ask the user once: *"Two placeholders are empty: style-guide and glossary. Fill both, just one, or skip both?"* Don't chain-interview without consent.

## What this skill MUST NOT do

- Do NOT overwrite a filled file without explicit user confirmation.
- Do NOT preserve the `TBD-PLACEHOLDER:` marker after writing — its absence is what tells future agents the slot is filled.
- Do NOT auto-fill from training data. Every value in the file comes from the user's answers (or from existing chat context that the user has explicitly endorsed). Inventing brand voice rules is the worst-case failure mode of this skill.
- Do NOT chain-interview multiple slots without asking the user.
- Do NOT modify files outside `custom/projects/<active-slug>/`.
- Do NOT touch the global active-project pointer.
- Do NOT chain to `push-changes` automatically — user opts in.
- Do NOT shame the user for empty placeholders. State facts about the file, not judgments about the user.

## Skill family

- [`new-project`](../new-project/SKILL.md) — scaffolds the placeholders during project creation
- [`fill-project-gap`](../fill-project-gap/SKILL.md) — this skill; fills placeholders when needed
- [`load-worker`](../load-worker/SKILL.md) — detects + flags placeholders in the warmup confirmation
- [`create-worker`](../create-worker/SKILL.md) — the worker spec template's Step 1 also checks for placeholders before starting a task

The flow: `new-project` writes the stubs → time passes → an agent (load-worker, a worker mid-task, or the user) detects a placeholder → `fill-project-gap` fills it → future tasks have the content.
