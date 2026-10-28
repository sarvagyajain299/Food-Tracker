# n8n Workflow Log

Running record of workflows built/changed in this project, newest entry on top.
Read this at the start of a session to recover context. Append a new entry after
any create/update/publish action — don't edit past entries except to mark status changes.

Entry format:

```
## YYYY-MM-DD — <Workflow Name>
- Workflow ID / URL: 
- Change: what was created/modified
- Why: the requirement or bug behind it
- Status: draft / validated / tested / published
- Next: what's left, if anything
```

---

## 2026-07-27 — Log Food Entries (sub-workflow) — Fix daily-total drift on manual deletes
- Workflow ID: `YOUR_WORKFLOW_ID`
- Change: "Read Today Totals" renamed to "Read Today Tracking Rows" and repointed from the
  `total daily macros tracking table` to the raw `tracking table` (still filtered to today's
  date). "Compute New Totals" no longer reads the existing total and adds the newly logged
  entries on top — it now re-sums Calories/Protein/Carbs/Fats fresh from every row currently
  in the tracking table for today, then upserts that as the new total.
- Why: discovered during a review of the separately-existing "CLAUDE - Daily Macro Totals
  (3AM)" workflow (`uHjm43xfiXsYnleo`, not previously logged here — predates this log) that
  the incremental add-on-top design meant a manually deleted row in `tracking table` was
  never reflected in the daily total until the 3AM nightly re-sum caught up the next day.
  Re-summing from source on every log closes that gap same-day instead of waiting overnight.
- Status: published (active). Snapshot of pre-change state saved to
  `workflow_versions/Log_Food_Entries_(sub-workflow)/2026-07-27_pre-resum-fix.json`. Verified
  via `test_workflow` with pinned data: two today-rows (300/20/40/10 and 200/30/0/5) summed
  to the correct 500/50/40/15 in `Compute New Totals` and `Build Summary`.
- Next: the 3AM workflow (`uHjm43xfiXsYnleo`) is being kept active as a low-cost nightly
  safety net for the one remaining edge case this fix doesn't cover — deleting the *only*
  remaining log for a day, with no further log that day to trigger a recompute. No other
  changes needed to it.

## 2026-07-27 — Food Tracker v2 — Reply Formatting
- Workflow ID: `YOUR_WORKFLOW_ID`
- Change: Rewrote the Calorie Logging Agent system prompt to emit HTML-formatted
  replies (bold headers/totals, bullet lines, italic assumption notes) and to
  tag every food line with its source (📋 database match, 🌐 fresh web lookup).
  Post-log messages now report both "Today so far" (all four macros) and
  "Remaining" (calories/protein), not remaining alone. Removed the HTML-escaping
  on Telegram: Send Message's output expression, which had been stripping the
  agent's own `<b>`/`<i>` tags.
- Why: Replies were plain and gave no indication of whether a food's macros
  came from the trusted local database or an unverified web search.
- Status: published and confirmed working live in Telegram. Tested via
  `test_workflow` with pinned data for a database-only log, a web-lookup log
  (agent correctly called `lookup_food_on_web` then `add_new_food_to_database`,
  tagged 🌐), and a confirmed-log turn (post-log totals matched the pinned
  `log_food_entries` return exactly). One deploy snag hit and fixed:
  `update_workflow` only updates the draft version — the node changes didn't
  reach production until `publish_workflow` was called explicitly, even though
  the workflow was already active. User confirmed the fix live afterward.
- Next: None outstanding for this change. Snapshot of the pre-change state is
  at `workflow_versions/Food_Tracker_v2/2026-07-27_1900_pre-reply-formatting.json`.

