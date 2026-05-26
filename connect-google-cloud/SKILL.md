---
name: connect-google-cloud
description: Walks the user through setting up Google Cloud credentials so Omnipresence can access their Google Search Console, Google Analytics 4, Drive, Docs, Sheets, and other Google APIs. Trigger when the user says any of "connect google", "connect google cloud", "set up GSC", "set up search console", "connect search console", "connect GA4", "connect google analytics", "connect drive", "connect google drive", "set up google", "wire up google APIs", "set up google service account", "I want omni to access my GSC / GA4 / Drive / Docs / Sheets". This is the ONE canonical Google Cloud credential setup. Two paths: service account (recommended for resources the user admins) and OAuth (fallback for granted-user-only resources). The flow is incremental — set up ONE integration, test it live, then move on — so users get a fast feedback loop instead of finding setup is broken after a 20-minute click marathon. OAuth path uses a local-server flow (browser opens, user signs in, refresh token captured automatically) — NOT the OAuth Playground, which testing showed members find overwhelming. Key file handling is path-based — user provides the file path to the downloaded JSON, agent reads from disk and moves into place, key contents never enter the chat transcript. Saves credentials locally to ~/.claude/.omni-google-sa.json and/or ~/.claude/.omni-google-oauth.json. Idempotent — safe to re-run to add more integrations, refresh an expired OAuth token, or rotate credentials.
---

# Connect Google Cloud — Step-by-Step, One Integration At a Time

This skill connects Omnipresence to your Google Cloud account so it can read GSC / GA4 / Drive / Docs / Sheets on your behalf. Two credential types supported (service account for resources you own, OAuth for resources you only have user access to). The skill walks through ONE integration at a time and tests it live before moving on — so you confirm each piece works before doing the next.

## How to talk to the user during this skill

