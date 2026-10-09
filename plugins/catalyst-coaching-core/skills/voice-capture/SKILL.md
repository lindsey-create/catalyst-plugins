---
name: voice-capture
description: Compares what the coach actually SENT this week to what the system DRAFTED (email track), and compares the coach's edited vision statements to the versions the system generated (vision-voice track). Reads from the coach's private state folder and writes both anonymized style-delta files to the coach's view-only Delta folder ONLY, so the coach's edits teach the system without client content leaving the coach's account. Runs weekly in the coach's account; Catalyst reads the deltas and updates the two voice profiles separately. Use when a scheduled task or the coach says "run voice capture", "what did the coach change", "voice delta", or "vision voice".
---

# Voice Capture (edit capture)

**Plugin version: 0.3.2.** Load `coaching-guardrails` first. This skill reads and writes files. It never drafts to a client and never sends.

**Write boundary (non-negotiable):** voice-capture READS from the coach's private `state_folder_name` and WRITES only to the coach's view-only `delta_folder_name` - every output, including the run heartbeat. It has no write path into the state folder. If the Delta folder is missing it HALTS and writes nothing; it never falls back to the state folder (2026-08-27 delta split; Option B; fail-safe).

## Why this exists

The system's drafts only get better if the coach's edits come back. The coach edits in Gmail (email drafts) and in the vision-statement Google Doc (vision statements), not in a form. So the system logs what it produced - email drafts by `eod-recap`, vision Part A baselines by `vision-statement` - and this skill compares each to what the coach actually kept, inside the coach's own account. Client content never leaves the coach's Google account; Catalyst receives only the anonymized delta files, in the view-only Delta folder.

Two independent tracks. The **email track** learns the coach's everyday email voice. The **vision-voice track** learns how the coach edits AI-drafted vision statements - a different, higher-stakes voice. They produce separate deltas and separate proposed profiles, and are never blended.

## Setup

