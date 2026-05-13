---
name: push-changes
description: Commits and pushes the user's customizations in custom/ and overrides/ to their private synapse fork on GitHub. Trigger when the user says any of "push my synapse changes", "save my synapse work", "commit my synapse", "push omnipresence", "push my customizations", "save what I added to synapse". This is the ONE canonical save-and-push flow. Only commits files in custom/ and overrides/ - never touches core/, even if the user accidentally edited it. Safe to run after editing methodology / process / skill files in the member zones.
---

# Push Changes — Save Member Customizations to Their Fork

This skill commits and pushes the user's work in `custom/` and `overrides/` to their GitHub fork. ONE canonical flow. Refuses to push `core/` edits.

## Prerequisites

- The user has already run `getting-started` (their local synapse exists, upstream remote is wired).
- They've edited files in `custom/` and/or `overrides/`.

## Execute these steps in order

### Step 1: Locate the synapse clone

Same as sync-omnipresence Step 1 — read `~/.claude/skills/.omnipresence-path` or search common locations or ask once.

### Step 2: Verify it's a synapse fork

```
cd <path>
git remote get-url upstream
```

Must point to omnipresence-os/synapse. If not, redirect to getting-started.

### Step 3: Check what's staged for commit

```
cd <path>
git status --porcelain
```

- **If output is empty**, tell the user: "Nothing to commit — your customizations are already saved. Did you mean to sync from upstream? Try 'sync omnipresence'."  STOP.

- **Otherwise**, parse the file paths from the status output.

### Step 4: Enforce the core/ rule

Separate the changed files into two groups:
- **Member zone:** paths starting with `custom/` or `overrides/`
- **Core zone:** paths starting with `core/` or anywhere else (`docs/`, `scripts/`, root-level files)

- **If the core zone is non-empty**, tell the user:
  > "You've got edits in these `core/` files (or other upstream-managed locations):
  > 
  > <list>
  > 
  > These will conflict with future updates from upstream. The rule is: edit only `custom/` and `overrides/`.
  > 
  > Options:
  > 
  > 1. **Move the changes:** I can help you relocate these edits to `overrides/` (which preserves them and won't conflict with upstream). Reply 'move them'.
  > 2. **Discard the core/ edits:** I can revert them so the file matches upstream again. Reply 'discard them'.
  > 3. **Push only member-zone changes for now:** I'll commit and push `custom/` + `overrides/` changes, leaving the `core/` edits uncommitted on your machine. You can deal with them later. Reply 'just push the safe stuff'.
  > 
  > Which option?"

  - **On "move them":** for each `core/` file the user changed, copy the changed file to the same path under `overrides/`, then revert the `core/` version with `git checkout core/<file>`. Then continue to Step 5.
  - **On "discard them":** run `git checkout core/ <other-non-member-paths>` to revert each non-member-zone change. Then continue to Step 5.
  - **On "just push the safe stuff":** do nothing to the core changes; they stay uncommitted on disk. Continue to Step 5 — only stage member-zone files.
  - **On anything else:** repeat the question once. If still unclear, STOP and tell the user to message Jonathan.

### Step 5: Stage member-zone files only

```
cd <path>
git add custom/ overrides/
```

Use `git add` with explicit paths — never `git add -A` or `git add .` (which would catch core/ changes too).

### Step 6: Ask for a commit message

Ask the user, once:
> "What's a short description of what you changed? (One sentence, like 'Added my client's brand voice profile' or 'Customized refresh thresholds for low-traffic sites'.)"

If they don't reply with anything substantive, use a default: `"Member customizations: <comma-separated top-level dir names under custom/ and overrides/>"`.

### Step 7: Commit

```
git commit -m "<their message>"
```

### Step 8: Push to their fork

```
git push origin main
```

- **If push fails with "non-fast-forward":** their fork has changes they don't have locally (pushed from another machine, or upstream was synced elsewhere). Tell them: "Your fork has changes I don't have locally. Run 'sync omnipresence' first to pull them, then I'll push your new work on top. Try the sync, then re-run push."
- **If push succeeds:** continue.

### Step 9: Report success

Tell the user:
> "✅ Pushed. Your customizations are saved to github.com/THEIR-USERNAME/synapse.
> 
> Committed: <N> files changed.
> Message: '<their message>'."

If any `core/` edits were left uncommitted in step 4 ("just push the safe stuff" branch), remind them:
> "Note: you still have unsaved changes in `core/`. Those won't sync to your fork. Move them to `overrides/` or discard them next time."

### Stop here.

## What this skill MUST NOT do

- Do NOT stage or commit anything outside `custom/` and `overrides/`. Period.
- Do NOT use `git add -A` or `git add .`.
- Do NOT auto-resolve the core/-edits scenario silently. The user must consent to one of the three options.
- Do NOT force-push.
- Do NOT amend previous commits (always create a NEW commit, even if the user re-runs immediately after a typo).
- Do NOT push to upstream (origin = the user's fork; upstream = omnipresence-os, never pushed to).