**Critical UX rules:**
- Do NOT show shell commands or paste credential contents in your replies. Run commands silently and explain in plain English.
- Show clickable URLs with `https://` prefix. Never echo any part of the service account JSON or OAuth secrets back to the user.
- After each integration finishes setup, **always run a live API call** to prove it works before asking what's next. Don't claim success without evidence.
- Surface the SA email address (it's not a secret — it's the value users paste into "Share with" dialogs throughout Google's UIs).
- Never show: the SA JSON file contents, the `private_key` field, the OAuth `client_secret`, terminal commands as user instructions.

## Two paths, both supported

**Default: service account.** When the user has admin / owner rights on the resources Omni needs to access (their own Drive, their own GSC, their own GA4), service account is the right path: no token expiry, no Google verification process, no weekly re-auth. One JSON key unlocks every API the SA gets shared with.

**Fallback: OAuth.** When the user is a granted user (not admin) on a resource — typical for client GSC properties they've been given access to — service account can't be added by them. OAuth handles this by using the user's own Google identity. Caveats: token refreshes every 7 days in Testing mode (re-run the local OAuth flow), and unverified apps show a "this app isn't verified" warning that the user clicks through. Both are minor; not deal-breakers.

Neither path leaks key contents into the chat: SA uses path-based file handoff, OAuth uses a local server flow (no contents to paste).

## Workflow

### Step 0: Check existing setup

Silently check both credential files:
- `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows)
- `~/.claude/.omni-google-oauth.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-oauth.json` (Windows)

Branch on what exists:

- **Neither exists:** new setup. Continue to Step 1.
- **SA only:** ask: *"You've already got a Google Cloud service account connected (the email is `<client_email>`). What do you want to do? — (a) add a new integration to the existing setup, (b) set up OAuth for resources you don't admin, (c) test the existing connection, (d) full re-setup, (e) something else."* Branch accordingly: (a) jump to Step 5, (b) jump to Step 8, (c) run a connectivity check on whatever the SA was set up for, (d) continue to Step 1 (warn that this will replace the existing files), (e) wait for clarification.
- **OAuth only:** similar — *"You've got OAuth credentials but no service account. What do you want to do? — (a) add service account for resources you admin, (b) refresh the OAuth token (it expires every 7 days in Testing mode), (c) test the existing connection, (d) full re-setup."* Branch accordingly.
- **Both exist:** offer all four options including refresh-OAuth.

### Step 1: Pick or create a Google Cloud project

Ask:

> *"To set up Google Cloud credentials, you'll need a Google Cloud project. This is a free container that holds the APIs and service accounts — it doesn't bill anything unless you actually use the APIs, and the free tiers are generous.*
>
> *Do you already have a Google Cloud project, or do you need to create one?"*

- **If they need to create:** *"Go to https://console.cloud.google.com/projectcreate and create a project. Name it 'Omnipresence' or similar. After it's created, you'll see the Project ID near the top (kebab-case, looks like `omnipresence-12345`). Copy that ID and paste it here — I'll use it to build the rest of the links."*
- **If they have one:** *"Great. Paste the Project ID here — it's the kebab-case string near the top of the Google Cloud console (NOT the display name)."*

Wait for the project ID. Validate it looks reasonable (lowercase, hyphens, digits). Store as `<project-id>`.

### Step 2: Enable the Google APIs

Default to enabling all five (Search Console, Analytics 4, Drive, Docs, Sheets). Tell the user:

> *"I'll have you enable all 5 Google APIs we use: Search Console, Analytics 4, Drive, Docs, Sheets. None of them charge unless you actually use them, and you can opt-out of any later in the GCP console. Open each link, click **Enable**, come back. Each takes ~5 seconds:*
>
> *- Search Console: https://console.cloud.google.com/apis/library/searchconsole.googleapis.com?project=<project-id>*
> *- Analytics 4: https://console.cloud.google.com/apis/library/analyticsdata.googleapis.com?project=<project-id>*
> *- Drive: https://console.cloud.google.com/apis/library/drive.googleapis.com?project=<project-id>*
> *- Docs: https://console.cloud.google.com/apis/library/docs.googleapis.com?project=<project-id>*
> *- Sheets: https://console.cloud.google.com/apis/library/sheets.googleapis.com?project=<project-id>*
>
> *Tell me when all are enabled. If you only want a subset, say so — but enabling all isn't a commitment, just turns on the APIs."*

Wait for confirmation. **STOP gate.**

### Step 3: Create the service account

> *"Now create the service account. Open https://console.cloud.google.com/iam-admin/serviceaccounts/create?project=<project-id>.*
>
> *Fill in:*
> *- **Name**: `Omnipresence Agent` (or whatever you like — auto-generates the ID)*
> *- **Description**: `Service account for Omnipresence to read GSC / GA4 / Drive on my behalf` (optional)*
>
> *Click **Create and continue**. On the "Grant this service account access to project" step, just click **Continue** without picking any role — we grant access per-resource, not at the project level. Click **Done**.*
>
> *Once the service account is created, copy its email from the SA list page (looks like `omnipresence-agent@<project-id>.iam.gserviceaccount.com`) and paste it here so I can confirm the format."*

Wait for the email. Validate it ends with `.iam.gserviceaccount.com`. If not, ask them to re-copy from the GCP console. Store as `<sa-email>`.

### Step 4: Generate and save the JSON key (path-based handoff)

> *"Now generate a JSON key. Open https://console.cloud.google.com/iam-admin/serviceaccounts?project=<project-id>, click the SA you just created, go to the **Keys** tab, click **Add Key → Create new key**, pick **JSON**, click **Create**.*
>
> *A JSON file will download automatically. **Don't open it. Don't paste its contents.** It contains a private key, and pasting it would leak the key into the chat transcript.*
>
> *Instead, give me the FILE PATH to the downloaded file (the path is non-secret):*
> *- **Windows:** in File Explorer, find the file in Downloads, right-click → **Copy as path**. Paste here.*
> *- **Mac:** in Finder, right-click → hold Option → **Copy "<filename>" as Pathname**. Paste here.*
> *- **Linux:** the file's at `~/Downloads/<filename>.json` — type that.*"

Wait for the path. Strip surrounding quotes. **Read the file from disk** (Read tool / Python / bash — whatever the harness supports). Validate it parses as JSON and has `type: "service_account"`, `project_id`, `private_key`, `client_email`, `client_id`.

If validation fails, tell the user: *"That file doesn't look like a valid service account JSON. Re-download from the Keys tab and try again."* Don't proceed.

**Move the file** to `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows). True move (rename or copy-then-delete) — the original shouldn't sit in Downloads. On Mac/Linux, `chmod 600`.