## 2026-07-27 — Food Tracker v2
- Workflow ID: `YOUR_WORKFLOW_ID`
- Change: Full rebuild of the food tracker as a single AI agent (Gemini 2.5 Flash) behind
  a Telegram Trigger. Photo messages branch through Telegram: Get File → a vision chain
  (Gemini Vision, separate model from the agent) that describes visible foods in plain
  text; text messages skip straight to formatting. Both paths converge through Combine
  Inputs → Combine All Data (merge, 3 inputs) alongside two Google Sheets reads (food
  database, weekly macro goals), then Build Prompt Data (Code) assembles the full food
  list + today's target into the prompt. The Calorie Logging Agent has three tools:
  `log_food_entries` (calls the sub-workflow below), `add_new_food_to_database` (Google
  Sheets Tool, appends with a random `f_xxxxxx` Food ID), and `lookup_food_on_web`
  (Tavily). Chat memory is a buffer window keyed on Telegram chat ID. Both sheet reads use
  `onError: continueErrorOutput` wired to a "Notify Data Fetch Failure" Telegram message.
- Why: Replace the older "Food Tracker" (v1, now inactive) with a faster, simpler design —
  one agent instead of multiple LLM calls, database preloaded into the prompt instead of
  a lookup tool, target under 8s for text / 15s for photo.
- Status: published (active), not yet exercised end-to-end. `get_workflow_details` flags
  "This workflow has no production triggers" despite the Telegram Trigger node and
  `active: true` — worth confirming the Telegram webhook is actually registered before
  relying on it.
- Next: Send a real Telegram message (text and photo) and confirm the agent logs
  correctly via the sub-workflow, confirmation flow works, and the error-fetch fallback
  triggers if a sheet read is broken deliberately. Resolve the "no production triggers"
  warning above.

## 2026-07-27 — Log Food Entries (sub-workflow)
- Workflow ID: `YOUR_WORKFLOW_ID`
- Change: Built as the `log_food_entries` tool called by Food Tracker v2. Takes
  `entries: [{food_id, quantity, unit}]` via Execute Workflow Trigger, reads the food
  database, computes per-entry macros in Code (throws if `food_id` isn't found — by
  design, forces the agent to add unknown foods first), appends rows to the tracking
  table, reads today's existing totals (filtered by date, `executeOnce` + always output
  data so a first-entry-of-the-day empty read doesn't break the merge), sums into new
  daily totals, and upserts them (matched on Date) into the daily totals sheet. Returns
  logged entries + today's running totals to the agent.
- Why: Keep macro math and sheet writes out of the agent's own reasoning — deterministic
  Code node computes the arithmetic, agent only decides *what* to log.
- Status: published (active), untested end-to-end (depends on Food Tracker v2 test above).
- Next: Verify the "Unknown food_id" error surfaces cleanly through the parent agent
  rather than as a raw failure, and confirm the daily-totals upsert doesn't double-count
  on same-day repeated calls.

## 2026-07-27 — Food Tracker v2 - Error Alerts
- Workflow ID: `YOUR_WORKFLOW_ID`
- Change: Minimal catch-all error workflow — Error Trigger → Telegram message to chat ID
  `YOUR_TELEGRAM_CHAT_ID` with workflow name, last node executed, error message, and IST-formatted
  timestamp, plus a prompt to retry the last message.
- Why: Per CLAUDE.md rule (§5) every webhook/unattended workflow needs an error path;
  this is the instance-level catch-all for anything that escapes per-node
  `continueErrorOutput` handling in Food Tracker v2.
- Status: published (active). Not yet confirmed wired as the **Error Workflow** setting on
  Food Tracker v2 itself (a workflow only calls its error workflow if that setting points
  here — creating this workflow alone doesn't attach it).
- Next: Open Food Tracker v2's workflow settings and confirm "Error Workflow" is set to
  this workflow; then force a node failure to verify the Telegram alert actually fires.

## Legacy / inactive workflows (not actively maintained, kept for reference)
- **Food Tracker** (`3YA09ACCL61hjdw7`) — original v1, inactive since v2 replaced it.
- **Error in Food Tracker** (`wvAAxnMS1qGhh9ZP`) — old error workflow for v1, still active;
  should probably be deactivated once v2's error workflow is confirmed wired, to avoid
  duplicate/conflicting alerts.
- **Food Tracker (May 12 at 22:11:07) backup** (`w1yO00Xas9jtPTso`) — manual backup snapshot,
  inactive.
