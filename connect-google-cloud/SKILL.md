---
name: connect-google-cloud
description: Walks the user through setting up Google Cloud credentials so Omnipresence can access their Google Search Console, Google Analytics 4, Drive, Docs, Sheets, and other Google APIs. Trigger when the user says any of "connect google", "connect google cloud", "set up GSC", "set up search console", "connect search console", "connect GA4", "connect google analytics", "connect drive", "connect google drive", "set up google", "wire up google APIs", "set up google service account", "I want omni to access my GSC / GA4 / Drive / Docs / Sheets". This is the ONE canonical Google Cloud credential setup. Two paths supported: service account (recommended when the user has admin rights on the resources — no expiry, no verification gauntlet) and OAuth (fallback for resources where the user is only a granted user, not an admin). Both can coexist. Key file handling is path-based — the user provides the file path to the downloaded JSON, the agent reads from disk and moves into place, key contents never enter the chat transcript. Saves credentials locally to ~/.claude/.omni-google-sa.json (service account) and/or ~/.claude/.omni-google-oauth.json (OAuth refresh token). End-to-end ~10 minutes for first-time SA setup; OAuth fallback adds ~5 minutes for the consent screen + OAuth Playground flow. Idempotent.
---

# Connect Google Cloud — One Service Account, All Google APIs

This skill sets up Google Cloud credentials so Omni can read your Google Search Console, Google Analytics, Drive folders, Docs, Sheets, and any other Google API on your behalf. Service-account auth — no consent screens, no OAuth verification, no weekly re-auth.

## How to talk to the user during this skill

**Critical UX rule:** do NOT show shell commands or paste credential contents in your replies. Run commands silently and explain in plain English. Show clickable URLs with `https://` prefix. Never echo any part of the service account JSON back to the user.

✅ Good: *"Open https://console.cloud.google.com/apis/library, search for 'Search Console API', click Enable. Tell me when done."*

❌ Bad: *"Paste your private_key into a file at..."* or *"Run `gcloud iam service-accounts create...`"*

Only show:
- Plain-English progress + clickable URLs.
- The service account's email address (this is not a secret — it's the value users paste into "Share with" dialogs throughout Google's UIs).

Never show: the JSON file contents, the `private_key` field, terminal commands.

## Two paths, both supported

**Default: service account.** When the user has admin rights on the resources Omni needs to access (GSC properties, GA4 properties, Drive folders), service account is the right path: no token expiry, no Google verification process, no weekly re-auth. One JSON key unlocks every API the SA gets shared with.

**Fallback: OAuth.** When the user is a granted user (but not admin) on a resource — e.g., a guest on a client's GSC property — service account can't be added by them. OAuth handles this by using the user's own identity as the auth principal. Same Omni capability, different credential type. Caveats are real but not deal-breakers: token refreshes every 7 days in Testing mode (re-run the OAuth flow), and unverified apps show a "this app isn't verified" warning during OAuth that the user clicks through.

The skill picks the path based on what the user actually has access to in Step 5. If they admin everything → SA. If they're a granted user on some resources → OAuth for those. Mixed? Both files coexist, downstream skills check each.