Tell the user: *"Saved to `~/.claude/.omni-google-sa.json`. The original in Downloads is gone. Key never entered this chat — it went disk-to-disk. To revoke this key globally later, delete the SA in the GCP console."*

### Step 5: Pick your first integration to wire up and test

Now we wire ONE integration at a time, testing each before moving on. This way you confirm the setup actually works after each step instead of stacking unknowns.

Ask the user, in a single AskUserQuestion (4 options max):

| Option | When to pick |
|---|---|
| Drive (recommended) | Most common starting point. Quick to wire up; gives you Doc/Sheet creation immediately. |
| Search Console | You want Omni to read your GSC properties. Requires owner rights for SA path; otherwise OAuth (Step 8). |
| Analytics 4 | You want Omni to read GA4 data. Requires Administrator role for SA path; otherwise OAuth. |
| Skip — set up OAuth instead | You don't admin any of these resources; jump straight to OAuth (Step 8). |

Branch to Step 6 with the chosen integration.

### Step 6: Wire up and test the chosen integration

#### Step 6 — Drive

**Sharing approach choice (Drive only):**

| Option | When to pick |
|---|---|
| Catch-all `Omnipresence` folder | Simplest. Create one folder, share it, drop everything inside. |
| Specific existing folders | Share folders you've already organized. |

**If catch-all:**

> *"In https://drive.google.com, click **+ New → Folder**, name it `Omnipresence`, click Create. Right-click the folder → **Share**. Email: `<sa-email>`. Role: **Editor**. Uncheck 'Notify people'. Click **Share**.*
>
> *Then open the folder, copy its URL from the address bar (looks like `https://drive.google.com/drive/folders/abc123XYZ...`), paste it here."*

**If specific folders:**

> *"For EACH folder you want Omni to access:*
> *1. https://drive.google.com → right-click folder → **Share***
> *2. Email: `<sa-email>` · Role: **Viewer** / **Commenter** / **Editor** as desired*
> *3. Uncheck 'Notify people'. Click **Share**.*
> *4. Open the folder, copy URL, paste here. One per line if multiple. **Note:** if it's a subfolder (not a root folder), say so when you paste — disambiguates the share scope."*

Wait for URL(s). For each, extract the `folder_id` from the URL.

**Save to drive-access.md** in the synapse fork. Find the fork at `~/.claude/skills/.omnipresence-path` (or search common locations). Write `<fork>/custom/google/drive-access.md`:

```yaml
---
shared_folders:
  - name: "<folder name if known, else 'Folder shared on YYYY-MM-DD'>"
    url: "<URL>"
    folder_id: "<id>"
    purpose: "<optional, ask later if needed>"
    shared_role: "<editor/viewer/commenter>"
    notes: "<optional, e.g. 'Subfolder, not root'>"
---

# Google Drive folders shared with Omnipresence

This file lists Drive folders/files shared with the Omnipresence service
account (see ~/.claude/.omni-google-sa.json for the SA email). Omni reads
this file when it needs to know which folders to query.

To add more folders: share them with the SA at drive.google.com, then add
an entry to `shared_folders` above with the folder URL. Re-run
`connect google cloud` if you want guided help.
```

