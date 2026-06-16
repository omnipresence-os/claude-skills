---
name: connect-openai
description: Walks the user through connecting their OpenAI API key so any Omnipresence skill that supports the `provider:openai` toggle (today — dataforseo-ai-visibility's Method A/B OpenAI-direct path; future — additional AI-search collectors) can call OpenAI directly instead of going through DataForSEO's wrapper. Trigger when the user says any of "connect openai", "set up openai", "connect my openai", "add my openai key", "wire up openai", "save my openai api key", "configure openai for omnipresence", "use openai for GEO research", "switch GEO to openai". This is the ONE canonical OpenAI credential-setup flow. Walks them through retrieving an API key at https://platform.openai.com/api-keys, saving credentials locally to ~/.claude/.omni-openai.json (never to the synapse fork, never to our servers), and validating the key with one zero-cost GET to /v1/models. ~2 minutes total. Idempotent — safe to re-run to rotate keys or update the account.
---

# Connect OpenAI — API Key for Direct AI-Search Citation Calls

This skill connects the user's OpenAI API key so Omnipresence skills that support direct OpenAI calls — primarily the `dataforseo-ai-visibility` skill with `provider:openai` — can use OpenAI instead of DataForSEO's wrapper for AI-search citation research. DataForSEO costs ~$0.045/probe; OpenAI direct (Method B on `gpt-5`) costs ~$0.027–0.035/probe — roughly 25–40% cheaper at fidelity-comparable settings, more on the `gpt-4o-mini-search-preview` (Method A) cheap-bulk path.

DataForSEO remains the default for the GEO Source Finder worker and other AI-visibility consumers; connecting OpenAI is the opt-in cost lever.

## How to talk to the user during this skill

**Critical UX rule:** do NOT show shell commands or paste API credentials in your replies. Run commands silently and explain in plain English. Show clickable URLs (always with `https://` prefix). Never echo the API key back to the user.

Only show:
- Plain-English progress.
- Clickable URLs.
- Connectivity confirmation.

Never show: API keys in any output (not even partially masked), shell commands, the contents of the saved credentials file.

## What this skill does — execute these steps in order

### Step 0: Check existing setup

Silently check `~/.claude/.omni-openai.json` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-openai.json` (Windows).

- **If it exists and is valid JSON with an `api_key` field**, ask: *"You've already got OpenAI connected on this machine. Want to update the key (rotate or change account), or test the existing connection?"*
  - **Update:** continue to Step 1.
  - **Test:** skip to Step 4 and validate the existing key.
- **If it doesn't exist**, continue to Step 1.

### Step 1: OpenAI account check

Ask in plain English:

> *"To run AI-search citation probes directly against OpenAI (cheaper than DataForSEO's wrapper for bulk GEO research), Omnipresence needs an OpenAI API key. It's pay-per-call; typical AEO/GEO work runs ~$0.03/probe on a fidelity model (`gpt-5` + web_search tool) or ~$0.01–0.02/probe on the cheap bulk path (`gpt-4o-mini-search-preview`). Do you already have an OpenAI account with API billing enabled, or do you need to set one up?"*

- **If they need to sign up:** *"Go to https://platform.openai.com/signup and create an account. Then add a payment method at https://platform.openai.com/account/billing. The API surface is separate from ChatGPT Plus — having a ChatGPT subscription doesn't grant API access. Come back and say 'done' when billing is active."* Wait for confirmation.
- **If they have one:** *"Great. Let's grab an API key next."*

### Step 2: Retrieve the API key

> *"Open https://platform.openai.com/api-keys while signed in. Click 'Create new secret key', give it a name like 'omnipresence' so it's easy to identify in the dashboard, and copy the value — it starts with `sk-`. The key is shown ONCE; if you close the dialog without copying, you'll need to create a new one. Tell me when you've got it copied to your clipboard."*

Wait for confirmation. **STOP gate** — the key needs to exist before continuing.

### Step 3: Collect and save the key

Ask:

> *"Paste your OpenAI API key (starts with `sk-`)."*

Wait for the key.

Validate it looks like an OpenAI key: starts with `sk-` (project keys start with `sk-proj-`; both are valid), length > 40 characters. If validation fails, ask again — don't proceed with a malformed key.

**Silently** write `~/.claude/.omni-openai.json` (or `%USERPROFILE%\.claude\.omni-openai.json` on Windows) with this exact shape:

```json
{
  "api_key": "<API Key>"
}
```

On Mac/Linux, `chmod 600` the file (owner-only read/write). On Windows, the default user-profile permissions are owner-only already.

Tell the user (without quoting the key back): *"Saved locally to your machine — never sent to our servers, never committed to git."*

### Step 4: Validate with a zero-cost test call

Silently make this curl call (or its equivalent):

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer <api_key>" \
  https://api.openai.com/v1/models
```

`/v1/models` is a free metadata endpoint — costs nothing and confirms the key authenticates.

Inspect the response:

- **HTTP 200:** success. Tell the user: *"Connected. From now on, any Omnipresence skill that supports the `provider: openai` toggle will route through your OpenAI account when you set the flag. The skill catalog will tell you which calls are eligible."*

- **HTTP 401 or 403:** the key is wrong, expired, or revoked. Tell the user: *"OpenAI rejected the key. Common causes: typo on paste, key was deleted from the dashboard, or billing isn't active on the account. Go back to https://platform.openai.com/api-keys to create or rotate a key, and verify billing is active at https://platform.openai.com/account/billing. Then re-run this skill."* Delete the invalid credentials file. STOP.

- **HTTP 429:** quota exceeded. Tell the user: *"OpenAI returned 'quota exceeded' — your account is past its usage limit or has no billing method. Add or refresh billing at https://platform.openai.com/account/billing, then re-run this skill."* Keep the saved key (it's valid; the quota is the issue).

- **Network error / 5xx:** tell the user: *"OpenAI is unreachable right now — could be their service or your network. Saved your key anyway; the next skill that uses it will validate. To re-test, paste `connect openai` again and pick 'Test'."*

### Step 5: Briefly explain what's now available

> *"You can now opt into OpenAI-direct on supported skills by setting `provider: openai`. The main consumer today:*
> *- `dataforseo-ai-visibility` skill — pass `provider: openai` to swap the discovery probe from DataForSEO's wrapper to direct OpenAI calls (Method A: `gpt-4o-mini-search-preview` for cheap bulk; Method B: `gpt-5` + web_search tool for fidelity).*
>
> *Default behavior across the catalog is unchanged — DataForSEO stays the default provider; OpenAI is opt-in per call. The GEO Source Finder worker continues to default to DataForSEO unless you override it.*
>
> *Recommended next step: run the 20-probe parity test (Method B on `gpt-5`) against a recent DataForSEO run on the same query set, compare cited-domain frequency, and only flip the default if the distributions match."*

### Stop here.

Do not propose other skills. Do not suggest other configurations. The user got credentials wired; they're done.

## What this skill MUST NOT do

- Do NOT save the key to the synapse fork, project files, environment variables in shell config, or anywhere git-tracked. Credentials ONLY live in `~/.claude/.omni-openai.json`.
- Do NOT echo the API key back to the user in any chat output, log, or written file outside the credentials file. Even partial masking (e.g., `sk-***xyz`) leaks pattern info; just don't show it.
- Do NOT proxy the key through Omnipresence's servers, post it to any Omnipresence endpoint, or store it in Supabase. It never leaves the user's machine.
- Do NOT proceed past Step 3 if the key doesn't start with `sk-` — that's the OpenAI key prefix.
- Do NOT skip the validation in Step 4. The user should leave this skill knowing the key works.
- Do NOT run a billable validation call (chat completion, responses, etc.) when `/v1/models` is free and answers the same "does this key authenticate" question.
- Do NOT propose alternatives to OpenAI (Anthropic, Gemini, etc.) — this skill is specifically the OpenAI credential setup. Other provider connectors are separate skills.

## Why ~/.claude/.omni-openai.json (not env vars or settings.json)

For posterity: this file lives outside the synapse fork on purpose. Putting credentials in the fork risks accidental git-add. Putting them in `~/.claude/settings.json` is Claude-Code-specific (won't transfer if the member moves to Cursor / another tool). Putting them in shell config (`.bashrc` / `.zshrc`) survives across tools but is harder to inspect, list, and rotate.

The shape `{ "api_key": "sk-..." }` mirrors the JSON-credentials pattern used by `~/.claude/.omni-dataforseo.json` (login + password) and `~/.claude/.omni-resend-key` (api_key). Members who connect multiple providers see a consistent file layout under `~/.claude/`.

The full convention is documented in `omnipresence-os/docs/credential-integrations.md`.
