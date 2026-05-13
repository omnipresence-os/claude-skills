---
name: sync-omnipresence
description: Pulls the latest methodology updates from omnipresence-os/synapse upstream into the user's local synapse fork and pushes the merge to their GitHub. Trigger when the user says any of "sync omnipresence", "update synapse", "pull latest", "get omnipresence updates", "update omni", "what's new in synapse", "refresh my synapse". This is the ONE canonical update flow. Safely stashes any uncommitted member work before fetching, handles merge conflicts by surfacing the core/-edit rule violation clearly, and reports what changed since the last sync. Idempotent.
---

# Sync Omnipresence — Pull Upstream Updates

This skill pulls the latest methodology, processes, and skills from `omnipresence-os/synapse` into the member's local clone and pushes the merge to their GitHub fork. ONE canonical flow. No alternatives.

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

### Step 9: Report what changed

```
git log --oneline PRE_SYNC_COMMIT..HEAD
```

- **If the output is empty** (no new commits), tell the user: "Already up to date — no new methodology since your last sync."
- **Otherwise**, summarize:
  > "Synced. <N> new commits from upstream:
  > 
  > <git log oneline output>
  > 
  > Your customizations are intact. Your fork at github.com/THEIR-USERNAME/synapse is now up to date with omnipresence-os/synapse."

### Stop here.

Do not propose next steps. Do not suggest running anything else. The user got an update; they're done.

## What this skill MUST NOT do

- Do NOT modify any file in `core/` directly.
- Do NOT modify any file in `custom/` or `overrides/` without explicit user consent.
- Do NOT force-push to origin. Ever.
- Do NOT skip the stash step — uncommitted member work must be preserved.
- Do NOT propose alternative sync strategies (rebase instead of merge, cherry-pick, etc.). One method.
- Do NOT proceed if the rule-violation recovery is declined; abort cleanly and let the user message Jonathan.
