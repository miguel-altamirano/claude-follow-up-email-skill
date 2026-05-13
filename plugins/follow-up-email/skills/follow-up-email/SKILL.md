---
name: follow-up-email
description: Draft a customer follow-up email in Miguel's exact voice after an external sales call, save it as a Gmail draft (never send), and only after Miguel approves in chat. The skill reads two reference files produced by the follow-up-email-trainer skill — a style document and a meeting dossier — and uses them as ground truth for voice, structure, subject format, and prior recipient context. Trigger whenever the user says any of: "write followup", "write follow up", "write follow-up", "draft a follow up", "draft followup", "follow up email for [Company]", "follow up [Company] [Date]", "write follow up [Company]", "follow up on my last call", "draft a post-call email", or any close variant about producing a customer follow-up email after a meeting. Trigger even when the user doesn't say "skill" — the moment they ask for a follow-up email to be written, this skill should run. Sales / GTM / CS only. This is for external customer communications, not internal Slack-style follow-ups.
---

# Follow-Up Email Executor

## What this skill does

Drafts a single follow-up email after a customer call, in Miguel's voice, saves it as a Gmail draft for him to send manually. The skill never sends email itself — connector-safety requires per-action approval on writes, and a misread of a call should land in Drafts where it's recoverable, not in the customer's inbox.

This skill depends on the follow-up-email-trainer having been run first. If the reference files don't exist, stop and tell the user.

## When to use

Use this any time the user wants a post-call follow-up email drafted. Two invocation patterns:

- **`write followup`** (no arguments) → draft a follow-up for the most recent external Granola meeting
- **`write follow up [Company] [Date]`** → draft a follow-up for the specified meeting

If the user types something ambiguous like "follow up Infineon" with no date, default to the most recent Infineon meeting in the dossier or Granola list.

## Required reference files

Both files must exist in this skill's `references/` directory (the trainer writes them there):

- `references/follow-up-email-style.md` — voice/structure/length patterns
- `references/follow-up-email-dossier.md` — prior meetings + prior follow-up emails by company

If either file is missing, stop and tell the user:

> The follow-up email reference files aren't set up yet. Run the follow-up-email-trainer skill first — that's the one-time setup that teaches me how you write.

Don't try to draft without these files. The result will sound generic.

## The workflow

### Step 1 — Identify the source meeting

**No arguments:** Pull the most recent meeting from Granola via `list_meetings` (time_range: `this_week`, fallback to `last_30_days`). Filter to meetings with at least one external attendee (non-@sluicebox.ai). Take the most recent.

**Company + date arguments:** Search the dossier file first by company name and date. The dossier is already analyzed — using it saves a transcript pull and gives you better prior context. If not in dossier, fall back to Granola `query_granola_meetings` or `list_meetings` with a date range around the requested date.

If you can't resolve the meeting, stop and ask — don't guess.

### Step 2 — Load context

Read in this order:

1. `references/follow-up-email-style.md` — keep the whole thing in context, it's small
2. `references/follow-up-email-dossier.md` — pull the entries for the same **company** as the source meeting. These are the strongest anchors for voice on this specific recipient.
3. The source meeting's transcript (Granola `get_meeting_transcript`) if not already in the dossier

### Step 3 — Synthesize the dossier block for this call

Before drafting, produce the same structured dossier block the trainer produces, for the source meeting. This forces grounding in actual call content rather than generic post-call platitudes. Fields: attendees, objective, what happened, objections, risks, qualification signals, next-step lever. Keep it short — bullets, not paragraphs. Use Miguel's terminology if it appeared in the call: Pilot, Foundation, Scale, BOM, PCF, EPD, LCA, MPN, primary data, etc.

You don't need to show this block to the user — it's scaffolding for a better draft.

### Step 4 — Draft the email

Apply the style document. The patterns to respect (these come straight from the style file, but reiterate for emphasis):

