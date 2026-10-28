# Food Tracker v2 — Reply Formatting & Source Labeling

## Problem

The Calorie Logging Agent's Telegram replies (confirmation previews and post-log
summaries) are plain, undifferentiated text. Two specific gaps:

1. No visual structure — hard to scan on mobile.
2. No distinction between food data that came from the pre-loaded food database
   (`Read Food Database`) versus a fresh web lookup (`lookup_food_on_web` /
   Tavily) for a food not yet in the database.

Additionally, the post-log confirmation only reports *remaining* calories/protein
for the day, not the running total already eaten.

## Scope

- Rewrite the `systemMessage` on the `Calorie Logging Agent` node
  (`Food Tracker v2`, workflow `YOUR_WORKFLOW_ID`) to specify:
  - HTML formatting (bold headers, bullet lists, italic assumption notes).
  - Per-food-item source tags: 📋 for a database match, 🌐 for a web lookup
    result not yet in the database. Once `add_new_food_to_database` has run for
    a food, it is 📋 on every subsequent reference (same or later conversation).
  - Confirmation message shows only the proposed entry's total (no pre-log
    running-total lookup — out of scope, see Rejected Alternatives).
  - Post-log message shows both today's running total ("Today so far") and
    remaining-vs-target, sourced from `log_food_entries`'s existing return
    value (no new data source needed).
- Fix `Telegram: Send Message`'s output-sanitizing expression, which currently
  HTML-escapes `<` and `>` unconditionally and would strip the agent's own
  `<b>`/`<i>` tags along with any incidental angle brackets in food names.

No new nodes, no changes to `connections`, no changes to `Log Food Entries
(sub-workflow)`.

## Message templates

**Confirmation (before logging):**
```
<b>Log this?</b>
• Cooked rice ~150g — 195 cal, 4g protein, 43g carbs, 0g fat 📋
• Paneer tikka ~1 serving — 250 cal, 18g protein, 6g carbs, 18g fat 🌐
<i>(assumed 1 serving paneer tikka — say if different)</i>

<b>Total:</b> 445 cal · 22g protein · 49g carbs · 18g fat

Reply yes to log, or tell me what to change.
```

**Post-log (after `log_food_entries` succeeds):**
```
✅ Logged.
<b>Today so far:</b> 950 cal · 35g protein · 40g carbs · 20g fat
<b>Remaining:</b> 250 cal · 10g protein
```

Formatting rules: `<b>` for section headers/totals, `•` bullet per food line,
`<i>` only for stated assumptions, `·` as the separator inside a macro line.
Existing tone rules stay in force (no preamble, no chat filler, no emoji beyond
the two functional source tags).

## Data already available to the agent

- Per-item source: the agent already knows, per turn, whether it matched a food
  against `foodList` (from `Read Food Database`) or called `lookup_food_on_web`
  — this is a prompt-instruction change, not a new data feed.
- Post-log totals: `log_food_entries` already returns "today's updated total
  calories, protein, carbs, and fats" (see its tool description) — the agent
  just needs to be told to report both total-so-far and remaining, not just
  remaining.

## Rejected alternatives

- **Pre-log running total in the confirmation message**: would require adding a
  fourth data source (a read of today's existing totals) into `Build Prompt
  Data`, feeding the main agent turn. Explicitly deferred — confirmation shows
  only the entry being proposed; running totals appear only after a successful
  log.
- **Grouped sections / footnote-only source labeling**: rejected in favor of
  inline per-item tags (📋/🌐), which keep source visible at the line level
  without restructuring the list into blocks.

## Testing

- `validate_node_config` on `Calorie Logging Agent` and `Telegram: Send
  Message` after edits.
- `test_workflow` with pinned data for: (a) an all-database-match text log, (b)
  a log requiring one web lookup, (c) a confirmed log to see the post-log
  message render with both totals.
- Manually verify in a real Telegram chat that `<b>`/`<i>` render as bold/italic
  rather than literal tags (confirms the escaping fix worked).

## Rollback

Before `update_workflow`, snapshot current `Calorie Logging Agent` and
`Telegram: Send Message` node JSON to
`workflow_versions/Food_Tracker_v2/<timestamp>_pre-reply-formatting.json` per
CLAUDE.md §-1.
