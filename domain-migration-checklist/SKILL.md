---
name: domain-migration-checklist
description: Walks a member through a domain-migration sign-off checklist when their site is moving from one domain to another (e.g., spellbook.legal → spellbook.com, oldbrand.com → newbrand.com). Asks 4-6 orienting questions (old domain, new domain, CMS, subdomain scope, migration window, owners), then writes a customized checklist markdown file to the active project's outputs/audits/ folder. The checklist covers 11 surface areas in three phases (Pre / During / Post) — redirects, internal links, canonicals, pre-migration cleanup, email + DNS, tracking + verification, subdomains, sitemaps + robots, schema, external listings, performance benchmarking — with per-item phase / owner / priority / status columns the member can tick off. Trigger when the user says any of "domain migration checklist", "site migration checklist", "I'm migrating my domain", "we're changing domains", "migrate from X to Y", "help me migrate my site", "domain change checklist", "rebrand migration", "switching domains", "moving to a new domain". This is a v1 STARTER — Jonathan will build a fuller domain-migration methodology / process later; this skill ships the actionable checklist now so members can stop winging migrations. Idempotent — re-running on the same project refreshes the file with current answers (prompts to confirm overwrite). Surfaces the top "do these first" Pre/P0 items in chat after writing, doesn't echo the full file. Pairs with push-changes to commit the customized checklist to the member's fork.
---

# Domain Migration Checklist — Walkthrough + Generated Checklist File

Asks a few orienting questions, writes a customized migration sign-off checklist to the active project, surfaces the top urgent items in chat.

## How to talk to the user during this skill

Plain English. ONE question at a time. Wait for each answer. Don't show shell commands. Don't paste raw frontmatter or the whole checklist content into chat — the deliverable is the file; surface the path + a tight summary.

This skill walks a CHECKLIST, not a strategy or methodology. Don't oversell — keep the framing factual: "Here's the file. Here are the three things to do right now. Edit the Status column as you go."

## Prerequisites

- An active project (resolved via [project-resolver](../project-resolver/SKILL.md)). If none, route the user to `switch-project` / `new-project` and STOP.
- Synapse fork path cached at `~/.claude/skills/.omnipresence-path`.

## Execute these steps in order

### Step 1 — Resolve active project

Follow [project-resolver](../project-resolver/SKILL.md). If no project, route to `new-project` / `switch-project` and STOP.

If resolved, surface in the first output line:

```
Generating domain migration checklist for project=<slug> (resolved from <chat-session | global-default>).
```

### Step 2 — Capture migration basics (4-6 inline questions)

Ask one at a time. Don't batch.

**Q1 — Old domain.** *"What's the OLD domain (the one you're migrating away from)? Include the TLD — e.g., `spellbook.legal` or `oldbrand.com`."*

Capture as `OLD_DOMAIN`. Validate it looks like a domain (`<word>.<tld>`); if not, ask again.

**Q2 — New domain.** *"What's the NEW domain (the one you're migrating to)? Same format — e.g., `spellbook.com` or `newbrand.com`."*

Capture as `NEW_DOMAIN`. Validate. If `OLD_DOMAIN == NEW_DOMAIN`, that's not a migration — surface the contradiction and ask the user to re-state.

**Q3 — CMS.** *"What's the site running on? Webflow / WordPress / Shopify / Squarespace / custom? Pick one — this affects the wording of a few items (e.g., 'footer symbol' is a Webflow concept; WordPress calls it a 'template part')."*

Capture as `CMS`. Default to "Webflow" if the user is unsure (the source checklist was Webflow-flavored).

**Q4 — Subdomain scope.** *"Any subdomains involved? Things like `blog.<old-domain>`, `app.<old-domain>`, `docs.<old-domain>`. List them, or say 'none'."*

Capture as `SUBDOMAINS` (free-form). If "none", that's fine — the subdomain section of the checklist still appears but with a single line: *"No subdomains in scope — skip section 7."*

**Q5 — Migration window.** *"When is the migration happening? Give me a date + a rough time window (e.g., 'Saturday 2026-06-08, 10am-12pm UTC'). If you don't know yet, say 'TBD'."*

Capture as `MIGRATION_WINDOW`. "TBD" is fine — the checklist still ships; the user fills it in later.

**Q6 (optional) — Owners.** *"Want me to assign owners to each item now, or leave the Owner column blank for you to fill in? If now, give me 2-5 names + roles (e.g., 'Jonathan: redirects + GSC', 'Sarah: DNS + email')."*

Capture as `OWNERS` (free-form, optional). If skipped, every Owner column entry is blank.

### Step 3 — Generate the checklist file

