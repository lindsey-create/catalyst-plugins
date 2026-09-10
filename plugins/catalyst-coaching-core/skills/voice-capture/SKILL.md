---
name: voice-capture
description: Compares what the coach actually SENT this week to what the system DRAFTED, and writes a voice-delta file so the coach's edits teach the system. Runs weekly in the coach's account; Catalyst reads the delta and updates the voice profile. Use when a scheduled task or the coach says "run voice capture", "what did the coach change", or "voice delta".
---

# Voice Capture (edit capture)

**Plugin version: 0.2.1.** Load `coaching-guardrails` first. This skill reads and writes files. It never drafts to a client and never sends.

## Why this exists

The system's drafts only get better if the coach's edits come back. The coach edits in Gmail, not in a form. So: every draft is logged as-created (by `eod-recap`), and this skill compares it to the coach's own sent mail inside the coach's account. Client content never leaves the coach's Google account; Catalyst receives only the anonymized delta file.

## Setup

1. Build `run_id` = `voice-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `timezone`, `coach_first_name`, `state_folder_name`, `signature_style`.
3. HEARTBEAT - the first write of the run: create `run-<run_id>-start.md` in the state folder with one line: `<run_id> started <time, coach tz> plugin <version>`. If this stalls or fails, nothing later in the run will work - stop and report.
4. Write telemetry `run_start`.
5. Window = the last 7 days, coach timezone.

## Step 1 - Pair drafts with sent emails

Read this window's `draft-*.md` files from the state folder. For each, look in Gmail `in:sent` (restricted to `gmail_label` if configured) for the sent email to the same recipient with the same or a closely edited subject, sent after the draft was created. Pair them. Unpaired drafts = not sent (count them; do not speculate why).

## Step 2 - Describe each edit

For each pair, describe the edit in style terms, not content terms:

- Length: shorter / longer / same (rough word counts).
- Greeting or sign-off changed? To what.
- Sentences removed entirely - what kind (detail, pleasantry, next-step, a figure).
- Sentences added - what kind.
- Word swaps that recur (e.g. "conversation" to "chat", "touch base" removed).
- Formatting: paragraphs merged or split, bullets added or removed.
- Anything the coach added that the system could not have known (a personal detail, a scheduling specific) - label it "coach-only knowledge", not a voice miss.

Never copy client content into the delta beyond the minimum phrase needed to show a word swap. No names of clients in the delta file - use "client A, B, C".

## Step 3 - Pattern summary

Across all pairs: the three most frequent edits. Then a one-paragraph proposed update to the voice profile in plain language ("drop the second paragraph of context; sign off with first name only; prefer 'chat' over 'conversation'"). Mark it PROPOSED. Catalyst decides whether to adopt it.

## Step 4 - Write the delta

Create `voice-delta-<run_id>.md` in the state folder: pairs found, unpaired drafts, per-pair edit descriptions (client-anonymized), the pattern summary, the PROPOSED profile update.

## Close

Telemetry `run_complete` with `rows_read` = drafts in window, `rows_due` = pairs found, `drafts_created` = 0, `detail` = unpaired count. Final message: pairs found, top three edits, where the delta file is. If fewer than two pairs exist, say "not enough sends to read a pattern yet" and stop - do not manufacture a pattern.

## Scheduled-task stub

Name: `<Coach> - Voice Capture`. Weekly, Sunday 6:00 PM, coach's timezone. Working folder: EMPTY. Run on your computer: OFF. Permissions: automatically approve (README section 5 - a task on manually approve stalls unattended). Connector tools per README: reads, Gmail create draft, Drive create file approved; send, delete, modify, label, calendar-write set to needs approval (does not bind under automatically approve - this skill's never-send rule is the control).

```
Run the voice-capture skill from the catalyst-coaching-core plugin.
Load coaching-guardrails first.
BRAIN: https://catalyst-brain.lindsey-1ad.workers.dev
KEY:   <coach key>
```
