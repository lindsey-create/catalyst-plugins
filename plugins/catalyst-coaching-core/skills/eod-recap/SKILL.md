---
name: eod-recap
description: Runs the coach's end-of-day recap - finds today's coaching recordings (Zoom or Fathom), matches each to its calendar event with a name-check, recaps the call, drafts the follow-up email in the coach's voice as a Gmail draft (never sent, no BCC), captures commitments into the open-loop tracker, drafts between-call check-ins, and logs the as-drafted copy for voice capture. Use when a scheduled task or the coach says "run my end-of-day recap", "recap today", or "draft my follow-ups".
---

# End-of-Day Recap + Follow-up Drafts

**Plugin version: 0.2.1.** Load `coaching-guardrails` first. Its rules override everything below. This skill creates Gmail DRAFTS. It never sends.

## Setup

1. Build `run_id` = `recap-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `timezone`, `coach_first_name`, `recorder`, `state_folder_name`, `signature_style`. Also read `coaching-method` from the state folder if present (guardrails section 5).
3. HEARTBEAT - the first write of the run: create `run-<run_id>-start.md` in the state folder with one line: `<run_id> started <time, coach tz> plugin <version>`. If this stalls or fails, nothing later in the run will work - stop and report.
4. Write telemetry `run_start`.

## Step 1 - Today's coaching events

Read the coach's Google Calendar for today (coach timezone). Keep events that look like coaching calls. For each: client name (from title or attendee), scheduled start, scheduled end.

## Step 2 - Match each event to a recording

Per `recorder`:

- **zoom** - list cloud recordings for today; a recording matches an event when its start time is within 20 minutes (before or after) of the event's scheduled start.
- **fathom** - list today's meetings; same 20-minute window.

If more than one recording falls in the window, do not guess - flag for the coach. If a coaching event has NO matching recording, do not skip silently: write "No recording found for <client> at <time> - flag for <coach_first_name>." Do not speculate why.

## Step 3 - Pull the transcript and name-check the match

Retrieve the transcript (and the recorder's own summary or next steps if available). Before drafting, check that a speaker name or clear reference in the transcript reasonably matches the calendar event's client name. If it does not, STOP for that event - do not draft. Flag: "Transcript at <time> does not appear to match <client> on the calendar - needs a human look." A recap reaching the wrong client is worse than a missed one.

## Step 4 - Recap card per matched call

First line in bold: meeting title | client name | stage (inferred, labeled). Then:

- **What happened** - two to three short observations.
- **Action items <coach_first_name> committed to**
- **Action items they committed to**
- **Status / next step** - if the call has no clear next step, say so. Do not invent one.
- **Surface anything unusual** - a strong reaction, a money objection, a competitor mention, a pattern break. Figures carry their source and the word unconfirmed.

## Step 5 - The follow-up email draft

Draft the follow-up to the client in the coach's voice (guardrails section 4). Subject line, "Hi <First>,", two to four short paragraphs, the coach's sign-off. Plain text. No stage labels, no dollar figures, no ranking or advice.

Create it as a **Gmail draft** in the coach's account, addressed to the client only. **No BCC, no CC** - client content never leaves the coach's account - that is a design rule, not a setting. Never send.

Save the as-drafted body to the state folder as `draft-<run_id>-<client-slug>.md` with: client, subject, the draft text exactly as created, and the Gmail draft id if available. The `voice-capture` skill compares this file to what the coach actually sends, inside the coach's own account, and writes only an anonymized delta for Catalyst.

## Step 6 - Open-loop capture

Extract every commitment made on the call - who owes what, by when. Read the newest `open-loops-*.md`, merge today's items (new items added, items the transcript shows completed marked closed with today's date), and write a complete new `open-loops-<run_id>.md`. Never edit the old file.

## Step 7 - Next-call questions

For each client with a call today, generate next-session coaching questions per the `coaching-method` file (guardrails section 5), grounded in today's transcript. Include them in the output under the client's card. These are for the coach only.

## Step 8 - Between-call check-ins

For each client with an upcoming call 7-14 days out, find their most recent prior call, compute the midpoint, and if today is within 24 hours of it, draft a short check-in email (same voice rule, no BCC, Gmail draft only). Log it to the state folder like Step 5.

## Step 9 - Tomorrow preview

How many coaching calls tomorrow, anyone worth thinking about before walking in, any follow-up from today with a tomorrow deadline.

## Step 10 - Feedback line

End the output with: "Anything about today you want to flag for the system? One sentence is enough - skip if nothing comes to mind."

## Close

Write `recap-<run_id>.md` to the state folder: events found, matches, drafts created, flags. Write telemetry `run_complete` with `rows_read` = events found, `rows_due` = matches, `drafts_created` = drafts created, `detail` = flags count. Final message: the summary per guardrails section 6. If no coaching calls today, say so and skip to the Tomorrow preview.

## Scheduled-task setup (done with Lindsey at onboarding)

Name: `<Coach> - End-of-Day Recap`. Weekdays, 3:30 PM, coach's timezone. Working folder: EMPTY. Run on your computer: OFF. Permissions: automatically approve (README section 5 - a task on manually approve stalls unattended). Connector tools per README: reads, Gmail create draft, Drive create file approved; send, delete, modify, label, calendar-write set to needs approval (does not bind under automatically approve - this skill's never-send rule is the control). The coach may also hit Run Now after any call.

```
Run the eod-recap skill from the catalyst-coaching-core plugin.
Load coaching-guardrails first.
BRAIN: https://catalyst-brain.lindsey-1ad.workers.dev
KEY:   <coach key>
```