Write to `<synapse-path>/custom/projects/<active-slug>/outputs/audits/<OLD_DOMAIN>-to-<NEW_DOMAIN>-migration.md`. Create the `outputs/audits/` folder if it doesn't exist.

Use the template in the **Checklist Template** section below. Substitute every placeholder (`[OLD_DOMAIN]`, `[NEW_DOMAIN]`, `[CMS]`, etc.) with the captured values. CMS-specific wording: if `CMS == "Webflow"`, use "footer symbol" / "auto-301" / "Webflow Optimize"; if `CMS == "WordPress"`, use "template part / footer template" / "redirect plugin (e.g., Redirection)" / "Optimizely or AB Tasty"; for other CMSes, use the generic "[CMS] footer template" / "[CMS] redirect mechanism" wording. Owners go in the Owner column if provided; otherwise leave the column blank.

Do NOT paste the file content into chat. Do NOT echo the placeholder substitution as a log.

### Step 4 — Surface a tight summary in chat

Tell the user, exactly:

```
✅ Domain migration checklist generated for <OLD_DOMAIN> → <NEW_DOMAIN>.

  File: custom/projects/<active-slug>/outputs/audits/<OLD_DOMAIN>-to-<NEW_DOMAIN>-migration.md

Three phases, 11 surface areas, <N> total sign-off items:
  • Pre  (T-72h to T-2h):   <N_pre> items   — <N_pre_p0> P0 / <N_pre_p1> P1 / <N_pre_p2> P2
  • During (T-2h to T+5m):  <N_during> items
  • Post (T+5m to T+30d):   <N_post> items  — <N_post_p0> P0 / <N_post_p1> P1 / <N_post_p2> P2

**Do these THREE first (Pre / P0):**
  1. Audit existing 301 rules in [CMS]; prune rules pointing to URLs no longer in the sitemap
  2. Replace the [CMS] footer/nav template links to the new domain in ONE edit (eliminates the majority of internal link instances)
  3. Configure email and DNS for [NEW_DOMAIN]; retain [OLD_DOMAIN] MX records 12+ months

Update the Status column as you complete each item. The file is yours — hand-edit freely.

Next prompts:
  • Push to GitHub:        Push my synapse changes.
  • Regenerate:            Re-run "domain migration checklist" with new answers.
  • Open the file:         (click the path above).
```

If `MIGRATION_WINDOW == "TBD"`, prepend a heads-up: *"Migration date is TBD in the file. Update the header once you've locked the window."*

### Step 5 — Stop

Do NOT propose next steps beyond the success message. The user iterates from the file directly.

## Checklist Template

The full template below is what gets written to disk in Step 3 with placeholders substituted. Keep this section in sync with the source checklist (see "Source" at the bottom).

