---
name: follow-up-email-trainer
description: One-time training skill that teaches Claude the user's exact follow-up email writing style by analyzing their recent external meetings and the follow-ups they sent afterward. Produces two reference files — a style document and a meeting dossier — that the follow-up-email skill reads as ground truth when drafting future post-call emails. Use whenever the user says "train you on my follow-up emails", "learn my email style", "analyze my follow-ups", "set up the follow-up email trainer", "build my email style guide", or any variant about teaching Claude how they personally write post-call emails. Trigger even if the user doesn't say "trainer" — if they ask Claude to study their past calls and emails so it can mimic their voice in future follow-ups, that's this skill. Sales / GTM / CS context throughout — this is for customer-facing email, not internal comms.
---

# Follow-Up Email Trainer

## What this skill does

Builds a one-time training corpus so the follow-up-email skill can draft post-call emails in the user's voice. Two output files, both editable markdown:

- `follow-up-email-style.md` — voice/structure/length patterns extracted from real sent emails
- `follow-up-email-dossier.md` — per-meeting history of what happened on each call and what email got sent (or not) afterward

These files are saved into the **follow-up-email skill's `references/` directory**, so the executor skill finds them automatically. See "Where files go" at the end.

This is a one-shot setup. After running it once, the user invokes the follow-up-email skill after each call.

## Speed and ordering

The skill is opinionated about ordering — the order minimizes wasted transcript pulls, which are the slowest step.

1. **Connectors first.** If any of the three categories is missing, stop.
2. **Granola meeting list as source of truth.** Skip the calendar — Granola already has the meetings the user actually attended (you wouldn't record a meeting you didn't engage with). Filter to external by participant domains in the Granola response.
3. **Email-first matching.** One broad Gmail query for the window. Match emails to meetings by recipient overlap + date. Only then pull transcripts for meetings that had a matched follow-up. This cuts transcript work by ~50% on a typical run, since meetings without follow-ups don't train the style anyway.
4. **Inline style analysis.** Email bodies are ~150 words each. The full corpus fits comfortably in context — no subagent needed.

Run all read-only. No external writes during training.

## Step 1 — Connector check

Three categories required:

- **Meeting recorder with transcript access** — Granola, Fireflies, Otter, or any MCP exposing call transcripts
- **Email provider** — Gmail, Outlook / Microsoft 365, or any email MCP
- **Calendar** — Google Calendar, Outlook Calendar, or any calendar MCP (used only as a fallback if Granola returns fewer than ~15 meetings)

Don't hardcode providers. Inspect which MCPs are connected and pick the first available in each category. If multiple are available, tell the user which you picked.

If any category has zero MCPs, stop and tell the user, with one to three sentences max on how to add them in Claude settings. Don't proceed past Step 1 if anything is missing.

## Step 2 — Pull external meetings from Granola

Call the Granola list-meetings tool for the last 90 days. From the response, filter to **external meetings**: those with at least one participant whose email domain differs from the user's own (`miguel@sluicebox.ai` → domain `sluicebox.ai`).

Cap at the **20 most recent external meetings**. The original spec said 30 but in practice style patterns saturate around 10-12 follow-up emails — going from 30 to 20 cuts work by a third with no measurable accuracy loss.

For each kept meeting, capture: granola_id, date, title, external attendee emails, internal attendee emails.

**Fallback to calendar if needed.** If Granola returns fewer than 15 external meetings in 90 days, supplement with calendar events (same filter rules: external attendees, exclude declined, exclude personal/holds). Otherwise skip the calendar pull entirely — it's the most expensive read in the flow and rarely adds signal.

Show the user the count and date range. Don't dump the full list.

## Step 3 — Pull sent emails in the window (one query)

One Gmail query for the entire window:

```
from:{user_email} in:sent after:{window_start_YYYY/MM/DD}
```

Paginate with the email MCP's page_token if results exceed one page. Capture for each thread: subject, recipients (to + cc), sent timestamp, snippet.

Don't pull full bodies yet — that's Step 5, and only for matched threads.

## Step 4 — Match emails to meetings

For each meeting, find candidate emails by:

1. **Time window:** sent between meeting end and meeting end + 5 days (120 hours)
2. **Recipient overlap:** at least one of the email's `to` or `cc` is in the meeting's external attendees list
3. **Semantic check on subject/snippet:** the subject or snippet should reference the meeting content — company name, person name, named deliverable, or specific topic. Generic "hello" or unrelated thread snippets fail this.

If multiple candidates, pick the one whose subject most directly references the meeting (company name in subject is the strongest signal). If none match, mark `email_status: not_sent` for that meeting.

Output of this step: a list of `(meeting_id, email_thread_id_or_null)` pairs.

## Step 5 — Pull transcripts and full bodies (only for matched pairs)

For meetings with a matched email:

1. Pull the Granola transcript via the transcript tool
2. Pull the full email body via `get_thread`
3. Determine reply status: any external recipient sent a message after the user's send → `got_reply`, else `no_reply`

For meetings **without** a matched email: skip the transcript. They become one-line gap entries in the dossier, not full blocks. Their value is "what didn't get followed up" — useful awareness, not useful for style training.

## Step 6 — Build the dossier file

Save to `follow-up-email-dossier.md` (path resolution in "Where files go").

Header (literal):

```markdown
# Follow-Up Email Dossier

Generated [date] from [N] external meetings (last 90 days). [N] had matched follow-up emails. [N] meetings had no follow-up sent — listed at the bottom as gap entries. Edit any entry to correct Claude's interpretation; the follow-up-email skill reads this file as ground truth.

---
```

Then meetings with matched emails, sorted date descending, each as:

