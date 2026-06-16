---
name: disconnect-openai
description: Removes the user's OpenAI API key from their machine. Trigger when the user says any of "disconnect openai", "remove openai credentials", "delete my openai key", "unhook openai", "remove openai api key", "tear down openai connection". Removes ~/.claude/.omni-openai.json. After running, any skill that needs OpenAI-direct access (e.g., dataforseo-ai-visibility with provider:openai) will route the user to re-run `connect openai`. DataForSEO-routed calls are unaffected (different credentials file). Reminds the user to also revoke the key at https://platform.openai.com/api-keys if they want full server-side cleanup. To rotate the key, prefer re-running `connect-openai` (idempotent — replaces the existing key) over this skill.
---

# Disconnect OpenAI — Remove Saved API Key

This skill removes the user's OpenAI API key from their machine. After running, no Omnipresence skill can use `provider: openai` until they re-run `connect-openai`. Other API integrations (DataForSEO, Resend, Google, Ahrefs, future skills) are untouched — they live in separate credential files.

## How to talk to the user during this skill

**Critical UX rule:** plain English, no shell commands or file paths read aloud unless the user explicitly asks. This is a destructive action — be clear about what's being removed and confirm before doing it.

## What this skill does — execute these steps in order

### Step 1: Check what's actually set up

Silently check `~/.claude/.omni-openai.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-openai.json` (Windows).

- **If it doesn't exist:** *"Nothing to disconnect — OpenAI isn't set up on this machine. If you meant a different integration, paste `connect <service>` to set one up, or `disconnect <service>` to remove one."* STOP.
- **If it exists:** continue.

### Step 2: Confirm with the user

Tell them exactly what's about to happen, then ask:

> *"This will remove your OpenAI API key from your machine (`~/.claude/.omni-openai.json`).*
>
> *After this, any Omnipresence skill that uses `provider: openai` will route you to re-run `connect openai`. DataForSEO-routed calls (the default for AI-visibility skills) are unaffected — those use separate credentials.*
>
> *Note: this does NOT revoke the key on OpenAI's side. To fully invalidate it, also delete the key at https://platform.openai.com/api-keys. Recommended if you're disconnecting because the key may have been exposed (e.g., in a shared chat transcript or screenshot).*
>
> *Proceed? (Yes / No)"*

Wait for an explicit Yes. On No, stop without changes.

### Step 3: Remove the file

Silently delete `~/.claude/.omni-openai.json` (or the Windows equivalent).

### Step 4: Report

> *"Disconnected. Key removed from your machine (it was never on our servers).*
>
> *Reminder: delete the key at https://platform.openai.com/api-keys if you suspect it was exposed. To set up fresh, paste `connect openai`."*

### Stop here.

Do not propose other skills. Do not suggest re-running connect-openai unless the user asks.

## What this skill MUST NOT do

- Do NOT touch any file outside `~/.claude/.omni-openai.json`. In particular: do NOT touch `~/.claude/.omni-dataforseo.json`, `~/.claude/.omni-resend-key`, `~/.claude/.omni-google-*.json`, `~/.claude/.omni-ahrefs.json`, or any other `~/.claude/.omni-*` file; do NOT touch the synapse fork.
- Do NOT attempt to revoke the API key on OpenAI's side. We don't have OAuth scope to do that and shouldn't try. Tell the user to delete it themselves at https://platform.openai.com/api-keys.
- Do NOT proceed without explicit user confirmation in Step 2.
- Do NOT show the API key (or any part of it) in any chat output, log, or written file.