**Now test the Drive connection live.** Three checks in sequence:

1. **Read test** — list contents of the shared folder. Use the Drive API directly via Python (install `google-api-python-client` + `google-auth` if needed, silently). If 0 items found, that's still a pass (folder is empty). If 403, tell the user the SA wasn't added correctly + how to fix.
2. **Create test** — create a Google Doc called `Hello World` in the folder with body text `Hello, world! This document was created by the Omnipresence service account on YYYY-MM-DD.`
3. **Surgical edit test** — do a `replaceAllText` on the Doc, changing `document` → `doc` in the body.

After all three pass, surface:

> *"Drive connection live. Confirmed:*
> *- Read: SA can list folder contents (`<N>` items found)*
> *- Write: created test Doc at https://docs.google.com/document/d/<id>/edit*
> *- Surgical edit: 1 occurrence replaced (document → doc)*
>
> *The test Doc is in your shared folder — you can keep it as proof or delete it. From now on, when you ask Omni to write to Drive or edit a Doc, this is the underlying mechanism.*
>
> *Want to wire up another integration, set up OAuth for resources you don't admin, or are you done for now?"*

Go to Step 7 for the loop.

#### Step 6 — Search Console (SA path)

Ask: *"Are you the **owner** of the GSC properties Omni should access? (Owner role in GSC is required to add the SA; lesser roles can't.)"*

- **Yes:**
  > *"For each GSC property:*
  > *1. https://search.google.com/search-console → pick the property*
  > *2. Settings (gear) → Users and permissions → Add user*
  > *3. Email: `<sa-email>` · Permission: **Full** (or **Restricted** for read-only)*
  > *4. Click Add*
  >
  > *Tell me when done."*

  Test: list GSC sites via the Search Console API. If the SA email shows up under at least one site → pass. Report the property list. Continue to Step 7.

- **No:** *"Service account can only be added by an owner. We'll use OAuth instead — same Omni capability, different auth. Skipping ahead to Step 8."* Jump to Step 8.

#### Step 6 — Analytics 4 (SA path)

Ask: *"Are you an **Administrator** of the GA4 properties Omni should access? (Lower roles can't add new users.)"*

- **Yes:**
  > *"For each GA4 property:*
  > *1. https://analytics.google.com → pick the property*
  > *2. Admin (gear) → Property Access Management → +**Add users***
  > *3. Email: `<sa-email>` · Role: **Viewer** (read-only) or **Editor**, your call*
  > *4. Click Add*
  >
  > *Tell me when done."*

  Test: call `runReport` against the property with a minimal request to confirm access. Report the property name + total events for the last 7 days. Continue to Step 7.

- **No:** *"Same as GSC — admin's required for SA. Skipping ahead to OAuth in Step 8."*

### Step 7: What's next? (Loop point)

After every integration test, ask:

| Option | Action |
|---|---|
| Add another integration | Go back to Step 5 — pick a different service to wire up. |
| Set up OAuth for non-admin resources | Continue to Step 8. |
| Done for now | Wrap up. Surface what's currently set up. |

If they pick "done", surface a summary:

> *"You're set up with:*
> *- Service account: `<sa-email>` for <list of services with admin grants>*
> *- OAuth: <list of OAuth scopes, if any>*
>
> *Going forward, when you ask Omni for anything in those services, it'll use these credentials automatically. To set up more later, just run `connect google cloud` again — it's idempotent."*

### Step 8: OAuth setup (for resources the user doesn't admin)

OAuth uses the user's own Google identity as the auth principal — for resources where they're a granted user but not the owner. No need to share the SA with anything; OAuth lets Omni "act as" the user.

We use a **local server flow** (NOT the OAuth Playground). The browser opens to Google's sign-in, user approves, the local script catches the redirect and captures the refresh token automatically. Tested to be dramatically cleaner UX than OAuth Playground.

#### Step 8.1: Configure the OAuth consent screen