```markdown
## [YYYY-MM-DD] [Company] — [Meeting title]

- **Attendees:** Name (Title, Company), ...
- **Objective:** One sentence.
- **What happened:** 3-6 bullets — decisions, demos, data shared, questions.
- **Objections raised:** Customer concerns. "None surfaced." if none.
- **Risks:** Specific stalling risks.
- **Qualification signals:** Budget, authority, timeline, named pain. Quote where useful.
- **Next-step lever:** Specific, not generic.

### Follow-up email sent
- **Sent:** [timestamp] ([N] hours after meeting)
- **Subject:** [subject]
- **To:** [recipients]
- **CC:** [recipients if present]
- **Reply status:** got_reply / no_reply
- **Body:**

> [full email body, blockquote-formatted, line breaks preserved]

---
```

Then a final section listing meetings with no follow-up sent (one line each, no full dossier):

```markdown
## Meetings with no follow-up sent

- [YYYY-MM-DD] [Company] — [Title] (external: [emails])
- [YYYY-MM-DD] [Company] — [Title] (external: [emails])
...
```

Field names are load-bearing — the executor skill parses them. Don't rename them.

GTM/sales framing throughout. Preserve customer terminology (Pilot, Foundation, Scale, BOM, PCF, EPD, LCA, MPN, ADU, MCDS, etc.).

## Step 7 — Build the style file

Save to `follow-up-email-style.md` (same path resolution).

Analyze all matched email bodies as a corpus. The corpus is small enough (~10-15 emails × ~150 words) to fit in context — do this inline, no subagent.

Use this top-level structure (section headers are load-bearing — the executor parses these):

```markdown
# Follow-Up Email Style Reference

Generated [date] from [N] matched follow-up emails across [N] external meetings.
Edit anything below to correct Claude's interpretation. Section headers are load-bearing — keep them as-is.

## Opening patterns

## Length norms

## CTA style

## Tone markers

## Personalization patterns

## Reply-correlated patterns (weak signal — label as such)

## Verbatim signature block
```

What goes in each section:

- **Opening patterns:** 2-4 verbatim opening lines from real emails + a short pattern summary.
- **Length norms:** Mean, median, range, paragraph count, bullet density.
- **CTA style:** How asks are made. 2-3 verbatim CTAs.
- **Tone markers:** Recurring words/phrases. Words/phrases never used (pull from user preferences if present — em dashes, "leverage", "synergize", "circle back", absolutes, etc.).
- **Personalization patterns:** What's customized vs templated. Subject line format. Sign-off pattern.
- **Reply-correlated patterns:** Compare the emails that got replies vs no reply. Label clearly as weak signal if sample < 10. Honest "insufficient sample" is a valid finding — don't invent patterns.
- **Verbatim signature block:** Copy exact sign-off from a recent email. Don't reconstruct.

Don't fabricate. If a section has weak evidence, say so. The file is editable — the user can fill gaps themselves.

## Step 8 — Confirm completion

Tell the user, plainly:

- Number of external meetings analyzed
- Number of follow-up emails matched (and how many had no follow-up)
- Where the two files are saved (clickable `computer://` links to both)
- That both files are editable markdown — they can correct attendees, fix style misreadings, add notes Claude missed, and the executor skill will pick up the edits

Then close with this exact line (verbatim, no rephrasing — the user wants this specific call-to-action so they remember the trigger phrase):

> Training complete. Simply tell me "write followup" in the future, and I will apply this to your latest call. Or you can specify "write follow up CUSTOMER NAME + DATE".

Keep the rest tight. No preamble, no celebration. The closing line is the only CTA.

## Where files go

The two files must land where the follow-up-email skill can find them. The executor reads from its own `references/` directory.

**At runtime, resolve the executor's path:**

1. Look at the `<available_skills>` list in the current session. Find the entry whose `name` is `follow-up-email`. Read its `location` field.
2. Write the two files to `{executor_location}/references/follow-up-email-style.md` and `{executor_location}/references/follow-up-email-dossier.md`. Create the `references/` directory if it doesn't exist.

**Fallback if the executor isn't installed:**

Write to `~/Library/Application Support/Sluicebox/follow-up-email/` (macOS) or `~/.sluicebox/follow-up-email/` (Linux). Tell the user the executor skill isn't installed yet and the files are at the fallback path — they'll need to install the follow-up-email skill and copy the files into its `references/` directory.

This keeps the two skills loosely coupled but tied to a known shared location.

## Constraints worth restating

- **Read-only on connectors.** No emails sent, no calendar events created. The connector-safety rule applies — if you find yourself about to write to any external system, stop.
- **20 meeting cap.** Hitting the cap means using the 20 most recent external meetings, not 20 from across the 90 days.
- **GTM context throughout.** Internal-only meetings shouldn't appear because Step 2 filters them out, but if any slip through, drop them.
- **Don't blend confirmed style facts with inferences.** Flag inferences clearly. The executor will treat unflagged claims as rules.
- **Editable markdown, stable section headers.** The user needs to correct things; the executor needs to parse them. Both need to be true.

## When to stop and ask

- Connectors missing — stop after Step 1.
- Zero matched emails across all meetings — something is wrong (wrong account connected, wrong domain). Tell the user before producing a useless style file.
- Granola returns fewer than 5 external meetings — likely the account is light or the time window is wrong. Tell the user before producing thin style data.

## When not to ask

- Don't ask which template to use for the style doc — it's specified above.
- Don't ask permission to skip transcript-less meetings — drop them or use them as gap entries, just don't pull a transcript.
- Don't ask whether to overwrite existing files — overwrite them. This is a retraining, the previous version is stale.
