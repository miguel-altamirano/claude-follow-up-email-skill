Train Claude on your post-call follow-up email style once. Then draft new follow-ups in your voice after every customer meeting.

Works in **Claude Code** and **Claude Cowork**. Reads your calls from Granola, your past emails from Gmail. Drafts go to Gmail Drafts — nothing is sent without your approval.

> *Add demo GIF here — 20-30 seconds showing `write followup` → draft in chat → save to Drafts.*

---

## What it does

Two skills, one workflow.

**`follow-up-email-trainer`** runs once. It scans your last 90 days of external calls, finds the follow-up email you sent (or didn't) after each one, and extracts your writing style: subject format, opening patterns, length, CTA shape, tone, sign-off, plus prior-recipient context per customer.

**`follow-up-email`** runs after every call. You type `write followup` (or `write follow up Infineon 2026-05-11`). Claude reads your style file, pulls the source call's transcript, and drafts the email in chat. You approve, it lands in Gmail Drafts.

---

## Install

### Claude Code
/plugin marketplace add miguel-altamirano/claude-follow-up-email-skill
/plugin install follow-up-email

### Claude Cowork

Cowork's plugin UI doesn't support individual GitHub marketplaces yet. Install manually in 3 steps:

**Step 1 — Add the trainer skill**
1. Open Cowork → Customize → Skills → Your organization
2. Create a new skill called `follow-up-email-trainer`
3. Paste the contents of [follow-up-email-trainer/SKILL.md](https://github.com/miguel-altamirano/claude-follow-up-email-skill/blob/main/plugins/follow-up-email/skills/follow-up-email-trainer/SKILL.md)

**Step 2 — Add the executor skill**
1. Create a second skill called `follow-up-email`
2. Paste the contents of [follow-up-email/SKILL.md](https://github.com/miguel-altamirano/claude-follow-up-email-skill/blob/main/plugins/follow-up-email/skills/follow-up-email/SKILL.md)

**Step 3 — Run the trainer**
In any Cowork session, say: `Train me on my follow-up email style`

---

## First-time setup (one-time)

After install, run the trainer once:
Train me on my follow-up email style

The trainer will:

- Verify your meeting recorder, email, and calendar MCPs are connected
- Analyze your last 90 days of external calls (capped at 20 most recent)
- Match each call to the follow-up email you sent after it
- Write two reference files into the skill's `references/` folder

Takes about 5-8 minutes the first time. You only run this once. Re-run it if your writing style changes meaningfully.

---

## After every call
write followup

Claude pulls your most recent external call, drafts the email, shows it in chat. Say "looks good" and it lands in Gmail Drafts.

For a specific call:
write follow up Infineon 2026-05-11

---

## What you need connected

Three MCPs:

1. **A meeting recorder with transcript access** — Granola, Fireflies, Otter, or equivalent
2. **An email provider** — Gmail or Outlook
3. **A calendar** — Google Calendar or Outlook Calendar

The skill is provider-agnostic — at runtime it inspects which MCPs you have and uses the first available in each category.

---

## Privacy

Everything runs locally. The trainer reads your calls and emails, writes two files (`follow-up-email-style.md` and `follow-up-email-dossier.md`) into your local skill folder, and never sends data anywhere.

The two reference files contain real customer names, transcripts, and email content. That folder is `.gitignore`d in this repo for a reason — never commit your trained data back to a public fork.

If you want the reference files to survive a reinstall, copy them somewhere else first.

---

## Editing your style

The two files are plain markdown. Open them, edit them, save them. The executor reads them fresh on every draft.

If Claude misread an objection on a call, fix the dossier entry. If your writing style is shifting, edit the style file. The executor picks it up next time.

Section headers in `follow-up-email-style.md` are load-bearing — keep them as-is. Content under each header is yours to rewrite.

---

## FAQ

**Does this send emails?** No. The executor creates Gmail drafts only. Sending is on you, by design.

**What if I don't use Granola?** The skill is provider-agnostic. If you have Fireflies or Otter connected instead, it'll use that. Same for Outlook vs Gmail, Outlook Calendar vs Google.

**What if my latest call has no transcript?** The executor will ask you for 2-3 bullets on what happened, then draft from your style file. It won't fabricate call content.

**Can I retrain?** Yes. Re-run the trainer any time. It overwrites the two reference files.

**Does it work on Claude.ai web?** No. Skills only run in Claude Code and Cowork.

**Does it cost anything?** No. This is MIT-licensed. The skill itself is free. You're paying for Claude (Code or Cowork) and your connectors (Granola, Gmail, etc.) separately, as you normally would.

**Can I fork and customize?** Please do. If you ship something good, tag me on LinkedIn ([Miguel Altamirano](https://www.linkedin.com/in/miguel-altamirano/)).

---

## Built by

[Miguel Altamirano](https://www.linkedin.com/in/miguel-altamirano/), founding AE at [Sluicebox](https://sluicebox.ai). Shipped in public.

If this saves you 30 minutes a week, the goodwill is enough. Issues and PRs welcome.
