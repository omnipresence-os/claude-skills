---
name: connect-email
description: Walks the user through connecting their own Resend account and verified sending domain, so Omnipresence can send email from their own brand to any recipient (not just to themselves via the catch-all). Trigger when the user says any of "connect my email", "set up my email", "set up resend", "set up email sending", "let omni email my clients", "connect a sending domain", "set up email deliverability", "configure email sending". This is the ONE canonical email-deliverability setup flow. Walks them through: Resend signup (if needed), adding their sending domain, DNS records at their registrar, domain verification, API key generation, sender identity configuration, and a test send. Saves the API key locally to ~/.claude/.omni-resend-key (never to the synapse fork, never to our servers) and saves the sender identity to custom/email/sender.md in their synapse fork. ~5 minutes of active work plus 5-60 minutes of DNS propagation wait. Idempotent — safe to re-run to update the key or change the sender identity.
---

# Connect Email — Member-Owned Resend + Verified Domain

This skill connects the user's own Resend account so Omni can send email FROM their brand TO any recipient (clients, prospects, anyone). Different from the catch-all (`omni@getomnipresence.com`), which only emails the user themselves.

## How to talk to the user during this skill

**Critical UX rule:** do NOT show shell commands or DNS record values in your replies. Run commands silently and explain in plain English. Show clickable URLs (always with `https://` prefix). When DNS records are involved, point at Resend's UI (which shows the records with copy buttons) — never paste record values into the chat.

✅ Good: *"Opening Resend's domain page for you... once you've added your domain, come back and tell me what your DNS provider is."*

❌ Bad: *"Add a TXT record with name `_resend.mail.acme.com` and value `re_verify_xyz...`"*

Only show:
- Plain-English progress.
- Clickable URLs.
- Specific click-paths once the user names their DNS provider.

Never show: API keys in any output, DNS record values, terminal commands.

## What this skill does — execute these steps in order

### Step 0: Locate the synapse fork

Read the cached path from `~/.claude/skills/.omnipresence-path` (or `%USERPROFILE%\.claude\skills\.omnipresence-path` on Windows).

- **If the file exists and the path is valid** (contains `core/` + `package.json`), use it.
- **If not found**, search common locations: `~/Documents/omnipresence/synapse`, `~/synapse`, `~/dev/synapse`, `~/Code/synapse`.
- **If still not found**, redirect: *"Omnipresence isn't set up yet on this machine. Run getting started first, then come back."* STOP.

### Step 1: Resend account check

Ask in plain English:

> *"To send email from your own brand, you'll need a Resend account. They're the email-sending platform we recommend — clean API, generous free tier (3,000/month), AI-friendly. Do you already have an account, or do you need to sign up?"*

- **If they need to sign up:** *"Go to https://resend.com/signup and create an account. Free tier is fine. Come back and say 'done' when you're signed in."* Wait for confirmation.
- **If they have one:** *"Great, you're signed in at https://resend.com? OK, on to the next step."*

### Step 2: Add the sending domain

Ask:

> *"What domain do you want to send from? Two options: (a) your main domain like `acme.com`, or (b) a subdomain like `mail.acme.com`. Most people use a subdomain — keeps your main domain's reputation separate from email sending. What would you like to use?"*

Wait for the domain. Store it.

Then:

> *"Open https://resend.com/domains, click **Add Domain**, paste `<their-domain>`, and click **Add**. Tell me when you've done that — Resend will then show you 3 DNS records to add."*

Wait for confirmation. **STOP gate** — the records don't exist until they add the domain.

### Step 3: DNS records

Ask:

> *"What's your DNS provider? Common ones: Cloudflare, Namecheap, GoDaddy, Squarespace, Google Domains, Route 53. If you don't know, check where you bought the domain."*

Wait for the provider, then give provider-specific click-paths:

- **Cloudflare:** *"Sign in to https://dash.cloudflare.com, click your domain, then **DNS → Records → Add record**. For each of the 3 records Resend is showing: pick the type (TXT or CNAME), paste the **Name** from Resend into Cloudflare's 'Name' field, paste the **Value** into Cloudflare's 'Content' field. Set **Proxy status** to 'DNS only' (gray cloud, NOT orange). Save. Repeat for all 3."*

- **Namecheap:** *"Sign in to https://ap.www.namecheap.com, click **Domain List → Manage** next to your domain, then **Advanced DNS → Add New Record**. For each of the 3 records: pick the type, paste the Host (the part before your domain) and the Value from Resend. Click the green check to save. Repeat."*

- **GoDaddy:** *"Sign in to https://account.godaddy.com/products, click **DNS** next to your domain, then **Add**. Pick the type, paste the name and value. Save. Repeat."*

- **Squarespace / Google Domains** (Google migrated to Squarespace in 2024): *"Sign in to https://account.squarespace.com/domains, click your domain, then **DNS Settings → Custom Records**. Add a row per record: type, host, data. Save."*

- **Route 53:** *"AWS Console → Route 53 → Hosted zones → your domain → Create record. Repeat for each of the 3."*

- **Other / unknown:** *"Look for **DNS Management**, **Advanced DNS**, or **DNS Records** in your registrar's dashboard. Resend gives 3 records (TXT and/or CNAME). For each, the 'Name' field on Resend matches the host/name field at your registrar; the 'Value' matches the content/value field."*

