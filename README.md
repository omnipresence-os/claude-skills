# Omnipresence Claude Code Skills

Three Claude Code skills that turn the Omnipresence member workflow into three plain-English prompts. Install once, then everything is prompt-driven — no GitHub UI, no git CLI, no clicking around.

## Install (one time, ~30 seconds)

Open [Claude Code](https://claude.com/code) in any folder. Paste this prompt and hit enter:

> Install the Omnipresence skills from github.com/omnipresence-os/claude-skills into my ~/.claude/skills directory, then run getting started.

Claude Code will:
1. Clone this repo into your local skills directory.
2. Trigger the `getting-started` skill, which forks `omnipresence-os/synapse` to your GitHub, clones it locally, and wires it up to receive future updates.

After that, you'll never have to think about installation again.

## The prompts

Once installed, these are the prompts you'll use:

**Setup & sync:**

| What you want to do | Prompt |
|---|---|
| **First-time setup** (auto-runs during install) | `Run getting started.` |
| **Pull the latest methodology updates** | `Sync omnipresence.` |
| **Save your customizations to your private fork** | `Push my synapse changes.` |

**Managing multiple clients / projects:**

| What you want to do | Prompt |
|---|---|
| **Create a new project** | `Create a new project for X.` |
| **List all your projects** | `List my projects.` |
| **Switch which project is active** | `Switch to project X.` |
| **See what's in the active project** | `What's in this project?` |

**Multi-week strategies (per project):**

| What you want to do | Prompt |
|---|---|
| **Plan a multi-week initiative** | `Create a new strategy for X.` |
| **List strategies for the active project** | `List my strategies.` |
| **Resume a strategy** | `Continue strategy X.` |

**Email:**

| What you want to do | Prompt |
|---|---|
| **Let Omni email things to you** (works out of the box, no setup) | `Email me a summary.` (any "email me..." phrasing) |
| **Set up sending from your own brand domain** (so Omni can email clients / prospects) | `Connect my email.` |

That's the whole vocabulary. Bookmark this section.

## What each skill does

### `getting-started`

The ONE first-time setup flow. Triggered by prompts like "run getting started," "set up omnipresence," "I'm a new member."

It:
- Installs the GitHub CLI if you don't have it.
- Signs you in to GitHub if you're not authenticated.
- Forks `omnipresence-os/synapse` to your account (private fork — only you can see it).
- Clones the fork locally to `~/Documents/omnipresence/synapse` (or a path you specify).
- Wires up the upstream remote so future sync prompts can pull updates.
- Installs npm dependencies.
- Validates everything.

Total time: ~2 minutes, of which 90 seconds is `npm install` in the background.

### `sync-omnipresence`

The ONE update flow. Triggered by prompts like "sync omnipresence," "update synapse," "pull latest."

It:
- Finds your local synapse clone.
- Safely stashes any uncommitted work.
- Pulls the latest from upstream.
- Merges into your fork.
- Pushes the merge to your GitHub.
- Restores your stashed work.
- Reports what changed since your last sync.

Total time: under 10 seconds for a typical sync. Handles merge conflicts (caused by accidentally editing `core/`) with a one-question recovery flow.

### `push-changes`

The ONE save flow. Triggered by prompts like "push my synapse changes," "save my synapse work."

It:
- Stages files in `custom/` and `overrides/` only.
- Refuses to commit edits to `core/` (it asks you to move them to `overrides/` instead).
- Asks for a short commit message.
- Pushes to your private fork.

### `new-project`

Stand up a new per-client / per-brand folder under `custom/projects/<slug>/`. Triggered by "create a new project for X," "set up a new client."

It creates the folder structure (`brand/`, `strategies/`, `notes.md`, `project-config.md`, `README.md`, `brand/style-guide.md` starter), optionally chains to project-config-generation if you want the structured 8-field config now, and sets the new project as the active one. ~2 minutes end-to-end.

### `list-projects`

Lists every project under `custom/projects/`. Triggered by "list my projects," "what projects do I have," "show me my clients."

Shows each project's display name, one-line description, number of strategies, last-modified date. Stars the currently active project. Read-only.

### `switch-project`

Sets the active project. Triggered by "switch to project X," "work on X," "set active project to X."

Updates `~/.claude/skills/.omnipresence-active-project` so every downstream project-aware skill sources from the right folder. Handles fuzzy matching when the input doesn't exactly match a folder name. Auto-picks the single project when there's only one and no active is set.

### `project-info`

Surfaces info about the active project. Triggered by "what's in this project," "what's the brand voice," "show me the project config," "show notes."

Routes the question to the right file under `custom/projects/<active>/` and surfaces the contents in plain English. Read-only.

### `new-strategy`

Creates a new multi-week strategy doc under the active project's `strategies/` folder. Triggered by "create a new strategy," "plan an initiative," "build a strategy for X."

Asks 5-7 scoping questions (context, success criteria, time horizon, phases, per-phase steps, autonomy mode, notes), drafts the strategy file with frontmatter + phased checkboxes + decisions log, updates the project README. ~5 minutes for the interview.

### `list-strategies`

Lists strategies for the active project (or all projects on request). Triggered by "list my strategies," "what strategies am I running," "list all strategies across projects."

Shows each strategy's slug, display name, status (in-progress / paused / done), % complete, last-modified. Read-only.

### `continue-strategy`

Resumes a strategy. Triggered by "continue strategy X," "resume strategy X," "what's next on strategy X."

Reads the strategy doc, finds the first unchecked step, executes or assists with it (autonomous or step-by-step per the strategy's settings), checks off the box on completion, appends to the Decisions log if relevant, reports the next pending step. Designed to be called repeatedly across many sessions over weeks.

### `connect-email`

The ONE email-deliverability setup flow. Triggered by "connect my email," "set up email sending," "set up Resend," "set up my email," "let Omni email my clients."

Walks you through: Resend account signup (if needed), adding your sending domain, the 3 DNS records at your registrar (Cloudflare, Namecheap, GoDaddy, Squarespace, Route 53 — provider-specific click-paths), domain verification, API key generation, sender identity (brand name + from-address + reply-to), and a test send. After this, prompts like "email this report to client@acme.com" route through your verified domain.

API key saves to `~/.claude/.omni-resend-key` — out of the synapse fork, never git-tracked. Sender identity saves to `custom/email/sender.md` in your fork (git-tracked, syncs across machines via `push my synapse changes`).

~5 minutes of active work plus 5-60 minutes of DNS propagation wait. Idempotent — safe to re-run to update the key or change the sender identity.

**Note:** "Email me X" (to yourself) works out of the box from the moment you complete getting-started — no email setup needed. The catch-all `omni@getomnipresence.com` channel handles that. `connect-email` is only needed for sending to anyone OTHER than yourself.

## The one rule

**Edit only in `custom/`, `overrides/`, or `custom/projects/`. Never edit `core/`.**

- `core/` is upstream-managed. The sync flow pulls updates there.
- `custom/` is yours — your own methodologies, processes, skills (net-new entries with unique slugs).
- `overrides/` is also yours — modified versions of `core/` files (same path = replacement).
- `custom/projects/<slug>/` is per-client info — brand voice, style guides, glossaries, strategies, project config.

Following this one rule means updates from upstream always merge cleanly. The skills enforce it too — `push-changes` will refuse to commit `core/` edits and offer to move them to `overrides/`.

## File-fetch priority — Claude Code reads local first, MCP serves canonical only

Two surfaces, two content models:

### Claude Code (these skills) — filesystem-first, 4 tiers + MCP fallback

Your local fork is the source of truth. Claude Code resolves methodology / processes / skills / project info from disk in strict priority order:

1. **Project files** — `custom/projects/<active>/...` for the currently active project. Per-client info: brand voice, style guide, glossary, banned phrases, strategies. The skills here (`switch-project`, `new-project`, `continue-strategy`) write the active slug to `~/.claude/skills/.omnipresence-active-project`.
2. **Custom** — `custom/methodologies/`, `custom/processes/`, `custom/skills/`. Your own additions with unique slugs.
3. **Overrides** — `overrides/methodologies/`, `overrides/processes/`, `overrides/skills/`. Path-mirrored replacements of core files (same path = replaces it).
4. **Core** — `core/...` from upstream `omnipresence-os/synapse`. The baseline.
5. **MCP (fallback only)** — used when local resolution fails (file not in any tier), for fuzzy keyword discovery across the canonical corpus, or to verify your fork isn't stale.

Local reads are immediate, network-free, and always reflect your most recent push. Same-relevance entries from a higher tier win — so `custom/methodologies/strategy/my-icp.md` beats the core `modern-icp.md` for queries that match both.

### claude.ai web, ChatGPT, mobile, any tool without a local fork — canonical Omni only

These surfaces hit the hosted Omnipresence MCP via the Omni connector. **The MCP serves canonical Omni only** — it does NOT see your fork's `custom/`, `overrides/`, or `custom/projects/`. There is no per-member MCP deployment; one canonical MCP serves all members.

What this means:

- **For canonical methodology:** every surface works (Claude Code, claude.ai, ChatGPT, etc.) — they all eventually read from the same canonical content.
- **For customization:** only Claude Code (or another IDE that can read your local fork) sees your customizations. Your brand voice files, your custom methodology, your project-specific strategies — these flow only through filesystem reads.

If you need customization on claude.ai web, the practical workaround is to paste relevant context into the chat. For real customization, do that work in Claude Code.

### Why this split

Per-member MCP deployments would solve the cross-surface customization problem but add per-member infrastructure cost. The simpler model: canonical MCP for canonical surfaces, filesystem for customization. Keeps the architecture lean and the canonical content trustworthy.

## Files in this repo

```
getting-started/SKILL.md      The first-time setup flow
sync-omnipresence/SKILL.md    The pull-updates flow
push-changes/SKILL.md         The save-and-push flow
new-project/SKILL.md          Stand up a new per-client project
list-projects/SKILL.md        Show all projects
switch-project/SKILL.md       Set the active project
project-info/SKILL.md         Surface project info (config, brand voice, etc.)
new-strategy/SKILL.md         Create a multi-week strategy under the active project
list-strategies/SKILL.md      Show strategies for the active project
continue-strategy/SKILL.md    Resume a strategy from the next unchecked step
LICENSE                       MIT (for these skills only)
README.md                     This file
```

Each skill is a single markdown file with frontmatter (the trigger description) and a workflow body. Claude Code reads them from `~/.claude/skills/` and triggers them based on the user's prompt matching the description.

## Troubleshooting

### "I can't see omnipresence-os/synapse on my account."

You haven't accepted the outside-collaborator invitation yet. Check your email for a GitHub invitation, click "Accept invitation," and re-run `getting started`. If the email isn't there, email **jonathan@getomnipresence.com** for a fresh invite.

### "Forking of private repos is disabled."

Jonathan needs to enable it in the omnipresence-os org settings. Email him and he'll fix it.

### "Something is wrong with my Omnipresence setup."

Paste exactly that into Claude Code. It'll diagnose the state and tell you what to do. If it can't figure it out, email **jonathan@getomnipresence.com** with the chat transcript.

## License

These three skill files are MIT-licensed — they're install instructions, not Omnipresence's methodology. The methodology itself (the content of `omnipresence-os/synapse`) is proprietary to active Omnipresence members and licensed separately. See `synapse/LICENSE` and `synapse/NOTICE.md` in the upstream repo for those terms.

## Maintained by

Jonathan Boshoff / Omnipresence. Issues / questions: **jonathan@getomnipresence.com**.