> *"Open https://console.cloud.google.com/apis/credentials/consent?project=<project-id>.*
>
> *Pick **External** for user type (only option for personal Google accounts). Click **Create**.*
>
> *On the App information page, fill in:*
> *- **App name**: `Omnipresence` (or any name)*
> *- **User support email**: your own Google email*
> *- **Developer contact**: your own Google email*
>
> *Skip App logo, App domain, etc. — those are for Google's verification process which we're not pursuing.*
>
> *Click **Save and Continue**. On the Scopes page, click **Save and Continue** without adding anything. On Test users, click **+ Add Users**, type the Google email you'll sign in with, click **Save and Continue**.*
>
> *Note: no authorized redirect URIs needed — we use a Desktop app client which auto-allows localhost.*
>
> *Tell me when you're back at the summary page."*

Wait for confirmation. Mention the 7-day Testing-mode expiry once, matter-of-fact:

> *"The app is in **Testing** mode. OAuth tokens expire after 7 days unless you submit for Google verification (multi-week). For personal use, just re-run `connect google cloud` when it expires — it'll detect the existing OAuth file and let you refresh just the token (no need to redo this step or the consent screen)."*

#### Step 8.2: Create the OAuth client ID (Desktop app)

> *"Open https://console.cloud.google.com/apis/credentials?project=<project-id>. Click **+ Create Credentials → OAuth client ID**.*
>
> *Application type: **Desktop app** (important — Web app doesn't work with the local-server flow we'll use next).*
>
> *Name: `Omnipresence OAuth Client`. Click **Create**.*
>
> *Google shows a dialog with Client ID + Secret. Click **Download JSON**. Paste the FILE PATH here (right-click → Copy as path on Windows, etc. — same as Step 4)."*

Wait for path. Read file from disk. Validate it's a Desktop client (has `installed` key, not `web`). If it's a Web client, tell the user: *"That's a Web application client. The local-server flow needs a Desktop app client. Either: (a) create a new Desktop client in the GCP console and re-paste, or (b) we can fall back to OAuth Playground for the Web client — but Desktop + local server is the cleaner experience."*

Extract `client_id` and `client_secret`. Hold in agent memory; don't save yet.

#### Step 8.3: Mint the refresh token (local-server flow)

Ensure `google-auth-oauthlib` is installed (pip install if missing). Run a script that:

1. Calls `InstalledAppFlow.from_client_secrets_file(<client-json-path>, scopes=[<scopes>])`.
2. Calls `flow.run_local_server(port=0, prompt='consent', access_type='offline')`.
3. The script prints the auth URL — Claude Code may also auto-open the browser; either way the user clicks the URL and is taken to Google sign-in.
4. User signs in with the Google account that has access to the target resources. Clicks through the "this app isn't verified" warning (Advanced → Go to Omnipresence (unsafe) — normal for Testing mode).
5. Reviews scopes, clicks Continue / Allow.
6. Browser redirects to localhost; the script captures the auth code, exchanges it for tokens, returns credentials.
7. The script extracts `creds.refresh_token` and writes it to `~/.claude/.omni-google-oauth.json`:

```json
{
  "client_id": "<from client JSON>",
  "client_secret": "<from client JSON>",
  "refresh_token": "<minted>",
  "scopes": ["<scope-1>", "<scope-2>", ...],
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

On Mac/Linux, `chmod 600`.

**Picking scopes:** ask the user upfront what they want OAuth for, then mint the minimum scopes:

| Service | Scope |
|---|---|
| GSC (read) | `https://www.googleapis.com/auth/webmasters.readonly` |
| GSC (write — sitemap submission etc.) | `https://www.googleapis.com/auth/webmasters` |
| GA4 (read) | `https://www.googleapis.com/auth/analytics.readonly` |
| Drive (read) | `https://www.googleapis.com/auth/drive.readonly` |
| Drive (write) | `https://www.googleapis.com/auth/drive` |
| Docs | `https://www.googleapis.com/auth/documents` |
| Sheets | `https://www.googleapis.com/auth/spreadsheets` |

