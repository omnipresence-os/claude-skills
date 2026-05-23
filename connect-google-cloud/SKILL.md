---
name: connect-google-cloud
description: Walks the user through setting up a Google Cloud service account so Omnipresence can access their Google Search Console, Google Analytics 4, Drive, Docs, Sheets, and other Google APIs — without OAuth's verification gauntlet or weekly re-consent. Trigger when the user says any of "connect google", "connect google cloud", "set up GSC", "set up search console", "connect search console", "connect GA4", "connect google analytics", "connect drive", "connect google drive", "set up google", "wire up google APIs", "set up google service account", "I want omni to access my GSC / GA4 / Drive / Docs / Sheets". This is the ONE canonical Google Cloud credential setup. Recommends service account auth (no expiry, no verification gauntlet, per-resource permissions); explains why OAuth is harder but accommodates users who explicitly insist. Saves the service account JSON key locally to ~/.claude/.omni-google-sa.json (never to the synapse fork, never to our servers). End-to-end ~10 minutes for first-time setup; subsequent additions of a new resource (new GSC property, new Drive folder) take <1 minute since the service account already exists. Idempotent — safe to re-run to add more API access or rotate the key.
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

## Why service account (and not OAuth)

Before we start, lock this in for the user: **service account is the right path for personal and small-team Omni use.** OAuth sounds simpler but isn't:

- **Verification gauntlet** — Google requires app verification for sensitive scopes (GSC, GA4, Drive). Verification is a weeks-long process involving privacy policy, terms of service, security questionnaires, and video demos. Not viable for personal use.
- **Test-mode 7-day expiry** — unverified OAuth apps refresh-tokens expire after 7 days, forcing re-consent weekly.
- **Per-user re-auth** — every Omni user has to do the OAuth dance individually; revoking is harder.
- **Scope changes mean re-auth** — adding GA4 to an existing GSC connection requires another consent flow.

Service account fixes all of this: one JSON key, no expiry, granular per-resource access. The setup is more clicks the first time, but it's a one-time cost.

If the user insists on OAuth, fine — say *"I'll do my best, but heads up that you'll need to add yourself as a test user in the OAuth consent screen and re-auth every 7 days, and any sensitive scope requires Google verification. Service account skips all of that. Sure you want OAuth?"* — and if they're still yes, refer them to Google's OAuth documentation. This skill doesn't implement the OAuth path.

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

### Step 4: Generate the JSON key

> *"Now generate a JSON key for that service account. Open https://console.cloud.google.com/iam-admin/serviceaccounts?project=<project-id>, click the SA you just created, go to the **Keys** tab, click **Add Key → Create new key**, pick **JSON**, click **Create**.*
>
> *A JSON file will download automatically. Don't open it — it contains a private key.*
>
> *Paste the ENTIRE contents of that file here (open with a plain-text editor if needed and select-all → copy → paste). It's a JSON object starting with `{` and ending with `}` — many lines."*

Wait for the user to paste the full JSON.

When they paste it:

1. Parse it. Validate that it has the required fields: `type: "service_account"`, `project_id`, `private_key`, `client_email`, `client_id`.
2. If any field is missing or `type` is not `"service_account"`, tell them: *"That doesn't look like a valid service account JSON. Re-download from the Keys tab and try again."* Don't proceed.
3. **Silently** save the JSON content to `~/.claude/.omni-google-sa.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-google-sa.json` (Windows). On Mac/Linux, set file permissions to `600` (owner-only read/write).
4. Tell them (without quoting the JSON back): *"Saved locally to your machine — never sent to our servers, never committed to git. If you ever paste a chat transcript that includes this JSON, generate a new key at the Keys tab and re-run this skill to replace it. (The old key can be deleted from the Keys tab too.)"*
5. Also tell them: *"If you ever want to revoke Omni's access to all your Google resources at once, delete this service account in the Google Cloud console — that invalidates the key globally."*

### Step 5: Grant the service account access to each resource

This is the per-service permission step. The SA email is the value the user pastes into Google's "Share with" / "Add user" dialogs throughout the Google Cloud UI.

Surface the SA email prominently:

> *"This is the email you need to share each resource with: `<sa-email>`. Copy it now; you'll paste it into a few places.*
>
> *For each Google service you enabled, here's how to grant access:"*

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

**Google Drive:**
> *"For each Drive folder Omni should access:*
> *1. Open https://drive.google.com*
> *2. Right-click the folder → Share*
> *3. Email: `<sa-email>` · Role: **Viewer**, **Commenter**, or **Editor** depending on what you want Omni to do*
> *4. Click Send (you can uncheck 'Notify people' — the SA is a bot, it doesn't have an inbox)*
>
> *Same pattern for individual files. Tell me when done."*

**Google Docs / Sheets:**
> *"Same as Drive — open the Doc or Sheet, click Share, add the SA email with the role you want. (Anything you've already shared a parent Drive folder with is auto-shared too.) Tell me when done."*

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

### Stop here.

Do not propose other skills. Do not suggest additional setup. The user has working Google Cloud credentials; they're done.

## What this skill MUST NOT do

- Do NOT save the service account JSON anywhere git-tracked (synapse fork, project files, shell config). It ONLY lives in `~/.claude/.omni-google-sa.json`.
- Do NOT echo the JSON contents, the `private_key` field, or any sensitive part of the key in any chat output, log, or written file.
- Do NOT proxy the credentials through Omnipresence's servers. They never leave the user's machine.
- Do NOT implement the OAuth path. If the user insists on OAuth despite the warnings, refer them to Google's OAuth docs — this skill is service-account-only.
- Do NOT skip Step 5 for any service the user enabled. Service account creation + key download alone gets you nothing without the per-resource permission grants.
- Do NOT skip the test send in Step 6 unless the user explicitly says "I trust it." A failing test surfaces config issues before they're discovered mid-real-work.
- Do NOT proceed past Step 4 if the pasted JSON fails the format validation.

## Why ~/.claude/.omni-google-sa.json (not env vars or settings.json)

Same rationale as the other Omnipresence credentials — see `omnipresence-os/docs/credential-integrations.md` for the canonical reasoning. Tool-agnostic file location, simple to inspect/rotate, never in git, easy to enumerate with `ls ~/.claude/.omni-*`.

This single file unlocks every Google API the SA has been granted access to — there's no separate "GSC key" vs "Drive key." One credential, many resources.