```markdown
# Domain Migration Checklist — [OLD_DOMAIN] → [NEW_DOMAIN]

**Project:** [PROJECT_SLUG]
**CMS:** [CMS]
**Migration window:** [MIGRATION_WINDOW]
**Subdomains in scope:** [SUBDOMAINS]
**Generated:** [YYYY-MM-DD]

## Migration phases

**Pre: T-72h to T-2h.** All prep work happens here: cleanup, audits, pre-configurations, decisions.

**During: T-2h to T+5 min.** Pause campaigns, flip the [CMS] domain, verify the swap.

**Post: T+5 min through T+30 days.** Configuration cascade, verification crawls, external updates, performance monitoring.

---

## 1. Redirects resolve to correct URL

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Pre** | Audit existing 301 rules in [CMS]; prune rules pointing to URLs no longer in the sitemap | [OWNER] | **P0** | [ ] |
| **During** | Set [NEW_DOMAIN] as default in [CMS] and publish; verify path preservation | [OWNER] | **P0** | [ ] |
| **Post** | Crawl the website to check [OLD_DOMAIN] URLs redirect to [NEW_DOMAIN] as expected | [OWNER] | **P0** | [ ] |
| **Post** | Submit GSC Change of Address (Settings → Change of Address; requires both [OLD_DOMAIN] and [NEW_DOMAIN] as verified properties) | [OWNER] | **P1** | [ ] |

## 2. Internal Links Cleanup

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Pre** | Replace footer / nav links in the [CMS] global template (one edit replaces dozens to hundreds of instances) | [OWNER] | **P0** | [ ] |
| **Pre** | Replace remaining body-content internal links pointing to redirect targets with their final destinations (per the Internal Links to 301s tab of the pre-migration audit) | [OWNER] | **P0** | [ ] |
| **Pre** | Replace hardcoded [OLD_DOMAIN] references in CMS body content, templates, custom code, and embed snippets | [OWNER] | **P0** | [ ] |
| **Post** | Run a post-launch crawl and check for any remaining [OLD_DOMAIN] references in body content, headers, or footers | [OWNER] | **P1** | [ ] |

## 3. Canonical Tags

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Post** | Spot check 6-10 high-traffic pages and 4 from each major content section to verify canonicals reference [NEW_DOMAIN] | [OWNER] | **P0** | [ ] |
| **Post** | Crawl the full website (Screaming Frog or similar) and verify all canonical URLs reflect the change | [OWNER] | **P0** | [ ] |

## 4. Pre-Migration Cleanup

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Pre** | Flatten any existing redirect chains (protocol/host normalization, double-hops) at the DNS / host level so the post-migration result is a single hop to [NEW_DOMAIN] (per Redirect Chains tab of pre-migration audit) | [OWNER] | **P1** | [ ] |
| **Pre** | Action all broken URLs returning 404: redirect, restore, or accept (per 404 URLs tab) | [OWNER] | **P1** | [ ] |
| **Pre** | Verify intent of any noindex paths — remove noindex from any that should be indexable on [NEW_DOMAIN] (per Noindex Paths tab) | [OWNER] | **P1** | [ ] |
| **Pre / Post** | Ship any pre-migration content consolidation redirects either BEFORE migration (PREFERRED — keeps the new domain's redirect table clean) or directly into the [NEW_DOMAIN] 301 table after migration | [OWNER] | **P1** | [ ] |

## 5. Email & DNS

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Pre** | Configure email and DNS for [NEW_DOMAIN]; retain [OLD_DOMAIN] MX records for 12+ months to catch in-flight email | [OWNER] | **P0** | [ ] |

## 6. Tracking & Verification

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Pre** | Add [NEW_DOMAIN] as a Domain property in GSC, GA4, and Bing Webmaster Tools | [OWNER] | **P0** | [ ] |
| **Pre** | Audit A/B testing tools (e.g., Webflow Optimize / Optimizely / AB Tasty / VWO), custom JS redirects, and paid landing pages for hardcoded [OLD_DOMAIN] URLs; replace with [NEW_DOMAIN] or relative paths | [OWNER] | **P0** | [ ] |
| **During** | Pause A/B testing campaigns (they often hold cached URL references that fight the redirect) | [OWNER] | **P1** | [ ] |
| **Post** | Reconfigure full tracking stack for [NEW_DOMAIN] (GA4 property, GTM container, conversion tracking, ad platform pixels) | [OWNER] | **P1** | [ ] |

## 7. Subdomain Scope & Migration

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---: | :---: | ----- |
| **Pre** | Confirm subdomain scope: which subdomains migrate to [NEW_DOMAIN] vs stay on [OLD_DOMAIN] (in scope: [SUBDOMAINS]) | [OWNER] | **P0 / blocking** | [ ] |
| **Pre** | Configure redirect mechanism for each in-scope subdomain ([CMS] typically does NOT auto-301 subdomains — you'll need DNS-level or host-level rules) | [OWNER] | **P0** | [ ] |
| **Post** | Verify in-scope subdomains: redirects resolve, canonicals updated, sitemaps reflect new URLs | [OWNER] | **P1** | [ ] |

## 8. Sitemaps & Robots.txt

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---- | :---- | ----- |
| **Pre** | Confirm [CMS] auto-sitemap is enabled (or that a manual sitemap exists and is current) | [OWNER] | P1 | [ ] |
| **Post** | Verify [NEW_DOMAIN] sitemap resolves correctly; submit to GSC and Bing | [OWNER] | P1 | [ ] |
| **Post** | Verify robots.txt on [NEW_DOMAIN] doesn't accidentally block crawl (a common migration footgun) | [OWNER] | P1 | [ ] |

## 9. Schema (Structured Data)

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---- | :---- | ----- |
| **Post** | Spot-check schema URLs reflect [NEW_DOMAIN] on 5 sample pages (per Schema URLs tab for full inventory) | [OWNER] | P2 | [ ] |
| **Post** | Verify any custom-coded schema in header embeds is updated manually ([CMS] auto-updates dynamic schema only — hand-written JSON-LD in custom code is yours to update) | [OWNER] | P2 | [ ] |

## 10. External Listings & Directories

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---- | :---- | ----- |
| **Post** | Update Tier 1 platforms to show [NEW_DOMAIN]: typically LinkedIn company page, Twitter/X profile, YouTube channel "about", Crunchbase, G2/Capterra (if applicable), industry-specific directories | [OWNER] | P2 | [ ] |
| **Post** | Update social meta on [NEW_DOMAIN] pages (og:url, twitter:url) so re-shares pull the new domain | [OWNER] | P2 | [ ] |

## 11. Performance Benchmarking

| Phase | Sign-off Item | Owner | Priority | Status |
| :---- | :---- | :---- | :---- | ----- |
| **Pre** | Capture baseline Core Web Vitals (CWV) on [OLD_DOMAIN] BEFORE migration — top 10 pages by traffic | [OWNER] | P2 | [ ] |
| **Post** | Capture post-migration CWV at T+24h, T+7d, T+30d on [NEW_DOMAIN]; compare to baseline | [OWNER] | P2 | [ ] |

---

## Notes

- **Priority key:** P0 = blocking / catastrophic if missed; P1 = important; P2 = nice-to-have / can be done in the weeks after migration.
- **Status checkbox:** mark `[x]` when done; leave `[ ]` when pending.
- **About this checklist:** this is the v1 starter generated by the `domain-migration-checklist` skill. A fuller methodology / process is on the roadmap — for now, use this as the sign-off surface and add project-specific items inline.

## Source

Generated by the [`domain-migration-checklist`](https://github.com/omnipresence-os/claude-skills/blob/main/domain-migration-checklist/SKILL.md) Claude Code skill on [YYYY-MM-DD]. Adapted from the Spellbook .legal → .com migration checklist (proprietary).
```