1. Build `run_id` = `voice-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `timezone`, `coach_first_name`, `state_folder_name`, `delta_folder_name`, `signature_style`.
3. Write telemetry `run_start` (before reading any source or writing any file).
4. **Resolve the Delta folder - before any Drive write.** Locate the folder named by `delta_folder_name`. If `delta_folder_name` is unset, or no such folder exists, HALT: write nothing to Drive (no heartbeat, no delta, and never anything to the state folder), emit telemetry `run_complete` with `detail=no-delta-dir`, and tell the coach in the final message that the view-only Delta folder named by `delta_folder_name` must be provisioned. Do not create the folder yourself; the coach provisions it.
5. HEARTBEAT - the first Drive write, and the Delta-writability probe: create `run-<run_id>-start.md` in the **Delta folder** with one line: `<run_id> started <time, coach tz> plugin <version>`. On failure, apply the guardrails section 3 retry-once rule; if it still fails, the Delta folder is not writable - stop, emit `run_complete` `detail=no-delta-dir`, and report. Nothing later in the run works without it.
6. Window = the last 7 days, coach timezone.

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

Never copy client content into the delta beyond the minimum phrase needed to show a word swap. No names of clients in the delta file - use "client A, B, C".

### E3 - Pattern summary

Across all pairs: the three most frequent edits. Then a one-paragraph proposed update to the EMAIL voice profile in plain language ("drop the second paragraph of context; sign off with first name only; prefer 'chat' over 'conversation'"). Mark it PROPOSED. Catalyst decides whether to adopt it into `coach-voice`.

### E4 - Write the email delta

Create `voice-delta-<run_id>.md` in the **Delta folder**: pairs found, unpaired drafts, per-pair edit descriptions (client-anonymized), the pattern summary, the PROPOSED email voice-profile update. This is the only email-track content that crosses to Catalyst - no client names, no client content beyond the minimum phrase needed to show a word swap.

## Vision-voice track (separate from the email track - never blended)

This track reads only Part A of vision statements - the shareable text. It never reads Part B or Part C (coach-only working notes). It is the mirror image of what `client-profile` reads from the same Docs (that skill reads the header and Part B only, never Part A), so the two never overlap.

### V1 - Find in-window vision baselines

Read `vision-baseline-<client-slug>-<run_id>.md` files (`source: vision-doc`) from the state folder - each holds the exact Part A a `vision-statement` run generated ("before"). A baseline is IN WINDOW if its matching vision Doc was modified in the last 7 days (the coach edited it this week). If no baselines are in window, record zero and skip to Close - there is nothing to diff.

### V2 - Pair each baseline with its edited Doc and extract Part A

For each in-window baseline, find the matching `vision-<client-slug>-<run_id>` Google Doc in the state folder (same client-slug and run_id from the baseline header). Read the current Doc and extract PART A ONLY: the text above the line "FOR <COACH_FIRST_NAME> ONLY - do not share below this line", starting at the first-person forward-looking opening chosen from `coach_vision_opening_patterns` and ending at whatever line the coach's `coach_vision_ai_disclosure` sets (or, if the disclosure row is empty, at the last non-empty paragraph before the do-not-share line). Skip the coach-facing header above it. NEVER read Part B or Part C. If the do-not-share line or the Doc is missing, skip that baseline and note the skip (no client detail). If the Doc is unchanged from the baseline, record "unedited - no delta" and skip.

### V3 - Describe each edit in style terms (stricter anonymization)

Diff the baseline Part A (system "before") against the Doc's current Part A (coach "after"). Describe edits exactly as E2 does - length, opening pattern, sentences removed/added and of what kind, recurring word swaps, formatting, tense or point-of-view drift. Anonymize HARDER than the email track, because a vision statement is a deeply personal first-person narrative: never a client name (use "client A, B, C"); never a life detail, figure, role, family specific, or named fear; quote at most the minimum phrase needed to show a word-level swap, never a full sentence of vision content. Anything the coach added that the system could not have known is "coach-only knowledge" - name its TYPE only, never its content.

### V4 - Vision-voice pattern summary + proposed profile

Across all pairs this run: the three most frequent edit patterns. Then a one-paragraph PROPOSED update to the VISION-VOICE profile in plain style language - kept separate from the email `coach-voice` profile. Mark it PROPOSED; Catalyst decides whether to adopt it into a vision-specific voice reference. If fewer than two pairs exist, say "not enough vision edits to read a pattern yet" and propose nothing - do not manufacture a pattern.

### V5 - Write the vision-voice delta

Create `vision-voice-delta-<run_id>.md` in the **Delta folder**: baselines found in window, pairs diffed, unedited/skipped baselines (counts only), per-pair anonymized style descriptions, the pattern summary, the PROPOSED vision-voice profile update. Same boundary as the email delta and stricter: no client names, minimum phrasing, and never any Part B or Part C content. This is a SEPARATE file from `voice-delta-*`.

## Close

Telemetry `run_complete` with `rows_read` = (email drafts in window) + (vision baselines in window), `rows_due` = (email pairs) + (vision pairs), `drafts_created` = 0, `detail` = `ok` (the halt path in Setup step 4/5 already emitted `no-delta-dir` and stopped). Final message: email pairs found + top three email edits; vision pairs found + top three vision edits; where both delta files landed (the Delta folder); and any flag raised. If a track has fewer than two pairs, say so for that track and do not manufacture a pattern there.

## Scheduled-task stub

Name: `<Coach> - Voice Capture`. Weekly, Sunday 6:00 PM, coach's timezone. Working folder: EMPTY. Run on your computer: OFF. Permissions: automatically approve (README section 5 - a task on manually approve stalls unattended). Connector tools per README: reads, Gmail create draft, Drive create file approved; send, delete, modify, label, calendar-write set to needs approval (does not bind under automatically approve - this skill's never-send rule is the control).

```
Run the voice-capture skill from the catalyst-coaching-core plugin.
Load coaching-guardrails first.
BRAIN: https://catalyst-brain.lindsey-1ad.workers.dev
KEY:   <coach key>
```
