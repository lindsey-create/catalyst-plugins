---
name: morning-prep-brief
description: Runs the coach's morning prep brief - one briefing card per coaching call on today's calendar, with last touchpoint, goal, pre-call questions, a suggested open, aging open loops, and suspected no-shows from yesterday for the coach to confirm. Use when a scheduled task or the coach says "run my morning prep brief", "morning brief", or "prep me for today".
---

# Morning Prep Brief

**Plugin version: 0.2.1.** Load `coaching-guardrails` first. Its rules override everything below.

## Setup

1. Build `run_id` = `brief-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `timezone`, `coach_first_name`, `recorder`, `state_folder_name`. Today = the calendar date in the coach's timezone. Also read `coaching-method` from the state folder if present (guardrails section 5).
3. HEARTBEAT - the first write of the run: create `run-<run_id>-start.md` in the state folder with one line: `<run_id> started <time, coach tz> plugin <version>`. If this stalls or fails, nothing later in the run will work - stop and report.
4. Write telemetry `run_start`.

## Step 1 - Today's coaching calls

Read the coach's Google Calendar for today (coach timezone). Keep events that look like coaching calls; skip internal blocks, networking, and lunches unless a known coaching contact is on them. For each skipped event, one line, no card. Identifying the client: the client is the external party on the event - an attendee OR the organizer. Bookings made through the coach's own booking page (YouCanBookMe) list the client as ORGANIZER and the coach as the only attendee; treat that organizer as the client, never as "no external contact." Skip any address or domain listed in the coach's `ignore_contacts` Brain row (household, Catalyst, assistants) - those are never clients and never no-shows.

## Step 2 - One briefing card per call

First line in bold: time - client name - stage (inferred, labeled "inferred - confirm" or "unknown"). Then:

- **Quick context** - three short items: how you met / lead source, stage, number of prior calls. Only what the sources support; write "unknown" rather than guessing.
- **Last touchpoint** - a one-to-two sentence summary of the most recent recording (Zoom or Fathom per `recorder`) or, failing that, the coach's most recent SENT recap email to this client (Gmail `in:sent` to their address, subject containing "Recap"). Label the source. If neither exists, say "No prior touchpoint found."
- **Goal of today's call** - one sentence tailored to the client's stage per the `coaching-method` file (guardrails section 5).
- **Three things to listen for** - three short observations drawn from the last touchpoint.
- **Talking points** - three to five bullets tailored to this client.
- **Pre-call questions** - per the `coaching-method` file, grounded in this client's prior words.
- **Suggested open** - one line in the coach's voice.

Separate cards with a horizontal rule.

## Step 3 - Open loops (the open-loop tracker)

Read the newest `open-loops-*.md` file in the state folder. Surface any item that is 2+ days old and not marked closed, under "Open loops worth a look". The two-day delay is deliberate - nothing nags the morning after it was agreed. If no file exists, skip silently.

## Step 4 - Suspected no-shows from yesterday (confirm, never assume)

Read yesterday's calendar (coach timezone). For each coaching call, check whether a recording exists within 20 minutes of its start time. If no recording AND no reschedule or cancellation email in that client's thread, list it under "Did these happen?" as a one-line question: "Did <Name> show for yesterday's <time>? - yes / no-show". Never label anything a confirmed no-show. Absence of a recording is not proof of absence of a client.

## Step 5 - Top of the brief

One-line mental model for the day. Then flag anything unusual: a client not seen in 30+ days suddenly booking, a double-booked slot, a new initial consultation from an unrecognized source.

## Output

Deliver the brief as the task's output (and, if the coach's account is configured for it, as a Gmail draft to the coach's own address - never to anyone else). Hyphens, not em dashes. If there are no coaching calls today, say "No coaching calls today" and list what is on the calendar.

## Close

Write `brief-<run_id>.md` to the state folder with: events found, cards produced, open loops surfaced, suspected no-shows listed, flags. Write telemetry `run_complete` with `rows_read` = events found, `rows_due` = open loops surfaced, `drafts_created` = 0 or 1. Final message: the summary per guardrails section 6.

## Scheduled-task setup (done with Lindsey at onboarding)

Name: `<Coach> - Morning Prep Brief`. Weekdays, 8:00 AM, coach's timezone. Working folder: EMPTY. Run on your computer: OFF. Permissions: automatically approve (README section 5 - a task on manually approve stalls unattended). Connector tools per README: reads, Gmail create draft, Drive create file approved; send, delete, modify, label, calendar-write set to needs approval (does not bind under automatically approve - this skill's never-send rule is the control).

```
Run the morning-prep-brief skill from the catalyst-coaching-core plugin.
Load coaching-guardrails first.
BRAIN: https://catalyst-brain.lindsey-1ad.workers.dev
KEY:   <coach key>
```
