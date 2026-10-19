---
name: weekly-metrics-digest
description: Runs the coach's weekly metrics digest - calls scheduled vs held, suspected no-shows, drafts created vs sent, open loops outstanding, and a short read on the week. Coach's own data only, never cross-coach. Use when a scheduled task or the coach says "run my weekly digest", "weekly metrics", or "how did my week go".
---

# Weekly Metrics Digest

**Plugin version: 0.3.3.** Load `coaching-guardrails` first. Its rules override everything below. This digest is for the coach's eyes. It never compares this coach to any other coach.

## Setup

1. Build `run_id` = `digest-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `timezone`, `coach_first_name`, `recorder`, `state_folder_name`.
3. HEARTBEAT - the first write of the run: create `run-<run_id>-start.md` in the state folder with one line: `<run_id> started <time, coach tz> plugin <version>`. If this stalls or fails, nothing later in the run will work - stop and report.
4. Write telemetry `run_start`.
5. Week = Monday through today, coach timezone.

## Sources (read only these)

- Google Calendar for the week - coaching events.
- Recorder (`zoom` or `fathom`) for the week - recordings.
- Gmail `in:sent` for the week - the coach's sent emails to the clients on this week's calendar (restricted to `gmail_label` if configured).
- The state folder - this week's `recap-*.md`, `brief-*.md`, `draft-*.md`, and the newest `open-loops-*.md`.

## Metrics

| Metric | How |
|---|---|
| Calls scheduled | Coaching events on the calendar this week |
| Calls held | Events with a matching recording (20-minute window) |
| Suspected no-shows | Scheduled, no recording, no reschedule or cancellation in the thread. Label suspected, never confirmed. Source: calendar vs recordings - NOT YouCanBookMe. |
| Drafts created | `draft-*.md` files this week |
| Drafts sent | Sent emails this week whose recipient and subject match a draft file |
| Edit rate | Of drafts sent, how many differ materially from the as-drafted copy (rough judgment is fine; voice-capture does the precise work) |
| Open loops outstanding | Items in the newest `open-loops-*.md` not marked closed, with age in days |
| Runs this week | From the state folder: how many brief and recap runs wrote a file. If a weekday is missing a run, say which - silence is not success. |

## The read

Three to five sentences in plain language: what moved, what stalled, one pattern worth noticing. No praise padding. If a metric cannot be computed, say so rather than estimating.

## Output

Deliver as the task output AND as an email sent to `coach_email` titled "Your week - <dates>" (the one send the guardrails permit - never to anyone else, no CC, no BCC). Send it through the Gmail connector's `htmlBody` field (minimal HTML: `<p>`, `<ul>`/`<li>`, and one plain `<table>` for the metrics - no CSS, no images); never markdown in the plain-text `body` field. If `coach_email` is missing, deliver as task output only and say so in the final message. If the send fails, follow the send-failure line in guardrails rule 1: task output plus at most one content-free notification, nothing else. Hyphens, not em dashes.

## Close

Write `<run_id>.md` to the state folder with the metrics table (the run_id already starts with `digest-`; never prepend it again). Telemetry `run_complete` with `rows_read` = calls scheduled, `rows_due` = open loops outstanding, `drafts_created` = 0 - the coach email is a send, not a draft, and is never counted. Final message: summary per guardrails section 6.

## Scheduled-task stub

Name: `<Coach> - Weekly Metrics Digest`. Weekly, Friday 4:00 PM, coach's timezone. Working folder: EMPTY. Run on your computer: OFF. Permissions: automatically approve (README section 5 - a task on manually approve stalls unattended). Connector tools per README: reads, Gmail create draft, Drive create file approved; Gmail send is used for exactly one message per run, to `coach_email` only; delete, modify, label, calendar-write set to needs approval (does not bind under automatically approve - guardrails rule 1 is the control).

```
Run the weekly-metrics-digest skill from the catalyst-coaching-core plugin.
Load coaching-guardrails first.
BRAIN: https://catalyst-brain.lindsey-1ad.workers.dev
KEY:   <coach key>
```
