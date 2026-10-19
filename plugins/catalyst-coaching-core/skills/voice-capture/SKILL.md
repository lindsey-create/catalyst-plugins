---
name: voice-capture
description: Compares what the coach actually SENT this week to what the system DRAFTED (email track), and compares the coach's edited vision statements to the versions the system generated (vision-voice track). Reads from and writes to the coach's own private state folder only, maintaining two separate compounding voice files - one for email voice, one for vision voice - that the drafting skills read. Nothing leaves the coach's account. The coach's own `coach-voice` file always overrides both. Use when a scheduled task or the coach says "run voice capture", "what did the coach change", "voice profile", or "vision voice".
---

# Voice Capture (edit capture)

**Plugin version: 0.3.3.** Load `coaching-guardrails` first. This skill reads and writes files. It never drafts to a client and never sends.

**Write boundary (non-negotiable):** voice-capture READS from and WRITES to the coach's private `state_folder_name` only - every output, including the run heartbeat. It has no write path out of the coach's account; nothing but counts-only telemetry ever reaches Catalyst. It NEVER creates or edits `coach-voice`, `voice-samples`, `coaching-method`, or any other coach-authored file - it writes only its own dated compounding files (2026-09-18 decision: the Delta folder is out; supersedes the 2026-08-27 delta split and Option B).

## Why this exists

The system's drafts only get better if the coach's edits come back. The coach edits in Gmail (email drafts) and in the vision-statement Google Doc (vision statements), not in a form. So the system logs what it produced - email drafts by `eod-recap`, vision Part A baselines by `vision-statement` - and this skill compares each to what the coach actually kept, inside the coach's own account. Client content never leaves the coach's Google account, and neither does the voice profile: the compounding files stay in the coach's state folder and are read by the drafting skills in that same account. Catalyst receives counts-only telemetry and never reads a coach's voice profile.

Two independent tracks. The **email track** learns the coach's everyday email voice. The **vision-voice track** learns how the coach edits AI-drafted vision statements - a different, higher-stakes voice. They produce separate compounding files and separate proposed profiles, and are never blended.

## Setup