Neither path leaks key contents into the chat: SA uses path-based file handoff (Step 4), OAuth uses OAuth Playground + a refresh token paste (a refresh token is opaque and rotatable; it's a reasonable thing to paste).

## What this skill does — execute these steps in order

### Step 0: Check existing setup

Silently check `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows).

- **If it exists and is valid JSON with a `client_email` field**, the user already has a service account configured. Ask: *"Looks like you've already got a Google Cloud service account connected (the email is `<client_email>`). What do you want to do?"*
  - **Add access to a new resource** (GSC property, Drive folder, GA4 property): skip to Step 5 with the existing SA email.
  - **Enable a new Google API** (e.g., previously had GSC, now want Drive too): skip to Step 2 with the existing project.
  - **Rotate the key** (security concern): skip to Step 4 to generate a fresh key, then re-save.
  - **Full re-setup**: continue from Step 1.
- **If it doesn't exist**, continue to Step 1.

### Step 1: Pick or create a Google Cloud project

Ask in plain English:

> *"To set up Google Cloud credentials, you'll need a Google Cloud project. This is a container that holds the APIs and service accounts — it's free, doesn't bill anything, and you can name it whatever you want.*
>
> *Do you already have a Google Cloud project, or do you need to create one? (If you've never used Google Cloud, you'll need to create one.)"*

- **If they need to create one:**
  > *"Go to https://console.cloud.google.com/projectcreate and create a project. Name it 'Omnipresence' or similar. After it's created, you'll see the Project ID near the top (looks like `omnipresence-12345`). Copy that ID and tell me what it is — I'll use it to build the right links for the next steps."*
- **If they have one:**
  > *"Great. What's the Project ID? It's the kebab-case string near the top of the Google Cloud console (NOT the display name — the ID, which is usually all lowercase with hyphens and digits)."*

Wait for the project ID. Store it as `<project-id>`.

### Step 2: Enable the APIs the user actually needs

Ask:

> *"Which Google services do you want Omni to access? Pick all that apply:*
>
> *1. **Google Search Console** — search performance data (queries, impressions, clicks, rankings)*
> *2. **Google Analytics 4** — site traffic and conversion data*
> *3. **Google Drive** — files in your Drive*
> *4. **Google Docs** — Doc creation, reading, editing*
> *5. **Google Sheets** — Sheet creation, reading, editing*
> *6. **All of them** — recommended for full Omni capability"*

Wait for the answer. For each selected service, give a clickable enable URL (pre-filled with the project ID):

| Service | Enable URL pattern |
|---|---|
| Search Console | `https://console.cloud.google.com/apis/library/searchconsole.googleapis.com?project=<project-id>` |
| Analytics 4 | `https://console.cloud.google.com/apis/library/analyticsdata.googleapis.com?project=<project-id>` |
| Drive | `https://console.cloud.google.com/apis/library/drive.googleapis.com?project=<project-id>` |
| Docs | `https://console.cloud.google.com/apis/library/docs.googleapis.com?project=<project-id>` |
| Sheets | `https://console.cloud.google.com/apis/library/sheets.googleapis.com?project=<project-id>` |

Tell them: *"Open each link, click **Enable**, come back. Each one takes about 5 seconds. Tell me when all are enabled."*

Wait for confirmation. **STOP gate.**

### Step 3: Create the service account

> *"Now create a service account. Open https://console.cloud.google.com/iam-admin/serviceaccounts/create?project=<project-id> — that opens the create form for your project.*
>
> *Fill in:*
> *- **Name**: `Omnipresence Agent` (or whatever you like)*
> *- **ID**: auto-generates from the name — fine as-is*
> *- **Description**: 'Service account for Omnipresence to read GSC / GA4 / Drive on my behalf' (optional)*
>
> *Click **Create and continue**. Then for the "Grant this service account access to project" step, just click **Continue** without picking any role — we don't grant project-level access; we grant per-resource access in Step 5. Then click **Done**.*
>
> *Tell me when the service account is created."*

Wait for confirmation.

Then:

> *"On the service accounts list page, copy the email of the SA you just created. It looks like `omnipresence-agent@<project-id>.iam.gserviceaccount.com`. Paste it here so I can confirm the format."*

Wait for the email. Validate it ends with `.iam.gserviceaccount.com`. If not, ask them to re-copy from the GCP console. Store as `<sa-email>`.

### Step 4: Generate the JSON key (path-based — key contents stay off the chat)

> *"Now generate a JSON key for that service account. Open https://console.cloud.google.com/iam-admin/serviceaccounts?project=<project-id>, click the SA you just created, go to the **Keys** tab, click **Add Key → Create new key**, pick **JSON**, click **Create**.*
>
> *A JSON file will download automatically — usually to your Downloads folder. **Don't open it. Don't paste its contents.** It contains a private key, and pasting it into chat would leak the key into the conversation transcript.*
>
> *Instead, give me the FILE PATH to the downloaded file. The path is safe — it's just `C:\Users\You\Downloads\filename.json` (Windows) or `/Users/you/Downloads/filename.json` (Mac), no secrets in it.*
>
> *Easiest ways to get the path:*
> *- **Windows:** in File Explorer, find the file in Downloads, right-click → 'Copy as path'. Paste the result here. (It'll be wrapped in quotes — that's fine, I'll strip them.)*
> *- **Mac:** in Finder, find the file, right-click → hold Option → click 'Copy "<filename>" as Pathname'. Paste here.*
> *- **Linux:** the file is at `~/Downloads/<filename>.json` — type that.*
>
> *Paste the path here."*

Wait for the user to paste the path.

When they paste it:

1. Strip surrounding quotes (Windows "Copy as path" wraps in `"`).
2. **Read the file FROM DISK at that path** (do not ask the user to paste contents). Use whatever filesystem-read mechanism is available in the current agent harness — `Read` tool, `cat`, Node `fs.readFileSync`, etc.
3. Parse the JSON. Validate that it has: `type: "service_account"`, `project_id`, `private_key`, `client_email`, `client_id`.
4. If any field is missing or `type` is not `"service_account"`, tell them: *"That file doesn't look like a valid service account JSON. The download from the Keys tab should be a JSON file starting with `{` and including a `private_key` field. Make sure you gave me the path to the right file (the one Google just downloaded) and try again."* Don't proceed.
5. **MOVE the file** to `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows). Use a true move (rename if same volume, copy-then-delete otherwise) — the original file shouldn't sit in Downloads where the user might accidentally email it or commit it later.
6. On Mac/Linux, `chmod 600` the new file (owner-only read/write). Windows inherits user-profile permissions, which are owner-only by default.
7. Tell the user: *"Saved to `~/.claude/.omni-google-sa.json` and the original in your Downloads folder is gone. The key never entered this chat — it went straight from disk to disk. If you ever want to revoke Omni's access to all your Google resources at once, delete this service account in the Google Cloud console — that invalidates the key globally."*

**Fallback if filesystem read fails** (rare — the agent harness doesn't support file reads, or the path doesn't resolve): tell the user: *"I can't read the file at that path — either the path is wrong or my agent harness can't access local files. If the path looks right but it still fails, paste the path again with forward slashes, or as a last resort paste the file contents (acknowledged tradeoff: key will be in chat history; rotate the key after via the Keys tab in GCP)."* This branch ONLY applies if path-based reading genuinely doesn't work — don't suggest paste-contents as the default.

### Step 5: Grant the service account access to each resource

This is the per-service permission step. The SA email is the value the user pastes into Google's "Share with" / "Add user" dialogs throughout the Google Cloud UI.

**Important caveat before proceeding:** sharing a resource with the SA requires admin / owner permission on that resource. Specifically:

- **GSC property** — you need "Owner" role (or "Full" user role with permission-management rights)
- **GA4 property** — you need "Administrator" role
- **Drive folder / Doc / Sheet** — you need "Editor" or "Owner" sharing rights

If the user is themselves a non-admin user on a resource (e.g., a guest user on a client's GSC property), they CAN'T add the SA to that resource. That's not a failure mode — it's a different auth model: **OAuth (via the user's own identity) is the right path for resources where the user isn't an admin.** Same Omni capability, different credentials.

Ask the user upfront:

> *"For each resource you want Omni to access, are you the admin / owner — or just a granted user? Pick whichever applies:*
>
> *(a) I'm the admin on all the resources I want Omni to access. → Service account, the path we're on.*
> *(b) I'm a non-admin user on some / all resources (e.g., I have access to a client's GSC property but I'm not the owner). → OAuth, the user-identity path. Service account doesn't work without admin rights to add the SA.*
> *(c) Mixed — admin on some, not on others. → Set up both. SA handles the ones you admin; OAuth handles the rest. Both can coexist."*

Wait for the answer.

- **If (a):** continue below with the SA-grant steps. Skip Step 7.
- **If (b):** skip ahead to Step 7 (OAuth fallback). You can leave the SA setup we just did in place — it's harmless if unused, and useful later if you ever get admin rights.
- **If (c):** continue below with SA-grant steps for the resources they admin, THEN do Step 7 (OAuth) for the rest.

#### Service account permission grant — for resources the user admins

Surface the SA email prominently:

> *"This is the email you need to share each resource with: `<sa-email>`. Copy it now; you'll paste it into a few places.*
>
> *For each Google service you enabled and have admin rights on, here's how to grant access:"*

Then walk through ONLY the services they enabled in Step 2:

**Google Search Console:**
> *"For each GSC property Omni should access:*
> *1. Open https://search.google.com/search-console*
> *2. Pick the property*
> *3. Settings (gear icon) → Users and permissions → Add user*
> *4. Email: `<sa-email>` · Permission: **Full** (or **Restricted** if you want read-only)*
> *5. Click Add*
>
> *Repeat for any other property you want Omni to access. Tell me when done."*

**Google Analytics 4:**
> *"For each GA4 property Omni should access:*
> *1. Open https://analytics.google.com*
> *2. Pick the property*
> *3. Admin (gear icon) → Property Access Management → click the + → Add users*
> *4. Email: `<sa-email>` · Roles: **Viewer** (read-only) or **Editor**, your call*
> *5. Click Add*
>
> *Tell me when done."*

**Google Drive — pick a scoping approach first:**

> *"For Drive access, two options. Pick whichever fits your workflow:*
>
> *(a) **Catch-all 'Omnipresence' folder.** Create a single folder in My Drive called `Omnipresence` (or any name), share THAT one folder with the SA, then drop anything you want Omni to access inside it. Simplest to manage — one share, one URL to track. Best when starting fresh.*
>
> *(b) **Specific existing folders.** Share each folder you want Omni to access individually. More clicks upfront, but no reorganization needed if your existing Drive structure works.*
>
> *Which approach? (a/b)"*

Wait for the answer.

**Critical bit either way: Omni needs the folder URL(s).** Sharing alone isn't enough — Drive doesn't auto-tell the SA "here's what you can see now." Omni has to be told WHERE to look. Always capture the URL after sharing and save it (see below).

**If (a) — catch-all folder:**

> *"In https://drive.google.com, click **+ New → Folder**, name it `Omnipresence`, click Create.*
>
> *Right-click the new folder → **Share**. Email: `<sa-email>`. Role: **Editor** (recommended — lets Omni create new files inside; can downgrade to Viewer later if you want). Uncheck 'Notify people' (the SA has no inbox). Click **Share**.*
>
> *Then open the folder and copy its URL from the address bar — it looks like `https://drive.google.com/drive/folders/abc123XYZ...`. Paste that URL here so I can save it."*

**If (b) — specific folders:**

> *"For EACH folder you want Omni to access:*
> *1. Open https://drive.google.com*
> *2. Right-click the folder → **Share***
> *3. Email: `<sa-email>` · Role: **Viewer** / **Commenter** / **Editor** depending on what you want Omni to do*
> *4. Uncheck 'Notify people'. Click **Share**.*
> *5. Open the folder and copy its URL from the address bar. Paste that URL here (one per line if you're doing several at once).*
>
> *Tell me when you've shared + collected all the URLs you want set up."*

Wait for the URLs.

**For both (a) and (b): save the URLs to a persistent file** at `<synapse-fork>/custom/google/drive-access.md` so Omni knows which folders to query on future sessions (without making the user re-state every time).

File shape:

```yaml
---
shared_folders:
  - name: "Omnipresence Hub"
    url: "https://drive.google.com/drive/folders/abc123..."
    folder_id: "abc123..."  # extract from the URL path segment after /folders/
    purpose: "Catch-all for everything Omni needs"
    shared_role: "editor"
  - name: "Q3 Content Briefs"
    url: "https://drive.google.com/drive/folders/xyz789..."
    folder_id: "xyz789..."
    purpose: "Q3 content sprint"
    shared_role: "viewer"
---

# Google Drive folders shared with Omnipresence

This file lists Drive folders/files shared with the Omnipresence service
account (see ~/.claude/.omni-google-sa.json for the SA email). Omni reads
this file when it needs to know which folders to query.

To add more folders: share them with the SA at drive.google.com, then add
an entry to `shared_folders` above with the folder URL. Re-run
`connect google cloud` if you want guided help.
```

The `name` and `purpose` fields can come from the user (ask conversationally) or be auto-filled from the folder name in Drive's URL response if you have an active session. Don't block on getting them — `name: "Folder shared on YYYY-MM-DD"` is a fine default.

Tell the user: *"Saved to `custom/google/drive-access.md` in your synapse fork. Omni will look here automatically when you ask for anything in Drive. To add more folders later, you can either re-run `connect google cloud` for guided help or edit that file directly."*

**Google Docs / Sheets:**
> *"Same as Drive — open the Doc or Sheet, click Share, add the SA email with the role you want. Anything you've already shared a parent Drive folder with is auto-shared too (Google inherits permissions). For one-off Docs/Sheets outside your shared folders, paste the URL here and I'll add it to your drive-access.md so Omni knows about it.*
>
> *Tell me when done."*

Wait for confirmation on each.

### Step 6: Test the connection

Pick the simplest API the user has enabled and run a test call.

**If GSC is enabled** (most common):
- Use the [gsc-connection](https://github.com/omnipresence-os/synapse/blob/main/core/skills/data-access/gsc-connection/SKILL.md) skill from the synapse corpus. Pass the SA JSON from `~/.claude/.omni-google-sa.json` and the user's GSC property URL. Expect a successful `sites.get` response.

**If GA4 is enabled** but not GSC:
- Make a `https://analyticsdata.googleapis.com/v1beta/properties/<property-id>:runReport` call with a minimal date-range body. Expect 200 OK.

**If Drive is enabled** but not GSC or GA4:
- Make a `https://www.googleapis.com/drive/v3/files?pageSize=1` call. Expect 200 OK with at least one file in `files[]` (or zero files if nothing's been shared yet, which is also a valid success).

**If the test succeeds:** *"Connected. Service account `<sa-email>` can now read your `<service>` data. From now on, when you ask Omni for things like 'pull my top GSC queries for last 30 days' or 'show me yesterday's traffic from GA4', it'll use these credentials automatically."*

**If the test fails with `403 Permission denied`:** *"Permission denied — the service account needs to be added to that specific property/folder. Re-check Step 5 for the resource you tried to access; the SA email needs to be in the access list."*

**If the test fails with `401 Unauthorized`:** *"The credentials Resend rejected. Re-run this skill — the JSON might have been corrupted on paste."* (Suggest re-downloading the key from the Keys tab.)

**If the test fails with `400 / 404 with API-not-enabled error`:** *"That API isn't enabled in the GCP project yet. Re-do Step 2 for `<service>`."*

### Step 7: OAuth fallback (only if Step 5 selected (b) or (c))

This step sets up OAuth credentials for resources the user can't admin with a service account. After this, the user has TWO credential files: `~/.claude/.omni-google-sa.json` (for admin'd resources) and `~/.claude/.omni-google-oauth.json` (for resources they only have user access on). Downstream Omni skills check both.

Walk through it directly, no preamble — the user is here because SA doesn't fit, that's fine.

#### Step 7.1: Configure the OAuth consent screen

> *"Open https://console.cloud.google.com/apis/credentials/consent?project=<project-id>.*
>
> *Pick **External** for user type (the only option for personal Google accounts). Click **Create**.*
>
> *On the App information page, fill in:*
> *- **App name**: `Omnipresence` (or any name)*
> *- **User support email**: your own Google email*
> *- **Developer contact**: your own Google email*
>
> *Skip the App logo, App domain, etc. — they're for verification, which we're not pursuing. Click **Save and Continue**.*
>
> *On the Scopes page, click **Save and Continue** without adding anything (we'll request scopes at OAuth time).*
>
> *On the Test users page, click **+ Add Users**, type the Google email you'll be signing in with (the account that has access to the GSC / GA4 / Drive resources). Click **Save and Continue**.*
>
> *Tell me when you're back at the summary page."*

Wait for confirmation.

> *"One important note before we continue: the app is now in **Testing** mode. That means OAuth tokens generated against it expire after 7 days unless you submit the app for Google verification (a multi-week process). For personal use, just re-run this OAuth flow every ~7 days when the token expires. Acknowledged tradeoff vs service account — but it's what works when you don't have admin access."*

#### Step 7.2: Create the OAuth client ID

> *"Open https://console.cloud.google.com/apis/credentials?project=<project-id>. Click **+ Create Credentials** → **OAuth client ID**.*
>
> *Pick **Desktop app** for application type. Name it `Omnipresence OAuth Client`. Click **Create**.*
>
> *Google shows you a Client ID and Client Secret in a dialog. Click **Download JSON** — that downloads the client config file. Don't paste contents into chat — give me the FILE PATH like in Step 4 (right-click → Copy as path on Windows, Option-Copy as Pathname on Mac)."*

Wait for the path. Read the file from disk (same mechanism as Step 4). Validate it has `installed.client_id` and `installed.client_secret`. Extract those two values; keep them in agent memory for the next sub-step. Don't save anywhere yet.

#### Step 7.3: Mint the refresh token via OAuth Playground

OAuth Playground is the simplest clickable way to get a refresh token without writing any code or running gcloud locally.

> *"Open https://developers.google.com/oauthplayground/ in a new tab.*
>
> *1. Click the **gear icon** (top right) → check **Use your own OAuth credentials** → paste your Client ID and Client Secret into the fields (I'll show them in a moment if you want to copy them from here, but easier: open the JSON you just downloaded and copy from there). Close the gear panel.*
>
> *2. On the LEFT panel, scroll to find the API scopes for what you need. Check the boxes for:*"

Then list the scopes the user needs based on which APIs they enabled in Step 2:

| API enabled | Scope to check in OAuth Playground |
|---|---|
| Search Console | `https://www.googleapis.com/auth/webmasters.readonly` |
| Analytics 4 | `https://www.googleapis.com/auth/analytics.readonly` |
| Drive (read-only) | `https://www.googleapis.com/auth/drive.readonly` |
| Drive (read + write) | `https://www.googleapis.com/auth/drive` |
| Docs | `https://www.googleapis.com/auth/documents` |
| Sheets | `https://www.googleapis.com/auth/spreadsheets` |

> *"3. Click **Authorize APIs** (blue button under the scope list).*
>
> *4. A Google sign-in popup opens. Sign in with the Google account you added as a test user in Step 7.1. You may see a warning page (`This app isn't verified`) — click **Advanced** → **Go to Omnipresence (unsafe)**. That warning is normal for unverified apps in Testing mode; it just means Google hasn't reviewed it.*
>
> *5. Review the scopes Google shows you. Click **Continue** / **Allow**.*
>
> *6. You're redirected back to OAuth Playground with an Authorization code. Click **Exchange authorization code for tokens** (button on the left, under Step 2).*
>
> *7. On the right pane, you'll now see a **Refresh token** and an **Access token**. Copy the **Refresh token** value (long string, not the access token).*
>
> *8. Paste the refresh token here."*

Wait for the refresh token. Validate it looks like a refresh token (starts with `1//` typically, base64-ish, > 50 chars).

#### Step 7.4: Save the OAuth credentials

**Silently** write `~/.claude/.omni-google-oauth.json` with this shape:

```json
{
  "client_id": "<from step 7.2>",
  "client_secret": "<from step 7.2>",
  "refresh_token": "<from step 7.3>",
  "scopes": ["<list of scopes the user checked in 7.3>"]
}
```

On Mac/Linux, `chmod 600`.

Tell the user: *"Saved. Omni now has OAuth credentials at `~/.claude/.omni-google-oauth.json`. For any GSC property, GA4 property, or Drive folder you can access (whether you're admin or not), Omni can read it using these credentials. Reminder: this token will expire in ~7 days because the app is in Testing mode — when that happens, just re-run this OAuth flow (Step 7.3 + 7.4). You don't need to redo 7.1 or 7.2 unless you create a new GCP project."*

#### Step 7.5: Test the OAuth path

Run the same test call pattern as Step 6, but use the OAuth credentials instead of the SA. Expected: same success behavior.

If the test fails, the most common cause is the user not having access to the property they tested against. Different from the SA failure mode (which would be "SA not added"). OAuth failures are usually "user not on property" or "scope not granted at OAuth time."

### Stop here.

Do not propose other skills. Do not suggest additional setup. The user has working Google Cloud credentials (SA, OAuth, or both); they're done.

## What this skill MUST NOT do

- Do NOT save any credential file anywhere git-tracked (synapse fork, project files, shell config). SA lives ONLY in `~/.claude/.omni-google-sa.json`; OAuth lives ONLY in `~/.claude/.omni-google-oauth.json`.
- Do NOT ask the user to paste the service account JSON contents into chat. The path-based handoff in Step 4 is the canonical path. Only fall back to paste-contents if file reading literally doesn't work in the current agent harness — and explicitly warn before doing so.
- Do NOT echo the JSON contents, the `private_key` field, the OAuth `client_secret`, or any sensitive part of the credentials in any chat output, log, or written file. (Refresh tokens are exception-OK to display momentarily during Step 7.3 since the user just minted one and is about to paste it back — but don't store them in chat-visible state.)
- Do NOT proxy the credentials through Omnipresence's servers. They never leave the user's machine.
- Do NOT skip Step 5's admin/OAuth branching question. Service account creation + key download alone gets nothing useful if the user can't grant access to the resources.
- Do NOT treat OAuth as a worse option in the user-facing flow. It's a different tool for a different situation (no admin access). Surface the 7-day token expiry as a fact, not as a complaint.
- Do NOT skip the test in Step 6 (for SA) or Step 7.5 (for OAuth) unless the user explicitly says "I trust it." A failing test surfaces config issues before they're discovered mid-real-work.
- Do NOT proceed past Step 4 if the read-from-disk JSON fails the format validation.

## Why ~/.claude/.omni-google-*.json (not env vars or settings.json)

Same rationale as the other Omnipresence credentials — see `omnipresence-os/docs/credential-integrations.md` for the canonical reasoning. Tool-agnostic file location, simple to inspect/rotate, never in git, easy to enumerate with `ls ~/.claude/.omni-*`.

Two credential files, one or both may be present:

- `~/.claude/.omni-google-sa.json` — service account JSON. Unlocks every Google API the SA has been granted access to. Use when the user has admin rights on the target resources.
- `~/.claude/.omni-google-oauth.json` — OAuth client + refresh token. Unlocks every Google API the user has access to as a person. Use when the user is a granted user but not an admin.

Downstream Omni skills (gsc-connection, future ga4-connection, etc.) check both. SA takes precedence for resources the SA has been granted; OAuth fills the gap for the rest.
