---
name: send-feedback
description: Lets the coach send Lindsey a change request or feedback about their Catalyst system without editing the plugin - asks up to three short questions, saves the request to the coach's state folder, drafts an email to Lindsey for the coach to review and send, and logs a count (never the content) to telemetry. Use when the coach says "send feedback", "I want to change how X works", "request a change", "this should be different", or starts to customize the plugin.
---

# Send Feedback

**Plugin version: 0.3.3.** Load `coaching-guardrails` first. This skill creates one Gmail DRAFT to Lindsey. It never sends, and it never reads client data.

## Why this path

Edits made inside the plugin stop receiving updates and are lost on the next release. A request sent this way gets into the next version for everyone, and anything that is yours-only goes into your `coach-voice` file, which no update touches.

## Setup

1. Build `run_id` = `feedback-<YYYYMMDD-HHMM UTC>`.
2. Fetch config (`mode=brain`). Resolve `coach_first_name`, `state_folder_name`.
3. Write telemetry `run_start` (with `plugin_version`).

## Step 1 - Ask, briefly (max three questions, one at a time)

1. "What would you like to work differently?" (what)
2. "What happened that made you want this - a specific draft, a brief, a call?" (why / use case)
3. "Is this something only for you, or would other coaches want it too?" (scope)

If the coach already answered any of these in their first message, do not re-ask. Never ask a fourth question.

## Step 2 - Sort it

- Style only (how drafts sound: openers, sign-offs, words to avoid, length): offer to add it to `coach-voice` in the state folder right now (create the file if it does not exist; if it exists, create `coach-voice-<run_id>` with the addition and tell the coach to merge - you cannot edit existing files). Still send the request to Lindsey so she knows.
- Anything else (what a skill does, a new check, a new output): request to Lindsey.

## Step 3 - Save and draft

- Write `feedback-<run_id>.md` to the state folder: date, what, why, scope, the coach's own words.
- Create a Gmail DRAFT to lindsey@catalystai-solutions.com, subject "Change request from <coach_first_name>: <five-word summary>", body: the three answers in plain text, then "Sent from my Catalyst system - reply to me directly." No client names, no client content - if the coach's example includes a client, describe the situation without the name.
- Tell the coach: "Draft is in your Gmail drafts - read it, edit it, send it when you're ready. Lindsey replies within two business days."

## Close

Telemetry `run_complete` with `rows_read` = 0, `rows_due` = 0, `drafts_created` = 1, `detail` = `feedback`. Final message per guardrails section 6.

## Not scheduled

This skill has no scheduled-task stub. The coach invokes it by name or phrase.
