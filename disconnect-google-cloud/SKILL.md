---
name: disconnect-google-cloud
description: Removes the user's Google Cloud credentials (service account JSON and/or OAuth refresh token) from their machine. Trigger when the user says any of "disconnect google", "disconnect google cloud", "remove google credentials", "delete google service account", "unhook google", "remove google cloud key", "tear down google connection", "remove GSC connection", "remove GA4 connection", "disconnect drive". Removes ~/.claude/.omni-google-sa.json and/or ~/.claude/.omni-google-oauth.json — after which Omni can no longer access any Google service (GSC, GA4, Drive, Docs, Sheets) until they re-run `connect google cloud`. Confirms with the user before deleting. Reminds them that local removal does NOT revoke the credential globally — to fully invalidate, they must also delete the service account key (or OAuth client) in the Google Cloud console. To rotate just the credential, re-run `connect google cloud` instead — it's idempotent.
---

# Disconnect Google Cloud — Remove Service Account + OAuth Credentials

This skill removes the user's Google Cloud credentials from their machine. After running, any Omni skill that needs Google APIs (GSC, GA4, Drive, Docs, Sheets) will route the user back to `connect google cloud`.

## How to talk to the user during this skill

**Critical UX rule:** plain English, no shell commands or file paths read aloud unless the user explicitly asks. This is a destructive action — be clear about what's being removed and confirm before doing it.

## What this skill does — execute these steps in order

### Step 1: Check what's actually set up

Silently check both credential files:

- **SA file:** `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows)
- **OAuth file:** `~/.claude/.omni-google-oauth.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-oauth.json` (Windows)

Possible states:

- **Neither exists:** *"Nothing to disconnect — Google Cloud isn't set up on this machine. If you meant a different integration, paste `connect <service>` or `disconnect <service>`."* STOP.
- **Only SA exists:** read `client_email` from the JSON for the confirmation message. Continue.
- **Only OAuth exists:** read `client_id` from the JSON for the confirmation message. Continue.
- **Both exist:** read both, mention both in the confirmation. Continue.

### Step 2: Confirm with the user

Tell them exactly what's about to happen, then ask. Tailor the message to what's actually installed:

**If SA only:**
> *"This will remove your Google Cloud service account key (`<client_email>`) from your machine (`~/.claude/.omni-google-sa.json`).*
>
> *After this, Omni can no longer access ANY Google service — Search Console, Analytics, Drive, Docs, Sheets — using the service account path until you re-run `connect google cloud`. Other API integrations (Resend, DataForSEO, etc.) are untouched.*
>
> *Important: removing this file does NOT revoke the key globally. The service account still exists in Google Cloud and the key is still valid. To fully invalidate it, delete the key (or the entire service account) at https://console.cloud.google.com/iam-admin/serviceaccounts.*
>
> *Proceed? (Yes / No)"*

**If OAuth only:**
> *"This will remove your Google Cloud OAuth credentials (client `<client_id>`) from your machine (`~/.claude/.omni-google-oauth.json`).*
>
> *After this, Omni can no longer access Google services using your user identity until you re-run `connect google cloud`. Other API integrations untouched.*
>
> *Important: removing this file does NOT invalidate the OAuth client or refresh token globally. To fully invalidate, go to https://myaccount.google.com/permissions and revoke the OAuth app, OR delete the OAuth client at https://console.cloud.google.com/apis/credentials.*
>
> *Proceed? (Yes / No)"*

**If both exist:**
> *"This will remove BOTH Google Cloud credential files from your machine:*
> *- Service account key (`<client_email>`) at `~/.claude/.omni-google-sa.json`*
> *- OAuth credentials (client `<client_id>`) at `~/.claude/.omni-google-oauth.json`*
>
> *After this, Omni can no longer access ANY Google service until you re-run `connect google cloud`. Other API integrations untouched.*
>
> *Important: removing these files does NOT revoke either credential globally. To fully invalidate:*
> *- SA key: delete at https://console.cloud.google.com/iam-admin/serviceaccounts*
> *- OAuth: revoke at https://myaccount.google.com/permissions and/or delete the OAuth client at https://console.cloud.google.com/apis/credentials*
>
> *Want to remove both, or just one? (Both / SA only / OAuth only / Cancel)"*

Wait for explicit confirmation. On Cancel / No, stop without changes.

### Step 3: Remove the file(s)

Silently delete whichever the user confirmed. Don't touch anything they didn't confirm.

### Step 4: Report

Tailor to what was deleted:

> *"Disconnected. Removed:*
> *- [the file(s) the user actually deleted]*
>
> *Reminder: the credentials are still valid on Google's side. If you suspect exposure, follow the revoke links above. To set up fresh, paste `connect google cloud`."*

### Stop here.

Do not propose other skills. Do not suggest re-running connect-google-cloud unless the user asks.

## What this skill MUST NOT do

- Do NOT touch any file outside `~/.claude/.omni-google-sa.json` and `~/.claude/.omni-google-oauth.json`. In particular: do NOT touch `~/.claude/.omni-resend-key`, `~/.claude/.omni-dataforseo.json`, or any other `~/.claude/.omni-*` file. Do NOT touch the synapse fork.
- Do NOT delete BOTH files if the user said "SA only" or "OAuth only" — respect the explicit scope.
- Do NOT attempt to revoke credentials on Google's side. We don't have the auth to do that and shouldn't try. Tell the user to do it themselves at the appropriate Google console.
- Do NOT proceed without explicit user confirmation in Step 2.
- Do NOT show any sensitive credential field (`private_key`, `client_secret`, `refresh_token`) in any chat output, log, or written file. The `client_email` and `client_id` fields ARE okay to surface (they're public-facing identifiers).