- **Subject line:** Default to `Month Day - [Company] & Sluicebox` for recurring-cadence meetings. For topic-anchored calls (specific milestone, specific deliverable), use a content-led subject like `[Specific topic]: [Company] [Date]`. Match what prior emails to this company used if there's a pattern.
- **Opening:** `Hi <FirstName>,` (first names only). Then a one-line reaction or thanks tied to something specific from this call. Then frontload the deliverable — `Attached the [deck/matrix/reports]` or `Sharing as promised:`.
- **Length:** Aim for 110-160 words. Median in the corpus is ~149. Hard cap at 200 unless there's a substantive reason.
- **Body structure:** Bullets, not prose paragraphs. Light section labels in plain text (`Next Steps`, `Next week's focus`, `Timeline pending confirmation`). High bullet density.
- **CTA:** Exactly one closing question, named recipient if multiple addressees, time-boxed where natural. Yes/no confirmation is the most common pattern.
- **Tone:** Specific numbers and dates land well. Telegraphic openings (`Attached the X`). Use `&` not `and` in subjects. Spaced hyphens never em dashes.
- **Never:** em dashes, "leverage", "synergize", "circle back", "only X", "difficult to hit", absolutes, "prepared for" labels, CTA buttons.
- **Sign-off:** Copy the verbatim signature block from the style file. Don't reconstruct.
- **CC:** Per Miguel's preferences, customer-facing emails CC Sarah, Elmar, and Piriya. Confirm this is desired before writing (the style file may show variation by meeting type).

### Step 5 — Show the draft in chat

Show the full email in chat, formatted exactly as it'll appear in Gmail:

```
Subject: [subject line]
To: [comma-separated external recipients]
CC: [sarah@sluicebox.ai, elmar@sluicebox.ai, piriya@sluicebox.ai] (or omit if not applicable)

[body]
```

Below the email, list 2-3 things the user might want to change. Examples:

- Length: 138 words. Want shorter / longer?
- CTA defaults to a yes/no on next session date. Want a different ask?
- Anchor proof point: [which one was used and why]. Want to swap?

Ask one question: **"Draft this to Gmail as-is, or do you want to edit anything?"**

Do not call `create_draft` until the user says yes / approves / "looks good" / equivalent. Per connector-safety, an in-conversation explicit approval on this specific draft is required.

### Step 6 — Create the Gmail draft on approval

Once approved, call `mcp__<gmail>__create_draft` with the email content. Never use `send_message` directly. Drafts are recoverable, sends aren't.

Confirm to the user:

> Draft saved to Gmail. Open it to review and send.

Include a Gmail link to the Drafts folder if available.

If the user requested edits, apply them in chat and re-show the draft. Loop until approved or they say to drop it.

## Edge cases

**No recent meeting found.** If `write followup` is invoked but no external Granola meeting exists in the last 7 days, ask: "I don't see a recent external call in Granola. Which meeting do you want me to draft from?" Don't draft from an internal call.

**Source meeting was internal.** Don't draft a customer follow-up from an internal call. Tell the user, name the meeting, ask which external meeting they meant.

**Multiple meetings with same company on same date.** Pick the one that has external attendees if only one does. Otherwise ask.

**Dossier has no prior emails for this company.** That's fine — fall back to the style document alone. The style file is the universal anchor; the dossier entries are just stronger recipient-specific signal when available. Flag it lightly in chat: "First follow-up to [Company] I see in your history — drawing from your general style only."

**Customer is in your DISQUALIFIERS list** (La Farga / copper-Sphera-default, Indra/Tecan-type satellite industries, fellow vendors). Draft the email but flag: "This customer is in your disqualifier list per your preferences — confirm you want to keep this thread warm?"

**Transcript missing.** If the source meeting has no Granola transcript, ask the user for 2-3 bullets on what happened, then draft from those + the style doc. Don't fabricate call content.

## Hard rules

- Read-only on connectors until the user explicitly approves the draft. Then exactly one write call: `create_draft`. Never `send_message`.
- Never name a customer relationship or outcome that isn't in the dossier or transcript. Don't fabricate proof points.
- Never quote a Vishay/Infineon/AWS/Western Digital number outside the values in Miguel's preferences file.
- Never describe Apollo Fire as "committed" — it's Evaluate stage.
- Don't volunteer Apollo Fire pricing ($75k → $625k by 2030) unless the customer is Apollo.
- Match proof points to ICP per Miguel's preferences (component manufacturer → Vishay; OEM → WD; EPD-driven → Distech/Qalex; hyperscaler → AWS TÜV).
- If unsure about a delivery timeline, don't commit one — flag it and ask Elmar/Piriya per Miguel's ownership rules.

## When not to ask

- Don't ask the user to confirm the source meeting choice if there's a clear most-recent external call. Just pick it and show the draft.
- Don't ask for the recipient list — it's in the meeting attendees.
- Don't ask which style to use — there's exactly one style file.
- Don't ask about subject format — apply the default unless the user overrides.

## When to ask

- Multiple plausible source meetings.
- Disqualifier customer (confirm intent).
- Missing transcript (ask for 2-3 bullets).
- User mentions a specific framing or proof point that conflicts with what the dossier shows for that customer.