If they want multiple, pass all scopes to the flow — single consent covers all.

**Fallback to OAuth Playground:** if the local-server flow fails for any reason (corporate firewall blocks localhost binding, Python deps refuse to install, etc.), tell the user: *"Local-server flow didn't work — falling back to OAuth Playground. This requires one more setup step: open the OAuth consent screen, scroll to **Authorized redirect URIs**, click **+ Add URI**, paste `https://developers.google.com/oauthplayground`, click Save. Then I'll walk through Playground."* Then run the Playground flow: open https://developers.google.com/oauthplayground/, gear icon → Use your own OAuth credentials → paste client ID + secret, pick scopes on the left, Authorize APIs, sign in, exchange code for tokens, copy the refresh token, paste it back to chat. The agent saves the same JSON shape as the local-server flow.

#### Step 8.4: Test the OAuth connection

For the service(s) OAuth was set up for, run a test call using the OAuth credentials:

- **GSC:** list sites. Report which properties OAuth can access (with their permission levels — `siteOwner` / `siteFullUser` / `siteRestrictedUser`).
- **GA4:** list account summaries. Report which properties OAuth can access.
- **Drive:** list files. Report file count and sample names.

Surface the result + the 7-day expiry reminder.

### Step 9: Loop or wrap

After OAuth setup + test, go back to Step 7 (What's next?). User can add more SA grants, more OAuth scopes, or finish.

### Stop here.

When the user picks "done", stop. Do not propose other skills. Do not suggest additional setup.

## What this skill MUST NOT do

- Do NOT save any credential file anywhere git-tracked (synapse fork's tracked content, project files, shell config). SA lives ONLY in `~/.claude/.omni-google-sa.json`; OAuth lives ONLY in `~/.claude/.omni-google-oauth.json`.
- Do NOT ask the user to paste the service account JSON contents into chat. The path-based handoff is the canonical path. Only fall back to paste-contents if file reading literally doesn't work in the current agent harness — and explicitly warn before doing so.
- Do NOT default to the OAuth Playground flow. User testing showed members find it overwhelming. Local-server flow is the default; Playground is the fallback when local flow doesn't work.
- Do NOT echo any sensitive credential field (`private_key`, `client_secret`, `refresh_token`) in any chat output, log, or written file.
- Do NOT proxy the credentials through Omnipresence's servers. They never leave the user's machine.
- Do NOT skip the live test after each integration. Setup-then-test-everything-at-the-end was the previous flow and produced bad UX (member completes 20 min of clicks before discovering something's wrong). The new flow tests after every integration so failures surface immediately.
- Do NOT treat OAuth as a worse option in user-facing language. It's a different tool for a different situation (no admin access). Surface the 7-day token expiry as a fact, not a complaint.
- Do NOT proceed past Step 4 if the read-from-disk JSON fails format validation.
- Do NOT show authorized redirect URI config in Step 8.1 — the local-server flow doesn't need it. Only mention it as part of the Playground fallback in Step 8.3.

## Why ~/.claude/.omni-google-*.json (not env vars or settings.json)

Same rationale as the other Omnipresence credentials — see `omnipresence-os/docs/credential-integrations.md` for the canonical reasoning. Tool-agnostic file location, simple to inspect / rotate, never in git, easy to enumerate with `ls ~/.claude/.omni-*`.

Two credential files, one or both may be present:
- `~/.claude/.omni-google-sa.json` — service account JSON. Used for resources the user admins.
- `~/.claude/.omni-google-oauth.json` — OAuth client + refresh token. Used for resources the user is a granted user on.

Downstream Omni skills (`gsc-connection`, `drive-list`, etc.) check both. SA takes precedence for resources the SA has been granted; OAuth fills the gap.
