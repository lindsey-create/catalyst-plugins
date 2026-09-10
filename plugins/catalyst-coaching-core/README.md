# Catalyst Coaching Core (v0.2.1)

The Core tier of the Catalyst coaching ops system, packaged as a Claude plugin and distributed from Catalyst's marketplace. Installed into the coach's own Claude account; runs as Cowork scheduled tasks against the coach's own Gmail, Google Calendar, and recorder (Zoom or Fathom) connectors.

**Nothing is ever sent on the coach's behalf. Every email lands as a Gmail draft for the coach to review and send.**

## What ships in this plugin

| Skill | Schedule | What it does |
|---|---|---|
| `coaching-guardrails` | loaded by the others | The non-negotiable rules, the Brain protocol, the voice rule, the coaching-method rule. Never scheduled on its own. |
| `morning-prep-brief` | weekdays 8:00 AM, coach's timezone | One card per coaching call today: context, last touchpoint, goal, pre-call questions, suggested open. Surfaces open loops and suspected no-shows for confirmation. |
| `eod-recap` | weekdays 3:30 PM, coach's timezone | Finds today's recordings (Zoom or Fathom), matches them to calendar events with a name-check, recaps each call, drafts the follow-up email in the coach's voice, captures commitments (open loops), drafts between-call check-ins, logs the as-drafted copy for voice capture. No BCC - client content stays in the coach's account. |
| `weekly-metrics-digest` | Fridays 4:00 PM | Calls held vs scheduled, suspected no-shows, drafts created vs sent, open loops outstanding. No cross-coach data, ever. |
| `voice-capture` | Sundays 6:00 PM | Compares what the coach actually SENT to what the system DRAFTED, and writes a voice-delta file for Catalyst to fold into the voice profile. The coach's edits teach the system. |

## What the plugin does NOT contain

- No coach-specific data. Timezone, first name, signature style, recorder, and state-folder name come from the Catalyst Brain at run time, keyed per coach.
- No credentials. The coach's key goes in the scheduled-task prompt, nowhere else.
- No send capability by design. Gmail is used to create drafts only.

## Setting up a coach (Lindsey's side)

1. Add `KEY_<coach>` as a Script Property on the Brain endpoint. Verify with `/v1/ping/<anything>?key=...&mode=ping`.
2. Add the coach's `brain` rows: `coach_first_name`, `timezone`, `signature_style`, `recorder` (zoom | fathom), `state_folder_name`, `ignore_contacts` (comma-separated addresses/domains that are never clients - pre-seed with Lindsey's addresses, since Catalyst books coaches through their own booking pages).
3. Coach installs this plugin from the Catalyst marketplace and connects Gmail, Calendar, and recorder in their own browser.
4. Create the four scheduled tasks using the stubs in each skill's "Scheduled-task stub" section. **Working folder: empty. "Run on your computer": off.**
5. **Approvals (verified 2026-08-20 - a task left on "manually approve" stalls when the app is closed):**
   - Task-level permission: **automatically approve** on each scheduled task.
   - Connector tool permissions (Settings > Connectors > each connector): set **read** tools and Gmail **create draft** to *approved*; leave **send, delete, modify, label, archive, calendar-write** tools on *needs approval* or unconnected. Note: under automatically approve these do not bind (verified 2026-09-04) - the skill's never-send rule is the control. Never enable an org-wide "skip all approvals."
   - Drive: *create file* approved; nothing else.
6. Run each once with Run Now. Read the first three drafts before the coach does.

## Versioning and releases

Versions follow semver: a change to an existing skill bumps the patch number, a new skill bumps the minor number. The version on your plugin card is the version that runs - every run also reports it in its first line.

**A push does not reach installed plugins on its own timeline.** Verified 2026-08-20: an installed plugin kept reporting the old version through an app restart, a disable/enable, and a plugin uninstall/reinstall, because plugin installs read the app's cached marketplace clone. The release process:

1. Test on the Catalyst test account first (`INSTALL-TEST` task, confirm the `SKILL LOADED` version line).
2. Push to GitHub `main`. Drag-upload the skill files (folder contents, no zip); hidden `.claude-plugin` manifests do not drag-upload, so bump `version` in `plugin.json` and `marketplace.json` in GitHub's web editor. Verify both on GitHub.
3. Tell every coach the same day, with this one step: **Settings > Plugins > Personal > `...` next to catalyst-marketplace > Check for updates > then Update on the Catalyst card.** ~30 seconds. Browser (claude.ai) users: Settings > Plugins > Personal, wait out the "Still syncing" pop-up until the row reads "updated X minutes ago", reload the page; no version or Update button is shown in the browser - confirm via the next run's `SKILL LOADED` line.
4. Verify from the next run's `SKILL LOADED` line. A coach still on the old version has not clicked Update.

Do not uninstall/reinstall the plugin to update (stale cache). Do not remove the marketplace (uninstalls the plugin).

**Silent-day troubleshooting:** a day with no `run_start` row at all = fetch-layer block or scheduler miss, not a skill bug. First check that `BRAIN/robots.txt` returns plain text (the fetch tool blocks the whole domain with ROBOTS_DISALLOWED when it can't read robots.txt - hit 2026-08-24). The Worker carries an explicit robots.txt route (added 2026-09-09); before that, Cloudflare's managed robots.txt on workers.dev answered the path on its own. Keep both true.

## Changelog

- **0.2.1 (2026-09-09)** - coaching-method file replaces the in-plugin CLEAR reference (each coach's own method, placed at onboarding; skills degrade to general questions if absent); state-write retry + `err-drive`; client-as-organizer rule + `ignore_contacts` row; heartbeat file per run. Test-account run 2026-09-09 (method file read, fallback, heartbeat). Readability pass done - 2026-09-09.
- **0.2.0 (2026-09-08)** - superseded by 0.2.1 before release; never shipped to a coach.
- **0.1.2 (2026-08-21)** - telemetry `run_complete` retries on fresh suffixes, falls back to `detail=err`; release process documented; browser update path verified 2026-08-21.

- **0.1.1 (2026-08-20)** - per-CALL unique Brain path segments with literal shapes; `detail` limited to one token; version line in every skill; approvals guidance; timezone is per coach (no `*` row). Findings from the install test on a personal Pro account (Q1-Q3 + RT all passed).
- **0.1.0 (2026-08-20)** - initial scaffold.
