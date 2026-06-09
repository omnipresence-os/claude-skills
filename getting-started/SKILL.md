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

## How to talk to the user during this skill

**Critical UX rule:** do NOT show the user shell commands or terminal output in your replies. Run commands silently and explain what's happening in plain English. The user is not a coder; they're a marketer / SEO operator. Shell commands look scary and lose them.

✅ Good: *"Checking if you have GitHub CLI installed... already there, good."*
❌ Bad: *"Running `gh --version`... output: gh version 2.40.1"*

✅ Good: *"I'll authenticate you with GitHub using a Personal Access Token. Easier than the alternatives. Here's exactly what to do..."*
❌ Bad: *"You can run `gh auth login` and select your preferred protocol..."*

Only show URLs and instructions that involve the user clicking something in their browser. Never show terminal commands as instructions for the user to type.

## What this skill does — execute these steps in order

### Step 1: Verify prerequisites (run silently)

Run `gh --version` and `git --version` in the background. Don't show the output to the user.

- **If gh is not installed**, tell the user in plain English: *"I need to install the GitHub command-line tool first. This takes about 30 seconds and runs in the background."* Then install silently:
  - **Windows:** `winget install --id GitHub.cli` (or point at https://cli.github.com if winget isn't available)
  - **Mac:** `brew install gh` (or point at https://cli.github.com if brew isn't available)
  - **Linux (Debian/Ubuntu):** `sudo apt update && sudo apt install gh`
  
  After install, ask the user to restart their terminal if `gh --version` still doesn't work post-install.

- **If git is not installed**, tell the user: *"You'll need to install Git first — that's the standard tool for working with code repos. Download it from https://git-scm.com/downloads, install with all defaults, then come back here and I'll continue."* STOP and wait.

### Step 2: Authenticate with GitHub (Personal Access Token method — DEFAULT, do not deviate)

**Do NOT run `gh auth login`. Do NOT run `gh auth login --web`. Do NOT recommend interactive terminal flows.** Those have multi-question prompts that confuse non-technical users.

Run `gh auth status` silently to check current state.

- **If the user is already authenticated** (output mentions "Logged in to github.com"), tell them: *"You're already signed in to GitHub as `<USERNAME>`. Moving on."* Skip to Step 3.

- **If not authenticated**, walk them through the Personal Access Token flow. Tell them, verbatim:

  > "I need to connect Claude Code to your GitHub. We'll use a Personal Access Token — it's the simplest method, no scary terminal stuff. Here's exactly what to do:
  >
  > **1.** Click this link to open GitHub's token-creation page in your browser:
  >
  > https://github.com/settings/tokens/new?scopes=repo,read:org&description=Omnipresence+(Claude+Code)
  >
  > **2.** If you're not already signed in to GitHub, sign in first — the link will reload after.
  >
  > **3.** The form is mostly pre-filled. Two boxes are already checked under "Select scopes": `repo` and `read:org`. Don't touch those.
  >
  > **4.** **Set "Expiration" to 90 days** (or longer if you prefer — but not "No expiration"; we want this to rotate eventually).
  >
  > **5.** Scroll to the very bottom and click the green **"Generate token"** button.
  >
  > **6.** A token will appear at the top of the next page. It starts with `ghp_` followed by a long string. **Copy the whole thing.** GitHub only shows it once — if you lose it, you have to make a new one.
  >
  > **7.** Come back here and **paste the token as your next message.** Just the token, nothing else."

  Wait for the user's next message. When they paste:

  - **Validate loosely:** check that the message starts with `ghp_` or `github_pat_` and is at least 40 characters. If it doesn't look like a token, ask once more politely: *"That doesn't look like a GitHub token — they start with `ghp_` followed by a long string of letters and numbers. Try copying it again, making sure you get the whole thing."*
  
  - **Authenticate silently** by piping the token to `gh auth login --with-token`:
    - **Windows PowerShell:** `"<TOKEN>" | gh auth login --with-token`
    - **Mac/Linux bash:** `echo "<TOKEN>" | gh auth login --with-token`
    
    Do NOT print the command or the token in your reply. Just run it.
  
  - **Verify** with `gh auth status` silently. Expected output mentions "Logged in to github.com account <username>".
  
  - **Confirm to the user:** *"✅ Connected to GitHub as `<USERNAME>`. Your token is now stored securely in GitHub's command-line tool. It'll auto-renew through that tool until the 90-day expiration, at which point I'll ask you to make a new one."*
  
  - **Security note to the user (one line, then move on):** *"Heads up: the token you pasted is now in this chat's history. If you ever share this conversation, regenerate the token first at github.com/settings/tokens."*

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

### Step 8.5: Install the Omnipresence Claude Code skills (canonical clone)

**This step exists because skipping it is the #1 silent install failure.** The prompt-driven Claude Code skills the member uses every day (`new-project`, `switch-project`, `create-worker`, `load-worker`, `sync-omnipresence`, etc.) live in a separate repo (`omnipresence-os/claude-skills`) and must be installed as a `git clone` into `~/.claude/skills/`. If this step is skipped, future agents will see "skill missing" and **fall back to `skill-creator`** to hand-write the skill files — producing local-only files that don't update on `sync-omnipresence` and silently drift from canonical.

Run silently — do not paste the commands into chat. Tell the user *"Installing the Omnipresence Claude Code skill set..."* then run the three-case handler:

**Case 1: `~/.claude/skills/` doesn't exist.** Fresh clone:

```
git clone https://github.com/omnipresence-os/claude-skills.git ~/.claude/skills
```

**Case 2: `~/.claude/skills/` exists AND is a proper clone of omnipresence-os/claude-skills.** Verify with `git -C ~/.claude/skills/ remote get-url origin` — if it returns a URL containing `omnipresence-os/claude-skills`, skip with *"Omnipresence Claude Code skills already installed and properly linked."*

**Case 3: `~/.claude/skills/` exists but is NOT a proper clone** (no `.git/` dir, OR `remote get-url origin` returns something other than omnipresence-os/claude-skills). This is the bad state — the directory has been hand-populated by a previous agent that fell into the skill-creator trap. Recover:

```
mv ~/.claude/skills ~/.claude/skills.backup.<YYYY-MM-DD-HHMMSS>
git clone https://github.com/omnipresence-os/claude-skills.git ~/.claude/skills
```

Surface the backup path to the user: *"Your existing `~/.claude/skills/` looked hand-written rather than cloned. I backed it up to `~/.claude/skills.backup.<timestamp>/` so you can recover anything custom from there, then installed the canonical version from `omnipresence-os/claude-skills`."*

**After the clone (any case):** tell the user *"Restart Claude Code so it discovers the newly installed skills (Claude Code reads the skill set at process start, not per-session)."*

**Windows path note:** `~/.claude/skills` resolves to `%USERPROFILE%\.claude\skills` on Windows.

**⚠️ Critical guard.** If at any point an agent thinks "the skills are missing, I should create them" and reaches for `skill-creator` — STOP. The Omnipresence skills are canonical and vendored from `omnipresence-os/claude-skills`. They must be cloned, never authored. If this step's git clone fails for network/auth reasons, surface the specific error and ask the user to fix it, but do NOT invoke `skill-creator` as a fallback.

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

### Step 11: Report success and tell them how to open their new folder

Tell the user, exactly:

```
✅ Omnipresence is set up.

  Your private fork:  github.com/THEIR-USERNAME/synapse
  Local clone:        <chosen-path>
  Upstream link:      omnipresence-os/synapse

──────────────────────────────────────────────────
Next step — open the synapse folder in your IDE

You're currently in a launchpad folder. The actual synapse content
lives at the path above. To start working with it:

  1. In VS Code (or Cursor): File → Open Folder
  2. Navigate to <chosen-path>
  3. Click "Open"
  4. You can close this current window once the new one opens —
     the launchpad folder has served its purpose.

(Tip: if you're on a Mac, the shortcut is Cmd+O. On Windows/Linux,
Ctrl+K then Ctrl+O.)
──────────────────────────────────────────────────

Three prompts you'll use going forward (in your new synapse folder):

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
- Do NOT skip Step 8.5 — without the `~/.claude/skills/` clone, every future "I need to create a project" / "I need to load a worker" prompt will fall back to `skill-creator`, which produces local-only hand-written files that drift from canonical and can't be repaired by `sync-omnipresence`. The clone is non-negotiable.
- Do NOT invoke `skill-creator` to author any Omnipresence skill (load-worker, create-worker, sync-omnipresence, switch-project, etc.) under ANY circumstances. Those skills are canonical and vendored from `omnipresence-os/claude-skills`. They must be cloned (Step 8.5), never hand-written. If a skill appears missing in a future session, the install was incomplete — re-run getting-started or `sync-omnipresence`, do not "create" the skill.
- Do NOT edit anything in the user's clone beyond running `npm install` and `npm run manifest`.
- Do NOT push anything to the user's fork (this is setup, not their first commit).
