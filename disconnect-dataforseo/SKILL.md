---
name: disconnect-dataforseo
description: Removes the user's DataForSEO API credentials from their machine. Trigger when the user says any of "disconnect dataforseo", "remove dataforseo credentials", "delete my dataforseo config", "unhook dataforseo", "remove SEO data credentials", "tear down dataforseo". Removes ~/.claude/.omni-dataforseo.json. After running, any skill that needs DataForSEO data will route the user to re-run `connect dataforseo`. Reminds the user to also reset their API password in the DataForSEO dashboard if they want full server-side revocation (the local removal doesn't invalidate the password on DataForSEO's side). To rotate just the password, prefer re-running `connect-dataforseo` (idempotent — replaces the existing credentials) over this skill.
---

# Disconnect DataForSEO — Remove Saved Credentials

This skill removes the user's DataForSEO credentials from their machine. After running, no Omnipresence skill can call DataForSEO until they re-run `connect-dataforseo`. Other API integrations (Resend, future skills) are untouched.

## How to talk to the user during this skill

**Critical UX rule:** plain English, no shell commands or file paths read aloud unless the user explicitly asks. This is a destructive action — be clear about what's being removed and confirm before doing it.

## What this skill does — execute these steps in order

### Step 1: Check what's actually set up

Silently check `~/.claude/.omni-dataforseo.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-dataforseo.json` (Windows).

- **If it doesn't exist:** *"Nothing to disconnect — DataForSEO isn't set up on this machine. If you meant a different integration, paste `connect <service>` to set one up, or `disconnect <service>` to remove one."* STOP.
- **If it exists:** continue.

### Step 2: Confirm with the user

Tell them exactly what's about to happen, then ask:

> *"This will remove your DataForSEO credentials from your machine (`~/.claude/.omni-dataforseo.json`).*
>
> *After this, any Omnipresence skill that needs DataForSEO data will route you to re-run `connect dataforseo`. Other API integrations (Resend, GitHub, etc.) are untouched.*
>
> *Note: this does NOT revoke the API password on DataForSEO's side. To fully invalidate it, also reset the API password at https://app.dataforseo.com/api-dashboard. Recommended if you're disconnecting because the password may have been exposed (e.g., in a shared chat transcript).*
>
> *Proceed? (Yes / No)"*

Wait for an explicit Yes. On No, stop without changes.

### Step 3: Remove the file

Silently delete `~/.claude/.omni-dataforseo.json` (or the Windows equivalent).

### Step 4: Report

> *"Disconnected. Credentials removed from your machine (they were never on our servers).*
>
> *Reminder: reset the API password at https://app.dataforseo.com/api-dashboard if you suspect it was exposed. To set up fresh, paste `connect dataforseo`."*

### Stop here.

Do not propose other skills. Do not suggest re-running connect-dataforseo unless the user asks.

## What this skill MUST NOT do

- Do NOT touch any file outside `~/.claude/.omni-dataforseo.json`. In particular: do NOT touch `~/.claude/.omni-resend-key`, do NOT touch any other `~/.claude/.omni-*` file, do NOT touch the synapse fork.
- Do NOT attempt to revoke the API password on DataForSEO's side. We don't have credentials to do that and shouldn't try. Tell the user to reset it themselves at the DataForSEO dashboard.
- Do NOT proceed without explicit user confirmation in Step 2.
- Do NOT show the API password (or any part of it) in any chat output, log, or written file.
