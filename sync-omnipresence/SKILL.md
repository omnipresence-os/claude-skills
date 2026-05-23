---
name: sync-omnipresence
description: Updates both halves of an Omnipresence install — the methodology corpus (omnipresence-os/synapse upstream → user's fork) and the prompt-driven Claude Code skill set (omnipresence-os/claude-skills → ~/.claude/skills). Trigger when the user says any of "sync omnipresence", "update synapse", "pull latest", "get omnipresence updates", "update omni", "what's new in synapse", "refresh my synapse", "update my skills", "pull latest skills". This is the ONE canonical update flow for both. Safely stashes any uncommitted member work before fetching, handles merge conflicts on the fork by surfacing the core/-edit rule violation clearly, fast-forwards the skills install in place, and reports what changed across both. Idempotent. Tells the member to restart Claude Code if any new Claude Code skills landed (Claude Code discovers skills at process start, not per-session).
---

# Sync Omnipresence — Pull Upstream Updates

This skill updates BOTH halves of an Omnipresence install:

1. **Methodology corpus** — `omnipresence-os/synapse` upstream → the member's local synapse fork → their GitHub. Pulls new beliefs, processes, executor skills.
2. **Prompt-driven Claude Code skills** — `omnipresence-os/claude-skills` → `~/.claude/skills/`. Pulls new "Claude Code commands" like `connect-google-cloud`, `new-project`, etc.

ONE canonical flow. No alternatives.

## How to talk to the user during this skill

**Critical UX rule:** do NOT show the user shell commands or terminal output in your replies. Run commands silently and explain what's happening in plain English. The user is not a coder. Shell commands look scary and lose them.

✅ Good: *"Pulling the latest updates from upstream... 12 new files, no conflicts."*
❌ Bad: *"Running `git fetch upstream && git merge upstream/main`..."*

Only show:
- Plain-English progress updates ("Stashing your work-in-progress safely..." / "Merging upstream changes...").
- Human-readable summaries of what changed (e.g., "Synced. 12 new commits — 3 new methodologies, 2 new skills, 7 updates").
- Clear yes/no recovery prompts if a conflict happens.

Never show: `git status` output, raw commit hashes, merge marker syntax, remote URLs, branch names. Translate to human terms before surfacing.

## Prerequisites

- The user has already run `getting-started` (their local synapse exists, upstream remote is wired). If not, redirect them: "It looks like Omnipresence isn't set up yet. Run getting started first."

## Execute these steps in order

### Step 1: Locate the synapse clone

Read the cached path from `~/.claude/skills/.omnipresence-path` (or `%USERPROFILE%\.claude\skills\.omnipresence-path` on Windows).

- **If the file exists and the path is valid** (contains a `.git` folder + `core/` + `package.json`), use it.
- **If the file doesn't exist OR the path is invalid**, search common locations:
  - `~/Documents/omnipresence/synapse`
  - `%USERPROFILE%\Documents\omnipresence\synapse`
  - `~/synapse`, `~/dev/synapse`, `~/projects/synapse`
- **If none found**, ask the user once:
  > "I can't find your synapse clone. Paste the absolute path where you cloned it (or run getting started if you haven't set up Omnipresence yet)."
  
  Once they provide a valid path, cache it to `.omnipresence-path` for next time.

### Step 2: Verify it's a synapse fork

```
cd <path>
git remote get-url upstream
```

Expected: `https://github.com/omnipresence-os/synapse.git` (or git@ form).

- **If `upstream` remote doesn't exist or points elsewhere**, tell the user: "This doesn't look like a synapse fork — the upstream remote is missing or wrong. Run getting started to set up Omnipresence correctly."

### Step 3: Save the user's uncommitted work safely

```
cd <path>
git status --porcelain
```

- **If output is empty** (working tree clean), proceed without stashing.
- **If there are uncommitted changes**, stash them:
  ```
  git stash push -u -m "auto-stash before sync-omnipresence"
  ```
  Remember to unstash at the end (step 8).

Tell the user: "Found uncommitted work — I've stashed it safely. It'll come back after the sync."

### Step 4: Capture pre-sync state

```
git rev-parse HEAD
```

Save as `PRE_SYNC_COMMIT` so we can show the user what changed.

### Step 5: Pull from upstream

```
git fetch upstream
git checkout main
git merge upstream/main --no-edit
```

### Step 6: Handle merge conflicts (the only real failure mode)

After the merge command, check for conflicts:
```
git diff --name-only --diff-filter=U
```

- **If no conflicts** (empty output), continue to Step 7.

- **If conflicts**, every conflicting file SHOULD be in `core/` (member's rule violation). Tell the user:
  > "You've got conflicts on these files:
  > 
  > <list>
  > 
  > These are in `core/`, which is reserved for upstream. Edits to `core/` will conflict every time you sync. To recover, I can discard your `core/` edits and adopt the upstream version. Your `custom/` and `overrides/` work is untouched.
  > 
  > Proceed with the recovery? (Yes/No)"

  - **On Yes:**
    ```
    git checkout --theirs core/
    git add core/
    git commit --no-edit
    ```
    Continue to Step 7.
  
  - **On No:** abort the merge and surface the situation:
    ```
    git merge --abort
    ```
    Then tell the user: "Merge aborted. Your `core/` edits are still in place but you can't sync until you resolve them. Easiest path: copy any `core/` changes you want to keep into `overrides/` (same file path), then re-run sync. If you're stuck, email jonathan@getomnipresence.com." STOP here.

- **If conflicts are in `custom/` or `overrides/`** (unusual — only happens if upstream renamed a member-zone file): preserve the user's version:
  ```
  git checkout --ours custom/ overrides/
  git add custom/ overrides/
  git commit --no-edit
  ```
  Note in the report: "Upstream changed some custom/ or overrides/ structure — kept your version."

### Step 7: Push the merge to the user's fork

```
git push origin main
```

- **If push fails with "non-fast-forward":** the user pushed something to their fork from another machine. Tell them: "Your fork has changes I don't have locally. Run sync again and I'll pick those up too." Stop here.
- **If push succeeds:** continue.

### Step 8: Restore the stash (if step 3 stashed)

```
git stash pop
```

- **If `git stash pop` reports conflicts** (rare — only if a stashed change collides with an upstream change), tell the user: "Your stashed work conflicts with the upstream update. I've left the stash in place — run `git stash list` to see it, or email jonathan@getomnipresence.com for help." STOP.

### Step 9: Refresh the Claude Code skills install

The synapse fork holds methodology + processes + executor skills. Separately, `~/.claude/skills/` holds the prompt-driven Claude Code skills — `connect-google-cloud`, `new-project`, `sync-omnipresence` itself, etc. Different repo (`omnipresence-os/claude-skills`); doesn't get touched by the synapse fork sync above. This step keeps it current too.

**Verify the install is a git clone of claude-skills:**

```
cd ~/.claude/skills    # or %USERPROFILE%\.claude\skills on Windows
git remote get-url origin
```

- **If origin = `https://github.com/omnipresence-os/claude-skills.git` (or git@ form):** it's a clone — proceed.
- **If `git remote get-url origin` fails or returns something else:** the install was done with an older copy-files method that can't be auto-updated. Tell the user:
  > "Your Claude Code skills were installed in a way that can't be auto-updated to pick up new prompts. To fix it: paste `Install the Omnipresence skills from github.com/omnipresence-os/claude-skills into my ~/.claude/skills directory` — that'll set up a clone, after which future syncs will keep them current automatically. Run that first, then re-run sync."
  
  Skip the rest of this step and continue to Step 10 with `SKILLS_UPDATED = false`.

**Capture pre-pull state for diffing:**

```
cd ~/.claude/skills
PRE_SKILLS_COMMIT=$(git rev-parse HEAD)
PRE_SKILLS_FOLDERS=$(find . -maxdepth 1 -type d -not -name '.*' | sort)
```

**Pull latest with fast-forward only:**

```
git pull origin main --ff-only
```

- **If pull fails with non-fast-forward error** (rare — only if someone committed locally to `~/.claude/skills/`): tell the user *"Your Claude Code skills folder has local edits, which is unusual — usually skills are read-only on the member side. Skipping the skills update for safety. If this happens repeatedly, email jonathan@getomnipresence.com."* and continue to Step 10 with `SKILLS_UPDATED = false`.
- **If pull succeeds:** continue.

**Diff the new skill folders:**

```
POST_SKILLS_FOLDERS=$(find . -maxdepth 1 -type d -not -name '.*' | sort)
NEW_SKILLS=$(comm -13 <(echo "$PRE_SKILLS_FOLDERS") <(echo "$POST_SKILLS_FOLDERS"))
POST_SKILLS_COMMIT=$(git rev-parse HEAD)
```

Save `NEW_SKILLS` (list of new skill folder names) and whether the commit advanced (`PRE_SKILLS_COMMIT != POST_SKILLS_COMMIT`) for Step 10's combined report.

### Step 10: Report what changed (both halves)

```
git -C <synapse-path> log --oneline PRE_SYNC_COMMIT..HEAD
```

Assemble a single combined report:

**If nothing changed in either** (synapse no new commits AND skills already current):
> "Already up to date — no new methodology, no new Claude Code skills since your last sync."

**Otherwise:**
> "Synced.
>
> **Methodology (synapse fork):** <N> new commits from upstream:
> 
> <git log oneline output, or 'no changes' if empty>
> 
> **Claude Code skills:** <one of the following>
> - *No new skills landed.* (if synapse advanced but skills didn't)
> - *<N> new skill(s) landed: <comma-separated list from NEW_SKILLS>. Restart Claude Code (full app restart, not a new chat) so they get discovered.* (if NEW_SKILLS non-empty)
> - *Existing skills updated, no new ones. No restart needed.* (if skills commit advanced but no new folders)
>
> Your customizations in `custom/`, `overrides/`, and `custom/projects/` are intact. Your local fork at github.com/THEIR-USERNAME/synapse is now up to date with omnipresence-os/synapse.
>
> **What this affects:** Claude Code reads your local fork directly, so the new upstream methodology is available to your prompts immediately. New Claude Code skills require an app restart to be discoverable. Note: the hosted Omnipresence MCP (used by claude.ai web, ChatGPT, mobile) serves canonical content only and doesn't see your fork — customizations live in Claude Code only."

### Stop here.

Do not propose next steps. Do not suggest running anything else. The user got an update; they're done.

## What this skill MUST NOT do

- Do NOT modify any file in `core/` directly.
- Do NOT modify any file in `custom/` or `overrides/` without explicit user consent.
- Do NOT force-push to origin. Ever.
- Do NOT skip the stash step — uncommitted member work must be preserved.
- Do NOT propose alternative sync strategies (rebase instead of merge, cherry-pick, etc.). One method.
- Do NOT proceed if the rule-violation recovery is declined; abort cleanly and let the user message Jonathan.
- Do NOT use a non-`--ff-only` pull on `~/.claude/skills/` — that directory is supposed to be read-only on the member side; if it has divergent commits something is off, surface it rather than auto-merging.
- Do NOT silently skip the claude-skills refresh — always report whether it happened and what landed, even if "nothing changed" or "install method can't be auto-updated."
