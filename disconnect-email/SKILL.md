---
name: disconnect-email
description: Removes the user's Resend API key and sender identity from their machine, returning them to the catch-all-only email state. Trigger when the user says any of "disconnect my email", "remove my email setup", "delete my Resend config", "remove my Resend key", "unhook email sending", "tear down email setup". Removes ~/.claude/.omni-resend-key (the local API key) and custom/email/sender.md (the sender identity in the synapse fork). After running, "email me X" still works (that uses the catch-all omni@getomnipresence.com channel — different infrastructure), but third-party sends will route the user to re-run `connect my email`. Reminds the user to also revoke the API key in the Resend dashboard if they want full server-side cleanup. To rotate just the key, prefer re-running `connect-email` (idempotent — replaces the existing key) over this skill.
---

# Disconnect Email — Remove Resend Key + Sender Identity

This skill removes the user's member-owned Resend configuration from their machine. After running, the user can no longer send to third parties from their domain until they re-run `connect-email`. System emails to themselves (the catch-all) continue to work — they live on entirely separate infrastructure.

## How to talk to the user during this skill

**Critical UX rule:** plain English, no shell commands or file paths read aloud unless the user explicitly asks. This is a destructive action — be clear about what you're about to delete and confirm before doing it.

## What this skill does — execute these steps in order

### Step 0: Locate the synapse fork

Read the cached path from `~/.claude/skills/.omnipresence-path` (or `%USERPROFILE%\.claude\skills\.omnipresence-path` on Windows). If missing, search common locations: `~/Documents/omnipresence/synapse`, `~/synapse`, `~/dev/synapse`, `~/Code/synapse`.

If still not found, the user has no Omnipresence install at all on this machine. Tell them: *"No Omnipresence setup found on this machine — nothing to disconnect."* STOP.

### Step 1: Check what's actually set up

Silently check:
- `~/.claude/.omni-resend-key` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-resend-key` (Windows) — the API key
- `<synapse-fork>/custom/email/sender.md` — the sender identity

Three possible states:

- **Neither exists:** *"Nothing to disconnect — you haven't set up Resend on this machine. If you meant 'email me X' (the catch-all), that's a different system that doesn't need disconnecting."* STOP.

- **Only key exists (no sender.md):** Half-configured — probably a setup that didn't finish. Continue to Step 2 and clean up the key.

- **Only sender.md exists (no key):** Probably a fresh clone of the fork on a new machine where the key was never pasted. Continue to Step 2 and clean up the sender.md.

- **Both exist:** Normal disconnect. Continue.

### Step 2: Confirm with the user

Tell them exactly what's about to happen, then ask:

> *"This will remove:*
> *- Your Resend API key from your machine (`~/.claude/.omni-resend-key`)*
> *- Your sender identity from your synapse fork (`custom/email/sender.md`)*
>
> *After this, 'email me X' (the catch-all) still works, but third-party sends will need you to re-run `connect my email`.*
>
> *Note: this does NOT revoke the API key on Resend's side. To fully invalidate it, also revoke it at https://resend.com/api-keys. (Recommended if you're disconnecting because the key may have been exposed.)*
>
> *Proceed? (Yes / No)"*

Wait for an explicit Yes. On No, stop without changes.

### Step 3: Remove the key

Silently delete `~/.claude/.omni-resend-key` (or the Windows equivalent). If the file doesn't exist (per Step 1's check), skip without error.

### Step 4: Remove the sender identity

Silently delete `<synapse-fork>/custom/email/sender.md`. If the parent `custom/email/` directory is now empty, leave it (an empty directory in git is fine and the user may set up again later).

### Step 5: Report

> *"Disconnected. Both files removed:*
> *- API key (local-only, was never on our servers)*
> *- Sender identity (was in your synapse fork — run `push my synapse changes` to commit the removal to your GitHub fork)*
>
> *Reminder: revoke the key at https://resend.com/api-keys if you suspect it was exposed. To set up fresh, paste `connect my email`."*

### Stop here.

Do not propose other skills. Do not suggest re-running connect-email unless the user asks.

## What this skill MUST NOT do

- Do NOT touch any file outside the two paths above (`~/.claude/.omni-resend-key` and `<synapse-fork>/custom/email/sender.md`). In particular: do NOT touch other `~/.claude/` files, do NOT touch other `custom/` files, do NOT touch `core/` or `overrides/`.
- Do NOT attempt to revoke the API key on Resend's side. We don't have credentials to do that and shouldn't try. Tell the user to revoke it themselves at the Resend dashboard.
- Do NOT proceed without explicit user confirmation in Step 2.
- Do NOT show the API key (or any part of it) in any chat output, log, or written file.
- Do NOT skip the synapse-fork check in Step 0 — if there's no fork, there's nothing to clean up under `custom/`.
