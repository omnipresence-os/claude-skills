---
name: connect-dataforseo
description: Walks the user through connecting their DataForSEO account so Omnipresence can pull SERP, keyword, backlink, on-page, and LLM-mention data without an Ahrefs or Semrush subscription. Trigger when the user says any of "connect dataforseo", "set up dataforseo", "connect my dataforseo", "set up SEO data", "connect a SEO data source", "wire up dataforseo", "add my dataforseo credentials", "set up AIO measurement". This is the ONE canonical DataForSEO credential-setup flow. Walks them through: DataForSEO signup (if needed), retrieving their API login + API password (separate from the dashboard login password) from https://app.dataforseo.com/api-dashboard, saving credentials locally to ~/.claude/.omni-dataforseo.json (never to the synapse fork, never to our servers), and validating with one cheap test call to the user_data endpoint. ~3 minutes total. Idempotent — safe to re-run to rotate the password or update the login.
---

# Connect DataForSEO — API Credentials for SERP / Keyword / Backlink / AIO Data

This skill connects the user's DataForSEO account so any Omnipresence skill that needs SEO data (SERP scrapes, keyword volumes, backlink intel, on-page audits, AIO panel lookups, LLM-mention tracking) can call the API. DataForSEO is the recommended third-party data source for members who don't have an Ahrefs API or Semrush API subscription.

## How to talk to the user during this skill

**Critical UX rule:** do NOT show shell commands or paste API credentials in your replies. Run commands silently and explain in plain English. Show clickable URLs (always with `https://` prefix). Never echo the API password back to the user.

Only show:
- Plain-English progress.
- Clickable URLs.
- Connectivity confirmation and credit balance.

Never show: API passwords in any output (not even partially masked), shell commands, the contents of the saved credentials file.

## What this skill does — execute these steps in order

### Step 0: Check existing setup

Silently check `~/.claude/.omni-dataforseo.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-dataforseo.json` (Windows).

- **If it exists and is valid JSON with `login` + `password` fields**, ask: *"You've already got DataForSEO connected as `<login email>`. Want to update it (rotate password or change account), or test the existing connection?"*
  - **Update:** continue to Step 1.
  - **Test:** skip to Step 4 and validate the existing creds.
- **If it doesn't exist**, continue to Step 1.

### Step 1: DataForSEO account check

Ask in plain English:

> *"To pull SEO data — SERP, keywords, backlinks, AIO panels, LLM mentions — Omnipresence needs DataForSEO credentials. It's pay-per-call (no subscription, no minimum); typical AIO/SERP work runs a few cents per session. Do you already have a DataForSEO account, or do you need to sign up?"*

- **If they need to sign up:** *"Go to https://dataforseo.com/register and create an account. They give every new account a small free credit to try it. Come back and say 'done' when you're signed in."* Wait for confirmation.
- **If they have one:** *"Great. Let's grab your API credentials next."*

### Step 2: Retrieve the API credentials

> *"Open https://app.dataforseo.com/api-dashboard while signed in. You'll see two values near the top: an **API Login** (it's your account email, but copy it from the page to be sure) and an **API Password** (a long alphanumeric string — this is DIFFERENT from your dashboard login password). Click the copy buttons next to each. Tell me when you've got both copied to your clipboard."*

Wait for confirmation. **STOP gate** — both values need to exist before continuing.

### Step 3: Collect and save the credentials

Ask, exactly in this order:

> *"1. Paste your **API Login** (your account email per the dashboard)."*

Wait for the login. Validate it looks like an email (contains `@` and at least one `.`).

> *"2. Paste your **API Password** (the long alphanumeric string from the dashboard)."*

Wait for the password.

**Silently** write `~/.claude/.omni-dataforseo.json` (or `%USERPROFILE%\.claude\.omni-dataforseo.json` on Windows) with this exact shape:

```json
{
  "login": "<API Login>",
  "password": "<API Password>"
}
```

On Mac/Linux, `chmod 600` the file (owner-only read/write). On Windows, the default user-profile permissions are owner-only already.

Tell the user (without quoting the password back): *"Saved locally to your machine — never sent to our servers, never committed to git."*

### Step 4: Validate with a test call

Silently make this curl call (or its equivalent):

```bash
curl -s -u "<login>:<password>" \
  https://api.dataforseo.com/v3/appendix/user_data
```

Inspect the response:

- **`status_code == 20000` with a `money.balance` field:** success. Tell the user: *"Connected. Your remaining credits balance is $<balance>. From now on, any Omnipresence skill that needs DataForSEO data — SERP scrapes, keyword volumes, AIO panels, LLM mentions — will use these credentials automatically."*

- **`status_code == 401` or `403`** (or HTTP 401): the password is wrong. Tell the user: *"DataForSEO rejected the credentials. The most common cause is pasting the dashboard login password instead of the API Password — they're different. Go back to https://app.dataforseo.com/api-dashboard and copy the API Password (the long alphanumeric string under 'API access'). Then re-run this skill."* Delete the invalid credentials file. STOP.

- **Network error / 5xx**: tell the user: *"DataForSEO is unreachable right now — could be their service or your network. Saved your credentials anyway; the next skill that uses them will validate. To re-test, paste `connect dataforseo` again and pick 'Test'."*

### Step 5: Briefly explain what's now available

> *"You can now ask Omni to do things like:*
> *- 'Run an AIO readiness audit on <URL> for query <X>'*
> *- 'Show me SERP results for <query> in <location>'*
> *- 'Check search volume for these keywords: <list>'*
> *- 'Pull backlinks for <domain>'*
> *- 'Track LLM mentions of <brand> across ChatGPT, Claude, Gemini, Perplexity'*
>
> *Costs run a few cents per session for most work. Heavy bulk runs (10k+ keyword volumes) can cost more — Omni will tell you before any expensive call. To check your balance anytime, just ask: 'What's my DataForSEO balance?'"*

### Stop here.

Do not propose other skills. Do not suggest other configurations. The user got credentials wired; they're done.

## What this skill MUST NOT do

- Do NOT save the credentials to the synapse fork, project files, environment variables in shell config, or anywhere git-tracked. Credentials ONLY live in `~/.claude/.omni-dataforseo.json`.
- Do NOT echo the API password back to the user in any chat output, log, or written file outside the credentials file. Even partial masking (e.g., `abc***xyz`) leaks pattern info; just don't show it.
- Do NOT proxy the credentials through Omnipresence's servers, post them to any Omnipresence endpoint, or store them in Supabase. They never leave the user's machine.
- Do NOT proceed past Step 3 if the API Login doesn't look like an email — DataForSEO's API Login is always the account email.
- Do NOT skip the validation in Step 4. The user should leave this skill knowing the credentials work.
- Do NOT propose alternatives to DataForSEO (Serper, ScrapingBee, manual scraping, etc.). One platform — DataForSEO — for the recommended third-party SEO data source. Members who have Ahrefs or Semrush should use those skills instead; this skill is the fallback path.

## Why ~/.claude/.omni-dataforseo.json (not env vars or settings.json)

For posterity: this file lives outside the synapse fork on purpose. Putting credentials in the fork risks accidental git-add. Putting them in `~/.claude/settings.json` is Claude-Code-specific (won't transfer if the member moves to Cursor / another tool). Putting them in shell config (`.bashrc` / `.zshrc`) survives across tools but is harder to inspect, list, and rotate.

The full convention is documented in `omnipresence-os/docs/credential-integrations.md`.