Then:

> *"Once all 3 records are in at `<provider>`, go back to Resend's domain page and click **Verify**. Verification takes 5 to 60 minutes (DNS propagation). The domain status will turn green when ready. Tell me when it's verified."*

**STOP gate:** Do NOT proceed until the user confirms the domain is verified (green) in Resend. Sends from unverified domains fail.

**If verification stalls past an hour:**
- Suggest https://mxtoolbox.com/SuperTool.aspx as a sanity check ("DNS Lookup" tool, pick TXT or CNAME).
- Common gotchas: typo in the name, double-domain (some registrars auto-append the apex), Cloudflare proxy enabled on a CNAME (must be 'DNS only').
- If still stuck: *"Email jonathan@getomnipresence.com with the domain and a screenshot of your DNS records."*

### Step 4: Generate the API key

> *"Go to https://resend.com/api-keys, click **Create API Key**, name it `omnipresence-<your-brand>`, set **Permission** to 'Sending access', and (recommended) restrict it to the domain you just verified. Click **Create**. Copy the key — it's only shown once. Paste it here."*

Wait for the user to paste.

When they paste it:

1. Validate the format — must start with `re_` followed by alphanumeric characters. If not, ask them to re-copy from Resend.
2. **Silently** save to `~/.claude/.omni-resend-key` (Mac/Linux) or `%USERPROFILE%\.claude\.omni-resend-key` (Windows). Single line, no whitespace, no quoting. On Mac/Linux, set file permissions to `600` (owner-only read/write).
3. Tell the user (without quoting the key): *"Saved on your machine only — never sent to our servers. If you ever paste a chat transcript that includes this key, regenerate it at https://resend.com/api-keys."*

### Step 5: Sender identity

Ask three things in one turn:

> *"Three quick questions to set your sender identity:*
> *1. **Brand display name** — what should appear as the sender name? (e.g., 'Clean Green Cars')*
> *2. **From address** — which address on your verified domain? (e.g., 'hello@mail.cleangreencars.com'). Must be on the domain you just verified.*
> *3. **Reply-to** — where should replies go? Press Enter to use your Omnipresence signup email."*

Wait for answers.

Validate that the from-address domain matches the verified domain (or is a subdomain of it). If it doesn't, surface the mismatch and ask them to either pick an address on the verified domain or add the second domain to Resend.

Then **silently** write `<synapse-fork>/custom/email/sender.md`:

```
---
from: "<brand-display-name> <<from-address>>"
reply_to: "<reply-to or empty>"
verified_domain: "<verified-domain>"
---

# Email Sender Identity

This file is read by the `resend-send` skill when Omni sends email
on your behalf. Edit the frontmatter to change your sender identity.
The `verified_domain` field is informational; Resend enforces
verification at send time.
```

Tell the user: *"Sender identity saved to your synapse fork at `custom/email/sender.md`. Run `push my synapse changes` later to back this up to your GitHub fork."*

### Step 6: Test send

Ask:

> *"Last step — let's send a test email so we know everything works. Where should I send it? Use a real inbox you can check (your own address is fine)."*

Wait for the recipient.

Invoke a send via the Resend API with:
- API key from `~/.claude/.omni-resend-key`
- From: the configured `from` in `sender.md`
- To: the test recipient
- Subject: `Omnipresence email setup test`
- Body (plain text): `If you're reading this, your Omnipresence email setup is working. You can now ask Omni to send email from <verified-domain> to any recipient.`

**Handle the response:**

- **200 with a message id:** *"Sent. Check `<recipient>` in the next minute or two. If it lands in spam, mark it 'Not spam' — that helps your domain's sender reputation. From now on, when you say things like 'email this report to client@acme.com', I'll send through your domain. The catch-all (`omni@getomnipresence.com`) is still available for emails to yourself."*

- **401 / 403:** the saved key is wrong. *"Resend rejected the API key. Re-run this skill to regenerate and re-save."* Then loop back to Step 4.

- **422 with a domain validation error:** *"Resend says the domain isn't verified yet, even though you said it was. Go back to https://resend.com/domains and confirm the status is green. Then re-run this skill from the verify step."* STOP.

- **Anything else:** surface the verbatim error. *"Send failed: `<error>`. If this looks like a config issue, email jonathan@getomnipresence.com with this message."*

### Stop here.

Do not propose next steps. Do not suggest other skills. The user got email setup; they're done.

## What this skill MUST NOT do

- Do NOT save the API key to the synapse fork, project files, environment variables in shell config, or anywhere git-tracked. Keys ONLY live in `~/.claude/.omni-resend-key`.
- Do NOT show the API key in any chat output, log, or written file outside `~/.claude/.omni-resend-key`.
- Do NOT proxy the key through Omnipresence's servers, post it to any Omnipresence endpoint, or store it in Supabase. The key never leaves the user's machine.
- Do NOT proceed past Step 4 if the API key fails format validation.
- Do NOT proceed past Step 3 if the user has not confirmed the domain is verified in Resend.
- Do NOT propose alternatives to Resend (Postmark, SendGrid, Mailgun, SES, etc.). One platform — Resend.
- Do NOT skip the test send. The setup is not complete until a real send succeeds.
