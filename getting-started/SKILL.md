---
name: getting-started
description: Sets up a new Omnipresence member with their own private fork of omnipresence-os/synapse on GitHub and a local clone wired to pull upstream updates. Trigger when the user says any of "run getting started", "set up omnipresence", "install omnipresence", "onboard me to omnipresence", "fork synapse", "I'm a new member", "set me up", "get me started with omni". This is the ONE canonical first-time setup flow for Omnipresence members. Forks omnipresence-os/synapse to the user's GitHub account, clones to a sensible local path, configures upstream remote, installs npm deps, and validates with npm run manifest. Self-installs the gh CLI if missing. Self-authenticates via gh auth login if the user isn't signed in. Idempotent - safe to re-run if interrupted.
---

# Getting Started — Omnipresence Onboarding

This skill is THE canonical first-time setup flow for an Omnipresence member. Follow it exactly. Do not invent alternatives, do not propose other approaches, do not let the user pick options — there is one method.

## Prerequisites this skill handles automatically

- GitHub CLI (`gh`) — installs via winget (Win), brew (Mac), or apt (Linux) if missing.
- GitHub authentication — runs `gh auth login --web` if the user isn't signed in.
- git — assumed installed; if not, instructs the user to install from git-scm.com.

## What this skill does — execute these steps in order

### Step 1: Verify prerequisites

```
gh --version
```

- **If gh is not installed**, install it:
  - **Windows:** `winget install --id GitHub.cli`
  - **Mac:** `brew install gh`
  - **Linux (Debian/Ubuntu):** `sudo apt update && sudo apt install gh`
  - **Anything else:** point the user at https://cli.github.com/manual/installation and ask them to install + restart their terminal.

Then re-run `gh --version` to confirm.

Run `git --version`. If git isn't installed, tell the user: "Install git from https://git-scm.com/downloads and re-run me." Stop here.

### Step 2: Authenticate with GitHub

```
gh auth status
```

- **If the user is not authenticated**, run `gh auth login --web` and walk them through:
  > "I'm going to open a browser window where you sign in to GitHub. When you see a one-time code on screen, copy it and paste it into the browser. After you authorize, come back here — I'll continue automatically."

- After `gh auth login --web` completes, re-run `gh auth status` to confirm.

### Step 3: Check membership-access prerequisite

```
gh repo view omnipresence-os/synapse --json name 2>&1
```

- **If this succeeds** (returns JSON with `"name": "synapse"`), the user has access. Continue.
- **If this fails with 404** or similar, the user has not yet accepted their outside-collaborator invitation. Tell them:
  > "I can't see omnipresence-os/synapse on your account. That usually means you haven't accepted Jonathan's invitation yet. Check your email inbox for a message from GitHub titled '[GitHub] @boshify invited you to collaborate' or similar — click 'Accept invitation' in that email, then ask me to run getting started again. If you can't find the invitation, email jonathan@getomnipresence.com and ask for a fresh invite."
  > 
  > STOP. Do not proceed.

### Step 4: Fork omnipresence-os/synapse to the user's account

```
gh repo fork omnipresence-os/synapse --clone=false --remote=false
```

- This creates `github.com/THEIR-USERNAME/synapse` (private by default, inheriting from upstream).
- **If the fork already exists**, gh reports this and continues without error. That's fine — continue.

Capture the user's GitHub username from `gh api user --jq .login`.

### Step 5: Choose a local clone path

Default path:
- **Windows:** `%USERPROFILE%\Documents\omnipresence\synapse`
- **Mac/Linux:** `~/Documents/omnipresence/synapse`

Ask the user, once:
> "I'll clone synapse to `<DEFAULT_PATH>`. Press Enter to accept, or paste a different absolute path."

If they press Enter, use the default. If they paste something, use that. Don't ask twice.

Make sure the parent directory exists (`mkdir -p` equivalent for the chosen platform).

### Step 6: Clone the fork

```
gh repo clone THEIR-USERNAME/synapse <chosen-path>
```

If the directory already exists and is a valid synapse clone, tell the user:
> "Synapse is already cloned at `<chosen-path>`. I'll wire up upstream and validate it."

Don't error out — just continue to Step 7 with the existing clone.

### Step 7: Wire up upstream

```
cd <chosen-path>
git remote add upstream https://github.com/omnipresence-os/synapse.git 2>/dev/null || git remote set-url upstream https://github.com/omnipresence-os/synapse.git
git fetch upstream
```

The `||` handles re-runs gracefully — if `upstream` already exists, we update its URL instead of failing.

Verify with `git remote -v` — the user should see both `origin` (their fork) and `upstream` (omnipresence-os).

### Step 8: Install dependencies

```
cd <chosen-path>
npm install
```

This takes ~30-60 seconds. Tell the user what's happening:
> "Installing dependencies (about 30 seconds)..."

### Step 9: Validate

```
cd <chosen-path>
npm run manifest
```

Expected output: `Manifest written: ... XXX entries (XX methodology, XX process, XX skill)`.

If the manifest builder reports any validation errors, surface them and stop — the clone is corrupted and the user should email Jonathan.

### Step 10: Cache the synapse path for future skills

Write the chosen path to `~/.claude/skills/.omnipresence-path` so the `sync-omnipresence` and `push-changes` skills can find it later without asking the user again.

On Windows: `%USERPROFILE%\.claude\skills\.omnipresence-path`
On Mac/Linux: `~/.claude/skills/.omnipresence-path`

### Step 11: Report success

Tell the user, exactly:

```
✅ Omnipresence is set up.

  Your private fork:  github.com/THEIR-USERNAME/synapse
  Local clone:        <chosen-path>
  Upstream link:      omnipresence-os/synapse

Three prompts you'll use going forward:

  • To pull the latest methodology from upstream:
    "Sync omnipresence."

  • To save your customizations to your GitHub fork:
    "Push my synapse changes."

  • To get help if something goes wrong:
    "Something is wrong with my Omnipresence setup."

One rule: edit only in custom/ and overrides/. Never edit core/.
Full onboarding doc: <chosen-path>/docs/MEMBER-ONBOARDING.md
```

### Stop here. Do NOT offer additional steps.

The user is done. They can close Claude Code or move to a different task. Do not propose follow-up actions, do not suggest tutorials, do not chain into other workflows.

## Failure-handling rules

- **`gh repo fork` returns 403:** the user has access to the source repo but cannot fork private repos. Tell them: "Forking of private repos is disabled on your account. Ask Jonathan to verify that omnipresence-os has 'Allow forking of private repositories' enabled in org member-privileges settings. Once he confirms, ask me to run getting started again."
- **Network failure anywhere:** report the specific failed step and tell the user to check their connection + re-run.
- **Permission denied on the chosen path:** ask the user for a different path; don't try to chmod or sudo.
- **npm install fails:** report the error verbatim. Most likely cause is they don't have Node.js installed. Point at https://nodejs.org and ask them to install + re-run.

## What this skill MUST NOT do

- Do NOT propose alternative onboarding methods.
- Do NOT manually walk the user through the GitHub website UI ("click Fork on github.com").
- Do NOT skip the upstream-remote step (without it, sync-omnipresence breaks later).
- Do NOT edit anything in the user's clone beyond running `npm install` and `npm run manifest`.
- Do NOT push anything to the user's fork (this is setup, not their first commit).
