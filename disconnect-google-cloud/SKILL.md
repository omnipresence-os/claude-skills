---
name: disconnect-google-cloud
description: Removes the user's Google Cloud service account JSON from their machine. Trigger when the user says any of "disconnect google", "disconnect google cloud", "remove google credentials", "delete google service account", "unhook google", "remove google cloud key", "tear down google connection", "remove GSC connection", "remove GA4 connection", "disconnect drive". Removes ~/.claude/.omni-google-sa.json — Omni can no longer access ANY Google service (GSC, GA4, Drive, Docs, Sheets) until they re-run `connect google cloud`. Reminds the user that local removal does NOT revoke the key globally — to fully invalidate, they must also delete the service account or rotate the key in the Google Cloud console. To rotate just the key (most common need), re-run `connect google cloud` instead — it's idempotent.
---

# Disconnect Google Cloud — Remove the Service Account Key

This skill removes the user's Google Cloud service account JSON from their machine. After running, any Omni skill that needs Google APIs (GSC, GA4, Drive, Docs, Sheets) will route the user back to `connect google cloud`.

## How to talk to the user during this skill

**Critical UX rule:** plain English, no shell commands or file paths read aloud unless the user explicitly asks. This is a destructive action — be clear about what's being removed and confirm before doing it.

## What this skill does — execute these steps in order

### Step 1: Check what's actually set up

Silently check `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows).

- **If it doesn't exist:** *"Nothing to disconnect — Google Cloud isn't set up on this machine. If you meant a different integration, paste `connect <service>` or `disconnect <service>`."* STOP.
- **If it exists:** read the `client_email` field so we can mention it in the confirmation. Continue.

### Step 2: Confirm with the user

Tell them exactly what's about to happen, then ask:

> *"This will remove your Google Cloud service account key (`<client_email>`) from your machine (`~/.claude/.omni-google-sa.json`).*
>
> *After this, Omni can no longer access ANY Google service — Search Console, Analytics, Drive, Docs, Sheets — until you re-run `connect google cloud`. Other API integrations (Resend, DataForSEO, etc.) are untouched.*
>
> *Important: removing this file does NOT revoke the key globally. The service account still exists in Google Cloud and the key is still valid. To fully invalidate it, either:*
> *- Delete the key from https://console.cloud.google.com/iam-admin/serviceaccounts (recommended if the key may have been exposed), OR*
> *- Delete the entire service account (also revokes ALL keys at once)*
>
> *Proceed? (Yes / No)"*

Wait for an explicit Yes. On No, stop without changes.

### Step 3: Remove the file

Silently delete `~/.claude/.omni-google-sa.json` (or the Windows equivalent).

### Step 4: Report

> *"Disconnected. Service account key removed from your machine (it was never on our servers).*
>
> *Reminder: the key is still valid on Google's side. If you suspect it was exposed, go to https://console.cloud.google.com/iam-admin/serviceaccounts, click your `<client_email>` service account, go to the Keys tab, and delete the key (or delete the entire service account to revoke everything at once).*
>
> *To set up fresh, paste `connect google cloud`."*

### Stop here.

Do not propose other skills. Do not suggest re-running connect-google-cloud unless the user asks.

## What this skill MUST NOT do

- Do NOT touch any file outside `~/.claude/.omni-google-sa.json`. In particular: do NOT touch `~/.claude/.omni-resend-key`, `~/.claude/.omni-dataforseo.json`, or any other `~/.claude/.omni-*` file. Do NOT touch the synapse fork.
- Do NOT attempt to revoke the key on Google's side. We don't have credentials to do that and shouldn't try. Tell the user to do it themselves at the Google Cloud console.
- Do NOT proceed without explicit user confirmation in Step 2.
- Do NOT show any part of the service account JSON (especially `private_key`) in any chat output, log, or written file. The `client_email` field IS okay to surface (it's the public-facing identifier).