1. Build `run_id` = `voice-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `timezone`, `coach_first_name`, `state_folder_name`, `signature_style`.
3. Write telemetry `run_start` (before reading any source or writing any file).
4. HEARTBEAT - the first Drive write: create `run-<run_id>-start.md` in the **state folder** with one line: `<run_id> started <time, coach tz> plugin <version>`. On failure, apply the guardrails section 3 retry-once rule; if it still fails, stop and report. Nothing later in the run works without it.
5. Window = the last 7 days, coach timezone.

## Email track

### E1 - Pair drafts with sent emails

Read this window's `draft-*.md` files from the state folder. For each, look in Gmail `in:sent` (restricted to `gmail_label` if configured) for the sent email to the same recipient with the same or a closely edited subject, sent after the draft was created. Pair them. Unpaired drafts = not sent (count them; do not speculate why).

### E2 - Describe each edit

For each pair, describe the edit in style terms, not content terms:

- Length: shorter / longer / same (rough word counts).
- Greeting or sign-off changed? To what.
- Sentences removed entirely - what kind (detail, pleasantry, next-step, a figure).
- Sentences added - what kind.
- Word swaps that recur (e.g. "conversation" to "chat", "touch base" removed).
- Formatting: paragraphs merged or split, bullets added or removed.
- Anything the coach added that the system could not have known (a personal detail, a scheduling specific) - label it "coach-only knowledge", not a voice miss.

Never copy client content into the compounding file beyond the minimum phrase needed to show a word swap. No names of clients in the compounding file - use "client A, B, C".

### E3 - Pattern summary

Across all pairs: the three most frequent edits. Then a one-paragraph proposed update to the EMAIL voice profile in plain language ("drop the second paragraph of context; sign off with first name only; prefer 'chat' over 'conversation'"). Mark it PROPOSED. It is written into the compounding email-voice file, which the drafting skills read; the coach's own `coach-voice` file overrides it. Neither this skill nor Catalyst ever edits `coach-voice`.

### E4 - Write the email compounding file

Read the newest `compounding-voice-email-*.md` in the state folder (list, sort filename dates descending, take newest; none = first run, empty inheritance). Create `compounding-voice-email-<YYYY-MM-DD>.md` in the **state folder** - if one for today already exists, suffix with the run_id. Inherit-then-append, never a diff: the newest file is always complete on its own (the `client-profile` snapshot contract, guardrails section 3). Contents, in order: the header block below verbatim; inherited patterns from the prior file; this window's pairs found and unpaired drafts; per-pair edit descriptions (client-anonymized); the pattern summary; the PROPOSED email voice-profile update. Anonymization still applies even though nothing leaves the account - no client names, no client content beyond the minimum phrase needed to show a word swap.

    # Compounding voice profile - EMAIL track
    SYSTEM-MAINTAINED - written by the voice-capture skill each week.
    Your own `coach-voice` file OVERRIDES everything here.
    To change how your drafts sound, edit `coach-voice` - not this file.
    Generated: <YYYY-MM-DD> | Prior: <prior filename, or "none"> | Plugin <version from this file's header>

## Vision-voice track (separate from the email track - never blended)

This track reads only Part A of vision statements - the shareable text. It never reads Part B or Part C (coach-only working notes). It is the mirror image of what `client-profile` reads from the same Docs (that skill reads the header and Part B only, never Part A), so the two never overlap.

### V1 - Find in-window vision baselines

Read `vision-baseline-<client-slug>-<run_id>.md` files (`source: vision-doc`) from the state folder - each holds the exact Part A a `vision-statement` run generated ("before"). A baseline is IN WINDOW if its matching vision Doc was modified in the last 7 days (the coach edited it this week). If no baselines are in window, record zero and skip to Close - there is nothing to diff.

### V2 - Pair each baseline with its edited Doc and extract Part A

For each in-window baseline, find the matching `vision-<client-slug>-<run_id>` Google Doc in the state folder (same client-slug and run_id from the baseline header). Read the current Doc and extract PART A ONLY: the text above the line "FOR <COACH_FIRST_NAME> ONLY - do not share below this line", starting at the first-person forward-looking opening chosen from `coach_vision_opening_patterns` and ending at whatever line the coach's `coach_vision_ai_disclosure` sets (or, if the disclosure row is empty, at the last non-empty paragraph before the do-not-share line). Skip the coach-facing header above it. NEVER read Part B or Part C. If the do-not-share line or the Doc is missing, skip that baseline and note the skip (no client detail). If the Doc is unchanged from the baseline, record "unedited - no delta" and skip.

### V3 - Describe each edit in style terms (stricter anonymization)

Diff the baseline Part A (system "before") against the Doc's current Part A (coach "after"). Describe edits exactly as E2 does - length, opening pattern, sentences removed/added and of what kind, recurring word swaps, formatting, tense or point-of-view drift. Anonymize HARDER than the email track, because a vision statement is a deeply personal first-person narrative: never a client name (use "client A, B, C"); never a life detail, figure, role, family specific, or named fear; quote at most the minimum phrase needed to show a word-level swap, never a full sentence of vision content. Anything the coach added that the system could not have known is "coach-only knowledge" - name its TYPE only, never its content.

### V4 - Vision-voice pattern summary + proposed profile

Across all pairs this run: the three most frequent edit patterns. Then a one-paragraph PROPOSED update to the VISION-VOICE profile in plain style language - kept separate from the email `coach-voice` profile. Mark it PROPOSED. It is written into the compounding vision-voice file, which `vision-statement` reads; the coach's own `coach-voice` file overrides it. Neither this skill nor Catalyst ever edits `coach-voice`. If fewer than two pairs exist, say "not enough vision edits to read a pattern yet" and propose nothing - do not manufacture a pattern.

### V5 - Write the vision compounding file

Read the newest `compounding-voice-vision-*.md` in the state folder (same lookup as E4). Create `compounding-voice-vision-<YYYY-MM-DD>.md` in the **state folder**, same inherit-then-append contract. Contents: the header block below verbatim; inherited patterns; baselines found in window; pairs diffed; unedited/skipped baselines (counts only); per-pair anonymized style descriptions; the pattern summary; the PROPOSED vision-voice profile update. Same anonymization as the email file and stricter: no client names, minimum phrasing, never any Part B or Part C content. This is a SEPARATE file from `compounding-voice-email-*` and the two are NEVER merged.

    # Compounding voice profile - VISION track
    SYSTEM-MAINTAINED - written by the voice-capture skill each week.
    Your own `coach-voice` file OVERRIDES everything here.
    To change how your vision drafts sound, edit `coach-voice` - not this file.
    Generated: <YYYY-MM-DD> | Prior: <prior filename, or "none"> | Plugin <version from this file's header>

## Close

Telemetry `run_complete` with `rows_read` = (email drafts in window) + (vision baselines in window), `rows_due` = (email pairs) + (vision pairs), `drafts_created` = 0, `detail` = `ok`. Final message: email pairs found + top three email edits; vision pairs found + top three vision edits; which two compounding files were written, and where; and any flag raised. If a track has fewer than two pairs, say so for that track and do not manufacture a pattern there.

## Scheduled-task stub

Name: `<Coach> - Voice Capture`. Weekly, Sunday 6:00 PM, coach's timezone. Working folder: EMPTY. Run on your computer: OFF. Permissions: automatically approve (README section 5 - a task on manually approve stalls unattended). Connector tools per README: reads, Gmail create draft, Drive create file approved; send, delete, modify, label, calendar-write set to needs approval (does not bind under automatically approve - this skill's never-send rule is the control).

```
Run the voice-capture skill from the catalyst-coaching-core plugin.
Load coaching-guardrails first.
BRAIN: https://catalyst-brain.lindsey-1ad.workers.dev
KEY:   <coach key>
```