## Edge cases

**The user has no active project.** Route to `new-project` (if zero projects exist) or `switch-project` (if multiple) per the standard project-resolver flow. Don't generate the checklist into an orphaned location.

**The checklist file already exists for this domain pair.** Ask: *"A checklist for `<OLD_DOMAIN>` → `<NEW_DOMAIN>` already exists at `<path>`. Refresh it (overwrites — you'll lose any Status / Owner edits) or open the existing one?"* Default to "open" — don't clobber checked-off items.

**The user is mid-migration and wants a quick re-check rather than a fresh checklist.** This skill ALWAYS generates fresh. If the user wants "what's still left to do?", direct them to open the existing file and look at the unchecked rows. (A future `domain-migration-status` skill could parse the file and surface remaining items, but that's out of v1 scope.)

**The OLD_DOMAIN and NEW_DOMAIN are subdomains of the same root** (e.g., `app.example.com` → `www.example.com`). Generate the checklist normally — most items still apply. Add a note in the file header: *"NOTE: This migration is between subdomains of the same root. Items 5 (Email & DNS) and 10 (External Listings) may be partially or fully N/A."*

**The user picks a CMS the skill doesn't have CMS-specific wording for** (e.g., "Ghost", "Hugo", "custom Python"). Use the generic "[CMS] footer template" / "[CMS] redirect mechanism" phrasing and add a one-line note above section 2: *"NOTE: [CMS] specifics not built into this checklist. Verify the redirect mechanism and global-template-substitution flow for your stack."*

**Owners list is partially provided (some items the user has owners for, others not).** Fill in what they gave you; leave the rest blank. Don't try to infer owners from chat context.

**The user wants to add custom items to the checklist after generation.** Tell them: *"Edit the file directly — add rows to any of the 11 sections, or add a section 12 for project-specific items. The skill is a starter; the file is yours."*

## What this skill MUST NOT do

- Do NOT paste the full checklist content into chat — it's a file.
- Do NOT skip the migration-basics interview (Step 2) — those answers drive the placeholder substitution.
- Do NOT auto-execute any migration actions (this skill produces a checklist; it doesn't redirect anything, modify DNS, or touch the CMS).
- Do NOT modify files outside `custom/projects/<active-slug>/outputs/audits/`.
- Do NOT touch the global active-project pointer.
- Do NOT chain to `push-changes` automatically — the user opts in via the success message.
- Do NOT freestyle the checklist content. The template above is the source of truth; substitute placeholders and ship.

## Why this exists

Domain migrations are high-stakes (a missed redirect can torch organic rankings for months) and members often wing them — there was no canonical sign-off checklist in the Omni library, so each migration's quality depended on whoever was running it remembering everything. This skill bakes the 11-surface-area, 3-phase checklist into a generator that produces a customized file in 60 seconds.

It's a v1 starter. Jonathan is building a fuller domain-migration methodology / process — when that ships, this skill will chain to it for deeper guidance per item. For now, the checklist alone is enough to stop the wing-it failure mode.

## Skill family

- [`new-project`](../new-project/SKILL.md) — sets up the project folder where the checklist file lands.
- [`switch-project`](../switch-project/SKILL.md) — resolves which project the checklist belongs to.
- [`push-changes`](../push-changes/SKILL.md) — commits the generated checklist to the member's fork.
- [`fill-project-gap`](../fill-project-gap/SKILL.md) — sibling pattern (interactive file generator) for project setup gaps.
