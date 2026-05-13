# Omnipresence Claude Code Skills

Three Claude Code skills that turn the Omnipresence member workflow into three plain-English prompts. Install once, then everything is prompt-driven — no GitHub UI, no git CLI, no clicking around.

## Install (one time, ~30 seconds)

Open [Claude Code](https://claude.com/code) in any folder. Paste this prompt and hit enter:

> Install the Omnipresence skills from github.com/omnipresence-os/claude-skills into my ~/.claude/skills directory, then run getting started.

Claude Code will:
1. Clone this repo into your local skills directory.
2. Trigger the `getting-started` skill, which forks `omnipresence-os/synapse` to your GitHub, clones it locally, and wires it up to receive future updates.

After that, you'll never have to think about installation again.

## The three prompts

Once installed, these are the only three prompts you ever need:

| What you want to do | Prompt |
|---|---|
| **First-time setup** (auto-runs during install) | `Run getting started.` |
| **Pull the latest methodology updates** | `Sync omnipresence.` |
| **Save your customizations to your private fork** | `Push my synapse changes.` |

That's it.

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

## The one rule

**Edit only in `custom/` and `overrides/`. Never edit `core/`.**

- `core/` is upstream-managed. The sync flow pulls updates there.
- `custom/` is yours — your own methodologies, processes, skills.
- `overrides/` is also yours — modified versions of `core/` files, same path. Synapse loads these first.

Following this one rule means updates from upstream always merge cleanly. The skills enforce the rule too — `push-changes` will refuse to commit `core/` edits and offer to move them to `overrides/`.

## Files in this repo

```
getting-started/SKILL.md      The first-time setup flow
sync-omnipresence/SKILL.md    The pull-updates flow
push-changes/SKILL.md         The save-and-push flow
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
